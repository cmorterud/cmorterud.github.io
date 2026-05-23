---
layout: post
title:  "TeraSort and the Cost of Moving Data"
date:   2026-05-23 10:45:00 -0400
categories: systems
---

# Introduction
TeraSort is a sorting benchmark. At small scale, the interesting part is the
comparison function. At cluster scale, the interesting part is data movement.

The project I worked on used Chapel to sort TeraSort-style records across
multiple locales. The goal was to produce one globally sorted output, even
though the input was spread across machines.

This is a useful systems problem because local correctness is not enough.

## Local Sort Is Not Global Sort
Suppose two locales each sort their local records:

```text
locale 0: 30, 40
locale 1: 10, 20
```

Each local array is sorted, but the full output is still wrong. The records
with keys `10` and `20` need to appear before the records with keys `30` and
`40`.

The distributed version has to answer a second question:

```text
Where should each record live in the final output?
```

That question matters more than the local sorting code.

## Chapel Locales
Chapel exposes distributed execution through locales. A locale is a place
where computation and memory live. The project used Chapel locales for the
sort itself, while the provided generator and validator used MPI.

The first step was local work: each locale sorted the records it already had.
That made each local range easier to reason about, but it did not decide the
final owner of each record.

## Sampling And Splitters
The implementation used sampling to estimate good partition boundaries.

After sorting locally, each locale sampled keys from its local range. Those
samples were gathered and sorted. From that sorted sample set, the code chose
splitters.

Conceptually:

```text
splitters: 25, 50, 75

locale 0: keys < 25
locale 1: 25 <= keys < 50
locale 2: 50 <= keys < 75
locale 3: keys >= 75
```

The splitters do not have to be perfect, but they need to be good enough to
avoid giving one locale most of the records. Bad splitters turn a distributed
sort back into a bottleneck.

## Moving Records
Once the splitters were known, each record was assigned a destination locale.
This is where the project became more about systems than sorting.

The implementation then had to move records to the locale responsible for
their key range. Sorting locally first helped because records for the same
destination tended to be grouped by key range rather than appearing as a
completely arbitrary stream.

That is the practical lesson. Distributed algorithms often spend their time
not on the core operation, but on arranging the data so the core operation is
cheap enough to run.

## Final Sort
After records were moved to their destination locales, each locale sorted its
final partition again.

That final local sort is still necessary because a destination locale receives
records from multiple source locales. Each source range may be sorted, but
concatenating sorted ranges does not guarantee one sorted result.

For example:

```text
from locale 0: 10, 30
from locale 1: 20, 40

combined:      10, 30, 20, 40
sorted output: 10, 20, 30, 40
```

Once each destination partition is sorted, the global output is sorted because
the splitter boundaries define the order between locales.

## Validation
Validation was an important part of the assignment. It is easy to write a
distributed sort that looks plausible on small input and fails when data is
skewed.

The validator checked that the output contained the right number of records
and that the global ordering was correct.

The count check matters. A distributed sort can produce sorted output and
still be wrong if it drops or duplicates records while moving data.

## Conclusion
TeraSort is a good distributed systems exercise because the algorithm is
simple to state and easy to get wrong in the details.

The useful part was not learning that sorting exists. It was dealing with
partitioning, load balance, record ownership, and validation across locales.
Those are the same concerns that show up in backend systems that shard data,
rebalance work, or move state between workers.

Please feel free to message me on [LinkedIn]({{ site.linkedin_url }})
with any questions or concerns.
