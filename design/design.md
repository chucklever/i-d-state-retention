# Design Notes: State Recovery for NFS Gateway Servers

Companion to `outline.md`.  Covers outline sections 5 through 7
only: the mechanism, the wire extension, and gateway behavior.
Its job is to settle the open questions and to serve as the
reference when the draft is written and reviewed.

Each statement below is one of three kinds, and is marked where
the kind is not obvious:

- **RFC**: checked against RFC 8881 text in this pass, with the
  section cited.
- **Proposal**: a design choice made here.  Open to challenge.
- **Unverified**: an assumption about what implementations can
  do that nobody has checked yet.  The design must not depend on
  the internal choices of any one implementation; a check
  against one is evidence of feasibility, not a constraint.

## 1. The Mechanism in One Page

A gateway is an NFSv4.2 client of a backend server.  It holds
opens and byte-range locks at the backend on behalf of its own
clients ("derived state").

1. **Negotiate.**  The gateway sets a new flag in EXCHANGE_ID.  A
   backend that supports the extension and is willing to do this
   for the requesting principal echoes the flag.
2. **Retain.**  For a client ID established with the flag, the
   backend does not release opens and byte-range locks when the
   lease expires or when a new instance of the same client owner
   is confirmed.  The state stays in place and continues to
   conflict with requests from other clients.
3. **Reclaim.**  The new gateway instance sends ordinary
   reclaim-type requests (OPEN with CLAIM_PREVIOUS, LOCK with
   reclaim set).  The backend accepts them outside its grace
   period when they match retained state, and moves that state to
   the new client ID.
4. **Complete.**  The gateway sends RECLAIM_COMPLETE when its own
   front-side grace period ends.  The backend releases whatever
   retained state was not reclaimed.

The backend bounds two intervals: how long retained state blocks
other clients for a gateway that has not come back (the "absence
limit"), and how long the unreclaimed remainder blocks them once
a new instance has a session (the "reclaim cap").  Within the
cap, the new instance's lease renewals keep the remainder in
place until RECLAIM_COMPLETE.  The backend advertises both
limits as attributes, so the gateway can fit its front-side
grace period inside the cap.

The mechanism is not specific to gateways.  Any NFSv4.2 client
that can reconstruct its lock state after a restart could use it
(a user-space NFS proxy, an SMB server exporting an NFS mount).
The draft should define it as client-restart state retention and
present the gateway as the motivating use.

## 2. What RFC 8881 Says Today

These are the rules the extension changes or builds on.

- **Client restart discards state.**  EXCHANGE_ID with a known
  client owner and principal but a new verifier is the "client
  restart" case.  Once CREATE_SESSION confirms the new client ID,
  byte-range locks and share reservations "should be released
  immediately" (Section 18.35.4, case 5; Section 8.4.1).
- **Precedent for retention exists, for delegations only.**  A
  server MAY support CLAIM_DELEGATE_PREV.  If it does, it MUST NOT
  remove delegations when the new client ID is confirmed and MUST
  keep them for at least lease_time so the restarted client can
  reclaim them.  DELEGPURGE discards the rest (Section 10.2.1).
  The extension does for opens and locks what this does for
  delegations.
- **Expired state must yield to conflicts.**  A server may keep an
  expired client's locks, but they "do not prevent such a
  conflicting lock from being granted" and MUST be revoked on
  conflict (Section 8.4.3).  The extension overrides this for
  retaining clients, up to a limit (D3).
- **Reclaim outside grace is refused.**  After a partition the
  server "will not allow the client to reclaim locks, because the
  server will not be in its recovery grace period" (Section
  8.4.3).  The error is NFS4ERR_NO_GRACE.
- **RECLAIM_COMPLETE gates new locks.**  A client with a new
  client ID MUST send a global RECLAIM_COMPLETE before its first
  non-reclaim locking operation; otherwise the operation fails
  with NFS4ERR_GRACE.  It can be sent once per server instance
  (Section 18.51.3, 18.51.4).
- **Non-reclaim requests during grace can be allowed.**  The
  server may grant them when it "can reliably determine (through
  state persistently maintained across restart instances) that
  granting any such lock cannot possibly conflict with a
  subsequent reclaim" (Section 8.4.2.1).  The exception applies
  to a server in its own grace period.  It does not lift the
  RECLAIM_COMPLETE gate above.
- **A lock belongs to an open.**  A byte-range lock is
  "associated with a lock-owner and an open-owner, the latter
  being the open-owner associated with the open file under which
  the LOCK operation was done" (Section 9.1.1).
- **Unknown EXCHANGE_ID flags are rejected.**  "Bits not defined
  above cannot be set in the eia_flags field.  If they are, the
  server MUST reject the operation with NFS4ERR_INVAL" (Section
  18.35.3).  Bits in use: 0x1, 0x2, 0x100, 0x10000, 0x20000,
  0x40000, 0x40000000, and 0x80000000 in RFC 8881 (Section
  18.35.1), plus 0x4, EXCHGID4_FLAG_SUPP_FENCE_OPS, from RFC
  7862 (Section 14.1).
- **Fencing delays the reply to a restarted client.**  When
  EXCHGID4_FLAG_SUPP_FENCE_OPS is in effect, the server does not
  reply to an EXCHANGE_ID "on the same client owner with a new
  verifier until all operations in progress on the client ID's
  session are completed or aborted" (RFC 7862 Section 14.1.4).  That is the EXCHANGE_ID a
  returning gateway sends, and the extension leaves the rule in
  place.

## 3. Decisions

### D1. Negotiate with EXCHANGE_ID flags

**Proposal.**  Two new flags.

- `EXCHGID4_FLAG_RETAIN_STATE`: set by the client to request
  retention for this client owner.  Set by the server in
  `eir_flags` when it will retain.
