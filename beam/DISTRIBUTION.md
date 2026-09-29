---
title: "BEAM: Distribution"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Distribution

*Send a message to a process on another machine with the same syntax as a local one. The network is an implementation detail the language hides.*

---

## Location transparency

`Pid ! message` looks the same whether the receiver is on this node or
another continent. All the routing information lives in the process
identifier; Erlang guarantees pids are unique across the network. Code
written for one machine runs on a dozen unchanged — and a program
designed for a dozen machines can be tested on a laptop.

This one property changes how systems get designed: communication
between machines stops being a threshold to cross and becomes the
normal state of things.

---

## Nodes

A **node** is a running VM configured for distribution, named
`nodename@hostname`:

| Flag | Name form | Use |
|------|-----------|-----|
| `-name` | `simple_cache@mybox.home.net` | Normal networks with working DNS |
| `-sname` | `simple_cache@mybox` | Short names, same subnet, no DNS |

Long and short names **cannot be mixed** in one cluster — they are
different communication modes.

---

## How nodes find each other

- **EPMD** (Erlang Port Mapper Daemon, port 4369) maps node names to
  ports on each host. A node wanting to reach another asks the remote
  host's EPMD.
- Nodes do not discover each other automatically: one node must look
  for another. Once two connect, they **exchange everything they know**
  about other nodes, so the cluster becomes fully connected.
- The cluster is a **fully connected mesh**; the practical ceiling is a
  couple of dozen nodes — connection overhead grows quadratically.
  **Hidden nodes** connect for inspection without joining the mesh
  proper.

---

## The magic cookie

A node refuses traffic from any node that does not know its cookie — a
shared secret, generated into `~/.erlang.cookie` on first start. This is
**authorization, not security**: the default distribution model assumes
a trusted network. For anything crossing an untrusted boundary, use
TLS distribution, IPsec, or your own protocol — not the plain
distribution port.

---

## The operational realities

| Reality | Consequence |
|---------|-------------|
| Sending is fire-and-forget | A robust sender handles no-reply the same way whether the receiver died or the network partitioned — "not responding" is one failure mode |
| Network adds nondeterminism | Message ordering and delivery guarantees weaken across nodes; design for it |
| Cookie mismatch is the #1 connection failure | After firewalls. Check the cookie before anything else |
| Distribution ≠ data replication | Mnesia (or your own design) is what replicates; distribution only connects |

---

## Rules of thumb

- Reach for distribution when one machine cannot hold the work —
  not before.
- Keep clusters small and homogeneous (one name mode, one cookie,
  one trust domain).
- Prefer process groups (`pg`) or registries over ad-hoc node-name
  addressing; node names churn in fleets.
- In fleet deployments, pinned TLS distribution with short names beats
  long-name DNS assumptions that break in containers.
