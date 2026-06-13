---
layout: post
title:  "Turning Inputs Into a Movement Vector For Bots"
date:   2026-06-13 09:00:00 -0400
categories: projects ai systems
---

# Introduction

[Mass Arena](https://mass-arena.com/) has bots so the game does not feel empty when only a few real
players are online. The bots are not driven by a neural network, a planning
system, or a heavyweight behavior tree. They use a small steering system.

That still counts as game AI in the practical sense. The bot observes the
world, evaluates nearby objects, combines competing desires, and turns those
inputs into one output: a movement direction.

The core idea is:

```text
world inputs -> weighted steering influences -> output direction vector
```

The bot does not decide on a sentence like "go eat that pellet" or "run from
that player." It builds an acceleration-like vector from many small pushes and
pulls. Pellets pull the bot toward food. Large enemies push it away. Smaller
players pull it into a chase. Arena walls push it back toward the playable
space. Random wandering adds some variation.

At the end, all of those influences collapse into a normalized direction
vector.

## Why Use Steering Instead of a State Machine?

A simple state machine might look like this:

```text
if threatened:
    flee
else if prey nearby:
    chase
else:
    eat pellets
```

That can work, but it tends to create brittle behavior. The bot is always in
one mode, and the boundaries between modes become awkward. What should happen
when food is nearby, prey is also nearby, and a larger player is approaching
from the side? A strict state machine usually has to pick one behavior and
throw the others away.

The steering approach is softer. Each input contributes to the final movement
vector. Threats can dominate because they have higher weight, but pellets,
wandering, prey, and arena bounds can still shape the result.

That is useful for a multiplayer arena game because the environment changes
continuously. The bot does not need a perfect plan. It needs to move in a way
that is plausible, responsive, and cheap enough to run every tick.

## Inputs to the Bot

The steering function has a few categories of input:

```text
bot position
bot mass
bot personality
pellet positions
powerup positions
other entities
arena bounds
short-term bot memory
```

The output is much smaller:

```text
bot.dx
bot.dy
```

That is the important design constraint. No matter how much information the
bot considers, the game loop only needs a direction.

The function starts by choosing a personality configuration:

{% highlight typescript %}
const config = BOT_CONFIG[bot.botPersonality ?? "GRAZER"];
{% endhighlight %}

That configuration controls the weights and thresholds used later. A grazer
can care more about pellets. A more aggressive bot can use a lower chase
threshold or a higher prey weight. The decision structure stays the same, but
the personality changes how strongly the bot reacts to each signal.

## Food as an Attraction Vector

The simplest behavior is pellet seeking.

The bot scans all pellets and finds the nearest one:

{% highlight typescript %}
for (const pellet of room.pellets) {
  const d = distanceSquared(bot.x, bot.y, pellet.x, pellet.y);
  if (d < nearestPelletDistance) {
    nearestPelletDistance = d;
    nearestPellet = pellet;
  }
}
{% endhighlight %}

Distance is compared using squared distance. That avoids a square root inside
the search loop. The exact distance is only needed after the nearest pellet has
already been chosen.

Once a target exists, the code converts it into a unit direction:

{% highlight typescript %}
const dist = Math.sqrt(Math.max(1, nearestPelletDistance));

ax += ((nearestPellet.x - bot.x) / dist) * config.pelletWeight;
ay += ((nearestPellet.y - bot.y) / dist) * config.pelletWeight;
{% endhighlight %}

This is the basic pattern used throughout the steering logic:

```text
direction = target_position - bot_position
unit_direction = direction / distance
weighted_direction = unit_direction * behavior_weight
```

Then the weighted direction is added to the accumulator.

```text
ax += weighted_x
ay += weighted_y
```

The accumulator is the intermediate decision. It is not yet the final movement
direction. It is the sum of everything the bot currently wants or wants to
avoid.

## Threats as Repulsion Vectors

Other players are more complicated than pellets because they can be either
dangerous or attractive depending on relative mass.

For each nearby entity, the bot computes two ratios:

{% highlight typescript %}
const botMass = effectiveMass(bot, now);
const otherMass = effectiveMass(other, now);
const botMassRatio = botMass / Math.max(1, otherMass);
const otherMassRatio = otherMass / Math.max(1, botMass);
{% endhighlight %}

These ratios answer two different questions:

```text
How much bigger am I than the other entity?
How much bigger is the other entity than me?
```

If the other entity is large enough and close enough, it becomes a threat:

{% highlight typescript %}
if (otherMassRatio >= config.fleeRatio && d < THREAT_SENSE_DISTANCE_SQ) {
  threatDetected = true;
  ax -= towardX * config.threatWeight;
  ay -= towardY * config.threatWeight;
  continue;
}
{% endhighlight %}

The direction `towardX, towardY` points from the bot toward the other entity.
To flee, the bot subtracts that direction. A threat therefore pushes the
accumulator away from itself.

This is a clean way to represent avoidance:

```text
toward threat = +direction
away from threat = -direction
```

The `continue` matters too. Once an entity is classified as a threat, the bot
does not also consider it as prey or as a similar-sized body. Threat handling
has priority.

## Prey as a Deferred Attraction

If the bot is large enough compared with another entity, that entity can
become prey:

{% highlight typescript %}
if (botMassRatio >= config.chaseRatio && d < PREY_SENSE_DISTANCE_SQ) {
  if (d < bestPreyDistance) {
    bestPreyDistance = d;
    bestPrey = other;
  }
  continue;
}
{% endhighlight %}

Notice that the code does not immediately add every prey vector to the
accumulator. It chooses the nearest prey first, then applies one chase vector
after the scan.

That avoids a common steering problem. If three smaller players are around the
bot in different directions, summing all three chase vectors could point the
bot toward the average of them, which might be toward none of them in
particular. For prey, a single target is usually more believable.

After the loop, the chosen prey contributes its own attraction:

{% highlight typescript %}
if (bestPrey) {
  const dist = Math.sqrt(Math.max(1, bestPreyDistance));

  ax += ((bestPrey.x - bot.x) / dist) * config.preyWeight;
  ay += ((bestPrey.y - bot.y) / dist) * config.preyWeight;
}
{% endhighlight %}

Food is an always-available background preference. Prey is a focused target.
Threats can override both by pushing the bot away with a stronger weight.

## Similar-Sized Players

The bot also avoids entities that are close to its own mass:

{% highlight typescript %}
if (
  botMassRatio > 0.90 &&
  botMassRatio < 1.10 &&
  d < SIMILAR_AVOID_DISTANCE_SQ
) {
  ax -= towardX * config.similarAvoidWeight;
  ay -= towardY * config.similarAvoidWeight;
}
{% endhighlight %}

This is a small but important detail. A similar-sized player is not clearly
prey and not clearly a threat. If bots ignored that case, they could drift
into ambiguous collisions that do not look intentional.

This rule gives the bot a mild preference for separation. It makes the bot
look less careless without requiring a complex tactical model.

## Wander as Short-Term Memory

If the bot only reacted to pellets and players, its movement could become too
deterministic. It would always snap toward the nearest target. The wander logic
adds low-frequency variation.

The bot does not pick a new random direction every frame. It checks only on an
interval:

{% highlight typescript %}
if (!threatDetected && (!bot.nextWanderCheckAt || now >= bot.nextWanderCheckAt)) {
  bot.nextWanderCheckAt = now + BOT_DECISION_INTERVAL_MS;
  if (Math.random() < WANDER_CHANCE_PER_DECISION) {
    const dir = normalize(Math.random() * 2 - 1, Math.random() * 2 - 1) ?? { x: 1, y: 0 };
    bot.wanderUntil = now + WANDER_DURATION_MS;
    bot.wanderDx = dir.x;
    bot.wanderDy = dir.y;
  }
}
{% endhighlight %}

That gives the bot short-term memory:

```text
wander direction
wander expiration time
next decision time
```

If the wander is active, it contributes another weighted vector:

{% highlight typescript %}
if (bot.wanderUntil && now < bot.wanderUntil) {
  ax += (bot.wanderDx ?? 0) * WANDER_WEIGHT;
  ay += (bot.wanderDy ?? 0) * WANDER_WEIGHT;
}
{% endhighlight %}

The `!threatDetected` condition is also important. Randomness should not make
the bot casually wander into danger. Threat avoidance gets priority.

## Arena Bounds as Environmental Pressure

The arena itself contributes a steering vector:

{% highlight typescript %}
const margin = 220;
const arenaAvoidance = arenaAvoidanceVector(room.arena, bot.x, bot.y, margin);
ax += arenaAvoidance.x;
ay += arenaAvoidance.y;
{% endhighlight %}

This is another example of treating the environment as an input vector. The
bot does not need a separate "near wall" state. When it gets too close to the
edge, the arena adds pressure back toward the playable area.

That pressure is just another influence in the accumulator.

## Normalizing the Decision

After all inputs have been processed, the accumulated vector is normalized:

{% highlight typescript %}
const dir = normalize(ax, ay);
{% endhighlight %}

This turns the accumulated desire into a direction. Without normalization, a
bot with many active influences could move faster than a bot with only one
active influence. In this game, the steering function is responsible for
direction, not speed.

The size of the accumulated vector still matters indirectly. Weights determine
which direction wins before normalization. But once the final direction is
chosen, movement speed can stay controlled elsewhere in the simulation.

## Smoothing the Output

The function does not immediately replace the bot direction with the new
direction. It blends the old direction with the new one:

{% highlight typescript %}
bot.dx = bot.dx * 0.85 + dir.x * 0.15;
bot.dy = bot.dy * 0.85 + dir.y * 0.15;
{% endhighlight %}

Then it normalizes again:

{% highlight typescript %}
const smoothed = normalize(bot.dx, bot.dy);

if (smoothed) {
  bot.dx = smoothed.x;
  bot.dy = smoothed.y;
}
{% endhighlight %}

This smoothing step keeps motion from looking jittery. If the nearest pellet
changes from one side of the bot to another, or if a prey target becomes
available for a single tick, the bot does not instantly snap to the new
direction. It turns gradually.

That makes a simple rule-based system feel more physical.

## The Shape of the AI

The full system can be summarized as a pipeline:

```text
1. Gather nearby objects.
2. Convert each relevant object into a direction.
3. Weight the direction by behavior type and personality.
4. Add attraction and repulsion vectors into one accumulator.
5. Normalize the accumulator into an output direction.
6. Smooth the output direction over time.
```

This is not an AI model in the machine learning sense. There is no training
data, no gradient descent, and no learned policy. But it is still a useful AI
technique for games because it turns many local facts into one believable
action.

The useful part is the abstraction:

```text
Every behavior is a vector.
Every personality is a set of weights.
Every frame produces one movement direction.
```

That makes the system easy to extend. Powerups, hazards, team behavior, map
objectives, and temporary buffs can all be added as new influences. The
commented-out powerup block in the code is an example of that direction: find a
nearby powerup, compute a preference weight, and add another vector.

## Tradeoffs

This approach has limits.

It is reactive. The bot does not plan a route around obstacles. It does not
predict where a human player will be in two seconds. It does not learn from
previous matches.

For Mass Arena, that is acceptable. The bots are there to keep rooms active,
create motion, provide targets, and make low-traffic games playable. They do
not need to be optimal. In fact, optimal bots might make the game worse.

The goal is not perfect intelligence. The goal is useful behavior at the right
complexity level.

## Closing Thoughts

The interesting part of this bot code is not any one rule. It is the way the
rules compose.

A pellet is an attraction. A threat is a repulsion. Prey is a focused
attraction. Similar-sized players are mild repulsion. Wandering is temporary
noise. The arena edge is environmental pressure. Personality changes the
weights.

All of that becomes one output vector.

For a small real-time game, that is a good tradeoff. The behavior is
understandable, cheap to compute, easy to tune, and expressive enough to make
the arena feel alive.

