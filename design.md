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

The backend bounds only one interval: how long it holds state for
a gateway that has not come back (the "absence limit").  Once the
new instance has a session, its lease renewals keep the remaining
retained state alive until RECLAIM_COMPLETE.

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
  retaining clients, up to the absence limit.
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
  server may grant them when it "can reliably determine ... that
  granting any such lock cannot possibly conflict with a
  subsequent reclaim" (Section 8.4.2.1).
- **Unknown EXCHANGE_ID flags are rejected.**  "Bits not defined
  above cannot be set in the eia_flags field.  If they are, the
  server MUST reject the operation with NFS4ERR_INVAL" (Section
  18.35.3).  Bits in use: 0x1, 0x2, 0x100, 0x10000 through
  0x40000, 0x40000000, 0x80000000 in RFC 8881, plus 0x4 from
  RFC 8435 (from memory, not rechecked).

## 3. Decisions

### D1. Negotiate with EXCHANGE_ID flags

**Proposal.**  Two new flags.

- `EXCHGID4_FLAG_RETAIN_STATE`: set by the client to request
  retention for this client owner.  Set by the server in
  `eir_flags` when it will retain.
- `EXCHGID4_FLAG_STATE_RETAINED_R`: result only.  Set when the
  server currently holds retained state from a prior instance of
  this client owner.

Why: a server without the extension returns NFS4ERR_INVAL, which
is an unambiguous signal, and the gateway retries without the
flag.  The flag is recorded with the client record, so the
server knows at lease expiry how to treat the state.  No new
operation is needed.

`STATE_RETAINED_R` is a hint.  The authoritative answer is the
result of each reclaim.  The gateway uses the hint to decide what
to tell its clients (D8).

Rejected: a new operation.  It would carry more (for example the
absence limit) but adds an operation number and XDR for
information the gateway does not need in advance (D3).

### D2. What is retained

**Proposal.**

- Retained: opens, with the share access and deny modes the
  backend holds for them, and byte-range locks.  Deny modes a
  gateway did not propagate are not there to retain (D12).
- Not retained: layouts, sessions, the old client ID's stateids.
- Delegations: unchanged from RFC 8881.  See D7.

Retained state behaves as if still held.  Other clients get
NFS4ERR_DENIED or NFS4ERR_SHARE_DENIED, not a grace error.  This
is the explicit override of the Section 8.4.3 MUST.

### D3. Two intervals, one timer

**Proposal.**

- **Absence interval:** from lease expiry of the old instance
  until a new instance is confirmed.  Bounded by the backend's
  absence limit, a local policy value.  When it runs out, the
  backend releases the state as it would today.
- **Reclaim interval:** from confirmation of the new instance
  until its RECLAIM_COMPLETE.  No separate timer.  Remaining
  retained state is kept as long as the new instance's lease is
  renewed.  The backend MAY impose a cap.

Why: this removes the need to advertise a window length.  The
gateway needs to know only whether state survived, which D1
tells it.  The front-side grace period can be whatever the
gateway chooses, because the gateway is renewing its lease
throughout.

Consequence: a partitioned (not restarted) gateway that returns
with the same verifier inside the absence limit resumes with its
state intact.  For a retaining client the absence limit acts as
a longer lease.

Rejected: a single retention window advertised as an attribute.
It forces the gateway to fit its front-side grace into a number
chosen by the backend.

### D4. Reuse existing reclaim operations

**Proposal.**  OPEN with CLAIM_PREVIOUS and LOCK with reclaim set
are accepted from a client that has retained state, whether or
not the server is in grace.  No new claim type.

The server can tell the cases apart: it is either in its own
grace period (ordinary reclaim, nothing retained) or it holds
retained state for this client owner.

Rejected: a new claim type modelled on CLAIM_DELEGATE_PREV.  It
is the cleaner analogy, but LOCK has only a boolean, so locks
would need a new operation.

### D5. Matching rule for reclaims

**Proposal.**

- A LOCK reclaim succeeds when the prior instance's retained
  locks for the same lock-owner string and file cover the
  requested range with a compatible type.  The open stateid it
  is presented under need not come from a reclaimed open.
- An OPEN reclaim succeeds when the retained open for the same
  open-owner string and file includes the requested access and
  deny bits.
- No match: NFS4ERR_RECLAIM_BAD.  Nothing retained any longer:
  NFS4ERR_NO_GRACE.

