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
are still running and will attempt to recover that state.  This
document extends NFSv4.2 so that a server can retain the state of
a client that has restarted and permit the new instance of that
client to reclaim it.


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
RFC8881}}).  For most clients this is the correct outcome.  The
applications that held the state restarted along with the client,
and nothing remains that could reclaim it.

A gateway restart is different.  The gateway's clients did not
restart.  They observe only that their server was unreachable for
a time, and when it returns they begin state recovery: NFSv4
clients reclaim their opens and locks during the gateway's grace
period, and NFSv3 clients reclaim their NLM locks when the
gateway's status monitor announces the restart.  Each of those
reclaims can succeed only if the gateway can re-establish the
corresponding state on the backend.  The backend, however, has
already released that state, and it accepts reclaim-type requests
only during its own grace period, which it did not enter because
it did not restart ({{Section 8.4.3 of RFC8881}}).  The gateway
has no way to restore its clients' state, and in the interval
since the restart the backend may have granted conflicting opens
or locks to other clients.

NFSv4.1 already retains one kind of state across a client restart.
A server that supports CLAIM_DELEGATE_PREV keeps a restarted
client's delegations for at least a lease period so that the new
instance of the client can reclaim them ({{Section 10.2.1 of
RFC8881}}).  This document applies the same idea to opens and
byte-range locks.  It defines client-restart state retention: a
server that supports it and a client that has requested it agree
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
it introduces applies only to client IDs that have negotiated it.
Servers and clients that do not implement the extension are
unaffected.

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

TODO Terminology


# Problem Summary {#problem}

TODO Problem Summary


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
