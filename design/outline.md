# Outline: State Retention Across Client Restart for NFSv4.2

Working outline for a standalone NFSv4.2 extension (RFC 8178).
Intended status: Standards Track.
Draft name: draft-cel-nfsv4-state-retention.

Decisions cited as D1 through D12 are in `design.md`.  The
problem itself is described in `problem-statement.md`; this
document summarizes it and specifies the mechanism.

## Abstract

- An NFSv4 server discards a client's open and lock state when
  the client restarts.
- Some clients hold that state on behalf of parties that survive
  the restart.  An NFS gateway, which re-exports a file system it
  accesses as a client of a backend server, is the motivating
  case.
- This document extends NFSv4.2 so that a server retains the
  state of a restarting client and permits the new client
  instance to reclaim it outside the server's grace period.

## 1. Introduction

- Client restart in RFC 8881: the server releases opens and locks.
- Why that fails for a gateway: its clients are still running
  and expect to reclaim.
- The mechanism is client-restart state retention.  It is usable
  by any NFSv4.2 client that can reconstruct its state; the
  gateway is the use case developed here.
- Precedent: retention of delegations for a restarted client
  (CLAIM_DELEGATE_PREV, RFC 8881 Section 10.2.1).
- Summary: negotiate, retain, reclaim, complete.
- Relationship to the problem statement document and to
  draft-haynes-nfsv4-flexfiles-v2-proxy-server.

## 2. Requirements Language

## 3. Terminology

- Gateway server, backend server, gateway client, direct client.
- Front side, back side.
- Front-side state, derived state.
- Client instance (one incarnation, identified by the verifier).
- Retaining client.
- Retained state.
- Absence interval, absence limit, reclaim interval, reclaim cap.
- Synthetic open-owner.
- Pass-through and local-only deny modes.

## 4. Problem Summary

Short; refers to the problem statement for the full analysis.

### 4.1. Deployment Model

- Figure: gateway clients, gateway, backend, direct clients.
- Front side: NFSv3 with NLM/NSM, or any NFSv4 minor version.
- Back side: NFSv4.2.

### 4.2. Failure Cases Addressed

- Gateway restart: the case the extension exists for.
- Both restart, backend restart, lease loss: gateway and backend
  behavior, no new protocol elements.
- Both restart: no silent loss is required; continuity is best
  effort.

### 4.3. Non-Goals

- Delegations granted by a gateway (D7).
- Layouts.
- A back side other than NFSv4.2.
- Failover of derived state between distinct gateway hosts.
- Gateways behind gateways.

## 5. Protocol Overview

- The four steps.
- The two intervals and their limits (D3).
- Message sequence: gateway restart with an NLM client.
- Message sequence: gateway restart with an NFSv4.1 client.
- Message sequence: gateway returns before its lease expires.
- Message sequence: gateway returns after the absence limit.
- Message sequence: gateway restarts twice (D10).

## 6. Protocol Extension

- Kind of extension: two previously unassigned bits in the
  EXCHANGE_ID flag word and two new attributes, XDR extensions
  that RFC 8178 Section 4.2 allows within a minor version.
- The flags are not the whole extension.  Negotiating them
  changes how later operations behave for that client ID.  The
  draft has to present every such behavior as the defined
  meaning of the new flag bits, because RFC 8178 Section 5 says
  changes outside the XDR extension framework, behavioral
  changes among them (Section 5.2), "can only be made in a new
  minor version".  Precedent, published as an NFSv4.2 extension
  under RFC 8178: the OPEN share_access flags of RFC 9754, which
  change required behavior in OPEN, CB_GETATTR, and SETATTR.
  Limit of the precedent: those flags act on one open or
  delegation and take nothing away from other clients.
- Behavior the flags select, in each case only for a client ID
  established with the echoed flag:
  - EXCHANGE_ID and CREATE_SESSION, client restart case: state
    of the prior instance is retained, not released.
  - Expired state does not yield to conflicting requests until a
    limit is reached (RFC 8881 Section 8.4.3).
  - The lease-expired stable-storage record is made when
    retained state is released (RFC 8881 Section 8.4.3).
  - OPEN and LOCK reclaims are accepted outside the server's
    grace period.
  - Non-reclaim OPEN is accepted before RECLAIM_COMPLETE (RFC
    8881 Sections 18.51.3 and 8.4.2.1).
  - RECLAIM_COMPLETE releases unreclaimed retained state.