This makes owner continuity a requirement: the new instance must
present the same owner strings the old one did, so the gateway
derives them deterministically from front-side identity (D9).

Why match on owners at all: the gateway lost its own record of
who held what.  With owner matching, the backend is that record,
and a confused or dishonest gateway client cannot take another
client's lock during reclaim.  Without it, the first reclaim to
arrive wins.

**Unverified:** that a server can move a retained lock to a lock
stateid under a different open.  In NFSv4 a lock stateid is tied
to (lock-owner, open, file).

### D6. Relax the RECLAIM_COMPLETE gate

**Problem.**  Per Section 18.51.3 the new instance cannot send a
non-reclaim OPEN until it sends RECLAIM_COMPLETE.  But it cannot
send RECLAIM_COMPLETE until front-side grace ends, and in the
meantime it may need fresh opens: LOCK requires an open stateid,
so reclaiming a lock for an NLM client requires an open, and a
gateway that opens files to serve NFSv3 READ and WRITE needs
those as well.  (A gateway could instead do that I/O with a
special stateid; the protocol should not force the choice.)

**Proposal.**  For a client with retained state, the backend
permits non-reclaim OPEN and LOCK before RECLAIM_COMPLETE and
checks them against retained state like any other conflict.
This is the Section 8.4.2.1 exception: the backend holds the
complete retained state, so it can determine that a grant cannot
conflict with a later reclaim.

Rejected:
- Gateway stalls all I/O until front-side grace ends.  Simple,
  but turns a gateway restart into a full-lease outage for
  NFSv3 I/O that today is not interrupted.
- A separate completion operation for retained state.  Leaves
  RECLAIM_COMPLETE semantics untouched at the cost of a new
  operation.  This is the fallback if the working group objects
  to relaxing the gate.

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

### D8. What the gateway tells its clients

**Proposal.**

- State retained: run a front-side grace period, notify NLM
  clients with SM_NOTIFY, forward reclaims.
- State not retained (limit exceeded, or backend lacks the
  extension): still notify, and refuse every reclaim.  A refused
  reclaim is the only way an NLM client learns a lock is gone.

### D9. Owner derivation

**Proposal.**  The draft requires determinism and gives a
recommended construction, not a mandatory one, since the backend
treats owner strings as opaque.

- NLM: lock-owner derived from the caller name and the NLM owner
  handle.
- NFSv4: owner derived from the front-side client owner string
  and the front-side open-owner or lock-owner.

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
normal operation as well as after a restart.

**Proposal.**

- The extension guarantees continuity for derived state only.
  For state the gateway enforces locally, a restart leaves the
  guarantee exactly as weak as it was before: the gateway
  re-grants it during its own grace period and the backend is
  not involved.
- A gateway that uses the extension SHOULD propagate deny modes:
  the derived open carries at least the deny bits of the
  front-side open or NLM_SHARE it stands for.  Then D2 retains
  them and D5 reclaims them.
- A gateway that cannot propagate a deny mode has two
  defensible behaviors, and the draft should name both: refuse
  the request, or grant it and document that it binds gateway
  clients only.  The draft does not pick, since the gap exists
  without this extension.

Interaction with D9: a deny mode conflicts with opens by other
open-owners, including other owners of the same client.  With
one back-side open-owner per front-side owner, the backend
arbitrates deny modes among gateway clients as well as against
direct clients.  A gateway that aggregates many front-side
owners under one back-side open-owner must arbitrate among its
own clients itself, and after a restart it cannot rely on D5 to
tell their opens apart.

Rejected: requiring propagation (MUST).  It would make the
extension unusable by a gateway whose back-side client cannot
send deny modes, for no gain in lock recovery.

## 4. Worked Sequence: Gateway Restart, One NLM Client

L is the backend lease time.  C is an NFSv3 client, G the
gateway, B the backend, D a client that mounts B directly.

**Before the restart**

1. G to B: `EXCHANGE_ID(co_ownerid=G, verifier=v1,
   flags|=RETAIN_STATE)`.  B echoes the flag.  `CREATE_SESSION`.
   `RECLAIM_COMPLETE`.
2. C to G: `NLM_LOCK(fh, caller="c", oh=X, range, exclusive)`.
3. G to B: `PUTFH; OPEN(CLAIM_FH, access=BOTH)`, then
   `LOCK(WRITE_LT, range, reclaim=false, lock-owner=f("c", X))`.
4. G records C in its NSM monitor list on stable storage.

**Absence**

