# Outline: State Retention Across Client Restart for NFSv4.2

Working outline for a standalone NFSv4.2 extension (RFC 8178).
Intended status: Standards Track.  Draft name: TBD.

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
- Absence interval, absence limit, reclaim interval.

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

### 4.3. Non-Goals

- Delegations granted by a gateway (D7).
- Layouts.
- A back side other than NFSv4.2.
- Failover of derived state between distinct gateway hosts.
- Gateways behind gateways.

## 5. Protocol Overview

- The four steps.
- The two intervals and the single timer (D3).
- Message sequence: gateway restart with an NLM client.
- Message sequence: gateway restart with an NFSv4.1 client.
- Message sequence: gateway returns after the absence limit.
- Message sequence: gateway restarts twice (D10).

## 6. Protocol Extension

### 6.1. Capability Negotiation (D1, D11)

- EXCHGID4_FLAG_RETAIN_STATE: request and grant.
- EXCHGID4_FLAG_STATE_RETAINED_R: result only; a hint that
  retained state exists.
- Server policy: which principals may be retaining clients.
- A server without the extension returns NFS4ERR_INVAL; the
  client retries without the flag.

### 6.2. Retained State (D2)

- Retained: opens with the access and deny modes the server
  holds, and byte-range locks.
- Not retained: layouts, sessions, the old instance's stateids.
- Delegations: unchanged from RFC 8881.
- Retained state continues to conflict.  Other clients see
  NFS4ERR_DENIED and NFS4ERR_SHARE_DENIED.
- Override of the RFC 8881 Section 8.4.3 requirement that expired
  state yield to conflicting requests.

### 6.3. Absence Interval and Reclaim Interval (D3)

- Absence interval: from lease expiry until a new instance is
  confirmed.  Bounded by the absence limit, a server policy
  value.
- Reclaim interval: from confirmation until RECLAIM_COMPLETE.
  Governed by the new instance's lease.  Optional server cap.
- A retaining client that returns with the same verifier inside
  the absence limit resumes with its state intact.
- Change to EXCHANGE_ID and CREATE_SESSION processing for the
  client restart case (RFC 8881 Section 18.35.4, case 5).

### 6.4. Reclaim Outside the Grace Period (D4, D5)

- OPEN with CLAIM_PREVIOUS and LOCK with reclaim set are accepted
  from a client that has retained state.
- Matching rule for LOCK: lock-owner, file, range, type.  The
  open it is presented under need not be a reclaimed open.
- Matching rule for OPEN: open-owner, file, access and deny bits.
- A successful reclaim moves the state to the new client ID.
- Owner continuity as a consequence of the matching rule.

### 6.5. Operations Before RECLAIM_COMPLETE (D6)

- Non-reclaim OPEN and LOCK from the new instance are permitted
  before RECLAIM_COMPLETE and are checked against retained state.
- Basis: the exception in RFC 8881 Section 8.4.2.1.
- Change to RFC 8881 Section 18.51.3 for retaining clients.

### 6.6. Completing Reclaim

- RECLAIM_COMPLETE releases all retained state not yet reclaimed.
- Repeated restarts: retained state accumulates across instances
  until a RECLAIM_COMPLETE (D10).

### 6.7. Errors

- NFS4ERR_RECLAIM_BAD: no retained state matches.
- NFS4ERR_NO_GRACE: nothing is retained any longer.
- NFS4ERR_RECLAIM_CONFLICT.
- No new error codes expected.

### 6.8. Interaction with the Server's Grace Period

- A request is an ordinary reclaim when the server is in grace,
  and a retained-state reclaim otherwise.
- Server restart during an absence or reclaim interval: retained
  state need not persist; ordinary grace covers the case.
- A server may hold its grace period open for a known retaining
  client, bounded by the absence limit.  Cost to other clients.

## 7. Gateway Server Behavior

### 7.1. What Is Recoverable (D12)

- Only derived state is recoverable.
- Propagating deny modes to the backend (SHOULD).
- A gateway that cannot propagate a deny mode: refuse, or grant
  with a documented local-only guarantee.

### 7.2. Owner Derivation (D9)

- Requirement: deterministic across restarts.
- Recommended construction for NLM owners and for NFSv4 owners.
- Length limits, hashing, and the cost of a collision.
- One back-side owner per front-side owner versus aggregation.

### 7.3. Stable Storage

- The back-side client owner string.
- The record of front-side clients permitted to reclaim.

### 7.4. Restart Sequence (D8)

- Order: EXCHANGE_ID and CREATE_SESSION to the backend, begin
  front-side grace, forward reclaims, end front-side grace,
  RECLAIM_COMPLETE to the backend.
- State retained: notify clients and forward reclaims.
- State not retained: notify clients and refuse every reclaim.

### 7.5. NFSv3 Gateway Clients

- SM_NOTIFY and NLM reclaim; mapping to back-side LOCK reclaim.
- NLM_SHARE and deny modes.
- Opens that carry lock reclaims; I/O during front-side grace.

### 7.6. NFSv4 Gateway Clients

- Mapping front-side CLAIM_PREVIOUS and lock reclaims.
- When to send the back-side RECLAIM_COMPLETE.
- NFSv4.0 clients (no RECLAIM_COMPLETE).
- Delegations: a gateway MUST NOT grant them on the strength of
  this extension.  Path forward through CLAIM_DELEGATE_PREV (D7).

### 7.7. Backend Restart

- The gateway reclaims derived state itself.
- Mapping the backend's NFS4ERR_GRACE to each front-side protocol.
- Both restart: forwarding front-side reclaims as ordinary
  reclaims; the grace-period timing dependency.

### 7.8. Reporting Lost State

- Causes: absence limit exceeded, reclaim refused, lease lost
  during a partition.
- NFSv4 clients: revoked stateids and SEQ4_STATUS flags.
- NLM clients: no mechanism other than a refused reclaim.

## 8. Backend Server Behavior

- Absence limit: guidance on its value.
- Resource limits on retained state.
- Administrative release of retained state.
- Several retaining clients on one file system.
- Effect on direct clients.

## 9. XDR Description

- Extraction instructions; the two flag constants.

## 10. Security Considerations

- Entitlement to retention is a server policy decision (D11).
- SP4_MACH_CRED for the retaining client's client ID.
- Denial of service: a retaining client holds state past lease
  expiry for the absence limit.
- Reclaim of state the prior instance did not hold.
- The server cannot check which gateway client a reclaim is for.
- Owner collisions between gateway clients.
- Transport security between gateway and backend.

## 11. IANA Considerations

- Expected: none.  Confirm for the new flags.

## 12. Open Issues

From `design.md`, Section 7.

- Relaxing the RECLAIM_COMPLETE gate (D6), with a separate
  completion operation as the fallback.
- Moving a retained lock to a lock stateid under a different open
  (D5).
- Owner derivation collisions (D9).
- Direct clients blocked by an absent gateway.
- Absence limit guidance.
- Persistence of retained state across a backend restart.
- Uncommitted writes across a gateway restart.
- Strength of the deny-mode propagation requirement, and other
  locally enforced state (D12).

## References

- Normative: RFC 8881, RFC 7862, RFC 7863, RFC 8178.
- Informative: RFC 1813, NLM/NSM (Open Group XNFS), RFC 7530,
  the problem statement document,
  draft-haynes-nfsv4-flexfiles-v2-proxy-server.