- `EXCHGID4_FLAG_RECLAIMABLE_R`: result only.  Set when
  reclaim-type requests from this client owner can succeed once
  the new client ID is confirmed.  That is so in three cases:
  - the server holds retained state from a prior instance whose
    lease expired;
  - the server holds state of a prior instance whose lease has
    not expired, and will retain it when CREATE_SESSION confirms
    the new instance (Section 18.35.4 replaces the old record at
    that step, not at EXCHANGE_ID);
  - the server is in its own grace period and its stable storage
    lists this client owner as permitted to reclaim.

  The name says what the client can do, not why, because the
  client acts the same way in all three cases (D8).

The server sets either flag in `eir_flags` only when the request
set `RETAIN_STATE`.  RFC 8178 Section 6 allows an extension to
change the response to an existing operation only if the server
can determine "that it is aware of the existence of XDR changes"
before it responds, and the request flag is that evidence.

Why: a server without the extension returns NFS4ERR_INVAL, which
is an unambiguous signal, and the gateway retries without the
flag.  The flag is recorded with the client record, so the
server knows at lease expiry how to treat the state.  No new
operation is needed.

`RECLAIMABLE_R` is a hint, and it is given before
confirmation, so it is not proof that anything was retained.
The authoritative answer is the result of each reclaim.  The
gateway uses the hint to decide whether to run a front-side
grace period at all (D8).

RFC 8178 Section 4.4.3 qualifies that signal: NFS4ERR_INVAL
shows a flag bit is unknown only if the requester avoids the
other conditions under which the operation returns that error.
EXCHANGE_ID has such conditions.  The retry settles it: if the
same request succeeds without the flag, the flag was the cause.

Rejected:
- A new operation.  It would carry more (for example the
  absence limit) but adds an operation number and XDR for
  two integers that attributes can carry (D3).
- A new attribute that advertises support.  RFC 8178 Section 6
  offers this as a convenience, and RFC 9754 uses it
  (fattr4_open_arguments) for its OPEN flags.  An attribute is
  read per file system, with a filehandle.  Retention is a
  property of a client ID, and the gateway has to learn of it
  at EXCHANGE_ID, before it has a session.  A reviewer may still
  ask for one, so the draft should give this reason.  The
  objection is to learning of support that way.  The limits are
  wanted only once a session exists, and D3 does advertise them
  as attributes.

### D2. What is retained

**Proposal.**

- Retained: opens, with the share access and deny modes the
  backend holds for them, and byte-range locks.  Deny modes a
  gateway did not pass through are not there to retain (D12).
- Not retained: layouts, sessions, the old client ID's stateids.
- Delegations: unchanged from RFC 8881.  See D7.

Retained state behaves as if still held.  Other clients get
NFS4ERR_DENIED or NFS4ERR_SHARE_DENIED, not a grace error.  This
is the explicit override of the Section 8.4.3 MUST, and it lasts
until a limit in D3 is reached.  After that, Section 8.4.3
applies again.

### D3. Two intervals, two limits

**Proposal.**

- **Absence interval:** from lease expiry of the old instance
  until a new instance is confirmed.  Bounded by the backend's
  absence limit, a local policy value.
- **Reclaim interval:** from confirmation of the new instance
  until its RECLAIM_COMPLETE.  Bounded by the backend's reclaim
  cap, a local policy value that every backend MUST have.  The
  cap is measured from the confirmation that began the interval,
  and a later restart does not start it over (D10).  Until it is
  reached,
  retained state not yet reclaimed is kept as long as the new
  instance's lease is renewed.
- **What a limit does.**  When either limit is reached, the
  retained state it governs stops blocking conflicting requests.
  It need not be destroyed.  The backend MAY release it at once,
  or keep it as RFC 8881 Section 8.4.3 allows for expired state
  and revoke it when the first conflicting request arrives.  A
  reclaim that finds the state still there succeeds, and safely:
  nothing that conflicts was granted, so the invariant in D12
  holds.  A reclaim for state that was released fails (D5).
- **What the cap does not touch.**  State the new instance has
  already reclaimed is ordinary state of a live client.  The cap
  governs only the unreclaimed remainder.
- **Administrative release** of retained state has the effect of
  a limit followed by immediate release.

**Both limits are advertised.**  Two new attributes carry them,
retain_absence_limit and retain_reclaim_cap, each a count of
seconds and each read-only.  They are per-server attributes in
the manner of lease_time (RFC 8881 Section 5.8.1.11): a client
reads them with GETATTR on any filehandle and gets the same
answer.

- A backend that implements the extension MUST support both.
  A backend without the extension does not list them in
  supported_attrs.
- To a retaining client the backend reports the limits it
  applies to that client ID.  A backend whose policy differs by
  principal (D11) reports the values for the requester.  To any
  other client it reports its defaults, which bind nothing.
- The values are the configured limits, not the time remaining.
  The gateway knows when each interval began: its own
  CREATE_SESSION started the reclaim interval.
- The backend applies to each client ID the limits in effect
  when that client ID was confirmed.  It MUST NOT reach a limit
  earlier than the value it reported for that client ID.  A
  later change of configuration reaches client IDs confirmed
  after it.  Administrative release, a resource limit, and a
  restart of the backend are not limits and are not promised
  against.
- After repeated restarts (D10) the cap for state retained in an
  earlier round is already running.  The attribute is the full
  cap, so it overstates what is left for that remainder.