5. G crashes at t0.
6. At t0+L the lease expires.  B keeps the open and the lock.
7. D to B: conflicting `LOCK`.  B returns `NFS4ERR_DENIED`.

**Return (before the absence limit)**

8. G to B: `EXCHANGE_ID(co_ownerid=G, verifier=v2,
   flags|=RETAIN_STATE)`.  B returns `RETAIN_STATE |
   STATE_RETAINED_R`.
9. G to B: `CREATE_SESSION`.  B destroys the old sessions and
   client ID, and keeps the old instance's opens and locks as
   retained state.
10. G starts its front-side grace period and sends `SM_NOTIFY`
    to C.
11. C to G: `NLM_LOCK(reclaim=true, same owner and range)`.
12. G to B: `PUTFH; OPEN(CLAIM_FH, access=BOTH)`.  Permitted
    before RECLAIM_COMPLETE by D6.
13. G to B: `LOCK(WRITE_LT, range, reclaim=true,
    lock-owner=f("c", X))`.  B matches the retained lock (D5),
    moves it to the new client ID, returns a new stateid.
14. G to C: `NLM4_GRANTED`.
15. Front-side grace ends.  G to B:
    `RECLAIM_COMPLETE(rca_one_fs=false)`.  B releases all
    remaining retained state, including the old open from step 3.

**Return after the absence limit**

- Step 8 returns `RETAIN_STATE` without `STATE_RETAINED_R`.
- G still sends `SM_NOTIFY` (D8).  It sends `RECLAIM_COMPLETE`
  at once and denies C's reclaim.

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
  deny modes survive if the gateway propagated them (D12).
  Front-side lock reclaims map as in Section 4.
- The gateway sends the back-side RECLAIM_COMPLETE when every
  recorded front-side client has sent its own, or when the
  front-side grace timer ends.  NFSv4.0 clients have no
  RECLAIM_COMPLETE, so the timer governs.
- Delegation reclaims do not arise (D7).

## 6. Backend Restart

Claim under test: no wire change is needed.  It holds, with two
caveats.

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
  and has no retained state.  The gateway forwards front-side
  reclaims as ordinary reclaims.  This works only if the
  backend's grace period outlasts the gateway's.  A backend that
  records in its client-tracking database that a client owner is
  a retaining client can hold its grace period open until that
  client's RECLAIM_COMPLETE, which Section 8.4.2.1 already
  permits.  The cost is a longer grace period for every client
  of the backend, so this should be policy, bounded by the
  absence limit.

Recommendation: keep this material in the draft as normative
gateway behavior with no protocol elements.

## 7. Thorns Still Open

1. **D6 is the most likely objection.**  It changes a MUST in
   Section 18.51.3 for retaining clients.  The fallback is a
   separate completion operation.
2. **D5 lock transfer across opens** needs an implementer's
   check before the draft relies on it.
3. **Owner derivation collisions** (D9).
4. **Blocked direct clients.**  A dead gateway blocks direct
   clients for the whole absence limit.  The backend needs an
   administrative release, and the draft should say what clients
   holding the lost state then see.
5. **Absence limit guidance.**  Long enough for a reboot, short
   enough to be tolerable.  No number proposed yet.
6. **Persistence of retained state across a backend restart.**
   Not required here.  Caveat 2 covers the case without it.
7. **Uncommitted writes.**  A gateway restart changes the
   front-side write verifier and clients resend.  Outside this
   document, but a reader will ask.
8. **Locally enforced state** (D12).  Is SHOULD the right
   strength for propagating deny modes, and is there other
   front-side state that gateways enforce without derived state?

## 8. Changes This Implies for the Outline

- Reframe sections 1 and 6 around client-restart state retention,
  with the gateway as the use case.
- Section 6.1: two flags; drop "advertise the retention window".
- Section 6.2: split into absence interval and reclaim interval.
- Section 6.3: add the matching rule and the relaxed
  RECLAIM_COMPLETE gate.
- Section 6.6 and 7.5: fold in Section 6 above, including the
  backend grace-extension policy.
- Section 7.4: delegations become a prohibition plus a note on
  the path forward.
- Sections 7.3 and 7.4: add deny-mode propagation (D12) for
  NLM_SHARE and NFSv4 opens.
- Section 12: the six open questions are answered by D1, D3, D5,
  D7, and Section 6.  Failover between gateway hosts stays a
  non-goal.  Replace the list with Section 7 above.