- Compatibility argument:
  - A client that does not set the flag, and a server that does
    not echo it, see RFC 8881 behavior unchanged.
  - A server sets either flag in a result only when the request
    set RETAIN_STATE, so it knows the client is aware of the
    extension before it sends an extended response (RFC 8178
    Section 6).
  - A server without the extension answers NFS4ERR_INVAL, which
    is how a requester learns a flag bit is unknown (RFC 8178
    Sections 4.4.3 and 8.2).
  - Clients other than the retaining client see only errors they
    can already receive (NFS4ERR_DENIED, NFS4ERR_SHARE_DENIED),
    for longer.
- No "Updates" header for RFC 8881 is needed (RFC 8178 Section
  6).

### 6.1. Capability Negotiation (D1, D11)

- EXCHGID4_FLAG_RETAIN_STATE: request and grant.
- EXCHGID4_FLAG_RECLAIMABLE_R: result only; a hint that
  reclaims from this client owner can succeed after
  confirmation.  Set for retained state, for unexpired state of
  a prior instance that will be retained at CREATE_SESSION, and
  for a server in grace that has the client owner on record.
- The hint precedes confirmation; the result of each reclaim is
  authoritative.
- Both result flags are set only in reply to a request that set
  EXCHGID4_FLAG_RETAIN_STATE.
- Server policy: which principals may be retaining clients.  A
  server that declines clears the flag in the result and behaves
  per RFC 8881.
- A server without the extension returns NFS4ERR_INVAL; the
  client retries without the flag, which also separates an
  unknown flag from the other causes of that error (RFC 8178
  Section 4.4.3).
- Why no attribute advertises support (RFC 8178 Section 6, RFC
  9754): retention belongs to a client ID and is learned at
  EXCHANGE_ID, and an attribute is read per file system.  The
  limits are attributes all the same (Section 6.3): a client
  wants them only once it has a session.

### 6.2. Retained State (D2)

- Retained: opens with the access and deny modes the server
  holds, and byte-range locks.
- Not retained: layouts, sessions, the old instance's stateids.
- Delegations: unchanged from RFC 8881.
- Retained state continues to conflict.  Other clients see
  NFS4ERR_DENIED and NFS4ERR_SHARE_DENIED.
- Override of the RFC 8881 Section 8.4.3 requirement that expired
  state yield to conflicting requests, until a limit is reached
  (Section 6.3).

### 6.3. Absence Interval and Reclaim Interval (D3)

- Absence interval: from lease expiry until a new instance is
  confirmed.  Bounded by the absence limit, a server policy
  value.
- Reclaim interval: from confirmation until RECLAIM_COMPLETE.
  Bounded by the reclaim cap, a server policy value the server
  MUST have, measured from the confirmation that began the
  interval (Section 6.6 for repeated restarts).  Floor in lease
  periods.
- At a limit, retained state stops blocking conflicting
  requests.  The server may release it or keep it until the
  first conflict (RFC 8881 Section 8.4.3).  A reclaim of state
  still present succeeds.
- The cap governs unreclaimed state only, and is not paused by
  loss of lease during the reclaim interval.
- Both limits are advertised, as the per-server attributes
  retain_absence_limit and retain_reclaim_cap, read like
  lease_time.  Mandatory for a server with the extension.  The
  value is the configured limit for the requesting client ID,
  and the server does not reach a limit sooner than it reported.
- A retaining client that returns with the same verifier inside
  the absence limit resumes with its state intact.
- Change to EXCHANGE_ID and CREATE_SESSION processing for the
  client restart case (RFC 8881 Section 18.35.4, case 5).

### 6.4. Reclaim Outside the Grace Period (D4, D5)

- OPEN with CLAIM_PREVIOUS and LOCK with reclaim set are accepted
  from a client that has retained state.
- Matching rule for OPEN: open-owner, file, access and deny bits.
- Matching rule for LOCK: lock-owner, file, range, type.  It is
  presented under the reclaimed open it was acquired under.
- A successful reclaim moves the state to the new client ID and
  keeps the lock's association with its open.
- Owner continuity as a consequence of the matching rule.

### 6.5. Operations Before RECLAIM_COMPLETE (D6)

- Non-reclaim OPEN from the new instance is permitted before
  RECLAIM_COMPLETE and is checked against retained state.
