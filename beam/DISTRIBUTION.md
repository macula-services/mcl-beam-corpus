---
title: "BEAM: Distribution"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Distribution

*Connected BEAM nodes address each other's processes with the same send, link and monitor primitives used locally. Convenient, and built for a trusted network.*

---

## What it is

A node started with a name (`-name` or `-sname`) can connect to other
named nodes. Once connected, a pid on the remote node behaves like a
local one: `send/2`, links and monitors all work, and the pid itself
carries which node it lives on. Code written against pids does not care
where the process runs, which is what lets a cluster be tested on a
laptop.

---

## How it works

| Piece | Role |
|-------|------|
| Node name | `name@host`. `-name` uses fully qualified host names, `-sname` short ones. The two modes cannot talk to each other. |
| EPMD | A small daemon on each host (default TCP port 4369) that maps node names to the port each node listens on. |
| Cookie | A shared secret; nodes whose cookies do not match refuse the connection. |
| `net_kernel` | Manages connections. Connecting to one node also connects you to the nodes it knows (transitive connections), unless started with `-connect_all false`. |
| Hidden nodes | Started with `-hidden`; their connections are not transitive and they do not show up in `nodes()`. Useful for tooling and remote shells. |

Every node keeps a TCP connection to every other node it knows, so a
default cluster is a full mesh and connection count grows quadratically.
That, plus heartbeat traffic, is why default distribution suits tens of
nodes, not thousands.

---

## Semantics you must design for

- **Ordering holds per pair, delivery does not.** Signals from one
  process to another arrive in the order sent, but if the connection
  drops, messages in flight can be lost. Links fire with reason
  `:noconnection` even though the remote process may still be alive.
- **A missing reply is ambiguous.** Crash, overload and network
  partition look identical from the caller; use timeouts and monitors,
  and make remote operations safe to retry.
- **Distribution connects, it does not replicate.** Data lives on one
  node unless you replicate it (Mnesia, an event store, your own
  protocol).

## Security

The cookie handshake guards against accidental cross-talk, not
attackers: the OTP documentation describes it as not cryptographically
secure, and a connected node can run arbitrary code on its peers. Use
TLS distribution (`-proto_dist inet_tls`) and keep the distribution
ports off untrusted networks.

## Rules of thumb

- Use distribution when one node is not enough, and keep a cluster to
  one trust domain, one naming mode and one cookie.
- Address services by group or registry, not by hard-coded node names:
  `:pg` for cluster-wide groups, [REGISTRY](REGISTRY.md) locally.
- In container fleets, prefer explicit node names and TLS over
  assumptions about DNS and EPMD reachability.

Related: [SCHEDULER](SCHEDULER.md), [ETS](ETS.md), [CONCURRENCY_MODELS](CONCURRENCY_MODELS.md).

## Sources

- *Erlang and OTP in Action*, 1st edition, Martin Logan, Eric Merritt and Richard Carlsson, Manning, 2010. <https://www.manning.com/books/erlang-and-otp-in-action>
- Erlang/OTP Reference Manual, Distributed Erlang. <https://www.erlang.org/doc/system/distributed.html>
- Erlang/OTP Reference Manual, Processes (signal ordering, links, monitors). <https://www.erlang.org/doc/system/ref_man_processes.html>
- ERTS `epmd` reference. <https://www.erlang.org/doc/apps/erts/epmd_cmd.html>
- SSL application, Using TLS for Erlang Distribution. <https://www.erlang.org/doc/apps/ssl/ssl_distribution.html>
- Kernel `pg` reference. <https://www.erlang.org/doc/apps/kernel/pg.html>
