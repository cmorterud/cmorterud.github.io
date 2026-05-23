---
layout: post
title:  "MPI Collectives Are Communication Schedules"
date:   2026-05-28 09:00:00 -0400
categories: systems
---

# Introduction
MPI collectives look simple from the outside.

```c
MPI_Bcast(buffer, count, MPI_INT, root, comm);
MPI_Reduce(sendbuf, recvbuf, count, MPI_INT, MPI_SUM, root, comm);
MPI_Allgather(sendbuf, sendcount, MPI_INT,
              recvbuf, recvcount, MPI_INT, comm);
```

The project I worked on implemented several of these operations using only
point-to-point MPI calls. The implementation had to build the communication schedule
directly with sends, receives, nonblocking sends, and waits.

That is what made the assignment useful. A collective is not magic. It is a
pattern of messages.

## Broadcast
Broadcast sends one buffer from a root process to every other process.

The naive implementation is easy:

```text
root -> process 1
root -> process 2
root -> process 3
...
```

That works, but the root does all of the sending. A tree-based broadcast
spreads the work out.

```text
        0
      /   \
     1     2
    / \   / \
   3   4 5   6
```

Once a process receives the buffer, it can help forward it. The root is no
longer the only process doing useful communication.

The useful idea is to turn one root bottleneck into a fan-out schedule. The
details depend on the tree shape, message size, and number of processes, but
the core tradeoff is the same: more processes participate in forwarding the
data, and the root does less serial work.

## Scatter And Gather
Scatter and gather are opposites.

Scatter starts with the root holding a different chunk for each process:

```text
root: [A][B][C][D]

process 0 gets A
process 1 gets B
process 2 gets C
process 3 gets D
```

Gather collects one chunk from each process and assembles the full result at
the root.

The tree version is more interesting than a flat loop because each internal
process handles a range of the final buffer. In scatter, a process receives a
larger range and forwards the part that belongs to another subtree. In
gather, a process receives a range from a subtree and forwards a larger
combined range upward.

That makes ownership part of the algorithm. It is not enough to know who
sends to whom. The schedule also has to preserve which process owns each
piece of the final buffer.

## Reduce
Reduce combines values from every process into one result.

For this assignment, the reduce operation was limited to a simple summation
case. Even with that restriction, the interesting part is where the addition
happens.

A reduction tree lets processes combine partial results as messages move
toward the root. Some processes send their current partial result and then
drop out of the schedule. Other processes receive a partial result, combine
it with their own, and continue.

The useful part is that the data is reduced as it moves. The root does not
receive every process's raw input. It receives progressively larger partial
results.

## Allgather
Allgather is gather without a single final owner. Every process ends with the
full result.

The implementation copied each process's own chunk into the correct position
in the receive buffer, then used nonblocking sends and receives to exchange
chunks with the other ranks.

The code used `MPI_Isend`, `MPI_Irecv`, and `MPI_Waitall` so the sends and
receives could be posted before waiting for completion.

That matters because communication code can deadlock when every process waits
for another process to move first. Nonblocking operations let the schedule
describe all of the expected traffic before blocking on completion.

## Scatter-Allgather Broadcast
The second broadcast implementation used a scatter-allgather approach.

Instead of sending the whole buffer down a tree, the root first scattered
pieces of the buffer. Then all processes participated in an allgather so
every process received every piece.

This is a different cost model from tree broadcast. Tree broadcast is simple
and good for many cases. Scatter-allgather can reduce pressure on the root
for larger buffers because the data is divided before it is replicated.

## Testing
The tests compared each custom collective against the MPI collective with the
same behavior. The driver also measured performance by timing multiple trials
and reporting the slowest process time.

That last detail is important. In parallel code, the runtime of the operation
is the runtime of the slowest participant. If one rank is still communicating,
the collective is not done.

## Conclusion
This project made MPI collectives feel less like library calls and more like
distributed protocols.

Broadcast is fan-out. Gather is fan-in. Scatter is ownership transfer. Reduce
is fan-in with computation along the way. Allgather is shared replication.

Those same patterns show up outside HPC. Backend systems use the same ideas
when they shard work, aggregate results, replicate state, or coordinate
workers. The names change, but the questions are familiar: who owns the data,
who needs a copy, what can run in parallel, and where does the bottleneck
move?

Please feel free to message me on [LinkedIn]({{ site.linkedin_url }})
with any questions or concerns.
