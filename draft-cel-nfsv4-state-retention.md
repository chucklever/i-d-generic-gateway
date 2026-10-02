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
clients.  {{overview}} describes the sequence and {{extension}}
specifies it.

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


# Protocol Overview {#overview}

TODO Protocol Overview


# Protocol Extension {#extension}

TODO Protocol Extension


# Gateway Server Behavior

TODO Gateway Server Behavior


# Backend Server Behavior

TODO Backend Server Behavior


# XDR Description

TODO XDR Description


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

TODO IANA


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