- Non-reclaim LOCK before RECLAIM_COMPLETE: NFS4ERR_GRACE, as in
  RFC 8881.
- Stated explicitly as a meaning of the negotiated flag that
  differs from RFC 8881 Sections 18.51.3 and 8.4.2.1, in effect
  only for a retaining client.  Not an instance of the Section
  8.4.2.1 exception, though the safety argument is the same.

### 6.6. Completing Reclaim

- RECLAIM_COMPLETE releases all retained state not yet reclaimed.
- Repeated restarts: retained state accumulates across instances
  until a RECLAIM_COMPLETE (D10).
- Each piece of retained state keeps the reclaim-cap clock
  started by the first confirmation after it was retained; a
  later confirmation does not restart it (D10).

### 6.7. Errors

- NFS4ERR_RECLAIM_BAD: no retained state matches.
- NFS4ERR_NO_GRACE: nothing is retained any longer.
- State revoked after a limit gets one of those two;
  NFS4ERR_RECLAIM_CONFLICT is not used (D5).
- No new error codes expected.

### 6.8. Interaction with the Server's Grace Period

- Order of the test: server in grace, ordinary reclaim;
  otherwise retained state for this client owner, matched
  reclaim; otherwise NFS4ERR_NO_GRACE.
- Server restart during an absence or reclaim interval: retained
  state need not persist; ordinary grace covers the case.
- A server SHOULD record retaining clients in stable storage and
  SHOULD hold its grace period open for them, bounded by the
  absence limit or the reclaim cap.  Cost to other clients,
  and why this is not a MUST.
- A retaining client stays eligible to reclaim across a server
  restart for the whole absence interval.  The lease-expired
  record of RFC 8881 Section 8.4.3 is made when retained state
  is released, not at lease expiry.
- Server restart during the reclaim interval: the client
  reclaims what it has recovered; remaining reclaims become
  ordinary reclaims.

## 7. Gateway Server Behavior

### 7.1. What Is Recoverable (D12)

- Only derived state is recoverable.  The invariant: backend
  conflict protection held from the original grant to recovery.
- Two deny modes, the implementer's choice.  Pass-through: the
  derived open carries the deny bits and the invariant holds.
  Local-only: binds clients of this gateway only; a weaker
  service, not recovered share exclusion.
- A gateway may instead refuse requests that carry a deny mode.
- Locks and opens serviced locally under a back-side delegation
  are not derived state either (Section 7.6).
- A gateway client cannot detect the mode.
- The mode is stable across a restart.
- Pass-through needs one back-side open-owner per front-side
  owner (D9).

### 7.2. Owner Derivation (D9)

- Requirement: deterministic across restarts.
- Recommended construction for NLM owners and for NFSv4 owners.
- NLM: a synthetic open-owner per NLM lock owner, one backend
  open per synthetic open-owner and file, and the rule for its
  share access.
- Binding the derived owner to the identity the front-side
  protocol authenticates; what the gateway checks before it
  forwards a reclaim.
- Length limits, hashing, and the cost of a collision.
- One back-side owner per front-side owner versus aggregation.

### 7.3. Stable Storage

- The back-side client owner string.
- The record of front-side clients permitted to reclaim: the
  NFSv4 client list and the NSM monitor list, optionally with
  the RPC credential or source address of each NLM caller (D9).
- The deny mode in use, pass-through or local-only (D12).

### 7.4. Restart Sequence (D8)

- Order: EXCHANGE_ID and CREATE_SESSION to the backend, read the
  limit attributes, begin front-side grace, forward reclaims,
  end front-side grace, RECLAIM_COMPLETE to the backend.
- The front-side grace period ends inside the reclaim cap where
  the front side allows it.  A cap too short to fit is reported
  to the operator (D3).
- Hint set: notify clients and forward each reclaim; grant the
  front-side reclaim only if the back-side one succeeded.
- Hint clear, or no extension: notify clients, RECLAIM_COMPLETE
  at once, refuse every reclaim.
- The gateway need not know whether the backend answers from
  retained state or from its own grace period.

### 7.5. NFSv3 Gateway Clients

- SM_NOTIFY and NLM reclaim; mapping to a back-side OPEN reclaim
  of the synthetic open, then a LOCK reclaim under it.
- NLM_SHARE and deny modes.
- I/O during front-side grace: opens under D6, or a special
  stateid.

### 7.6. NFSv4 Gateway Clients

