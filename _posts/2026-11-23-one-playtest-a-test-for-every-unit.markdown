---
layout: post
title:  "One Playtest, a Test for Every Unit"
date:   2026-11-23 09:00:00 -0500
categories: projects testing
description: "A playtest of sim-engine turned up shells that seemed to land short and bombers that circled forever. Instead of fixing two units, I asked for tests that check every unit against its own data. Those tests found bugs I never saw while playing."
---

# Introduction

I have been working on [sim-engine](https://codymorterud.com/sim-engine/), a
real-time strategy game that runs in the browser. It has 117 units and
structures across land, sea, and air. Each one is defined by a row of data: its
weapons, range, damage, rate of fire, speed, cost, and so on.

A quick note on how it is built, because it matters for this article. I wrote
the original specification, I make the design calls, and I playtest it. AI
coding agents write the code. So most of my contribution is noticing when
something is wrong and deciding what "right" means.

This article is about one playtest session and what came out of it. The session
found two bugs. The tests that came out of it found more, and none of those
were bugs I had seen while playing.

## The Details

### 1. Nine problems from one playtest

On November 16, I played a match and wrote down everything that felt wrong. The
list had nine items. Most were small, like the minimap appearing upside down.
Two were not small.

The first was shells landing short. I ordered a group of medium tanks to attack
the enemy commander. He was in range, they were firing, and the shells were
landing short of him. I wrote that this was "a huge problem," and it would have
been, if it were true. A tank that cannot hit what it is aiming at is useless.

It turned out the shells were hitting. The renderer drew each explosion where
the shell was on its last tick in the air, one tick of flight short of where it
actually hit. The simulation was right. The picture was wrong.

The second was fighter-bombers that circled forever. They would make one pass,
and then circle the target without ever dropping another bomb. This one was
real. After a pass, a fighter-bomber flew 400 units past its target before
turning back, but its bombs carry 840 units forward after release. It could
never get far enough away to line up another drop, so it circled.

Both were fixed the same day. But both left me with the same question.

### 2. What about the other units?

I found the bomber bug because I happened to use fighter-bombers in that match.
I found the shell bug because I happened to watch tanks fire at a commander.
There are 117 units and structures in the game. I am not going to play enough
matches to watch every one of them do every one of its jobs.

But I do not have to. Every unit already says what it should do. Its data lists
its weapons, what each one can hit, its range, its damage, and how often it
fires. That data is a specification. So the tests can be written against the
data instead of against any particular unit.

So I asked for tests that every unit can hit each kind of target it is meant to
hit: every tank can damage a ground target at its maximum range, every bomber
drops a bomb on every pass, and every anti-air weapon hits an aircraft flying
over. And per weapon, not just per unit. A battleship has more than one weapon,
so it needs more than one check.

### 3. Damage per second covers every weapon at once

Hitting a target once is a low bar. A unit with three weapons could pass with
only one of them working.

So the next thing I asked for was a test that every unit deals the damage per
second its data says it should, against each kind of target it can attack. Both
standing still, and moving in a straight line beside a target that is moving
too.

The point of damage per second is that it checks every weapon at once. If any
weapon fails to fire, fires too slowly, or fires and misses, the total comes up
short.

A simplified version of the test looks like this:

{% highlight rust %}
#[test]
fn every_unit_deals_its_damage_a_second_to_each_kind_of_target() {
    let mut failures = Vec::new();
    for unit in blueprints() {
        for kind in [Kind::Land, Kind::Ship, Kind::Sub, Kind::Air] {
            let weapons = weapons_that_hit(&unit, kind);
            if weapons.is_empty() {
                continue;
            }
            for setting in [Setting::StandingStill, Setting::MovingAlongside] {
                let expected = damage_per_second(&weapons);
                let measured = measure(&unit, kind, setting);
                if !close_enough(measured, expected) {
                    failures.push(format!("{} {setting:?} vs {kind:?}", unit.key));
                }
            }
        }
    }
    assert!(failures.is_empty(), "{failures:#?}");
}
{% endhighlight %}

The real test has more detail: where to place the shooter and the target on
land or water, how long to measure, and a tolerance for scattered shots. But the
shape is the same. It loops over the data, and it collects every failure before
reporting, so one run shows every broken unit instead of only the first.

It came out to 91 cases.

### 4. Testing the tests

A test that never fails is not proving much. It could be checking nothing.

Two checks guard against that. First, the damage test asserts that it actually
checked at least 60 cases. If a mistake in its filters made it skip most units,
it fails instead of passing quietly. Second, a separate test checks that
specific units of each kind, such as a tank, an artillery piece, a battleship, a
gunship, an anti-air turret, and a submarine, are among the ones being checked.

The damage test was also checked by breaking things on purpose. Silencing the
first extra weapon on units that have more than one made 19 of the 91 cases
fail. So each weapon really does have to fire at its rate and land.

### 5. What the tests found that I never saw

This is the part that made it worth it. The tests found bugs that no playtest
had shown me.

| Test | What it found |
| --- | --- |
| Damage per second while moving | A unit given a new move order stopped for a tick while its route was built. Repeated orders made units lurch at half speed, and shots aimed using their speed missed. A moving frigate landed only 40% of its expected damage. |
| Every shield | A side with nothing left but an army skipped shield upkeep entirely, so its mobile shields would freeze. |
| Every cost | Building fell a hair short of done after a whole number of steps, and the extra step was charged in full. Everything cost up to 1.4% more than its data. A medium tank cost 40.33 instead of 40. |
| Every transport | A heavy transport loading 28 tanks took on 18 and left the rest waiting around it forever. |

None of these would be easy to find by playing. A 1.4% difference in cost is
invisible. Frozen shields need a side to lose every builder and structure while
keeping an army with mobile shields. A transport that half loads looks like a
pathing quirk.

The frigate is the closest. I might eventually have noticed ships missing more
than they should. But "ships seem to miss a lot" is hard to act on. "A moving
frigate deals 40% of its expected damage" is not.

### 6. One pattern, everywhere

After the weapon tests, the same pattern spread to the rest of the data within
about a day:

- every unit's build time, and every structure's build rate
- every unit's sight, and every radar's range
- every structure's income, and every death blast
- every unit's cost
- every transport's capacity
- every missile and torpedo defense
- every builder's capture, reclaim, and repair rates
- every enhancement's effects

Each one loops over the data and checks that the game does what the data says.

A nice side effect is that new units are tested automatically. When a unit is
added to the data, it is in every one of these tests the next time they run,
without anyone writing a test for it.

### 7. General lesson

Playtesting and these tests do different jobs.

Playtesting tells me whether the data itself is right. A test can confirm that
a bomber deals exactly the damage its data says. It cannot tell me whether that
damage is too much against a factory, which is exactly the kind of thing I
noticed while playing. That is a judgment call, and it needs someone who knows
how the game should feel.

The tests tell me whether the game does what the data says, for every unit,
every time the code changes. That is not a judgment call. It is just checking,
and it should be exhaustive rather than sampled.

This is close to what I do in my day job, where I work on verification and
validation for software releases. Each requirement should trace to a test, and
the useful question is not "did we test some of it?" but "which requirements
are verified, which are failing, and which have no test at all?" A unit's row of
data is a requirement. A test that loops over every row is requirement
coverage.

The other lesson is about the bugs themselves. The two I found by playing were
the visible ones. The ones the tests found were quieter, and they were probably
always going to be the larger group. If a playtest shows you one broken unit,
do not assume the problem stops at that unit. Test the whole category.

## Thanks!

Thanks for reading!