What the gateway does with them: after CREATE_SESSION it reads
both in its first GETATTR, as a client reads lease_time.  It
SHOULD end its front-side grace period, and send
RECLAIM_COMPLETE, before the reclaim cap elapses from its
CREATE_SESSION.  A front-side grace period cannot be shorter
than the front side needs (one front-side lease period for
NFSv4 clients), so a cap below that leaves the gateway unable to
fit.  The gateway then runs the grace period it needs, late
reclaims are refused as before, and the gateway SHOULD report
the mismatch to its operator at startup, which it could not
detect before.  Each reclaim's result stays authoritative (D8).

Why advertise, when the first draft did not: the hint in D1
tells a gateway whether reclaims can succeed, not for how long.
Without the cap a gateway sizes its front-side grace period
blind.  A chain of gateways needs the numbers more: the inner
gateway's grace period has to cover the outer gateway's, and the
backend's cap has to cover both.  With the limits readable each
tier computes that, and an inner gateway reports to the outer
gateway limits it can keep given what the backend reported to
it.

The floor stays: the default reclaim cap SHOULD be no shorter
than one backend lease period (Section 7, item 5).

Why the cap is mandatory: without it, a faulty gateway that
returns, renews its lease, and never sends RECLAIM_COMPLETE
blocks direct clients indefinitely with state nobody is going to
reclaim.  A gateway that reclaims the state holds it
legitimately; the cap is aimed at the remainder.

**Loss of lease during the reclaim interval.**  If the new
instance is partitioned after confirmation, both limits are
running.  The cap is not paused.  The unreclaimed remainder
stops blocking at the cap; the state the instance had already
reclaimed gets the full absence limit.

Consequence: a partitioned (not restarted) gateway that returns
with the same verifier inside the absence limit resumes with its
state intact.  For a retaining client the absence limit acts as
a longer lease.

Rejected:
- A single retention window advertised as an attribute.  The
  two intervals bound different failures, a gateway that is
  gone and a gateway that is back but not finished, and an
  operator wants to set them apart.
- Unadvertised limits.  The first version of this document chose
  that, on the argument that the reclaimable hint and each
  reclaim's result are enough.  They are enough for safety.
  They leave the gateway's grace period, and the nesting in a
  chain of gateways, to operator configuration.
- Carrying the limits in the EXCHANGE_ID reply.  The result has
  no spare field, and RFC 8178 does not allow an extension to
  change the XDR of an existing result.
- Advertising time remaining instead of the configured value.
  It is stale on arrival and the gateway can compute it.
- An optional cap on the reclaim interval.  Availability for
  direct clients would then depend on a policy the backend need
  not have.
- A limit that destroys state.  Simpler to describe and test,
  but it forbids the courtesy Section 8.4.3 allows, for no gain
  to direct clients.
- Pausing the cap while the lease is expired.  A gateway that
  flaps could stretch the bound to the cap plus several absence
  limits.

### D4. Reuse existing reclaim operations

**Proposal.**  OPEN with CLAIM_PREVIOUS and LOCK with reclaim set
are accepted from a client that has retained state, whether or
not the server is in grace.  No new claim type.

The server can tell the cases apart, and the order of the test
is fixed:

1. The server is in its own grace period: the request is an
   ordinary reclaim under RFC 8881.  Retained state is not
   required to persist across a server restart (Section 7, item
   6), so the server need not consult it.
2. Otherwise, the server holds retained state for this client
   owner: the request is matched against it (D5).
3. Otherwise: NFS4ERR_NO_GRACE.

The client sends the same request in every case and need not
know which one applies.

Rejected: a new claim type modelled on CLAIM_DELEGATE_PREV.  It
is the cleaner analogy, but LOCK has only a boolean, so locks
would need a new operation.

### D5. Matching rule for reclaims

**Proposal.**

- An OPEN reclaim succeeds when the retained open for the same
  open-owner string and file includes the requested access and
  deny bits.
- A LOCK reclaim succeeds when the prior instance's retained
  locks for the same lock-owner string and file cover the
  requested range with a compatible type, and the open stateid
  it is presented under comes from reclaiming the open the lock
  was acquired under (same open-owner string, same file).
- No match: NFS4ERR_RECLAIM_BAD.  Nothing retained any longer:
  NFS4ERR_NO_GRACE.
- State revoked for a conflicting request after a limit (D3) is
  no longer retained, so one of those two errors answers a
  reclaim for it.  NFS4ERR_RECLAIM_CONFLICT is not used: to
  return it, the backend would have to remember what it revoked.

A reclaimed lock keeps its original association with a
lock-owner, an open-owner, and a file.  The new instance
reclaims the open first and then the locks under it, which is
the order and the server path of an ordinary grace-period
reclaim.

This makes owner continuity a requirement: the new instance must
present the same owner strings the old one did, so the gateway
derives them deterministically from front-side identity (D9).
That includes the open-owner of an open the gateway created only
to carry an NLM client's locks.

Why match on owners at all: the gateway lost its own record of
who held what.  With owner matching, the backend is that record:
a reclaim succeeds only for an owner that held the state.
Without it, the first reclaim to arrive wins.

Owner matching is a record, not authentication.  The backend
sees opaque strings the gateway built from front-side values,
and it cannot check which gateway client a reclaim is for.  A
reclaim under the right owner string is honored whoever sent it
to the gateway.  Whether a gateway client can present another
client's owner is decided on the front side, by what the gateway
checks before it forwards (D9).  SP4_MACH_CRED (D11) protects
the gateway's identity toward the backend and says nothing about
the gateway's clients.

Rejected: reclaiming a lock under a fresh, non-reclaim open.  It
moves a retained lock to a lock stateid under a different open,
which changes the association Section 9.1.1 describes and has
unspecified effects on stateids, CLOSE, and LOCKU.  It also
needs a non-reclaim OPEN before RECLAIM_COMPLETE for every lock
reclaim (D6).

### D6. Relax the RECLAIM_COMPLETE gate for OPEN

