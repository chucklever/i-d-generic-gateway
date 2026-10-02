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
  RFC7862:
  RFC7863:
  RFC8178:
  RFC8881:

informative:
  RFC1813:
  RFC7530:
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
side of the gateway, a back side other than NFSv4.2, failover of
a gateway's state to a different gateway host, and a gateway
whose backend is itself a gateway.


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
has a session with which to read an attribute.

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

Neither limit is advertised to the client.  A client needs to
know only whether reclaims can succeed, which the EXCHANGE_ID
reply tells it, and the reply to each reclaim is authoritative.

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

TODO Completing Reclaim

## Errors {#errors}

TODO Errors

## Interaction with the Server's Grace Period {#grace}

TODO Interaction with the Server's Grace Period


# Gateway Server Behavior

TODO Gateway Server Behavior

## Owner Derivation {#owners}

TODO Owner Derivation


# Backend Server Behavior

TODO Backend Server Behavior


# XDR Description {#xdr}

TODO XDR Description


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

TODO IANA


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
9. G starts its front-side grace period and sends SM_NOTIFY to C.
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

TODO acknowledge.
