---
layout: post
title:  "Distance Vector Routing and Distributed State"
date:   2026-05-21 09:00:00 -0400
categories: systems
---

# Introduction
Distance vector routing is a routing algorithm where each node only knows
about its neighbors and the costs advertised by those neighbors.

The project I worked on simulated a network of nodes running Bellman-Ford.
Each node kept a local distance vector, received messages from neighboring
nodes, updated its own distances, and advertised changes back to its
neighbors.

This is interesting because no node has the full topology. Each node is
making a local decision based on partial information.

## The Problem
Suppose a node `A` has a direct link to `B` with cost 2. If `B` advertises
that it can reach `C` with cost 3, then `A` can infer that it can reach `C`
through `B` with cost 5.

Conceptually:

```text
A -> B = 2
B -> C = 3

A -> C through B = 2 + 3 = 5
```

The general rule is:

```text
distance_to_target = cost_to_neighbor + neighbor_distance_to_target
```

If that computed distance is better than the current known distance, the node
updates its distance vector.

```python
candidate = cost_to_neighbor + neighbor_distance_to_target

if candidate < current_distance_to_target:
    current_distance_to_target = candidate
```

That is the local Bellman-Ford update.

## Messages
In the simulator, nodes did not inspect the full topology. This was not a
Mininet project. It used a Python topology simulator where nodes exchanged
messages with their neighbors.

A node processed messages from neighbors and updated its own state.

A message can be thought of as:

```python
message = {
    "from": "B",
    "distances": {
        "A": 2,
        "C": 3,
        "D": 7
    }
}
```

When `A` receives that message from `B`, it combines the advertised distances
with the cost of the local link from `A` to `B`.

This is the part of the project that is useful outside of networking. The
node is not receiving commands. It is receiving another node's view of the
world, then reconciling that view with its own local state.

## Convergence
The algorithm is iterative. One node learns a better path, advertises it,
another node learns from that update, and so on.

For example:

```text
A -> B -> C -> D
```

At first, `A` may only know about `B`. After messages propagate, `A` can
learn that `D` is reachable through `B` and `C`.

This is why routing convergence is not instant. The correct answer is not
computed once in a central place. It emerges as updates propagate through the
network.

That is also why unnecessary messages matter. If every node advertises every
round, the system does extra work. If a node only advertises when its vector
changes, the simulation has a cleaner termination condition.

## Bad News Travels Too
The easy case is discovering a better path. The harder case is discovering
that a local rule can keep improving a route forever.

Distance vector protocols are vulnerable to stale information because nodes
learn indirectly. A node may continue to believe a route exists because a
neighbor still advertises it, even if the neighbor learned it from the
original node.

That is the classic count-to-infinity problem in real routing protocols.
This project did not model that full failure case with link removals,
split horizon, or poison reverse.

The implementation also did not track the next hop used for each destination.
Nodes advertised their current distance vector to neighbors without recording
which neighbor originally supplied each route. That is acceptable for the
static simulator, but it is exactly the extra protocol state a real
distance-vector protocol needs in order to avoid advertising a route back to
the neighbor it learned it from.

The project included topologies with negative edges and negative cycles. That
forced the implementation to handle routes whose cost can keep decreasing.
Without a boundary, a negative cycle can keep improving forever from the
point of view of the local update rule.

The implementation handled that by bounding very low costs, which effectively
acted as negative infinity for the simulator.

## Distributed State
The useful backend analogy is distributed state.

Many backend systems work like this:

* a service has local state
* a neighbor or dependency reports its state
* the service combines that report with local knowledge
* if local state changes, the service publishes an update

That pattern appears in routing, service discovery, caches, membership
systems, replicated metadata, and workflow coordination.

The risk is the same in each case. A message is not the full truth. It is one
node's current view of the truth. The receiver still has to decide whether
that view changes its own state.

## Testing
Useful tests for distance vector routing are mostly topology tests:

* a simple line topology
* multiple paths to the same node
* asymmetric link costs
* cycles
* negative edges
* negative cycles
* nodes that are not reachable from every other node

The logs are important because the final answer is not the only thing that
matters. It is also useful to see how the network reaches that answer.

For example, if a node advertises when nothing changed, the final distances
may still be correct, but the protocol is doing unnecessary work.

## Conclusion
Distance vector routing is a good example of distributed computation with
partial information. Each node has a small local rule, but the system-level
behavior depends on message propagation and convergence.

That is the part that carries over to backend systems. When services exchange
state, the receiving service should be explicit about what it believes, why
it changed, and when it needs to tell others.

Please feel free to message me on [LinkedIn]({{ site.linkedin_url }})
with any questions or concerns.
