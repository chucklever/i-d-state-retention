---
title: "State Retention Across Client Restart for NFSv4.2"
abbrev: "NFSv4.2 State Retention"
category: std

docname: draft-cel-nfsv4-state-retention-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Network File System Version 4"
keyword:
 - NFSv4.2
 - client restart
 - state reclaim
 - NFS gateway
venue:
  group: "Network File System Version 4"
  type: "Working Group"
  mail: "nfsv4@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/nfsv4/"
  github: "chucklever/i-d-state-retention"
  latest: "https://chucklever.github.io/i-d-state-retention/draft-cel-nfsv4-state-retention.html"

author:
 -
    fullname: Charles Lever
    email: cel-ietf@chucklever.net

normative:
  RFC4506:
  RFC7862:
  RFC7863:
  RFC8178:
  RFC8881:

informative:
  RFC1813:
  RFC7530:
  RFC9289:
  RFC9754:
  I-D.haynes-nfsv4-flexfiles-v2-proxy-server:

...

--- abstract

An NFS gateway is an NFS server that exports a file system that is
itself an NFS mount of a backend server.  When the gateway
restarts, the backend server releases the file open and lock state
the gateway established there, even though the gateway's clients
are still running and will attempt to reclaim the file open and
lock state they hold.  This document extends NFSv4.2 so that a
server can retain the state of a client that has restarted and
permit the new instance of that client to reclaim that state.


--- middle

# Introduction {#intro}

An NFS gateway is an NFS server that exports a file system that is
itself an NFS mount of a backend server.  The gateway serves one
set of clients, over any version of NFS, while it is itself an
NFSv4.2 client of the backend.  Every file open and byte-range
lock that a client of the gateway holds has a counterpart that the
gateway holds on the backend, established under the gateway's own
client ID and lease.

NFSv4 treats a client restart as the end of that client's state.
When a client presents a known client owner with a new verifier in
EXCHANGE_ID and then confirms the new client ID, the server
releases the opens and byte-range locks that the previous instance
held ({{Section 8.4.1 of RFC8881}}, {{Section 18.35.4 of
RFC8881}}).  For most clients, releasing that state is the correct
outcome.  The applications that held the state restarted along
with the client, and nothing remains that could reclaim the state.

A gateway restart is different.  The gateway's clients did not
restart.  The gateway's clients observe only that the gateway was
unreachable for a time, and when the gateway returns they begin
state recovery: NFSv4 clients of the gateway reclaim their opens
and locks during the gateway's grace period, and NFSv3 clients of
the gateway reclaim their NLM locks when the gateway's status
monitor announces the restart.  Each of those reclaims can succeed
only if the gateway can re-establish the corresponding state on
the backend.  The backend, however, has already released that
state.  The backend accepts reclaim-type requests only during the
backend's own grace period, and the backend did not restart, so
no grace period is in effect ({{Section 8.4.3 of RFC8881}}).  The
gateway has no way to restore the state of the gateway's clients,
and in the interval since the gateway restarted the backend may
have granted conflicting opens or locks to clients that access the
backend directly.

NFSv4.1 already retains one kind of state across a client restart.
A server that supports CLAIM_DELEGATE_PREV keeps a restarted
client's delegations for at least a lease period so that the new
instance of the client can reclaim them ({{Section 10.2.1 of
RFC8881}}).  This document applies the same idea to opens and
byte-range locks.  This document defines client-restart state
retention: a server that supports retention and a client that has
requested retention agree
that the client's opens and locks survive a restart of the client,
and that the new instance of the client may reclaim them outside
the server's grace period.  The client requests retention when it
establishes its client ID.  The server retains the client's state
when the client's lease expires or a new instance of the client
appears.  The new instance reclaims the state using the existing
reclaim operations, then signals that it has finished, at which
point the server releases whatever was not reclaimed.  At each
stage the server bounds how long retained state may block other
clients.  {{extension}} specifies the mechanism, and {{sequences}}
walks through complete recovery sequences.

The mechanism is not limited to gateways.  Any NFSv4.2 client that
can reconstruct its open and lock state after a restart can use
it, for example a user-space NFS proxy or an SMB server that
exports an NFS mount.  This document presents the gateway as the
motivating case and develops the gateway's behavior in detail.

This document specifies the mechanism as an extension to NFSv4.2
under the process described in {{RFC8178}}, in the hope that it
can be deployed without a new NFSv4 minor version.  The behavior
the extension introduces applies only to client IDs that have
negotiated the extension.  Servers and clients that do not
implement the extension are unaffected.

This document is distinct from
{{I-D.haynes-nfsv4-flexfiles-v2-proxy-server}}, which describes a
proxy that the backend's metadata server names in pNFS layouts.
Clients of such a proxy hold their state at the metadata server
rather than at the proxy, so the recovery problem described here
does not arise for them.  {{problem}} summarizes the gateway's
recovery problem, including kinds of state that this document does
not address.


# Requirements Language

{::boilerplate bcp14-tagged}


# Terminology

This document uses the terms defined in {{Section 1.7 of RFC8881}}
for NFSv4.1 state, leases, and recovery.  The following terms are
specific to this document.

Gateway server:
: A host that is an NFS server to one set of clients and an NFSv4.2
  client of a backend server, for the same file system.  Shortened
  to "gateway" where no confusion can result.

Backend server:
: The NFS server that the gateway mounts.  Shortened to "backend".

Front-side client:
: A client of the gateway server.  This document does not use
  "gateway client", which could as easily mean the NFS client
  that runs on the gateway host.

Direct client:
: A client of the backend server that does not go through the
  gateway.

Front side, back side:
: The two roles of a gateway.  On the front side the gateway is a
  server to front-side clients.  On the back side the gateway is a
  client of the backend.

Front-side state:
: Open, share reservation, and lock state that the gateway has
  granted to a front-side client.

Derived state:
: State that the gateway holds at the backend because of
  front-side state.  Each item of derived state stands for one or
  more items of front-side state.

Client instance:
: One incarnation of an NFSv4.1 client, identified by the verifier
  in its EXCHANGE_ID client owner.  A client restart begins a new
  client instance with the same client owner identifier and a new
  verifier.

Courtesy client:
: A client whose lease has expired but whose state the server has
  not yet released.  Its locks are courtesy locks
  ({{Section 9.6.3.1 of RFC7530}}), which the server MUST release
  when a conflicting request arrives ({{Section 8.4.3 of RFC8881}}).

Courteous server:
: A server that keeps the state of a courtesy client for some
  time after lease expiry rather than releasing it at once.

Retaining client:
: A client whose client ID was established with the extension in
  this document negotiated, so that the server retains the client's
  state across a restart of the client.

Retained state:
: Opens and byte-range locks that belonged to a previous instance
  of a retaining client and that the server continues to hold and
  enforce on that client's behalf.  Unlike courtesy locks, retained
  state does not yield to a conflicting request until the absence
  limit or the reclaim cap is reached.

Absence interval:
: The time from expiry of a retaining client's lease until a new
  instance of that client is confirmed.

Absence limit:
: The longest absence interval during which a server continues to
  enforce retained state against conflicting requests from other
  clients.  A local policy value of the server.

Reclaim interval:
: The time from confirmation of a new instance of a retaining
  client until that instance sends RECLAIM_COMPLETE.

Reclaim cap:
: The longest reclaim interval during which a server continues to
  enforce retained state that the new instance has not yet
  reclaimed.  A local policy value of the server.

Synthetic open-owner:
: An open-owner that a gateway constructs for an NLM client, which
  has no concept of an open, so that the gateway can hold a
  back-side open under which to take that client's byte-range
  locks.

Pass-through deny mode:
: A gateway mode in which a derived open carries at least the share
  deny bits of the front-side open or NLM_SHARE that it stands for,
  so that the backend enforces them against direct clients.

Local-only deny mode:
: A gateway mode in which a derived open carries no share deny
  bits.  The gateway enforces the front-side deny mode itself, and
  the deny mode binds front-side clients only.


# Problem Summary {#problem}

{{fig-deployment}} shows the deployment this document addresses.
The gateway's clients may use NFSv3 with NLM and NSM, or any
NFSv4 minor version.  The gateway accesses the backend as an
NFSv4.2 client, and all of the gateway's front-side clients share
that one client ID.
Direct clients, and other gateways, may access the same files,
and the backend is the only party that can arbitrate among all of
the parties that hold state there.

~~~
  front-side clients       gateway            backend
  +-----------+        +-------------+     +-----------+
  | NFSv3/NLM |------->| NFS server  |     |           |
  | NFSv4.x   |        |     |       |     |    NFS    |
  +-----------+        | NFS client  |---->|   server  |
                       +-------------+     |           |
  direct clients ------------------------->|           |
                                           +-----------+