**Problem.**  Per Section 18.51.3 the new instance cannot send a
non-reclaim OPEN until it sends RECLAIM_COMPLETE.  But it cannot
send RECLAIM_COMPLETE until front-side grace ends.  Lock reclaims
do not need a fresh open, since each lock is reclaimed under its
reclaimed open (D5).  New locking requests from gateway clients
do not need one either: the gateway's own grace period holds
them off.  What remains is a gateway that opens files to serve
NFSv3 READ and WRITE.  (A gateway could instead do that I/O with
a special stateid; the protocol should not force the choice.)

**Proposal.**  For a client with retained state, the backend
permits non-reclaim OPEN before RECLAIM_COMPLETE and checks it
against retained state like any other conflict.  Non-reclaim
LOCK before RECLAIM_COMPLETE still fails with NFS4ERR_GRACE.

This differs from Section 18.51.3 and from the matching
statement in Section 8.4.2.1.  RFC 8881 does not already permit
it.  The draft defines it as part of what `RETAIN_STATE` means,
in effect only for a client ID established with that flag, and
not as a change to RFC 8881 for all clients (Section 7, item
11).  The Section 8.4.2.1 exception covers a server in its
own grace period; here the server is not in grace.  The safety
argument is the same one, though: the backend holds the complete
retained state, so it can determine that a grant cannot conflict
with a later reclaim.

Rejected:
- Relaxing the gate for LOCK as well.  Nothing needs it once
  locks are reclaimed under reclaimed opens.
- Gateway stalls all I/O until front-side grace ends.  Simple,
  but turns a gateway restart into a full-lease outage for
  NFSv3 I/O that today is not interrupted.
- No relaxation; the gateway serves NFSv3 I/O with a special
  stateid until RECLAIM_COMPLETE.  Leaves Section 18.51.3
  untouched, but dictates how a gateway's back-side client does
  I/O.  This is the fallback if the working group objects to
  relaxing the gate.
- A separate completion operation for retained state.  Leaves
  RECLAIM_COMPLETE semantics untouched at the cost of a new
  operation.

### D7. Delegations are out for the first version

**Proposal.**  The extension does not change delegation handling.
A gateway MUST NOT grant delegations to its own clients on the
strength of this extension.

Why: a gateway client holding a write delegation has opens and
locks that neither the gateway nor the backend knows about.
After a gateway restart that delegation can be honored only if
the backend guaranteed no conflicting access in the interim,
which requires a retained back-side delegation.  RFC 8881
already has the mechanism for that (CLAIM_DELEGATE_PREV), but it
is optional and rarely implemented.

Path forward, to be noted in the draft: a gateway may grant a
front-side delegation only while it holds a back-side delegation
on the same file from a backend that supports
CLAIM_DELEGATE_PREV.

The prohibition covers front-side grants only.  A gateway may
hold back-side delegations for its own use.  RFC 9754 Section
5.1 describes that arrangement: an NFSv3 server that is an
NFSv4.2 client and holds delegations so it can answer attribute
queries without a GETATTR to the backend.  This extension does
not retain those delegations (D2), so a gateway restart loses
them.  The loss costs more than performance in two cases.

**Locks and opens under a back-side delegation.**  "When a
client holds an OPEN_DELEGATE_WRITE delegation, lock operations
are performed locally" (RFC 8881 Section 10.4.2).  A gateway
that does this with its clients' locks creates no derived state
for them.  The delegation was their only protection at the
backend, and it is not retained.  After a restart there is
nothing to reclaim, and direct clients were free to take
conflicting locks once the delegation was gone.  So a gateway
that uses the extension MUST create derived opens and locks at
the backend even while it holds a delegation on the file.  This
is another case of D12.  **Unverified:** that a back-side client
implementation can be made to send LOCK while it holds a write
delegation.

**Delegated timestamps** (RFC 9754 Section 5).  The gateway is
the authority for the access and modify times until it returns
the delegation.  A restart loses the values only the gateway
held.

- Access times for reads the gateway served from its cache.
- Modify times the gateway reported to its clients for data it
  had already written to the backend.  The backend has its own
  modify time for those writes, which differs by the flush delay
  and by the difference between the two clocks.  After the
  restart, gateway clients see the modify time change, possibly
  backward, with no other writer.  An NFSv3 client treats that
  as a changed file and drops its cached data.  No data is lost.
  (Inferred: RFC 9754 does not say the backend stops keeping its
  own times while the delegation is out.)
- A time that a gateway client set explicitly with SETATTR, if
  the gateway absorbed it under the delegation.  That time is
  lost although the gateway acknowledged it.  An NFSv3 server
  "must be able to recover without data loss", with unstable
  WRITE the only exception (RFC 1813 Section 4.8).

Rule for the third case: a gateway MUST send an explicit time
change to the backend before it replies.  The second case can be
avoided by sending a SETATTR of the delegated modify time in the
compound that writes the data.  **Unverified:** that RFC 9754
allows that SETATTR at any time while the delegation is held.
Its text constrains only the order relative to DELEGRETURN.

Why this is a gateway problem: when an ordinary client crashes,
the applications that saw its delegated times are gone too.  A
gateway's clients survive and remember what they were told.

### D8. What the gateway tells its clients

**Proposal.**

The decision is made once, from the EXCHANGE_ID result, and
then each reclaim is decided by the backend.

- `RECLAIMABLE_R` set: run a front-side grace period, notify
  NLM clients with SM_NOTIFY, and forward each reclaim.  The
  backend's answer to a forwarded reclaim is authoritative: the
  gateway grants the front-side reclaim only if the back-side
  one succeeded.  The gateway does not need to know whether the
  backend is answering from retained state or from its own grace
  period (D4); the requests are the same.  It withholds
  RECLAIM_COMPLETE until front-side grace ends.
