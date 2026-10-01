# Problem Statement: State Recovery Through NFS Gateway Servers

Draft text for a standalone Informational document.  Plain
markdown for now; conversion to kramdown-rfc is mechanical once
the docname and authors are chosen.  It describes the problem and
its scope and proposes no mechanism.  `design.md` holds the
mechanism for a subset of it.

Items marked TODO are claims not yet checked.

## Abstract

An NFS gateway exports to its own clients a file system that it
accesses as an NFS client of another server.  NFS state recovery
is defined between one client and one server.  With a gateway in
the path, the server that loses state and the server that offers
a grace period are not always the same server, and open, lock,
delegation, and layout state held through the gateway cannot be
recovered reliably.  This document describes the deployment, the
state involved, and the failure cases, and it sets the scope for
protocol work to address them.

## 1. Introduction

Re-exporting an NFS mount is used to place a caching server near
a group of clients, to bridge protocol versions (NFSv3 clients
reaching an NFSv4-only server), and to put a security or
administrative boundary between clients and a storage system.

Stateless operations pass through a gateway without difficulty.
Stateful operations do not.  A lock granted to a gateway client
exists twice: as state the gateway holds for its client, and as
state the backend server holds for the gateway.  The two copies
have different owners, different leases, and different recovery
events.  NFS recovery repairs one copy at a time and assumes that
the party asked to accept a reclaim is the party that lost the
state.

Without a protocol answer, a gateway implementation is left to
refuse the stateful features, to offer them with weaker
guarantees than a client expects, or to accept that state is
lost on restart (Section 8).  Each of these removes function
that applications depend on.

## 2. Terminology

Gateway server:
:  A host that is an NFS server to one set of clients and an NFS
   client of a backend server, for the same file system.

Backend server:
:  The NFS server the gateway mounts.

Gateway client:
:  A client of the gateway server.

Direct client:
:  A client of the backend server that does not go through the
   gateway.

Front side, back side:
:  The gateway's server-facing-clients role and its
   client-of-backend role, respectively.

Front-side state:
:  State the gateway has granted to a gateway client.

Derived state:
:  State the gateway holds at the backend because of front-side
   state.

## 3. Deployment Model

```
  gateway clients          gateway            backend
  +-----------+        +-------------+     +-----------+
  | NFSv3/NLM |------->| NFS server  |     |           |
  | NFSv4.x   |        |     |       |     |    NFS    |
  +-----------+        | NFS client  |---->|   server  |
                       +-------------+     |           |
  direct clients ------------------------->|           |
                                           +-----------+
```

- The front side may speak NFSv3 with NLM and NSM, or any NFSv4
  minor version.  The back side may likewise be NFSv3 or NFSv4.
  The problem exists in every combination.
- Direct clients, and other gateways, may access the same files.
  The backend is the only party that can arbitrate among them.
- A gateway may serve many clients over one back-side client
  identity.  The backend sees one client.

## 4. The Two-Party Recovery Model

NFSv4 recovery (RFC 8881, Section 8.4) rests on these rules.

- State belongs to a client instance and is kept alive by a
  lease.
- When a client restarts, the server discards the old instance's
  state.  There is nothing to reclaim, because the client has
  lost the applications that held it.
- When a server restarts, it offers a grace period during which
  clients reclaim what they held and no new conflicting state is
  granted.  Outside a grace period, reclaims are refused.
- When a lease expires without a restart, the server may revoke
  the state, and the client learns of the loss and reports it to
  applications.

NLM and NSM follow the same shape with notification
(SM_NOTIFY) in place of leases.

The second rule is the one a gateway breaks.  When a gateway
restarts, its back-side client instance has restarted, but the
applications that hold the state have not.  They are on the
gateway clients, still running, and they expect to reclaim.

## 5. State Held Through a Gateway

| State | Front side | Derived state at the backend |
|-------|-----------|------------------------------|
| Open, share reservation | NFSv4 OPEN; NLM_SHARE | OPEN, with matching access and deny |
| Byte-range lock | NLM lock; NFSv4 LOCK | LOCK |
| Delegation | Delegation granted by the gateway | Delegation held by the gateway, if any |
| Layout | Layout granted by the gateway | Layout held by the gateway, if any |

Two properties matter for recovery.

- **Opens and locks are mirrored.**  Each piece of front-side
  state has corresponding derived state, and the backend
  enforces it against direct clients.
- **Delegations and layouts hide state.**  A gateway client that
  holds a delegation opens and locks the file locally.  Neither
  the gateway nor the backend knows about that state until the
  delegation is returned.  A gateway client that holds a layout
  writes to storage devices without the gateway seeing the I/O.

## 6. Failure Cases

### 6.1. Backend Server Restart

The backend restarts and enters grace.  The gateway does not
restart and still knows all derived state.

- The gateway can reclaim derived state itself.  Gateway clients
  need not be told.
- Gateway clients see no grace period.  Requests for new state
  that the gateway forwards are refused by the backend until its
  grace period ends, and the gateway has to turn that into
  something each client protocol can express.
