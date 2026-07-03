---
layout: post
title:  "Not Everything Needs 60 FPS: WebSocket Update Rates in MassArena"
date:   2026-07-03 09:00:00 -0400
categories: projects web systems
description: "MassArena runs its server simulation at 60 Hz, but broadcasts world state, pellets, and leaderboard data at different rates. This article explains why simulation frequency and WebSocket update frequency are separate design choices."
---

# Introduction

When building a real-time game over WebSockets, a simple first
implementation may look something like this:

{% highlight text %}
Run the simulation.
Serialize everything.
Send everything to every client.
Repeat 60 times per second.
{% endhighlight %}

This is an attractive approach because there is only one clock to think about.
The game runs at 60 Hz, therefore the network sends at 60 Hz.

However, this is not always necessary.

In MassArena, the server simulation runs at 60 Hz. Client input is also sent at
60 Hz. But not every piece of state is broadcast to the browser at 60 Hz. World
state is sent at 30 Hz, pellets are sent at 5 Hz, and the leaderboard is sent at
2 Hz.

The important distinction is:

{% highlight text %}
Simulation frequency is not the same thing as network update frequency.
{% endhighlight %}

This article is about that distinction.

## The Details

### 1. The server still simulates at 60 Hz

MassArena is server authoritative. The client does not decide that it absorbed
another player. The client does not decide that a pellet was collected. The
client sends input intent, and the server decides what happened.

In the browser, the player moves the mouse, touches the screen, or presses dash.
The client sends a direction vector to the server. The server then advances the
world. This includes movement, collisions, pellet pickup, powerups, mass decay,
absorption, and win conditions.

A simplified version of the server loop is:

{% highlight typescript %}
setInterval(() => {
  const now = Date.now();
  for (const room of rooms.values()) {
    tickRoom(room, now, 1 / SIMULATION_HZ);
  }
}, 1000 / SIMULATION_HZ);
{% endhighlight %}

where `SIMULATION_HZ` is 60.

Why does this matter? Movement and collision rules are easier to reason about
when the server has a predictable tick rate. Dash timing is based on time.
Absorption depends on relative size and position. Bots also need to make
movement decisions against the current room state.

The simulation should be steady because it defines the actual game.

### 2. The network does not need one update rate

Even though the simulation runs at 60 Hz, there is no rule that says every
message over the WebSocket must be sent at 60 Hz.

The current frequencies are:


| Data type | Frequency | Reason |
| --- | ---: | --- |
| Simulation | 60 Hz | Keeps movement and collision logic consistent |
| Client input | 60 Hz | Captures player intent frequently |
| World snapshots | 30 Hz | Keeps visible movement responsive |
| Pellet snapshots | 5 Hz | High-cardinality state; small staleness is acceptable |
| Leaderboard | 2 Hz | Useful UI, but not gameplay-critical |

The server room stores separate timestamps for each kind of broadcast. The
general shape is:

{% highlight typescript %}
const now = Date.now();

tickRoom(room, now, dt);

if (now - room.lastBroadcastAt >= WORLD_INTERVAL_MS) {
  broadcastWorld(room);
}

if (now - room.lastPelletBroadcastAt >= PELLET_INTERVAL_MS) {
  broadcastPellets(room);
}

if (now - room.lastLeaderboardAt >= LEADERBOARD_INTERVAL_MS) {
  broadcastLeaderboard(room);
}
{% endhighlight %}

The exact code has more details, but the idea is simple. Each class of data has
its own freshness requirement.

### 3. World snapshots are more important

World snapshots are sent at 30 Hz. These snapshots include players, the arena,
powerups, the current tick, countdown information, and absorption hints.

This is the most latency-sensitive state. If another player is moving toward
you, a stale position feels like lag. If you are chasing a smaller player, the
position of that player is important. Players react to threats and opportunities
in real time.

So why not 60 Hz?

For this project, 30 Hz is a reasonable compromise. It is still frequent, and
the browser render loop runs independently. The canvas can redraw at the
display refresh rate using the newest world snapshot it has received.

Client-side interpolation would improve this further. I have not added that
yet, but the current split leaves room for it.

### 4. Pellets are allowed to be stale

Pellets are a different kind of state. There are around 1000 pellets in the
arena. Sending that list too often is wasteful.

The server is still authoritative. If a player eats a pellet, the server updates
the room immediately and respawns the pellet. But the client does not need to
receive the entire pellet view every simulation tick.

Pellet snapshots are sent at 5 Hz. The server also filters the pellet list for
each entity:

{% highlight typescript %}
function pelletsNear(room: Room, entity: Entity) {
  const maxDistanceSquared =
    (arenaMaxDimension(room.arena) * PELLET_SNAPSHOT_RADIUS_RATIO) ** 2;

  return room.pellets.filter((pellet) =>
    distanceSquared(entity.x, entity.y, pellet.x, pellet.y) <= maxDistanceSquared
  );
}
{% endhighlight %}

This is not a complicated interest management system. It is just a practical
filter. But it is already better than sending all pellets to every socket at 60
Hz.

A stale player position feels like lag. A slightly stale pellet often just
feels like nothing. Maybe a pellet appears a fraction of a second late. Maybe
one disappears slightly after the server has already consumed it.

That is not perfect, but it is acceptable for this kind of game.

### 5. Leaderboards can be even slower

The live leaderboard is sent at 2 Hz.

The leaderboard is important feedback. It tells players where they stand, who
is getting large, and whether they are climbing. But it is not gameplay logic.
If the displayed rank is a little behind reality, the game still works.
Movement still works. Collision still works. Absorption still works.

The server builds the leaderboard from alive entities, sorts by mass, and sends
the top entries. Twice per second is enough for this UI to feel current without
making it high-frequency traffic.

### 6. General lesson

The general lesson is that different data has different freshness requirements.
Treating all state equally is simple, but it can waste bandwidth and
serialization work.

This applies outside of games too. Collaborative editors, dashboards, presence
systems, stock or order-book UIs, monitoring tools, and other multiplayer apps
all have state that users react to immediately, and state that can be slightly
delayed.

The mistake is assuming there is one correct update rate for the whole system.
Some data is part of the user's immediate control loop. Some data is just
context.

MassArena is a small browser game, not a large distributed system. But the
tradeoff is real. The server can run a precise simulation without broadcasting
everything at simulation speed.

There are still obvious improvements. Client-side interpolation would make
movement smoother. Delta-compressed snapshots would avoid resending unchanged
data. A binary message format could reduce JSON overhead. Better interest
management could send each player only the entities near them. Load testing
would also be useful before making any real scaling claims.

## Thanks!

Thanks for reading!