- `RECLAIMABLE_R` clear: still notify, send RECLAIM_COMPLETE
  at once, and refuse every reclaim.  This covers a return after
  retained state was released, and a backend that restarted and
  either has left its grace period or has no record of this
  client owner.
- Backend lacks the extension (NFS4ERR_INVAL, retried without
  the flag), or declines retention (`RETAIN_STATE` not echoed,
  D11): as for `RECLAIMABLE_R` clear.

A refused reclaim is the only way an NLM client learns a lock is
gone, which is why the gateway notifies in every case.

### D9. Owner derivation

**Proposal.**  The draft requires determinism and gives a
recommended construction, not a mandatory one, since the backend
treats owner strings as opaque.

- NLM: lock-owner derived from the caller name and the NLM owner
  handle.  NLM has no opens, so the gateway also derives a
  synthetic open-owner from the same two values and holds one
  backend open per synthetic open-owner and file.  All of that
  NLM owner's locks on the file are taken under that open, and
  the gateway closes it when the last one is released.
- NFSv4: owner derived from the front-side client owner string
  and the front-side open-owner or lock-owner.

The synthetic open has to be reclaimable from an NLM reclaim
request alone, because the gateway has no other record after a
restart.  The request supplies the file handle, caller name,
and owner handle; the gateway must choose the open's share
access by a rule that needs nothing else (Section 7, item 2).

**Binding the owner to a front-side identity.**  Before it
forwards a reclaim, the gateway checks the reclaiming client
against whatever identity the front-side protocol authenticates,
and derives the owner from values tied to that identity.

- NFSv4: a reclaim arrives under a front-side client ID.  The
  gateway accepts it only from a client owner in its
  stable-storage list of clients permitted to reclaim, and
  under the principal RFC 8881 requires for that client owner.
  The derived owner includes the front-side client owner string,
  so it is bound to that check without carrying the principal
  itself.
- NLM: the caller name and owner handle are supplied in the
  request and nothing authenticates them.  The gateway accepts a
  reclaim only for a caller in its NSM monitor list.  It can
  also record the RPC credential or source address with that
  entry and require a match; with RPCSEC_GSS that is a real
  binding, with AUTH_SYS it is not.

Limit, to be stated in the draft: on an unauthenticated NLM
front side, a host that can reach the gateway and present
another client's caller name and owner handle during the grace
period can reclaim that client's lock.  The same is true of an
NLM server that is not a gateway, so the gateway adds no new
exposure, but owner matching does not remove it either.

Thorn: both front-side components can be up to 1024 bytes, and
so is the back-side limit.  The construction needs a hash, and
the draft must say what a collision costs (two front-side owners
sharing back-side state).

### D10. Repeated gateway restarts

**Proposal.**  Retained state accumulates.  If instance 2 crashes
after reclaiming some state but before RECLAIM_COMPLETE, the
backend retains both instance 2's state and the unreclaimed
remainder from instance 1.  RECLAIM_COMPLETE from any later
instance releases everything not yet reclaimed.

Each piece of retained state keeps the reclaim-cap clock (D3)
started by the first confirmation after it was retained.  A
later confirmation does not restart that clock.

- The remainder from instance 1 was retained when instance 2 was
  confirmed.  It stops blocking one reclaim cap after that
  confirmation, however many times the gateway restarts
  afterward.
- State instance 2 held, whether it reclaimed that state or
  acquired it new, is retained when instance 3 is confirmed.
  Its clock starts then.

Why: if every confirmation restarted the clock, a gateway that
crashes in a loop would block direct clients without bound with
state it never reclaims.  State the gateway does reclaim after
each restart is held legitimately each time (D3), so a fresh
clock for it costs direct clients nothing they were owed.

### D11. Who may ask for retention

**Proposal.**  Echoing `RETAIN_STATE` is a backend policy
decision keyed on the principal (and optionally the transport
peer).  A backend that declines clears the flag in `eir_flags`
and behaves per RFC 8881.  The draft recommends SP4_MACH_CRED so
that only the gateway's machine credential can establish the
next instance.

Why: any client that sets the flag can hold locks past lease
expiry for the absence limit.  Unrestricted, that is a
denial-of-service tool.

### D12. Only derived state is recoverable

**Finding.**  Retention preserves what the backend holds and
nothing else.  A gateway is free today to enforce some front-side
state in its own tables without creating derived state for it.
Share deny modes are the known case: a gateway can grant a deny
mode to its client and open the backend file with no deny mode.
Such state excludes other clients of the same gateway only, in
normal operation as well as after a restart.  Locks and opens
that a gateway services locally under a back-side delegation are
a second case (D7).  They are protected in normal operation, by
the delegation, and unprotected once a restart loses it.

**Proposal.**

- The extension guarantees continuity for derived state only.
  For state the gateway enforces locally, a restart leaves the
  guarantee exactly as weak as it was before: the gateway
  re-grants it during its own grace period and the backend is
  not involved.
- The invariant behind a successful front-side reclaim: the
  backend's conflict protection for that state remained in force
  from the original grant until the new instance recovered it.
  State with no derived counterpart cannot meet it.
- A gateway handles share deny modes in one of two modes.  The
  choice is the implementer's, and the draft defines both.
  - **Pass-through.**  The derived open carries at least the deny
    bits of the front-side open or NLM_SHARE it stands for.  The
    backend enforces them against direct clients, D2 retains
    them, and D5 reclaims them.  The invariant holds.
  - **Local-only.**  The derived open carries no deny bits.  The
    gateway enforces the deny mode in its own tables, and it
    binds clients of this gateway only, before and after a
    restart.  A front-side reclaim is re-granted from the
    gateway's own grace-period arbitration.  That is a weaker
    service, and the draft does not count it as recovered share
    exclusion.