- Mapping front-side CLAIM_PREVIOUS and lock reclaims.
- When to send the back-side RECLAIM_COMPLETE.
- NFSv4.0 clients (no RECLAIM_COMPLETE).
- Delegations: a gateway MUST NOT grant them on the strength of
  this extension.  Path forward through CLAIM_DELEGATE_PREV (D7).
- Back-side delegations the gateway holds for its own use (RFC
  9754 Section 5.1) are allowed and are not retained.
- Under a back-side delegation the gateway MUST still create
  derived opens and locks; RFC 8881 Section 10.4.2 would
  otherwise have it lock locally.
- Delegated timestamps: what a restart loses, and the rule that
  an explicit time change reaches the backend before the reply.

### 7.7. Backend Restart

- The gateway reclaims derived state itself.
- Mapping the backend's NFS4ERR_GRACE to each front-side protocol.
- Both restart: forwarding front-side reclaims as ordinary
  reclaims; the grace-period timing dependency.
- Safety rule (MUST): grant a front-side reclaim only if the
  back-side reclaim succeeded.  Continuity is best effort.
- Backend restart during the gateway's reclaim interval.

### 7.8. Reporting Lost State

- Causes: absence limit or reclaim cap reached and the state
  then released or revoked, administrative release, reclaim
  refused, lease lost during a partition.
- NFSv4 clients: revoked stateids and SEQ4_STATUS flags.
- NLM clients: no mechanism other than a refused reclaim.

### 7.9. Chained Gateways

- Not normative.  A gateway whose backend is a gateway: the
  inner gateway is a retaining client of the backend and a
  server with the extension to the outer gateway.
- The safety rule holds at each gateway.  Each restart case.
- Nesting of grace periods inside the backend's reclaim cap,
  computed from the limit attributes (D3).
- Hint propagation, deny modes, delegations.

## 8. Backend Server Behavior

- Absence limit and reclaim cap: guidance on their values.
- Resource limits on retained state.
- Administrative release of retained state, and what gateway
  clients see: a refused reclaim for unreclaimed state, ordinary
  revocation for reclaimed state.
- Stable-storage bookkeeping for state kept past a limit.
- Several retaining clients on one file system.
- Effect on direct clients.

## 9. XDR Description

- Extraction instructions; the two flag constants; the two
  attribute numbers and their types.

## 10. Security Considerations

- Entitlement to retention is a server policy decision (D11).
- SP4_MACH_CRED for the retaining client's client ID.
- Denial of service: a retaining client holds state past lease
  expiry for the absence limit, and unreclaimed state for the
  reclaim cap.  Both bounds are mandatory.
- Reclaim of state the prior instance did not hold.
- The server cannot check which gateway client a reclaim is for.
  Owner matching is a record of who held what, not
  authentication.
- Front-side identity: NFSv4 client ID and principal; NLM caller
  name and owner handle are client-supplied.  Limits for
  unauthenticated NLM deployments, no worse than a non-gateway
  NLM server.
- SP4_MACH_CRED protects the gateway's identity toward the
  server, not the identity of the gateway's clients.
- Owner collisions between gateway clients.
- Transport security between gateway and backend.

## 11. IANA Considerations

- Expected: none.  Confirm for the new flags.

## 12. Open Issues

From `design.md`, Section 7.

- Relaxing the RECLAIM_COMPLETE gate for OPEN (D6), with
  special-stateid I/O as the fallback.
- Share access of a synthetic open for NLM locks (D9).
- Owner derivation collisions (D9).
- Direct clients blocked by an absent or non-completing gateway.
- Absence limit and reclaim cap guidance.
- Bookkeeping for state kept past a limit (D3).
- Persistence of retained state across a backend restart.
- Uncommitted writes across a gateway restart.
- Other locally enforced state, and whether a local-only deny
  mode that clients cannot detect is acceptable (D12).
- Whether NFSv4 gateway clients tolerate NFS4ERR_GRACE from a
  gateway they did not see restart.
- Whether the behavior the flags select can be an extension
  under RFC 8178, or needs a new minor version (Section 6).

## References

- Normative: RFC 8881, RFC 7862, RFC 7863, RFC 8178.
- Informative: RFC 1813, NLM/NSM (Open Group XNFS), RFC 7530,
  the problem statement document,
  draft-haynes-nfsv4-flexfiles-v2-proxy-server, RFC 9754.