- If the gateway fails to reclaim some state, the front-side
  state is no longer backed by anything.  The gateway must
  report the loss.  NFSv4 can express that; NLM cannot.
- State hidden under a front-side delegation is safe only if the
  gateway reclaims the corresponding back-side delegation.

This case is mostly handled by the existing protocol.  What
remains is gateway behavior that no document specifies.

### 6.2. Gateway Server Restart

The gateway restarts and loses both front-side and derived state.
The backend does not restart.

- The backend sees its client restart, or sees the lease expire,
  and releases the derived state.  Direct clients may now be
  granted conflicting state.
- The gateway offers its own grace period.  Gateway clients
  reclaim.
- The gateway cannot honor those reclaims.  The backend is not
  in grace and refuses reclaim requests; a new, non-reclaim
  request may conflict with state granted in the interim, and
  even when it succeeds, nothing guaranteed that the file was
  untouched between loss and re-acquisition.

This is the central problem.  The existing protocol has no
provision for it.

### 6.3. Both Restart

The backend is in grace and the gateway has lost its state.
Reclaims from gateway clients can be forwarded as ordinary
reclaims, provided they reach the backend before its grace period
ends.  The two grace periods are not coordinated: the gateway's
starts later and the backend does not know to wait for it.

### 6.4. Gateway Loses Its Lease

A partition between gateway and backend outlasts the backend's
lease.  Neither party restarts.  The backend may revoke derived
state.  Gateway clients have renewed their own leases with the
gateway throughout and have done nothing wrong, yet their state
is gone.  The gateway must report a loss it did not cause, with
the same NLM limitation as in Section 6.1.

A partition between a gateway client and the gateway is the
ordinary two-party case and raises nothing new.

## 7. Analysis by State Type

### 7.1. Opens and Share Reservations

- Applies to NFSv4 gateway clients and to NLM_SHARE.  NFSv3 I/O
  carries no open state; opens the gateway creates to serve it
  are internal and need no recovery.
- On gateway restart the backend closes the derived opens.  A
  direct client can then obtain an open whose deny mode excludes
  the gateway client, and the reclaim fails.
- Deny modes are mirrored only if the gateway's back-side client
  sends them.  A gateway that enforces deny modes locally and
  opens the backend file with no deny mode gives
  its clients an exclusion that direct clients do not observe.
  That is a gap in normal operation, and a solution for
  recovery has to assume it is closed first.
- A file that was removed while open exists at the backend only
  as long as the open does.  When the backend closes the derived
  open, the file is destroyed, and the gateway client's reclaim
  finds a stale filehandle.

### 7.2. Byte-Range Locks

- On gateway restart the backend releases derived locks.  A
  direct client can acquire a conflicting lock and modify the
  protected range before the gateway client reclaims.
- A gateway that grants the reclaim anyway, by taking a fresh
  lock, tells the application its lock was continuous when it
  was not.
- NLM has no way to tell a client that a lock was lost other
  than refusing a reclaim.

### 7.3. Delegations

- A gateway may hold a back-side delegation for its own caching,
  with no front-side delegation.  Losing it on gateway restart
  costs performance only.
- A gateway may grant front-side delegations.  That is safe only
  while the gateway holds a back-side delegation covering the
  same file, since otherwise the backend may grant conflicting
  access to direct clients.
- On gateway restart the back-side delegation is lost unless the
  backend supports reclaim of delegations by a restarted client
  (CLAIM_DELEGATE_PREV, RFC 8881 Section 10.2.1), which is
  optional.  The gateway client then reclaims a delegation the
  gateway cannot vouch for, together with opens and locks that
  were never visible to the backend.
- On backend restart the gateway reclaims its delegation.  The
  backend may grant it as already recalled, and the gateway then
  has to recall the front-side delegation.
- Applies to directory delegations as well as file delegations.

### 7.4. Layouts

A gateway can be involved with pNFS in three ways.

1. **Back side only.**  The gateway is a pNFS client of the
   backend and serves its own clients without layouts.  Layout
   recovery is the gateway's own (RFC 8881 Section 12.7).  The
   one gateway-specific issue is durability: the gateway must
   not acknowledge data as stable to a gateway client until the
   back-side LAYOUTCOMMIT has succeeded, because a gateway
   restart leaves uncommitted data undefined (Section 12.7.1).
2. **Front-side layouts.**  The gateway acts as a metadata server
   to its clients, granting layouts that depend on layouts it
   holds from the backend.  After a gateway restart, a gateway
   client that wrote to storage devices recovers by sending
   LAYOUTCOMMIT in reclaim mode (Section 12.7.4).  The gateway
   must pass that to the backend, which is not in grace and has
   released, and possibly fenced, the gateway's layout.  This is
   the layout form of the problem in Section 6.2.
3. **Gateway as a storage device.**  The backend's metadata
   server names the gateway in layouts it grants to its own
   clients.  Clients hold their state at the metadata server,
   not at the gateway.  This is the arrangement in
   draft-haynes-nfsv4-flexfiles-v2-proxy-server and is not a
   re-export in the sense of this document.