- A gateway that wants neither can refuse a request that carries
  a deny mode.
- Only pass-through protects a gateway client against direct
  clients.  The draft says so plainly, because a gateway client
  has no way to learn which mode it was given: neither NFSv4 nor
  NLM_SHARE can signal it.

Two rules follow from making this a choice.

- **The mode is stable across a restart.**  A gateway that was
  local-only before the restart and pass-through after it sends
  a reclaim OPEN with deny bits the retained open lacks, and D5
  answers NFS4ERR_RECLAIM_BAD.  The mode is therefore persistent
  configuration.  A gateway whose mode did change retries the
  reclaim without deny bits and treats the result as local-only.
- **Pass-through needs one back-side open-owner per front-side
  owner** (D9).  A deny mode conflicts with opens by other
  open-owners, including other owners of the same client, so
  with that mapping the backend arbitrates deny modes among
  gateway clients as well as against direct clients.  A gateway
  that aggregates many front-side owners under one back-side
  open-owner is local-only for conflicts among its own clients,
  whatever bits it sends, and after a restart it cannot rely on
  D5 to tell their opens apart.

Rejected:
- Requiring pass-through (MUST).  It would make the extension
  unusable by a gateway whose back-side client cannot send deny
  modes, for no gain in lock recovery.
- Recommending pass-through (SHOULD) with local-only as the
  exception.  The gap exists without this extension, and two
  defined modes with stated guarantees say more than a SHOULD
  an implementer may not be able to follow.

## 4. Worked Sequence: Gateway Restart, One NLM Client

L is the backend lease time.  C is an NFSv3 client, G the
gateway, B the backend, D a client that mounts B directly.

**Before the restart**

1. G to B: `EXCHANGE_ID(co_ownerid=G, verifier=v1,
   flags|=RETAIN_STATE)`.  B echoes the flag.  `CREATE_SESSION`.
   `RECLAIM_COMPLETE`.
2. C to G: `NLM_LOCK(fh, caller="c", oh=X, range, exclusive)`.
3. G to B: `PUTFH; OPEN(CLAIM_FH, access=BOTH,
   open-owner=g("c", X))`, then `LOCK(WRITE_LT, range,
   reclaim=false, lock-owner=f("c", X))` under that open.
4. G records C in its NSM monitor list on stable storage.

**Absence**

5. G crashes at t0.
6. At t0+L the lease expires.  B keeps the open and the lock.
7. D to B: conflicting `LOCK`.  B returns `NFS4ERR_DENIED`.

**Return (before the absence limit)**

8. G to B: `EXCHANGE_ID(co_ownerid=G, verifier=v2,
   flags|=RETAIN_STATE)`.  B returns `RETAIN_STATE |
   RECLAIMABLE_R`.
9. G to B: `CREATE_SESSION`.  B destroys the old sessions and
   client ID, and keeps the old instance's opens and locks as
   retained state.
10. G starts its front-side grace period and sends `SM_NOTIFY`
    to C.
11. C to G: `NLM_LOCK(reclaim=true, same owner and range)`.
12. G to B: `PUTFH; OPEN(CLAIM_PREVIOUS, access=BOTH,
    open-owner=g("c", X))`.  B matches the retained open from
    step 3 (D5), moves it to the new client ID, returns a new
    stateid.
13. G to B: `LOCK(WRITE_LT, range, reclaim=true,
    lock-owner=f("c", X))` under the reclaimed open.  B matches
    the retained lock (D5), moves it to the new client ID,
    returns a new stateid.
14. G to C: `NLM4_GRANTED`.
15. Front-side grace ends.  G to B:
    `RECLAIM_COMPLETE(rca_one_fs=false)`.  B releases all
    remaining retained state.

An NFSv3 READ or WRITE from C between steps 9 and 15 needs no
reclaim.  If G opens the file to serve it, that is a non-reclaim
OPEN before RECLAIM_COMPLETE, permitted by D6.

**Return before the lease expires**

The common case: G reboots in less than L.

- Steps 6 and 7 do not occur as written.  The old instance's
  open and lock are ordinary unexpired state, and D's request is
  denied for that reason.
- Step 8 is the same request and the same result.  B sets
  `RECLAIMABLE_R` because it holds state of a prior instance
  that it will retain on confirmation (D1).  Nothing is retained
  yet.
- Step 9 is where retention happens: `CREATE_SESSION` confirms
  the new instance, and B keeps the old instance's opens and
  locks as retained state instead of releasing them.
- Steps 10 through 15 are unchanged.

**Return after the absence limit**

- This is the case where B released the state at the limit, or
  revoked it for a conflicting request afterward.  If B kept it
  and no conflict arrived, the sequence above runs unchanged
  (D3).
- Step 8 returns `RETAIN_STATE` without `RECLAIMABLE_R`.
- G still sends `SM_NOTIFY` (D8).  It sends `RECLAIM_COMPLETE`
  at once and denies C's reclaim.

**Second restart during the reclaim interval**

Add a second NFSv3 client, C2, that took a lock through G before
step 5 and is down throughout, so its lock is never reclaimed.
"Cap" is B's reclaim cap.

- Steps 8 through 14 run as written.  Step 9 happens at time T2.
  C's open and lock are reclaimed.  C2's open and lock remain
  retained, with a cap clock that started at T2.
- G crashes again before step 15, so instance 2 sends no
  `RECLAIM_COMPLETE`.  When its lease expires, B keeps C's open
  and lock as well.
- G returns as instance 3: `EXCHANGE_ID(verifier=v3)`, then
  `CREATE_SESSION` at time T3.  C's open and lock are retained
  with a clock that starts at T3.  The clock for C2's open and
  lock still runs from T2 (D10).