~~~
{: #fig-deployment title="Deployment model"}

NFSv4 recovery ({{Section 8.4 of RFC8881}}) assumes two parties.
State belongs to a client instance and is kept alive by its lease.
When the server restarts, it offers a grace period in which
clients reclaim what they held.  When the client restarts, the
server discards the old instance's state, because the
applications that held it are gone.  A gateway breaks that last
assumption.  Its back-side client instance restarts, but the
applications that hold the state are on the gateway's clients,
which have not restarted and will reclaim.  The backend, not in a
grace period, refuses the reclaims the gateway forwards, and in
the meantime may have granted conflicting state to direct
clients.

A gateway restart is the only failure that the existing protocol
cannot express, and the extension in this document exists for
that failure.  Three related failures need gateway and backend
behavior but no new protocol elements, and later sections describe
that behavior.  When the backend restarts alone, the gateway
reclaims the state the gateway holds on the backend and shields
the gateway's clients from the backend's grace period.  When both
the gateway and the backend restart, the gateway forwards the
reclaims of the gateway's clients into the backend's grace period
on a best-effort basis, since the two grace periods are not
coordinated.  When a partition outlasts the gateway's lease on the
backend, the backend may revoke the gateway's state, and the
gateway reports to the gateway's clients a loss that those clients
did not cause.

The extension described in this document does not retain
delegations.  A delegation that a gateway grants to one of its
clients can survive a gateway
restart only if the backend guarantees that no conflicting access
occurred in the interim, which requires a back-side delegation
retained under CLAIM_DELEGATE_PREV ({{Section 10.2.1 of
RFC8881}}).  A future version of this document, or a separate
one, may permit a gateway to grant a front-side delegation on
that condition.  Also out of scope are pNFS layouts on either
side of the gateway, a back side other than NFSv4.2, and failover
of a gateway's state to a different gateway host.  A gateway
whose backend is itself a gateway is not specified, but
{{chained}} describes how the mechanism behaves in that
configuration.


# Protocol Extension {#extension}

Client-restart state retention proceeds in four steps.  This
outline is not normative.  The subsections that follow specify
each step, and {{sequences}} walks through complete recovery
sequences.

Negotiate:
: The client sets a new flag, EXCHGID4_FLAG_RETAIN_STATE, in the
  EXCHANGE_ID that establishes its client ID.  A backend that
  implements the extension and is willing to retain state for the
  requesting principal echoes the flag in its reply.  The client is
  then a retaining client.  A backend that does not implement the
  extension returns NFS4ERR_INVAL, as {{Section 18.35.3 of
  RFC8881}} requires for an unknown flag, and the client retries
  without the flag.

Retain:
: When a retaining client's lease expires, or when a new instance
  of a retaining client is confirmed, the backend does not release
  the opens and byte-range locks of the old instance.  The state
  stays in place and continues to conflict with requests from
  other clients.

Reclaim:
: The new instance sends the reclaim operations that RFC 8881
  already defines, OPEN with CLAIM_PREVIOUS and LOCK with the
  reclaim flag set.  The backend accepts them outside its grace
  period when they match retained state, and moves the matched
  state to the new client ID.  The reply to each reclaim is the
  authoritative answer to whether that state survived.

Complete:
: When the new instance has finished reclaiming, it sends
  RECLAIM_COMPLETE.  The backend releases whatever retained state
  was not reclaimed.  For a gateway, this step comes at the end of
  the gateway's own front-side grace period.

The reply to EXCHANGE_ID also carries a second flag,
EXCHGID4_FLAG_RECLAIMABLE_R, which the backend sets when
reclaim-type requests from this client owner can succeed once the
new client ID is confirmed.  The flag is a hint that lets a
gateway decide whether to run a front-side grace period at all.
It is not proof that any particular item of state was retained.

The backend bounds how long it retains state with two limits, and
it advertises both as attributes ({{attrs}}).  A gateway reads
them once it has a session and uses them to fit its front-side
grace period inside the time the backend allows for reclaim.

## Capability Negotiation {#negotiate}

This extension defines two flags in the EXCHANGE_ID flag word,
using bits that {{Section 18.35.1 of RFC8881}} leaves unassigned.
{{xdr}} gives their values.

EXCHGID4_FLAG_RETAIN_STATE:
: A client sets this flag in eia_flags to request that the server
  retain the state of the client owner across a restart.  A server
  sets this flag in eir_flags when it will do so.  A client whose
  client ID was established by an EXCHANGE_ID in which both the
  request and the reply carried this flag is a retaining client.

EXCHGID4_FLAG_RECLAIMABLE_R:
: A server sets this flag in eir_flags when reclaim-type requests
  from this client owner can succeed once the new client ID is
  confirmed.  A client MUST NOT set this flag in eia_flags.

A server MUST NOT set either flag in eir_flags unless the request
set EXCHGID4_FLAG_RETAIN_STATE.  The request flag is how the
server knows, before it sends an extended reply, that the client
is aware of the extension ({{Section 6 of RFC8178}}).

A server that does not implement this extension rejects an
EXCHANGE_ID that sets EXCHGID4_FLAG_RETAIN_STATE with
NFS4ERR_INVAL, as {{Section 18.35.3 of RFC8881}} requires for an
unassigned flag bit.  A client that receives NFS4ERR_INVAL MUST
retry the EXCHANGE_ID without the flag.  If the retry succeeds,
the client knows that the flag was the cause of the error and that
the server does not support retention.  The retry is necessary
because EXCHANGE_ID returns NFS4ERR_INVAL for other reasons as
well ({{Section 4.4.3 of RFC8178}}).

Whether to retain state for a client owner is a server policy
decision.  A server that implements this extension MAY decline
the request.  A server that declines clears
EXCHGID4_FLAG_RETAIN_STATE in eir_flags and treats the client
owner as {{RFC8881}} specifies.  Any client that becomes a
retaining client can hold opens and locks past lease expiry for
the whole absence limit, so a server SHOULD restrict retention to
principals that are authorized to act as gateways.  A server
SHOULD require that a retaining client use the SP4_MACH_CRED state
protection mode, so that only the holder of the client's machine
credential can establish the next instance of that client owner
and reclaim its state.  {{security}} discusses both points.

A server records that a client owner is a retaining client along
with the rest of the client owner's record, so that the server
knows at lease expiry how to treat the state of that client owner.

A server sets EXCHGID4_FLAG_RECLAIMABLE_R in each of three cases:

- The server holds retained state from a prior instance of this
  client owner whose lease has expired.

- The server holds state of a prior instance of this client owner
  whose lease has not expired, and will retain that state when
  CREATE_SESSION confirms the new instance.  {{Section 18.35.4 of
  RFC8881}} replaces the prior instance's record at confirmation,
  not at EXCHANGE_ID, so nothing has been retained yet when the
  reply is sent.

- The server is in its own grace period and its stable storage
  lists this client owner as permitted to reclaim.

The flag names what the client can do rather than why, because
the client acts the same way in all three cases.  The flag is a
hint, given before confirmation.  It is not proof that any
particular item of state was retained.  The authoritative answer
for each item of state is the reply to the reclaim of that item.
A gateway uses the hint to decide whether to run a front-side
grace period at all.

This extension does not add an attribute that advertises support
for retention, although {{Section 6 of RFC8178}} offers that as a
convenience and {{RFC9754}} uses one for its OPEN flags.  An
attribute is read per file system, with a filehandle in hand.
Retention is a property of a client ID, and a client has to learn
whether retention is available at EXCHANGE_ID, before the client
has a session with which to read an attribute.  That reasoning
does not extend to the limits a server places on retention.  A
client has no use for those until it has a session, and this
extension does advertise them as attributes ({{attrs}}).

## Retaining State {#retain}

{{RFC8881}} has a server release a client's state in two
situations: when a new instance of the client is confirmed
({{Section 8.4.1 of RFC8881}}), and when the client's lease
expires and the server chooses not to keep the state as courtesy
locks ({{Section 8.4.3 of RFC8881}}).  For a retaining client,
a server MUST NOT release the opens and byte-range locks of the
client in either situation.  Instead, the server retains them.
Retention at the confirmation of a new instance happens when
CREATE_SESSION confirms the new client ID, which is the point at
which {{Section 18.35.4 of RFC8881}} has the server replace the
prior instance's record.  Nothing changes at EXCHANGE_ID.

The state that a server retains consists of:

- the client's opens, with the share access and share deny modes
  the server holds for each of them; and

- the client's byte-range locks.

The server does not retain the prior instance's sessions, its
client ID, or its stateids.  A new instance obtains new stateids
for the state it reclaims.  The server does not retain layouts.
Delegations are handled as {{RFC8881}} specifies, and in
particular a server that supports CLAIM_DELEGATE_PREV handles them
as {{Section 10.2.1 of RFC8881}} describes.  This extension does
not change delegation handling.

Retained state behaves as though the prior instance still held
it.  A request from another client that conflicts with a retained
open fails with NFS4ERR_SHARE_DENIED, and one that conflicts with
a retained byte-range lock fails with NFS4ERR_DENIED.  These are
the results the other client would have seen had the retaining
client not restarted.  The server MUST NOT return a grace-period
error for such a conflict.  This is a departure from
{{Section 8.4.3 of RFC8881}}, which requires that the state of a
client whose lease has expired yield to a conflicting request.
For a retaining client, that requirement is suspended until one
of the limits in {{limits}} is reached, after which it applies
again.

{{Section 8.4.3 of RFC8881}} also has a server record in stable
storage that a client's lease expired, so that the server can
reject the client's reclaims after a server restart with
NFS4ERR_NO_GRACE.  The reason for that record is that another
client could have acquired a conflicting lock in the interim.  For
a retaining client the reason does not hold while the state is
retained, because no conflicting lock is granted.  A server MUST
NOT make that record at lease expiry for a retaining client.  The
server makes the record when it releases or revokes retained
state, whether at a limit, at the first conflicting request after
a limit, or by administrative action.  {{grace}} describes the
consequences for a server restart.

Retained state accumulates across repeated restarts of the same
client owner.  If a new instance is confirmed, reclaims some of the
retained state, and then itself restarts before sending
RECLAIM_COMPLETE, the server retains both the state that instance
held and the remainder it never reclaimed.  A RECLAIM_COMPLETE
from any later instance releases everything not yet reclaimed, as
{{complete}} specifies.  Each item of retained state keeps the
reclaim-cap clock started by the first confirmation after it was
retained.  A later confirmation MUST NOT restart that clock.
Without that rule, a client that restarts in a loop could block
other clients without bound with state it never reclaims, since
every confirmation would start the cap over.  State the client
does reclaim after each restart is held legitimately each time,
so a fresh clock for it, started when the next instance is
confirmed, costs other clients nothing they were owed.

A server is not required to preserve retained state across a
restart of the server itself.  {{grace}} describes what happens
when a server restarts during an absence interval or a reclaim
interval.

## Intervals and Limits {#limits}

Retained state blocks other clients, so a server bounds how long
it enforces that state.  Two intervals are bounded separately.

The absence interval runs from expiry of the old instance's lease
until a new instance is confirmed.  A server MUST bound the
absence interval with an absence limit, a local policy value.
Within the absence limit, a retaining client that is partitioned
rather than restarted, and that returns with the same verifier,
resumes with its state intact.  For such a client the absence
limit acts as a longer lease.

The reclaim interval runs from confirmation of the new instance
until that instance sends RECLAIM_COMPLETE.  A server MUST bound
the reclaim interval with a reclaim cap, a local policy value.
Within the reclaim cap, the new instance's lease renewals keep the
unreclaimed remainder in place.  State the new instance has
already reclaimed is ordinary state of a live client, and the
reclaim cap does not apply to it.  Without a cap, a faulty client
that returns, renews its lease, and never sends RECLAIM_COMPLETE
would block other clients indefinitely with state nobody is going
to reclaim.

When either limit is reached, the retained state it governs MUST
stop blocking conflicting requests.  The server need not destroy
the state at that moment.  The server MAY keep the state as
{{Section 8.4.3 of RFC8881}} allows for expired state and revoke
it when the first conflicting request arrives.  A reclaim that
finds the state still in place succeeds, and does so safely,
because nothing that conflicts was granted in the meantime.

A server advertises both limits to its clients, as {{attrs}}
specifies.  The reply to each reclaim remains the authoritative
answer to whether an item of state survived.

## Limit Attributes {#attrs}

This extension defines two attributes, through which a server
reports the absence limit and the reclaim cap.  {{xdr}} gives
their numbers and types.

| Name                 | Id | Data Type | Acc |
|:---------------------|---:|:----------|:----|
| retain_absence_limit | 92 | uint32_t  | R   |
| retain_reclaim_cap   | 93 | uint32_t  | R   |
{: #tbl-attrs title="Limit attributes"}

retain_absence_limit:
: The absence limit, in seconds.

retain_reclaim_cap:
: The reclaim cap, in seconds.

Both are per-server attributes in the manner of lease_time
({{Section 5.8.1.11 of RFC8881}}).  A client can read them with
GETATTR on any filehandle the server provides, and the result
does not depend on the filehandle.  Neither can be set.

A server that implements this extension MUST support both
attributes.  A server that does not implement the extension does
not list them in supported_attrs, which a client can rely on
only as {{Section 6 of RFC8178}} describes.

To a retaining client, a server reports the limits it applies to
the client ID under which the request was sent.  A server whose
policy gives different principals different limits therefore
reports different values to different clients.  To a client that
is not a retaining client, a server reports its default limits.
Those values tell such a client how long retained state can block
it, and have no other effect on that client.

Each value is the configured limit, not the time remaining in an
interval.  A client knows when its reclaim interval began, since
its own CREATE_SESSION began it, and can compute what is left.

A server applies to each client ID the limits that were in effect
when that client ID was confirmed.  For the state of that client
ID, the server MUST NOT reach either limit earlier than the value
it reported.  A change to the server's configuration applies to
client IDs confirmed after the change.  Administrative release
({{admin}}), a resource limit ({{resources}}), and a restart of
the server ({{grace}}) can each end retention sooner, and the
attributes make no promise against them.

When a client owner restarts repeatedly, the reclaim cap for
state retained in an earlier round is already running ({{retain}}).
The value of retain_reclaim_cap applies in full only to state
retained at the most recent confirmation.

## Reclaiming Outside the Grace Period {#reclaim}

A new instance of a retaining client reclaims retained state with
the reclaim operations that {{RFC8881}} already defines: OPEN with
a claim type of CLAIM_PREVIOUS, and LOCK with the reclaim field
set.  This extension defines no new claim type.  A server MUST
accept these requests from a client owner for which it holds
retained state whether or not the server is in its grace period.

A server processes a reclaim-type request in a fixed order:

1. If the server is in its own grace period, the request is an
   ordinary reclaim and the server handles it as {{Section 8.4.2
   of RFC8881}} specifies.  The server need not consult retained
   state, which is not required to persist across a server
   restart.

2. Otherwise, if the server holds retained state for the client
   owner, the server matches the request against that state as
   described below.

3. Otherwise, the server returns NFS4ERR_NO_GRACE.

The client sends the same request in every case and need not know
which case applies.

An OPEN reclaim matches when the server holds a retained open for
the same open-owner string and the same file, and that retained
open includes the share access and share deny modes the request
asks for.  A LOCK reclaim matches when the retained byte-range
locks of the same lock-owner string on the same file cover the
requested range with a compatible lock type, and the open stateid
the request is presented under was obtained by reclaiming the
open under which the lock was originally acquired, that is, the
retained open with the same open-owner string on the same file.

When a reclaim matches, the server moves the matched state from
the prior instance to the new client ID and returns a new stateid
for it.  A reclaimed byte-range lock keeps its association with
its lock-owner, its open-owner, and its file, as {{Section 9.1.1
of RFC8881}} describes for any lock.  The new instance therefore
reclaims an open first and then the locks under it, which is the
order of an ordinary grace-period reclaim, and the server path is
the same.  A client MUST NOT attempt to reclaim a retained lock
under a non-reclaim open.  Doing so would move the lock to a lock
stateid under a different open, with consequences for CLOSE and
LOCKU that {{RFC8881}} does not define.

When a reclaim does not match any retained state of the client
owner, the server returns NFS4ERR_RECLAIM_BAD.  When the server no
longer holds any retained state for the client owner, it returns
NFS4ERR_NO_GRACE.  State that the server revoked for a
conflicting request after a limit in {{limits}} was reached is no
longer retained, so a reclaim for it receives one of these two
errors.  A server does not return NFS4ERR_RECLAIM_CONFLICT for
retained state, because to do so it would have to remember what it
revoked.

Matching on owner strings makes owner continuity a requirement.
The new instance MUST present the same open-owner and lock-owner
strings that the prior instance used for the state the new
instance reclaims.
A gateway that lost its own record of which front-side client held
what has, in the server's retained state, a record it can rely
on: a reclaim succeeds only for an owner that held the state, so
the gateway derives owner strings deterministically from
front-side identity, as {{owners}} specifies.  The derived owner
strings include the open-owner of an open the gateway created only
to carry an NLM client's locks.

Owner matching is a record, not authentication.  The server sees
opaque strings and cannot tell which front-side client a reclaim
is for, so it honors a reclaim under the right owner string
whoever presented that reclaim to the gateway.  Whether one
front-side client can present another's owner is decided on the
front side, by what the gateway checks before it forwards a
reclaim.  SP4_MACH_CRED protects the gateway's identity toward
the server and says nothing about the gateway's clients.
{{security}} discusses this further.

## Operations Before RECLAIM_COMPLETE {#before-complete}

{{Section 18.51.3 of RFC8881}} requires a client with a new client
ID to send a global RECLAIM_COMPLETE before its first non-reclaim
locking operation, and a server that receives a non-reclaim
locking operation before RECLAIM_COMPLETE returns NFS4ERR_GRACE.
A new instance of a retaining client cannot send RECLAIM_COMPLETE
until it knows that no more reclaims will arrive.  For a gateway,
that is the end of its front-side grace period, which can be a
full lease period after the gateway returns.

Little of what a gateway does in that interval needs a non-reclaim
locking operation.  Lock reclaims do not need a fresh open,
because each lock is reclaimed under its reclaimed open
({{reclaim}}).  New locking requests from front-side clients do
not need one either, because the gateway's own grace period holds
them off.  What remains is a gateway that opens a file in order to
serve NFSv3 READ and WRITE requests, which NLM does not gate on
recovery and which a gateway serves throughout its own grace
period.

For a client ID established with EXCHGID4_FLAG_RETAIN_STATE, a
server MUST accept a non-reclaim OPEN before the client's
RECLAIM_COMPLETE.  The server checks that OPEN against retained
state as it checks any OPEN against existing state: if the
request conflicts with a retained open's share deny mode, the
server returns NFS4ERR_SHARE_DENIED.  A non-reclaim LOCK before
RECLAIM_COMPLETE continues to fail with NFS4ERR_GRACE, as
{{RFC8881}} specifies.

This requirement differs from {{Section 18.51.3 of RFC8881}} and
from the corresponding statement in {{Section 8.4.2.1 of
RFC8881}}.  It is part of the meaning of
EXCHGID4_FLAG_RETAIN_STATE and is in effect only for a client ID
established with that flag.  It does not change the behavior
{{RFC8881}} requires of a server toward any other client.  The
exception in {{Section 8.4.2.1 of RFC8881}} applies to a server
in its own grace period, which is not the case here.  The safety
argument is the same one, though: the server holds the complete
retained state of the client owner, so it can determine that
granting the OPEN cannot conflict with a reclaim that arrives
later.

A gateway is not obliged to use this permission.  A gateway MAY
instead serve NFSv3 READ and WRITE requests with a special
stateid ({{Section 8.2.3 of RFC8881}}) until it has sent
RECLAIM_COMPLETE.  This extension does not dictate how a
gateway's back-side client performs I/O.

## Completing Reclaim {#complete}

A new instance of a retaining client signals that it has finished
reclaiming by sending RECLAIM_COMPLETE with rca_one_fs set to
FALSE, as any client does at the end of its reclaims
({{Section 18.51 of RFC8881}}).  This extension defines no
separate completion operation.

When a server receives that RECLAIM_COMPLETE from a retaining
client, it MUST release all retained state of that client owner
that has not been reclaimed.  The release ends the reclaim
interval.  State the instance reclaimed before sending
RECLAIM_COMPLETE is unaffected: it is ordinary state of a live
client and remains so.  Retained state released here is state
whose lease expired without being reclaimed, so this is the point
at which the server makes the stable-storage record that
{{retain}} defers from lease expiry.

A RECLAIM_COMPLETE with rca_one_fs set to TRUE pertains to a file
system transition and has no effect on retained state.

Once released, retained state cannot be reclaimed.  A reclaim-type
request that arrives after RECLAIM_COMPLETE finds no retained
state for the client owner and receives NFS4ERR_NO_GRACE, as
{{reclaim}} specifies.  A gateway whose front-side grace period
has ended has nothing further to forward, so this case arises only
from a reclaim that a front-side client sends late, and the
gateway refuses it as {{gateway-reclaim}} describes.

RECLAIM_COMPLETE from any instance of the client owner releases
everything retained and not yet reclaimed, including the remainder
from an earlier instance that restarted before sending its own
RECLAIM_COMPLETE.  {{retain}} describes how retained state
accumulates across repeated restarts and how the reclaim cap
bounds each part of it.

A new instance that learns from the EXCHANGE_ID reply that no
reclaim can succeed, because EXCHGID4_FLAG_RECLAIMABLE_R is
clear, sends RECLAIM_COMPLETE at once, as {{Section 18.51.3 of
RFC8881}} requires of a client that has nothing to reclaim.

## Errors {#errors}

This extension defines no new error codes.  The following existing
codes of {{RFC8881}} are returned in circumstances that this
extension introduces.  Each is specified in the subsection that
describes the circumstance; this list collects them.

NFS4ERR_INVAL:
: Returned by a server that does not implement this extension to
  an EXCHANGE_ID that sets EXCHGID4_FLAG_RETAIN_STATE
  ({{negotiate}}).

NFS4ERR_SHARE_DENIED, NFS4ERR_DENIED:
: Returned to a request from any client that conflicts with a
  retained open or a retained byte-range lock, respectively, while
  the retained state is enforced ({{retain}}).  These are the
  same results the requester would see had the retaining client
  not restarted, and they are returned in place of the yield to a
  conflicting request that {{Section 8.4.3 of RFC8881}} otherwise
  requires of expired state.  NFS4ERR_SHARE_DENIED is also
  returned to a non-reclaim OPEN from the new instance itself when
  that OPEN conflicts with retained state ({{before-complete}}).

NFS4ERR_GRACE:
: Returned to a non-reclaim LOCK from a new instance before it
  sends RECLAIM_COMPLETE, unchanged from {{Section 18.51.3 of
  RFC8881}}.  Not returned to a non-reclaim OPEN from a retaining
  client in that interval ({{before-complete}}), and not returned
  to another client whose request conflicts with retained state
  ({{retain}}).

NFS4ERR_RECLAIM_BAD:
: Returned to a reclaim-type request from a client owner for which
  the server holds retained state when no retained state matches
  the request ({{reclaim}}).

NFS4ERR_NO_GRACE:
: Returned to a reclaim-type request when the server is not in its
  grace period and holds no retained state for the client owner,
  whether because the state was never retained, was released at a
  limit or by administrative action, or was released by an earlier
  RECLAIM_COMPLETE ({{reclaim}}, {{complete}}).

A server does not return NFS4ERR_RECLAIM_CONFLICT on account of
retained state.  State revoked for a conflicting request after a
limit is no longer retained, and a reclaim for it receives
NFS4ERR_RECLAIM_BAD or NFS4ERR_NO_GRACE as above.  To return
NFS4ERR_RECLAIM_CONFLICT instead, the server would have to
remember what it had revoked ({{reclaim}}).

## Interaction with the Server's Grace Period {#grace}

Retention operates while the server is running normally.  This
subsection specifies what happens when the server itself restarts
while it holds retained state, or while a new instance of a
retaining client is reclaiming.

A server is not required to preserve retained state across a
restart of the server.  Where it does not, a server restart
converts the situation into one that {{Section 8.4.2 of RFC8881}}
already covers, as follows.

If the server restarts during an absence interval, the retaining
client has not yet returned.  When it does, the server is in its
grace period or has left it, and the state of the retaining
client is gone.  The new instance sends EXCHANGE_ID with
EXCHGID4_FLAG_RETAIN_STATE as usual.  If the server is in its
grace period and its stable storage lists the client owner as
permitted to reclaim, the server sets
EXCHGID4_FLAG_RECLAIMABLE_R, and the client's reclaims are
ordinary grace-period reclaims, which is the first case in the
order of {{reclaim}}.  If the server's grace period has ended,
the flag is clear and the client has nothing to reclaim.

If the server restarts during a reclaim interval, the new instance
of the retaining client has not restarted again.  It observes the
server restart as any client does, establishes a new client ID,
and reclaims during the server's grace period the state it had
already recovered.  Reclaims it has not yet made become ordinary
grace-period reclaims.  It sends RECLAIM_COMPLETE when it has
finished, which for a gateway is when its front-side grace period
ends.

The point at which a server records a retaining client's lease
expiry in stable storage, specified in {{retain}}, matters here.
{{Section 8.4.3 of RFC8881}} has a server make that record so
that, after a server restart, a reclaim from a client whose lease
had expired is rejected with NFS4ERR_NO_GRACE, because another
client could have acquired a conflicting lock in the meantime.  A
retaining client whose state was still retained when the server
restarted had no such lease expiry, since nothing conflicting was
granted, and the server has made no record.  A retaining client
therefore remains eligible to reclaim across a server restart for
the whole of an absence interval, although its lease expired
during that interval.

RFC 8881 does not promise that a server's grace period outlasts a
gateway's.  {{Section 8.4.2.1 of RFC8881}} says that "the server
may also terminate the grace period before all clients have done
a global RECLAIM_COMPLETE".  A gateway that returns after the
server's grace period has ended, or whose front-side reclaims are
still arriving when the server's grace period ends, cannot
recover the remainder.  To make that outcome rare, a server SHOULD
record in stable storage that a client owner is a retaining
client, and SHOULD hold the server's grace period open until that
client owner has sent
RECLAIM_COMPLETE, bounded by the absence limit while the client
has not returned and by the reclaim cap once it has.
{{Section 8.4.2.1 of RFC8881}} already permits a grace period
that lasts until every known client has completed its reclaims.

These are recommendations rather than requirements because holding
the grace period open delays every client of the server, by up to
the absence limit when the retaining client never returns.  A
server operator has to be able to decline that cost.  What the
protocol cannot leave to policy is safety: a gateway grants a
front-side reclaim only when the corresponding back-side reclaim
succeeded, as {{gateway-reclaim}} specifies, so a server that
ends its grace period early causes a front-side client to be told
that its state is lost, and never causes state to be re-granted
without protection.  Continuity across a restart of both the
gateway and the server is therefore best effort, and safety is
not.


# Gateway Server Behavior {#gateway}

This section specifies how a gateway uses the extension.  The
requirements here are on the gateway, in both of its roles.
{{extension}} places no requirement on a gateway beyond those of
any retaining client.

## What Is Recoverable {#recoverable}

Retention preserves what the backend holds and nothing else.  A
front-side reclaim can succeed with its protection intact only if
the backend's conflict protection for the corresponding derived
state remained in force from the original grant until the new
instance of the gateway recovered it.  Front-side state that has
no derived counterpart cannot meet that condition.  A gateway is
free today to enforce some front-side state in its own tables
without creating derived state for it, and for such state a
gateway restart leaves the guarantee exactly as weak as it was
before: the gateway re-grants the state during its own grace
period, and the backend is not involved.

Share deny modes are the known case.  A gateway handles them in
one of two modes, and the choice belongs to the implementer.

Pass-through:
: The derived open carries at least the share deny bits of the
  front-side open or NLM_SHARE it stands for.  The backend
  enforces those bits against direct clients, retains them with
  the open, and matches them on reclaim.  A front-side deny mode
  recovered in this mode has been protected throughout.

Local-only:
: The derived open carries no share deny bits.  The gateway
  enforces the front-side deny mode in its own tables.  The deny
  mode binds front-side clients only, before and after a restart,
  and a front-side reclaim is re-granted from the gateway's own
  grace-period arbitration.  This document does not count that as
  recovered share exclusion.

A gateway that implements neither mode MAY refuse a front-side
request that carries a deny mode.

Only pass-through protects a front-side client against direct
clients.  A front-side client has no way to learn which mode it
was given, because neither NFSv4 nor NLM_SHARE can signal it, so
an operator who needs share exclusion against direct clients has
to know which mode the gateway runs.

A gateway's deny mode MUST NOT change across a restart of the
gateway.  A gateway that was local-only before a restart and
pass-through after it would send a reclaim OPEN with deny bits
that the retained open lacks, and the backend would answer
NFS4ERR_RECLAIM_BAD ({{reclaim}}).  The mode is therefore
persistent configuration ({{stable}}).  A gateway whose mode has
nonetheless changed retries the reclaim without deny bits and
treats the result as local-only.

Pass-through requires one back-side open-owner per front-side
owner ({{owners}}).  A deny mode conflicts with opens by other
open-owners, including other open-owners under the same client
ID, so with that mapping the backend arbitrates deny modes among
the gateway's front-side clients as well as against direct
clients.  A gateway that
aggregates many front-side owners under one back-side open-owner
is local-only for conflicts among its own clients, whatever bits
it sends, and after a restart it cannot rely on owner matching to
tell their opens apart.

Opens and locks that a gateway services locally under a back-side
delegation are a second case of state with no derived counterpart.
{{gateway-nfsv4}} specifies how a gateway avoids it.

## Owner Derivation {#owners}

The matching rule in {{reclaim}} requires the new instance of a
gateway to present the same open-owner and lock-owner strings that
the prior instance used.  The gateway has no record of those
strings after a restart other than what it can reconstruct from
the reclaim request itself, so a gateway MUST derive each
back-side owner string deterministically from values that a
front-side reclaim request supplies.  The backend treats owner
strings as opaque, so this document requires determinism and
recommends a construction without mandating one.

For an NLM front-side client, the gateway derives the back-side
lock-owner from the NLM caller name and the NLM owner handle.
NLM has no opens, so the gateway also derives a synthetic
open-owner from the same two values and holds one back-side open
per synthetic open-owner and file.  All of that NLM owner's locks
on the file are taken under that open, and the gateway closes the
open when the last of them is released.  The synthetic open has
to be reclaimable from an NLM reclaim request alone, which
supplies the file handle, the caller name, and the owner handle.
The gateway therefore chooses the open's share access by a rule
that needs nothing else.

For an NFSv4 front-side client, the gateway derives the back-side
owner from the front-side client owner string and the front-side
open-owner or lock-owner.

Before it forwards a reclaim, a gateway MUST check the reclaiming
front-side client against whatever identity the front-side
protocol authenticates, and the derived owner MUST be built from
values tied to that identity.  For NFSv4, a reclaim arrives under
a front-side client ID.  The gateway accepts it only from a client
owner in its record of clients permitted to reclaim ({{stable}}),
and only under the principal that {{RFC8881}} requires for that
client owner.  The derived owner includes the front-side client
owner string, so it is bound to that check without carrying the
principal itself.  For NLM, the caller name and owner handle are
supplied in the request and nothing in NLM authenticates them.
The gateway accepts a reclaim only for a caller in its NSM monitor
list.  It MAY also record the RPC credential or the source address
with that entry and require a match.  With RPCSEC_GSS that is a
binding to an authenticated identity.  With AUTH_SYS it is not.

On a front side that NLM does not authenticate, a host that can
reach the gateway and present another client's caller name and
owner handle during the gateway's grace period can reclaim that
client's lock.  The same is true of any NLM server, so the gateway
adds no exposure that NLM did not already have, but owner matching
on the backend does not remove it either.  {{security}} discusses
this further.

Each front-side component of an owner can be up to 1024 bytes
long, and so can the back-side owner string, so the construction
needs a hash.  A collision between two front-side owners makes
them share back-side state: the backend treats their opens and
locks as belonging to one owner, so a conflict between them is
not detected and a reclaim by one can recover state of the other.
A gateway SHOULD use a hash whose output length makes an
accidental collision negligible.

## Stable Storage {#stable}

A gateway keeps the following in stable storage, in addition to
whatever its front-side and back-side implementations already
require:

- the back-side client owner string, so that each instance of the
  gateway presents the same client owner to the backend;

- the record of front-side clients permitted to reclaim, which is
  the NFSv4 client list that {{Section 8.4.3 of RFC8881}}
  requires of any NFSv4 server, and the NSM monitor list that NLM
  requires of any NLM server, optionally with the RPC credential
  or source address of each NLM caller ({{owners}}); and

- the deny mode in use, pass-through or local-only
  ({{recoverable}}).

## Restart Sequence {#gateway-reclaim}

When a gateway starts after a restart, it proceeds in this order:

1. The gateway sends EXCHANGE_ID with EXCHGID4_FLAG_RETAIN_STATE
   to the backend, then CREATE_SESSION.

2. The gateway reads retain_absence_limit and retain_reclaim_cap
   from the backend ({{attrs}}).

3. The gateway begins its front-side grace period and notifies
   its front-side clients: SM_NOTIFY to the callers in its NSM
   monitor list, and the ordinary NFSv4 restart indications to its
   NFSv4 clients.

4. The gateway forwards each front-side reclaim to the backend as
   a back-side reclaim and answers the front-side client from the
   backend's reply.

5. When the front-side grace period ends, the gateway sends
   RECLAIM_COMPLETE to the backend.

The reclaim cap runs from the gateway's CREATE_SESSION.  A gateway
SHOULD choose a front-side grace period that ends, with time left
to send RECLAIM_COMPLETE, before the reclaim cap elapses.  The
front side sets a floor on that choice.  NFSv4 front-side clients
need at least one lease period of the gateway in which to notice
the restart and reclaim, and NLM clients need time to act on
SM_NOTIFY.  When the reclaim cap is shorter than that floor, the
gateway runs the grace period its front side needs, and reclaims
that arrive after the cap are refused as {{reclaim}} specifies.
A gateway SHOULD report that condition to its operator when it
starts, because the condition is a configuration error that the
operator can correct at the gateway or at the backend.

The gateway decides once, from the EXCHANGE_ID reply, whether
reclaims can succeed, and then the backend decides each reclaim.

If the reply carries EXCHGID4_FLAG_RECLAIMABLE_R, the gateway runs
the sequence above in full.  The gateway MUST grant a front-side
reclaim only if the corresponding back-side reclaim succeeded, and
MUST refuse a front-side reclaim whose back-side reclaim failed.
The gateway does not need to know whether the backend answered
from retained state or from its own grace period, because the
requests are the same in both cases ({{reclaim}}).  The gateway
withholds RECLAIM_COMPLETE until its front-side grace period ends.

If the reply does not carry EXCHGID4_FLAG_RECLAIMABLE_R, the
gateway still notifies its front-side clients, sends
RECLAIM_COMPLETE at once, and refuses every front-side reclaim.
This covers a return after the retained state was released, and a
backend that restarted and has either left its grace period or
has no record of this client owner.  A gateway treats a backend
that does not implement the extension, detected by NFS4ERR_INVAL
and the retry without the flag, or that declines retention, in the
same way.  The gateway notifies in every case because a refused
reclaim is the only way an NLM client learns that a lock is gone.

## NFSv3 Front-Side Clients {#gateway-nfsv3}

An NFSv3 client learns of the gateway's restart from SM_NOTIFY and
reclaims each lock with an NLM_LOCK request that has the reclaim
field set.  The gateway maps that request to a back-side OPEN with
CLAIM_PREVIOUS for the synthetic open-owner derived from the
request, followed by a LOCK with the reclaim field set under the
reclaimed open ({{owners}}).  If both succeed, the gateway grants
the NLM reclaim.  If either fails, the gateway denies it.

An NLM_SHARE request carries a deny mode.  The gateway handles it
in the deny mode it runs ({{recoverable}}).  In pass-through mode,
the derived open carries the deny bits and is reclaimed with them.
In local-only mode, the gateway re-grants the share from its own
tables during its grace period.

NLM does not gate READ and WRITE on recovery, so NFSv3 I/O
continues to arrive during the gateway's grace period.  The
gateway serves it either by opening the file on the backend with
a non-reclaim OPEN, which {{before-complete}} permits before
RECLAIM_COMPLETE, or with a special stateid.

## NFSv4 Front-Side Clients {#gateway-nfsv4}

An NFSv4 client learns of the gateway's restart from
NFS4ERR_BADSESSION or NFS4ERR_STALE_CLIENTID, as it would from any
server, and reclaims during the gateway's grace period.  The
gateway maps a front-side OPEN with CLAIM_PREVIOUS to a back-side
OPEN with CLAIM_PREVIOUS under the derived open-owner, and a
front-side lock reclaim to a back-side LOCK with the reclaim field
set under the reclaimed open, as for NLM.  Share deny modes
survive the restart if the gateway passed them through.

The gateway sends the back-side RECLAIM_COMPLETE when every
front-side client in the gateway's record of clients permitted to
reclaim has sent its own RECLAIM_COMPLETE, or when the front-side
grace
timer ends, whichever comes first.  NFSv4.0 clients have no
RECLAIM_COMPLETE, so for a gateway that serves them the timer
governs.

A gateway MUST NOT grant a delegation to a front-side client on
the strength of this extension.  A front-side client holding a
write delegation has opens and locks that neither the gateway nor
the backend knows about.  After a gateway restart, that delegation
can be honored only if the backend guaranteed that no conflicting
access occurred in the interim, which requires a back-side
delegation retained under CLAIM_DELEGATE_PREV ({{Section 10.2.1
of RFC8881}}).  A future document may permit a gateway to grant a
front-side delegation on that condition.

The prohibition covers front-side grants only.  A gateway MAY hold
back-side delegations for its own use, in the arrangement that
{{Section 5.1 of RFC9754}} describes for an NFSv3 server that is
an NFSv4.2 client.  This extension does not retain those
delegations ({{retain}}), so a gateway restart loses them.

{{Section 10.4.2 of RFC8881}} has a client that holds a write
delegation perform lock operations locally.  A gateway that did
so with its front-side clients' locks would create no derived
state for them: the delegation would be their only protection at
the backend, and the delegation is not retained.  After a restart
there would be nothing to reclaim, and direct clients would have
been free to take conflicting locks once the delegation was gone.
A gateway that uses this extension MUST therefore create derived
opens and locks at the backend for its front-side clients' opens
and locks even while it holds a delegation on the file.

A gateway that holds a back-side delegation with delegated
timestamps ({{Section 5 of RFC9754}}) is the authority for the
file's access and modify times until it returns the delegation,
and a restart loses the values only the gateway held.  Access
times for reads the gateway served from its cache are lost.
Modify times the gateway reported to front-side clients for data
the gateway had
already written may differ from the backend's own modify time for
those writes, so that after the restart front-side clients see
the modify time change with no other writer, and an NFSv3 client
treats that as a changed file and drops its cached data.  No data
is lost in either case.  A time that a front-side client set
explicitly with a SETATTR is different: if the gateway absorbed it
under the delegation and then restarted, the time is lost although
the gateway acknowledged it, and an NFSv3 server is required to
recover without data loss ({{Section 4.8 of RFC1813}}).  A
gateway MUST send an explicit time
change to the backend before it replies to the front-side client
that requested it.

## Backend Restart {#gateway-backend-restart}

When the backend restarts and the gateway does not, the gateway
holds all of its derived state in memory and reclaims it during
the backend's grace period as any NFSv4 client does.  Front-side
clients are not notified.  During the backend's grace period, the
backend returns NFS4ERR_GRACE for new locking requests and for
I/O, and the gateway maps that to what each front-side protocol
can express: NLM4_DENIED_GRACE_PERIOD for NLM, NFS3ERR_JUKEBOX for
NFSv3 I/O, and NFS4ERR_GRACE for NFSv4.  If the gateway cannot
reclaim some derived state, for example because it was partitioned
from the backend through the backend's grace period, the
front-side state that derived state stood for is no longer
protected, and the gateway reports the loss as {{lost}} describes.

When both the gateway and the backend restart, the backend is in
its ordinary grace period and holds no retained state.  It sets
EXCHGID4_FLAG_RECLAIMABLE_R if it is in its grace period and has
the gateway's client owner on record, so the gateway follows the
sequence in {{gateway-reclaim}} and forwards front-side reclaims,
which the backend treats as ordinary reclaims.  If the backend's
grace period ended before the gateway returned, the flag is clear
and the gateway refuses every reclaim.  If the backend's grace
period ends while the gateway is still forwarding reclaims, the
remaining back-side reclaims fail with NFS4ERR_NO_GRACE and the
gateway refuses the corresponding front-side reclaims.
Continuity in this case therefore depends on the backend's grace
period outlasting the gateway's, which {{grace}} recommends but
{{RFC8881}} does not promise.  The safety rule of
{{gateway-reclaim}} holds regardless: nothing is re-granted on the
front side without a successful back-side reclaim.

When the backend restarts during the gateway's reclaim interval,
the gateway has not restarted again.  It observes the backend
restart, establishes a new client ID, and reclaims the derived
state it has already recovered as any client does.  Front-side
reclaims still arriving are forwarded as ordinary reclaims.  The
gateway sends RECLAIM_COMPLETE to the new backend instance when
the gateway's front-side grace period ends.  The same safety rule decides
each reclaim.

## Reporting Lost State {#lost}

A front-side client's state is lost when the backend released or
revoked the corresponding derived state after the absence limit
or the reclaim cap was reached, when an administrator released it,
when a back-side reclaim was refused, or when the gateway's lease
on the backend expired during a partition and the backend revoked
the state.  In the last case the front-side clients renewed their
own leases with the gateway throughout and did nothing wrong, yet
their state is gone.

To an NFSv4 front-side client, the gateway reports the loss with
the mechanisms {{Section 8.4.3 of RFC8881}} provides: the
affected stateids are revoked, and the SEQ4_STATUS flags in the
next SEQUENCE reply tell the client that some of its state has
been revoked.

To an NLM client, the gateway has no way to report a loss other
than to deny the client's reclaim after SM_NOTIFY.  NLM provides
no notification for a lock lost while the client was not
reclaiming.  This is a limitation of NLM, which this document
records and does not fix.

## Chained Gateways {#chained}

This subsection is not normative.  It describes how the mechanism
behaves when a gateway's backend is itself a gateway, a
configuration this document does not otherwise specify.

In a chain of two gateways, the outer gateway serves the
front-side clients and mounts the inner gateway, and the inner
gateway mounts the backend.  Both mounts use NFSv4.2.  The inner
gateway takes both roles this document defines.  Toward the
backend it is a retaining client.  Toward the outer gateway it is
a server that implements the extension, and the outer gateway is
its retaining client.  No protocol element beyond those in
{{extension}} is involved.

The safety rule of {{gateway-reclaim}} applies at each gateway
separately.  The outer gateway grants a front-side reclaim only if
the inner gateway granted the corresponding reclaim, and the inner
gateway grants that reclaim only if the backend did.  A front-side
reclaim therefore succeeds only if the backend's protection for
the state was continuous, however many gateways the chain has.
What the chain puts at risk is continuity, and the cases below
differ in how much of it survives.

When the outer gateway restarts alone, the inner gateway retains
the outer gateway's state by continuing to hold the derived state
on the backend.  The inner gateway's lease on the backend never
lapses, so the backend observes nothing and the backend's limits
do not come into play.  The inner gateway's absence limit and
reclaim cap are the only bounds on how long that state blocks
other clients of the backend.

When the inner gateway restarts alone, the outer gateway sees an
ordinary restart of its backend and behaves as
{{gateway-backend-restart}} describes.  It reclaims the derived
state it holds in memory during the inner gateway's grace period
and does not notify its front-side clients.  The inner gateway
forwards each of those reclaims to the backend, which answers from
retained state.

When both gateways restart, the reclaim intervals nest.  The outer
gateway withholds RECLAIM_COMPLETE from the inner gateway until
the outer gateway's front-side grace period ends.  The inner
gateway's grace period has to stay open until then, and the inner
gateway withholds RECLAIM_COMPLETE from the backend for as long as
its grace period is open.  The backend's reclaim cap therefore has
to cover the inner gateway's grace period, which in turn has to
cover the time the outer gateway takes to restart plus the outer
gateway's grace period.  Each further gateway in a chain adds its
own restart time and grace period to what the backend's reclaim
cap has to cover.  Each gateway can read the limits of the server
behind it ({{attrs}}).  An inner gateway can therefore report to
the outer gateway limits that fit inside the ones the backend
reported to it, and the outer gateway sizes its front-side grace
period from those.  Where the limits do not nest, the
reclaims still outstanding when the shorter limit is reached fail,
and the front-side clients that sent them are told that their
state is lost.

The recommendations in {{grace}} carry more weight for an inner
gateway than for a backend.  If the inner gateway restarts while
the outer gateway is absent, and the inner gateway does not hold
its grace period open for the outer gateway, the grace period ends
with no reclaim received.  The inner gateway then sends
RECLAIM_COMPLETE, and the backend releases all of the state.  An
inner gateway that records the outer gateway as a retaining client
and holds its grace period open avoids that outcome.  How long it
can wait is bounded by its own absence limit and, because it is
withholding RECLAIM_COMPLETE from the backend throughout, by the
backend's reclaim cap.

An inner gateway that has itself restarted can set
EXCHGID4_FLAG_RECLAIMABLE_R in its reply to the outer gateway only
if the backend set the flag in its reply to the inner gateway.
Otherwise the outer gateway would run a front-side grace period in
which every reclaim is refused.

Share deny modes are protected against direct clients only if
every gateway in the chain runs in pass-through mode
({{recoverable}}).  A single local-only gateway drops the deny
bits at that point in the chain, and the backend never sees them.
Owner derivation needs nothing further.  The inner gateway derives
its back-side owners from the outer gateway's client owner string
and from the owners the outer gateway derived ({{owners}}), so
each front-side owner still maps to one owner at the backend, and
the mapping is the same after any restart.

The inner gateway does not grant delegations to the outer gateway
({{gateway-nfsv4}}).  The outer gateway therefore holds no
back-side delegations, and it gives up the caching that such
delegations would have allowed.


# Backend Server Behavior {#backend}

{{extension}} specifies what a server does for a retaining
client.  This section covers the choices a server makes in doing
so, and their effect on the server's other clients.

## Choosing the Limits {#limit-values}

The absence limit bounds how long a gateway that has not returned
blocks direct clients.  It needs to be long enough to cover a
reboot of the gateway host, and short enough that an operator can
tolerate it when the gateway does not come back.  A server
SHOULD make the absence limit configurable.

The reclaim cap bounds how long a gateway that has returned but
not completed blocks direct clients from state it is not going to
reclaim.  A gateway's front-side grace period is typically at
least one lease period of the gateway, and a cap shorter than
that causes late front-side reclaims to be refused and their
clients told that state is lost.  A server SHOULD make the reclaim
cap configurable, and its default SHOULD be no shorter than the
server's own lease period.  A gateway reads the reclaim cap and
fits its front-side grace period inside it where it can
({{gateway-reclaim}}), but a gateway cannot shorten that grace
period below what its front side needs.  The default therefore
has to suit a gateway that nobody has tuned.

## Resource Limits {#resources}

Retained state occupies server resources after the lease that
would have released it has expired.  A server MAY limit the amount
of state it retains, per retaining client or in total.  When a
server declines to retain state because of a resource limit, it
releases that state as it would for a client that was not a
retaining client, and a later reclaim for it receives
NFS4ERR_RECLAIM_BAD or NFS4ERR_NO_GRACE ({{reclaim}}).  A server
that cannot retain the state of a client owner at all SHOULD
decline retention at EXCHANGE_ID ({{negotiate}}) rather than
accept and then release.

## Administrative Release {#admin}

An operator may need to release retained state before a limit is
reached, for example when a gateway is known not to be returning.
A server SHOULD provide a way to release the retained state of a
client owner.  Administrative release has the effect of a limit
reached followed by immediate release ({{limits}}), and the server
makes the stable-storage record of {{retain}} at that point.

What the gateway's front-side clients then see depends on whether
the gateway had already reclaimed the corresponding derived state.
For derived state not yet reclaimed, the
gateway's later reclaim is refused and the gateway denies the
front-side reclaim, which both NFSv4 and NLM can express.  For
state the new instance had already reclaimed, administrative
release is ordinary revocation of the gateway's own live state at
the backend, which the gateway reports to NFSv4 front-side clients
as {{lost}} describes and cannot report to NLM front-side clients
at all.

## Stable Storage {#backend-stable}

{{retain}} requires a server to make the lease-expired record of
{{Section 8.4.3 of RFC8881}} when it releases or revokes retained
state rather than at lease expiry.  For state released at a limit
or administratively, the record is made at release.  For state a
server chooses to keep past a limit, the record MUST be made when
the server first grants a request that conflicts with that state,
because from that moment another client may hold a conflicting
lock, which is the condition the record exists to capture.  A
server for which that bookkeeping is impractical releases retained
state outright when a limit is reached, which {{limits}} permits.

A server that follows the recommendation of {{grace}} also records
that a client owner is a retaining client, so that after a server
restart it can hold its grace period open for that client owner.

## Several Retaining Clients {#several}

A server may have more than one retaining client, for example two
gateways that export the same file system, or a gateway and
another client that can reconstruct its state.  Retention is per
client owner.  Each retaining client's state is retained, bounded,
and released independently, and the retained state of one blocks
the others exactly as it blocks direct clients.  Nothing in this
extension coordinates the recovery of one retaining client with
another.

## Effect on Direct Clients {#direct}

A direct client whose request conflicts with retained state
receives NFS4ERR_SHARE_DENIED or NFS4ERR_DENIED ({{retain}}).  A
direct client can already receive both errors, so a direct client
needs no change to operate against a server that implements this
extension.  What changes is duration: a conflict that
{{Section 8.4.3 of RFC8881}} would have resolved in the direct
client's favor at the retaining client's lease expiry now persists
until a reclaim moves the state to the new instance, a limit is
reached, or the state is released.  A direct client that blocks
on a byte-range lock, as {{Section 9.6 of RFC8881}} allows, waits
accordingly.  The limits in {{limit-values}} are how an operator
bounds that cost, and {{security}} considers it as a denial of
service.


# XDR Description {#xdr}

This extension adds two flag constants and two attributes to the
XDR description of NFSv4.2 in {{RFC7863}}.  No operations are
added, and the XDR of EXCHANGE_ID is unchanged: the flag constants
name bits in the existing eia_flags and eir_flags words.

~~~
/// /*
///  * EXCHANGE_ID flags defined by this document.  They are
///  * used in eia_flags and eir_flags alongside the flags
///  * defined in RFC 8881 Section 18.35.1 and RFC 7862
///  * Section 14.1.
///  */
/// const EXCHGID4_FLAG_RETAIN_STATE        = 0x00000008;
/// const EXCHGID4_FLAG_RECLAIMABLE_R       = 0x20000000;
///
/// /*
///  * Attributes defined by this document.  Each value is a
///  * count of seconds.
///  */
/// typedef uint32_t        fattr4_retain_absence_limit;
/// typedef uint32_t        fattr4_retain_reclaim_cap;
///
/// const FATTR4_RETAIN_ABSENCE_LIMIT       = 92;
/// const FATTR4_RETAIN_RECLAIM_CAP         = 93;
///
~~~

EXCHGID4_FLAG_RETAIN_STATE takes the next unassigned bit after the
capability flags of {{RFC8881}} and {{RFC7862}}, which it
resembles in being set by the client and echoed by the server.
EXCHGID4_FLAG_RECLAIMABLE_R takes a bit adjacent to
EXCHGID4_FLAG_CONFIRMED_R, which it resembles in being set only
by the server.  {{negotiate}} specifies the use of both.

{{attrs}} specifies the two attributes.  Their numbers are
provisional.  Other extensions to NFSv4.2 that are in progress
also assign attribute numbers, and the numbers here will change
if they collide with an assignment that is published first.

## Extraction of XDR {#extract}

The XDR description is embedded in this document in a way that
makes it simple for the reader to extract into a ready-to-compile
form, following the practice of {{RFC7863}} and {{RFC9754}}.  The
reader can feed this document into the following shell script to
produce the machine-readable XDR description of the new
constants and types:

~~~
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
~~~

That is, if the above script is stored in a file called
"extract.sh" and this document is in a file called "spec.txt",
then the reader can do the following:

~~~
sh extract.sh < spec.txt > state_retention_prot.x
~~~

The effect of the script is to remove leading blank space from
each line, plus a sentinel sequence of "///".  The extracted
definitions are written in the XDR language of {{RFC4506}} and
belong with the EXCHANGE_ID constants and the attribute
definitions in the nfs4_prot.x file generated from {{RFC7863}}.


# Security Considerations {#security}

The security considerations of {{Section 21 of RFC8881}} apply.
This extension changes nothing about how a server authenticates a
client or authorizes an operation.  What it changes is how long
state outlives the client instance that established it, and who
is able to recover that state.  Both raise considerations of their
own.

## Denial of Service {#dos}

A retaining client's opens and locks continue to block other
clients after its lease expires, for up to the absence limit, and
state the new instance never reclaims continues to block them for
up to the reclaim cap.  Any client that becomes a retaining client
therefore gains the ability to deny access to files for longer
than {{RFC8881}} otherwise permits, whether through a fault or by
intent.  Three things bound this.  Both limits are mandatory
({{limits}}), so the exposure is finite and under the server
operator's control.  Whether a client owner may be a retaining
client is a server policy decision ({{negotiate}}), and a server
SHOULD confine retention to principals that are authorized to act
as gateways, so that an arbitrary client cannot opt in.  A server
SHOULD require SP4_MACH_CRED state protection for a retaining
client, so that only the holder of that client's machine
credential can establish the next instance of the client owner
and so reclaim, or decline to reclaim, the retained state.

Administrative release ({{admin}}) is the operator's recourse when
a retaining client is not coming back and the absence limit is
too long to wait.

## Reclaiming State the Prior Instance Did Not Hold {#sec-reclaim}

A reclaim succeeds only when it matches retained state of the same
client owner under the same owner string ({{reclaim}}).  A new
instance of a retaining client therefore cannot recover state that
the prior instance did not hold, and no client can recover another
client owner's retained state.  SP4_MACH_CRED, where required,
ties the client owner to a credential.

The server cannot see further than the client owner.  A gateway
is one client owner to the server, and the server cannot tell
which front-side client a reclaim is for.  Owner matching is a
record of who held what, not authentication: a reclaim under the
right owner string is honored whichever front-side client
presented it to the gateway.  Whether one front-side client can
reclaim another's state is decided entirely on the front side, by
what the gateway checks before it forwards ({{owners}}).
SP4_MACH_CRED protects the gateway's identity toward the server
and says nothing about the identity of the gateway's clients.

## Front-Side Identity {#sec-front}

A gateway can identify a reclaiming front-side client only as
well as the front-side protocol allows.  NFSv4 ties a reclaim to
a client ID and to the principal that established it, and a
gateway accepts a reclaim only from a client owner in its record
of clients permitted to reclaim and under that principal.  NLM
identifies the holder of a lock by a caller name and an owner
handle that the client supplies, and nothing in NLM authenticates
either.  A gateway accepts an NLM reclaim only for a caller in its
NSM monitor list, and may bind the entry to an RPC credential or
a source address, which is an authenticated binding only under
RPCSEC_GSS.

On an unauthenticated NLM front side, a host that can reach the
gateway during its grace period and present another client's
caller name and owner handle can reclaim that client's lock.  Any
NLM server has this exposure, so a gateway using this extension
adds none that NLM deployments do not already accept.  What this
extension does not do is remove it: a successful back-side reclaim
shows that the owner string matched, not that the right front-side
client sent it.  Deployments that need stronger assurance use an
NFSv4 front side with RPCSEC_GSS, or an NLM front side where the
gateway binds reclaims to RPCSEC_GSS credentials.

Because the back-side owner string is a hash of front-side values
({{owners}}), a collision between two front-side owners makes
them one owner on the server.  A conflict between them is not
detected, and either can reclaim the other's state.  The hash
length SHOULD make an accidental collision negligible, and a
gateway on an untrusted front side SHOULD consider whether a
front-side client can choose its owner values so as to collide
with another's.

## False Assurance {#sec-assurance}

A gateway that granted a front-side reclaim without a successful
back-side reclaim would tell an application that its lock was
protected throughout the gateway's restart when it was not.  The
application might then act on data that another client changed in
the interim.  {{gateway-reclaim}} forbids this: a gateway MUST
grant a front-side reclaim only when the corresponding back-side
reclaim succeeded.  The same rule protects against a server that
ended its grace period early ({{grace}}) and against retained
state released at a limit: in each case the application learns
that its state was lost rather than proceeding on a false
assumption.  Local-only deny modes ({{recoverable}}) are the one
place a front-side client receives a weaker guarantee than it may
assume, and a front-side client cannot detect that mode.

## Transport Security {#sec-transport}

The connection between a gateway and its backend carries the
reclaims that decide whether every front-side client's state
survives.  A reclaim that is modified or injected in transit
changes which owner recovers which state.  Deployments SHOULD
carry that connection over an integrity-protecting channel, such
as RPCSEC_GSS with integrity or privacy protection, or RPC over
TLS ({{RFC9289}}).  This extension introduces no requirement on
the transport beyond those of {{RFC8881}}.


# IANA Considerations {#iana}

This document has no IANA actions.

The two EXCHANGE_ID flag bits that {{xdr}} assigns are not
managed by an IANA registry.  {{RFC8881}} assigns the existing
flag bits in its XDR description, and {{RFC7862}} added one in the
same way.  The two attribute numbers that {{xdr}} assigns are not
managed by an IANA registry either.  {{RFC8881}} and {{RFC7862}}
assign attribute numbers in their XDR descriptions, and
{{RFC9754}} added attributes in the same way.  This document
follows that practice.


--- back

# Example Recovery Sequences {#sequences}

This appendix is not normative.  It walks through the mechanism
as a gateway and a backend use it, so that the requirements in
{{extension}} can be read with a whole sequence in mind.

## Gateway Restart with an NLM Client

The sequence below has an NFSv3 client C, a gateway G, a backend
B, and a direct client D.  L is the backend's lease time.  The
owner strings that G derives from C's identity are written g(C)
for the open-owner and f(C) for the lock-owner.

Before the restart:

1. G sends EXCHANGE_ID with EXCHGID4_FLAG_RETAIN_STATE to B.  B
   echoes the flag.  G creates a session and sends
   RECLAIM_COMPLETE.
2. C sends NLM_LOCK for an exclusive byte range to G.
3. G sends OPEN with CLAIM_FH and open-owner g(C) to B, then LOCK
   with lock-owner f(C) under that open.  G records C in its NSM
   monitor list.

During the absence:

{: start="4"}
4. G crashes at time t0.
5. At t0 + L the lease expires.  B keeps the open and the lock.
6. D sends a conflicting LOCK to B.  B returns NFS4ERR_DENIED.

After the return, before the absence limit:

{: start="7"}
7. G sends EXCHANGE_ID with a new verifier and
   EXCHGID4_FLAG_RETAIN_STATE to B.  B replies with both
   EXCHGID4_FLAG_RETAIN_STATE and EXCHGID4_FLAG_RECLAIMABLE_R.
8. G sends CREATE_SESSION.  B destroys the old client ID and its
   sessions, and keeps the old instance's opens and locks as
   retained state.
9. G reads retain_reclaim_cap from B and chooses a front-side
   grace period that ends before the cap elapses from step 8.  G
   starts that grace period and sends SM_NOTIFY to C.
10. C sends NLM_LOCK with the reclaim flag set, for the same owner
    and range, to G.
11. G sends OPEN with CLAIM_PREVIOUS and open-owner g(C) to B.  B
    matches the retained open from step 3, moves it to the new
    client ID, and returns a new stateid.
12. G sends LOCK with the reclaim flag set and lock-owner f(C) under
    the reclaimed open.  B matches the retained lock, moves it to
    the new client ID, and returns a new stateid.
13. G grants C's reclaim.
14. G's front-side grace period ends.  G sends RECLAIM_COMPLETE to
    B.  B releases all remaining retained state.

An NFSv3 READ or WRITE from C between steps 8 and 14 needs no
reclaim.  If G opens the file to serve it, that is a non-reclaim
OPEN before RECLAIM_COMPLETE, which {{extension}} permits for a
retaining client.

## Variations

Return before the lease expires:
: This is the common case, a gateway that reboots in less than L.
  Steps 5 and 6 do not occur.  The old instance's open and lock
  are ordinary unexpired state, and D's request is denied for that
  reason.  B still sets EXCHGID4_FLAG_RECLAIMABLE_R in step 7,
  because B holds state of a prior instance that it will retain on
  confirmation.  Retention happens at step 8, where CREATE_SESSION
  confirms the new instance and B keeps the old instance's state
  instead of releasing it.  The remaining steps are unchanged.

Return after the absence limit:
: If B released the state at the limit, or revoked it for a
  conflicting request afterward, step 7 returns
  EXCHGID4_FLAG_RETAIN_STATE without EXCHGID4_FLAG_RECLAIMABLE_R.
  G still sends SM_NOTIFY, sends RECLAIM_COMPLETE at once, and
  denies C's reclaim.  If B kept the state past the limit and no
  conflict arrived, the sequence runs unchanged.

Second restart during the reclaim interval:
: Suppose a second NFSv3 client, C2, took a lock through G before
  step 4 and is down throughout, so its lock is never reclaimed.
  Steps 7 through 13 run as written, with step 8 at time T2.  C's
  open and lock are reclaimed.  C2's remain retained, with a
  reclaim-cap clock that started at T2.  G crashes again before
  step 14.  When its lease expires, B keeps C's open and lock as
  well.  G returns as a third instance and is confirmed at time
  T3.  C's open and lock are retained with a clock that starts at
  T3.  The clock for C2's state still runs from T2.  A later
  confirmation never restarts the clock on state retained earlier,
  so a gateway that crashes in a loop cannot block direct clients
  without bound with state it never reclaims.  At T2 plus the
  reclaim cap, C2's open and lock stop blocking, and B revokes them
  when it grants D's conflicting LOCK.  When G's front-side grace
  period ends, G sends RECLAIM_COMPLETE and B releases whatever is
  still retained.

## Gateway Restart with an NFSv4 Client

An NFSv4 front-side client differs from the NLM sequence in these
ways.  The front-side client learns of the gateway's restart from
NFS4ERR_BADSESSION or NFS4ERR_STALE_CLIENTID rather than from
SM_NOTIFY, and the gateway relies on its ordinary NFSv4 server
stable storage, the list of clients permitted to reclaim.  A
front-side OPEN with CLAIM_PREVIOUS maps to a back-side OPEN with
CLAIM_PREVIOUS under the derived open-owner, and share deny modes
survive if the gateway passed them through.  Front-side lock
reclaims map as in the NLM sequence.  The gateway sends the
back-side RECLAIM_COMPLETE when every recorded front-side client
has sent its own, or when the front-side grace timer ends.  NFSv4.0
clients have no RECLAIM_COMPLETE, so for them the timer governs.
Delegation reclaims do not arise, because a gateway using this
extension grants no delegations.


# Acknowledgments
{:numbered="false"}

Much of the prose in this document was generated by Claude, a large
language model, working from the editor's design, outline, and
instructions, and was reviewed and revised by the editor, who is
responsible for its content.  Claude is not an author of this
document.

The editor is grateful to
Bill Baker,
Greg Marsden,
and
Martin Thomson
for their input and support.

Special thanks to
Area Director
Gorry Fairhurst,
NFSv4 Working Group Chair
Brian Pawlowski,
and
NFSv4 Working Group Secretary
Thomas Haynes
for their guidance and oversight.