Layouts differ from the other state types in that they are not
reclaimed at all; recovery means committing or rewriting data.
How a metadata server validates a reclaim-mode LAYOUTCOMMIT, and
how it fences a failed client, depend on the layout type.

## 8. Current Practice

A gateway implementation has a small number of ways to respond
to the problem.  They are listed here as approaches.  Named
implementations are examples, and nothing in this document
depends on the choices any one of them made.

- **Refuse the stateful features.**  The gateway rejects lock
  requests and grants no delegations on a re-exported file
  system, so there is no derived state to lose.  Linux takes
  this approach for byte-range locks, NLM share reservations,
  and file delegations.
- **Enforce locally without derived state.**  The gateway tracks
  some state in its own tables and creates nothing at the
  backend.  The state then excludes other clients of the same
  gateway and no one else.  This avoids the recovery problem by
  not providing the guarantee.  Linux handles NFSv4 share deny
  modes this way.
- **Create derived state and accept the loss.**  The gateway
  forwards stateful requests, and after a restart it either
  refuses reclaims or re-acquires state with no guarantee of
  continuity.

TODO: survey implementations other than Linux (NFS-Ganesha with
its proxy back ends; commercial caching gateways) and record
which approach each takes.  Whether any implementation takes the
third approach is not known.

## 9. Scope

### 9.1. In Scope for This Problem Statement

All four state types and all four failure cases, as described
above, for every combination of front-side and back-side
protocol.

### 9.2. Proposed Scope for a First Solution

- Opens, share reservations, and byte-range locks.
- Gateway restart, both-restart, and the gateway behavior
  required for backend restart and lease loss.
- NFSv3 with NLM and all NFSv4 minor versions on the front side.
- NFSv4.2 on the back side, since that is the only version that
  can be extended (RFC 8178).

### 9.3. Delegations

The problem is stated here in full.  A first solution may
prohibit front-side delegations and leave their recovery to a
later revision or document.  The dependency on
CLAIM_DELEGATE_PREV support at the backend is the main reason
to stage it.

### 9.4. Layouts

Proposed division of work:

- **Here:** the layout-type-independent problem, namely the
  dependency of a front-side layout on a back-side layout, and
  the reclaim-mode LAYOUTCOMMIT that arrives at a backend not in
  grace.
- **Layout-type documents:** validation of recovered layout
  updates, fencing, and device mapping, per layout type.  For
  Flexible Files version 2 that is
  draft-haynes-nfsv4-flexfiles-v2-proxy-server.  The file
  layout (RFC 8881), Flexible Files version 1 (RFC 8435), SCSI
  (RFC 8154), and NVMe (RFC 9561) layouts would each need their
  own treatment if front-side layouts through a gateway are to
  be supported for them.
- **Not here:** arrangement 3 in Section 7.4.

Whether arrangement 2 is a deployment anyone needs is an open
question (Section 12).

### 9.5. Out of Scope

- Other re-export problems: filehandle size and construction,
  fsid mapping, export crossing.
- Replay protection across a gateway restart (reply cache and
  session state).  This is the ordinary server-restart problem.
- Write verifier handling across a gateway restart, beyond the
  durability point in Section 7.4.
- Timing of chained delegation and layout recalls (backend
  recalls from gateway, gateway recalls from its client) during
  normal operation.  Related, but not a recovery problem.
- Failover of a gateway's state to a different gateway host.
- Gateways behind gateways.
- NFSv3 on the back side: the problem is described, but NLM and
  NSM are not IETF protocols and cannot be extended here.

## 10. Requirements for a Solution

1. **No silent loss of exclusion.**  Between the loss of derived
   state and its recovery, the backend grants nothing that
   conflicts with it; or the gateway client is told the state
   was lost.
2. **Unmodified gateway clients.**  Gateway clients recover with
   the mechanisms their protocol already defines.
3. **Bounded cost to direct clients.**  State held for an absent
   gateway is released after a bounded time, and an
   administrator can release it sooner.
4. **Fallback.**  A gateway can detect a backend without the
   solution and fall back to refusing stateful requests.
5. **No new trust in gateway clients.**  Recovery does not let
   one gateway client obtain state another one held.
6. **Limited trust in gateways.**  The backend decides which
   clients are entitled to the special handling.

## 11. Security Considerations

- A backend that holds state for an absent gateway can be made
  to deny service to direct clients.  Entitlement has to be
  restricted to authenticated, authorized gateways.
- The backend sees one client where there are many.  It cannot
  check which gateway client a reclaim is for; it relies on the
  gateway, which has itself just lost its records.
- A gateway that grants reclaims it cannot back gives
  applications false assurance about data they believe was
  protected.

## 12. Open Questions on Scope

1. Do front-side layouts through a generic gateway (arrangement
   2) have a real deployment, or should the problem statement
   record the case and close it as not pursued?
2. Should delegations be in the first solution if the backend is
   required to support CLAIM_DELEGATE_PREV?
3. Is lease loss (Section 6.4) worth protocol work, or is
   reporting the loss all that can be done?
4. Should the problem statement be published on its own, or
   become the opening sections of the solution document?