- G notifies again, and C reclaims again as in steps 11 through
  14.
- At T2 plus the cap, C2's open and lock stop blocking.  D's
  conflicting `LOCK` can now be granted, and B revokes them
  when it grants it.
- Front-side grace ends.  G sends `RECLAIM_COMPLETE`, and B
  releases whatever is still retained.

Writing this out forced D3, D5, and D6.  None of the three was
settled by the outline.

## 5. NFSv4 Gateway Clients

The sequence differs from Section 4 in these ways.

- The client learns of the restart from NFS4ERR_BADSESSION and
  NFS4ERR_STALE_CLIENTID, not SM_NOTIFY.
- The gateway needs its ordinary NFSv4 server stable storage:
  the list of clients allowed to reclaim.
- A front-side `OPEN(CLAIM_PREVIOUS)` maps to a back-side
  `OPEN(CLAIM_PREVIOUS)` with the derived open-owner.  Share
  deny modes survive if the gateway passed them through (D12).
  Front-side lock reclaims map as in Section 4.
- The gateway sends the back-side RECLAIM_COMPLETE when every
  recorded front-side client has sent its own, or when the
  front-side grace timer ends.  NFSv4.0 clients have no
  RECLAIM_COMPLETE, so the timer governs.
- Delegation reclaims do not arise (D7).

## 6. Backend Restart

Claim under test: no wire change is needed.  It holds, with two
caveats and one limit: when both restart, continuity is best
effort.

- **Reclaim.**  The gateway holds all derived state in memory and
  reclaims it during the backend's grace period as any NFSv4
  client does.  Gateway clients are not notified.
- **Requests during back-side grace.**  The backend returns
  NFS4ERR_GRACE for new locks and for I/O.  The gateway maps
  that to NLM4_DENIED_GRACE_PERIOD, NFS3ERR_JUKEBOX, or
  NFS4ERR_GRACE.  **Unverified:** that NFSv4 clients tolerate
  NFS4ERR_GRACE from a server they did not see restart.
- **Caveat 1, lost locks.**  If the gateway cannot reclaim a lock
  (it was partitioned through the grace period), an NFSv4 client
  can be told through revoked stateids and SEQ4_STATUS flags.
  An NLM client cannot be told at all.  This is a limitation of
  NLM, to be documented, not fixed.
- **Caveat 2, both restart.**  The backend is in ordinary grace
  and has no retained state.  It sets `RECLAIMABLE_R` at
  EXCHANGE_ID because it is in grace and has the gateway's
  client owner on record (D1), so the gateway follows the same
  path as for a gateway restart (D8) and forwards front-side
  reclaims, which the backend treats as ordinary reclaims (D4).
  If the backend's grace period ended before the gateway
  returned, the flag is clear and the gateway refuses every
  reclaim.  If it ends while the gateway is still forwarding,
  the remaining reclaims fail with NFS4ERR_NO_GRACE and the
  gateway refuses those.  So continuity holds only if the
  backend's grace period outlasts the gateway's, and RFC 8881
  does not promise that: "The server may also terminate the
  grace period before all clients have done a global
  RECLAIM_COMPLETE" (Section 8.4.2.1).

**Proposal for both-restart: safety is required, continuity is
best effort.**

- **Safety (required).**  A gateway grants a front-side reclaim
  only if the back-side reclaim succeeded (D8).  If the backend
  left grace early, the reclaim fails, the gateway refuses, and
  the gateway client is told its state is lost.  Nothing is
  silently re-granted.  This needs no backend cooperation.
- **Continuity (best effort).**  A backend SHOULD record in
  stable storage that a client owner is a retaining client, and
  SHOULD hold its grace period open until that client's
  RECLAIM_COMPLETE.  Section 8.4.2.1 already permits a grace
  period that lasts until every known client has completed.
  The backend bounds the wait by the absence limit when the
  gateway has not returned, and by the reclaim cap once it has
  (D3).
- A retaining client stays eligible to reclaim across a backend
  restart for the whole absence interval, although its lease has
  expired.  This changes when a backend makes the stable-storage
  record Section 8.4.3 describes.  That record marks a client
  whose lease expired, because "another client could have
  acquired a conflicting" lock, and a reclaim from a marked
  client is rejected with NFS4ERR_NO_GRACE.  For a retaining
  client the reason does not hold until retained state is
  released, so the backend marks the client when it releases or
  revokes retained state (at a limit, at the first conflicting
  request after one, or administratively), not at lease expiry.

Why not required: holding grace open delays every client of the
backend, by up to the absence limit when the gateway never
returns.  A backend operator has to be able to decline that
cost.  What the draft cannot leave to policy is the safety
half.

**Backend restart during an absence or reclaim interval.**
Retained state is not required to persist (Section 7, item 6).
Where it does not, the restart converts the case into one of
these.

- During the absence interval: when the gateway returns, this is
  both-restart, as above.
- During the reclaim interval: the gateway has not restarted
  again.  It sees the backend restart, establishes a new client
  ID, and reclaims the derived state it has already recovered as
  any client does.  Front-side reclaims still arriving are
  forwarded as ordinary reclaims.  The gateway sends
  RECLAIM_COMPLETE to the new backend instance when front-side
  grace ends.  The same safety rule decides each one.

Recommendation: keep this material in the draft as normative
gateway and backend behavior with no new wire elements.

## 7. Thorns Still Open

1. **D6 is the most likely objection.**  It changes a MUST in
   Section 18.51.3 for retaining clients, now for OPEN only.
   The fallback is to drop it and have the gateway serve NFSv3
   I/O with a special stateid until RECLAIM_COMPLETE.  Whether
   that is acceptable for a gateway's back-side client needs an
   implementer's answer.
2. **Share access of a synthetic open** (D9).  The gateway must
   pick it from the NLM reclaim request alone.  Deriving it from
   the lock type works for one lock, but an NLM owner can hold
   shared and exclusive locks on one file, and the reclaims
   arrive in no fixed order.  Either the gateway always opens
   with the widest access the lock types can need, which fails
   on a file the caller can only read, or a later reclaim
   upgrades an open already reclaimed, and D5 has to say how an
   upgrade matches a retained open that has already moved.
3. **Owner derivation collisions** (D9).
4. **Blocked direct clients.**  A dead gateway blocks direct
   clients for the whole absence limit, and a returned gateway
   that never completes blocks them for the reclaim cap (D3).
   Beyond those bounds the backend needs an administrative
   release.  What gateway clients then see: for state not yet
   reclaimed, a refused reclaim, which NLM can express.  For
   state already reclaimed, release is ordinary revocation, with
   the NLM limitation of Section 6, caveat 1.
5. **Limit guidance.**  The absence limit: long enough for a
   reboot, short enough to be tolerable.  The reclaim cap: a
   floor in backend lease periods, long enough for a typical
   front-side grace period.  No numbers proposed yet.  Both
   limits are now advertised (D3), so a gateway can see a cap
   that does not fit, but the default still has to suit a
   gateway nobody tuned.
   Two points about the attributes are unsettled.  The attribute
   numbers in the draft are provisional and need to be checked
   against other NFSv4.2 extensions in progress.  And the
   reported value depends on the requesting client ID when
   policy differs by principal, which no existing attribute
   does.  If the working group objects, the fallback is one
   value per server and no per-principal limits.
6. **Persistence of retained state across a backend restart.**
   Not required here.  Section 6 covers the case without it, at
   the price of best-effort continuity when both restart.  Is
   that acceptable, or does a deployment need the grace
   extension to be a MUST?
7. **Uncommitted writes.**  A gateway restart changes the
   front-side write verifier and clients resend.  Outside this
   document, but a reader will ask.
8. **Locally enforced state** (D12).  Two cases are known: deny
   modes, and locks and opens serviced under a back-side
   delegation (D7).  Is there other front-side state that
   gateways enforce without derived state?  And will the working group accept a local-only mode
   that a gateway client cannot detect?
9. **Bookkeeping for state kept past a limit** (D3).  The
   backend has to mark the client in stable storage at the first
   conflicting grant, not at the limit.  Needs an implementer's
   check; if impractical, a limit releases the state outright.
10. **NFS4ERR_GRACE without a visible restart** (Section 6).
    Unverified that NFSv4 gateway clients tolerate it from a
    gateway they did not see restart.
11. **Extension or new minor version.**  RFC 8178 Section 4.2
    lets an extension add bits to a flag word.  Section 5 says
    changes outside the XDR extension framework, including the
    behavioral changes of Section 5.2, "can only be made in a
    new minor version".  This design adds two flag bits and
    attaches to them different behavior for lease expiry, client
    restart, reclaim outside grace, the RECLAIM_COMPLETE gate,
    and a stable-storage record.  The draft's position is that
    all of it is the defined meaning of the new bits.

    RFC 9754 is precedent, read in full for this pass.  It
    extends NFSv4.2 "using the process detailed in [RFC8178]"
    and cites RFC 8178 Section 4.4.2 as what lets it avoid a new
    minor version.  Its two OPEN share_access flag bits change
    more than the XDR.  With WANT_OPEN_XOR_DELEGATION, OPEN may
    return no open stateid.  With WANT_DELEG_TIMESTAMPS, the
    server "MUST query the client via a CB_GETATTR" and MUST
    accept or delay a SETATTR of the delegated times, so a flag
    carried in OPEN changes required behavior in operations
    whose own XDR is untouched.  That is the pattern this design
    follows.

    Where the precedent stops: RFC 9754's flags act on one open
    or delegation, and they take nothing away from other
    clients.  This design's flags act on a whole
    client ID, set aside requirements of RFC 8881 that protect
    other clients (Section 8.4.3), and change what those clients
    see, since they are denied for longer.  The working group may read that as a
    Section 5 change.  D6 is the most exposed part.

## 8. Where This Lands in the Outline

`outline.md` has been revised to match this document.  Section
numbers below are the outline's.

| Here | Outline |
|------|---------|
| Section 1, mechanism as client-restart state retention | Abstract, 1, 5 |
| D1, flags and the reclaimable hint | 6.1 |
| D2, what is retained | 6.2 |
| D3, two intervals, two limits, the limit attributes | 6.3, 7.4, 8, 9 |
| D4, reuse of reclaim operations, order of the test | 6.4, 6.8 |
| D5, matching rule, lock under its reclaimed open | 6.4 |
| D6, gate relaxed for OPEN | 6.5 |
| D7, delegations | 4.3, 7.6 |
| D8, what the gateway tells its clients | 7.4 |
| D9, owner derivation, synthetic open-owner, identity binding | 7.2, 7.5, 10 |
| D10, repeated restarts | 5, 6.6 |
| D11, who may ask for retention | 6.1, 10 |
| D12, pass-through and local-only deny modes | 7.1, 7.3, 7.5 |
| Section 4, worked sequences | 5 |
| Section 5, NFSv4 gateway clients | 7.6 |
| Section 6, backend restart, both restart | 4.2, 6.8, 7.7, 7.8 |
| Section 7, thorns | 12 |

Not from a decision here: the outline's Section 6 opening lists
every RFC 8881 behavior the extension changes and gives the
compatibility argument under RFC 8178.  Each entry traces to
Section 1, D2, D3, D4, D6, or Section 6 above.

Failover between gateway hosts stays a non-goal (outline 4.3).
