---
layout: post
title:  "Building Mass Arena"
date:   2026-06-06 09:00:00 -0400
categories: projects web systems
---

# Introduction

I recently built [Mass Arena](https://mass-arena.com), a no-login
real-time multiplayer browser game. The idea is simple. You pick a
nickname, join an arena, eat pellets, gain mass, absorb smaller players,
avoid bigger players, and try to climb the leaderboard.

This was not meant to be a huge game project. I wanted to build a small
web application that was real enough to deploy, test, and learn from. It
has a frontend, a backend, WebSockets, persistence, a domain name, HTTPS,
analytics, and a deployment story.

That is a lot more interesting than another local-only project that only
runs on my laptop.

The application is written in TypeScript. The frontend is React and
canvas. The backend is Node, Fastify, and WebSockets. The game state lives
in memory. PostgreSQL stores durable data like completed runs,
leaderboards, challenge links, and analytics events. The app is deployed
to an Azure VM behind Caddy.

This article is about the design of the project. It is not meant to be a
line-by-line tutorial, but I will walk through the main decisions.

## The Details

### 1. The product loop

Before talking about the code, it is worth talking about the loop.

The intended flow is:

{% highlight text %}
land -> enter nickname -> play -> die or win -> leaderboard -> respawn -> challenge a friend
{% endhighlight %}

Why does this matter? Because a game like this is not just the simulation.
The game loop also includes the screens around the simulation. The death
screen matters. The leaderboard matters. The ability to immediately
respawn matters.

I did not want the first version to require an account. Accounts are
useful later, but for this kind of project they also add friction. If a
person opens the site and has to create a password before trying the game,
a lot of them will simply leave.

So the constraints were roughly:

{% highlight text %}
no login
simple nickname
quickplay button
works on desktop and mobile
bots if the room is empty
persisted scores
daily/weekly leaderboard
shareable challenge links
{% endhighlight %}

Some of those are product decisions, but they also shape the architecture.
For example, if the game needs bots, then the server needs to simulate
non-human players. If the game needs leaderboards, then completed runs need
to be persisted somewhere. If the game needs challenge links, then a run
needs to be turned into a durable target another user can attempt to beat.

### 2. High-level architecture

The architecture is intentionally straightforward.

{% highlight text %}
Browser
React UI
Canvas rendering
Mouse/touch input

```
| HTTP
v
```

Fastify API
quickplay
leaderboards
challenges
health

```
| WebSocket
v
```

Node game server
in-memory rooms
server-authoritative simulation
bots
pellets
collisions
live leaderboard broadcasts

```
| queued writes
v
```

PostgreSQL
anonymous players
completed runs
challenges
events
{% endhighlight %}

Caddy sits in front of the Node server and handles HTTPS and WebSocket
proxying.

The browser talks to the backend in two ways. Normal HTTP is used for
things like creating or joining a game. WebSockets are used for the actual
real-time gameplay.

The most important boundary in the system is this:

{% highlight text %}
Live gameplay state stays in memory.
Durable product data goes to PostgreSQL.
{% endhighlight %}

Positions, velocities, inputs, pellets, and per-frame state do not belong
in the database. The database should not be involved in every movement
update. That would make the system slower and much more complicated.

The database stores the things I care about after something meaningful
happens: a completed run, a challenge link, a leaderboard entry, or a
coarse analytics event.

### 3. Server-authoritative gameplay

For the gameplay itself, I wanted the server to own the truth.

The client should not tell the server, "I am now at this position and I
absorbed this player." The client should send input intent. The server
should decide what actually happened.

Conceptually, the client sends something like:

{% highlight text %}
I want to move in this direction.
{% endhighlight %}

The server handles movement, collision detection, mass changes, death,
wins, and leaderboard state.

A simplified game loop looks like this:

{% highlight typescript %}
setInterval(() => {
    for (const room of rooms.values()) {
        updateBots(room);
        updateMovement(room);
        handlePellets(room);
        handleAbsorptions(room);
        maybeTriggerDomination(room);
        broadcastWorldState(room);
    }
}, TICK_MS);
{% endhighlight %}

This is simplified, but it captures the shape of the system.

Why does this matter? Because it keeps the rules in one place. It also
makes the leaderboard more trustworthy. If the browser is responsible for
awarding itself mass, then the leaderboard is not worth much.

The frontend still has plenty to do. It renders the canvas, handles mouse
and touch input, shows the landing screen, death screen, leaderboard, and
challenge UI. But the game rules belong on the server.

### 4. WebSockets

A normal web application can often get away with request/response HTTP.
This kind of game cannot.

Once the player is in the arena, the connection needs to stay open. The
client sends input. The server sends back world snapshots, pellet state,
leaderboard updates, and death/win events.

The protocol is deliberately small. Client messages are things like:

{% highlight text %}
join
input
respawn
{% endhighlight %}

Server messages are things like:

{% highlight text %}
world state
pellet state
leaderboard
death
respawned
room domination
{% endhighlight %}

The exact message names are not the important part. The important part is
the split in responsibility:

{% highlight text %}
The client sends intent.
The server sends truth.
{% endhighlight %}

That is a useful rule for this project.

### 5. Rooms and bots

One issue with multiplayer games is that they feel dead without
players.

If someone opens the game and sees an empty arena, they probably leave.
For that reason, Mass Arena uses bots to keep rooms playable at low
traffic.

This is not really an AI project. The bots are there for product reasons.
They give the player something to chase and something to avoid. They make
the arena feel alive even if there are only one or two real users online.

There is a tradeoff here. Bots should make the game more fun, but they
should not dominate the whole experience. They also should not pollute the
main persisted leaderboard as if they were real users.

Rooms live in memory. For the first version, one server can own many
rooms. If the app ever needs to scale horizontally, the natural model is:

{% highlight text %}
one room lives on one server
{% endhighlight %}

Splitting a single real-time room across multiple servers is a much harder
problem. I do not need that for an MVP.

### 6. Snowballing and domination

A game like this has a simple failure mode. One player gets huge, and then
the room becomes boring. New players spawn in, but the leader is so large
that the game is no longer interesting.

I could try to solve this with only balancing. For example, larger players
can move slower. Bots can be tuned. Spawn protection can help. Those are
all useful.

But there is a more direct product solution too: treat domination as a win
condition.

Instead of letting one huge player make the room stale forever, the server
can detect when a player has effectively dominated the arena. That player
gets a win, the active runs are completed, and the room resets after a
short countdown.

Conceptually, a completed run has outcome information like:

{% highlight text %}
ended reason
win flag
win type
win timestamp
duration
peak mass
final mass
{% endhighlight %}

This lets the application distinguish between different outcomes:

{% highlight text %}
absorbed by another player
disconnected
room reset
domination win
{% endhighlight %}

Why does this matter? Because it turns a broken state into a satisfying
ending. The huge player gets a victory, and everyone else gets a fresh
room.

### 7. Persistence

The database is intentionally not the live game engine.

The first version of the database has a few basic concepts:

{% highlight text %}
anonymous players
completed runs
challenges
events
{% endhighlight %}

Anonymous players give the browser a stable identity without forcing an
account system.

Completed runs store things like nickname, start time, end time, peak
mass, final mass, duration, rank, and outcome.

Challenges store "beat my score" links. A challenge has a target mass, and
then I can track visits, play starts, and completions.

Events store coarse analytics data.

What do I not store?

{% highlight text %}
every position
every input
every pellet eaten
every frame
every collision detail
{% endhighlight %}

That data would be noisy and expensive, and it would not answer the
questions I care about right now.

The questions I care about are more like:

{% highlight text %}
Did people click Play?
Did they actually spawn?
How long did they survive?
Did they respawn?
Did they view the leaderboard?
Did they copy challenge links?
Did challenge links bring in more players?
{% endhighlight %}

That is the level of persistence that makes sense for this stage.

### 8. Asynchronous database writes

The game should not freeze because the database is slow for a moment.

So database writes are queued. A completed run is important. An analytics
event is useful, but less important. If the system is under pressure,
gameplay should win.

A simplified version of the idea looks like this:

{% highlight typescript %}
type DbJob =
| { type: "event"; eventName: string; propertiesJson: string }
| { type: "run"; run: CompletedRun };

const queue: DbJob[] = [];

export function enqueueCompletedRun(run: CompletedRun) {
queue.unshift({ type: "run", run });
scheduleDrain();
}

export function enqueueAnalyticsEvent(eventName: string, properties: object) {
    queue.push({
        type: "event",
        eventName,
        propertiesJson: JSON.stringify(properties)
    });

    scheduleDrain();
}
{% endhighlight %}

The real implementation has more guardrails, but the principle is simple.
Do not put the real-time game loop on the critical path of a database
write.

This is a tradeoff. If I were writing a banking system, dropping an event
would be unacceptable. For a browser game MVP, it is reasonable to
prioritize live gameplay over low-value telemetry.

### 9. Leaderboards and challenge links

The leaderboard exists to give players a reason to replay.

An all-time leaderboard can become discouraging. Eventually someone has an
unreachable score. Daily and weekly leaderboards are more approachable.
They reset the mountain.

The main leaderboard is peak mass. Wins are tracked separately. This keeps
the meaning clean:

{% highlight text %}
Who got biggest?
Who won the most?
{% endhighlight %}

Those are related, but they are not exactly the same question.

Challenge links are the sharing loop. After a death or a win, a player can
create a link that says, in effect:

{% highlight text %}
Beat this score.
{% endhighlight %}

That is much better than a generic "share this game" button. It gives the
other person a concrete goal.

It also gives me a useful funnel:

{% highlight text %}
challenge created
challenge copied
challenge viewed
challenge play started
challenge completed
{% endhighlight %}

If the game ever gets organic traffic, I expect this loop to matter more
than almost anything else in the app.

### 10. Deployment

The deployment is boring on purpose.

The app runs in Docker Compose on an Azure VM. Caddy sits in front of the
app and handles HTTPS. PostgreSQL is an Azure Flexible Server with private
network access only.

The infrastructure is split conceptually into two parts:

{% highlight text %}
App infrastructure:
VM
public IP
network interface
network security rules

Data infrastructure:
virtual network
app subnet
delegated PostgreSQL subnet
private DNS
PostgreSQL Flexible Server
database
Key Vault secret for the database password
{% endhighlight %}

The app infrastructure is disposable. If I destroy the VM and recreate it,
the new VM can connect back to the existing private database. The data
infrastructure is the durable part.

The deploy script does the usual operational work:

{% highlight text %}
create or update resource groups
create or reuse the database password
deploy network and database infrastructure
deploy VM infrastructure
install Docker and rsync
sync code to the VM
write production environment variables
run migrations from the VM
start the app with Docker Compose
write a Caddyfile for the domain
{% endhighlight %}

One detail I care about: the PostgreSQL server is private-only. There is
no public database endpoint exposed. Migrations run from the VM because
the VM is inside the network that can reach the database.

A typical deploy command has this shape:

{% highlight sh %}
./deploy/azure/deploy.sh  <app-resource-group>  <data-resource-group>  <region>  <app-name>  <vm-size>  <admin-user>  <domain>
{% endhighlight %}

This is not Kubernetes. That is intentional. Kubernetes would add a lot of
machinery before I know whether the app needs it. A single VM is easy to
understand, easy to SSH into, and good enough for launch.

### 11. Smoke testing

I added a smoke test that simulates multiple clients.

The test calls the quickplay API, opens WebSocket connections, sends
movement input, receives world updates, and fails if clients do not receive
world state messages or if socket errors occur.

The command has this shape:

{% highlight sh %}
./scripts/smoketest.sh <url> <clients> <duration-ms>
{% endhighlight %}

The output includes things like:

{% highlight text %}
clients requested
clients receiving world state
rooms observed
world state messages
pellet state messages
leaderboard messages
socket errors
{% endhighlight %}

I also have a database smoke test that connects through the VM and counts
rows in the main tables.

This is not a full load test. It does not prove the system can handle
massive traffic. But it does catch the boring mistakes: broken deployment,
broken WebSocket routing, broken database connectivity, or migrations that
did not run.

That is valuable.

### 12. Analytics

I split analytics into two categories.

Google Analytics is for acquisition and high-level funnel metrics:

{% highlight text %}
landing viewed
play clicked
player spawned
player died
respawn clicked
challenge copied
challenge landing viewed
leaderboard viewed
{% endhighlight %}

PostgreSQL events are for more detailed product and gameplay analysis.

I do not want Google Analytics to become the gameplay database. I also do
not want to send raw nicknames, player IDs, challenge IDs, room IDs, or
high-cardinality values into GA.

For example, instead of sending exact mass values to GA, the frontend can
send buckets:

{% highlight text %}
0_99
100_249
250_499
500_999
1000_2499
2500_4999
5000_plus
{% endhighlight %}

That is enough to understand whether players are dying immediately or
surviving long enough to care.

### 13. Scaling later

The current architecture is a single VM. That is fine for launch.

The scaling rule I want to preserve is:

{% highlight text %}
One room lives on one server.
{% endhighlight %}

If the game needs more capacity later, I can add more game servers and
assign new rooms to different servers. Quickplay can return a WebSocket
URL, so the client does not need to know whether there is one server or
many.

Before doing that, there are simpler options:

{% highlight text %}
lower broadcast rate
send smaller payloads
drop slow WebSocket clients
move to a larger VM
optimize serialization
{% endhighlight %}

I do not want to build a distributed game fleet before I know whether the
game deserves one.

### 14. What I would build next

The next work is mostly product polish, not infrastructure.

Some candidates:

{% highlight text %}
better mobile controls
alternate maps, such as a donut arena or barrier map
better bot behavior
better challenge landing page
more leaderboard polish
basic analytics dashboard
cosmetics if people actually return
multi-server room routing only after real concurrency requires it
{% endhighlight %}

The hard part is not deploying a VM. The hard part is whether strangers
understand the game, respawn after dying, share challenge links, and come
back later.

## Thanks!

A small game still requires real architecture.

The game loop is simple, but the surrounding system still needs decisions
about trust, persistence, deployment, analytics, testing, and product
flow.

A few things I took away from this project:

1. Server-authoritative state keeps the rules understandable.
2. The database should not be in the live gameplay loop.
3. Bots are a practical solution for low-traffic multiplayer rooms.
4. A death screen is not just an ending; it is part of the retention loop.
5. Smoke tests are useful even for small applications.
6. Design scaling escape hatches, but do not overbuild before traffic exists.

Mass Arena is still an MVP, but it is a real deployed system. It has a
domain, HTTPS, WebSockets, persistence, deployment scripts, smoke tests,
and enough product surface to learn from actual usage.

That was the point of the project: build something small enough to ship,
but real enough to teach me something.

Please feel free to message me on [LinkedIn]({{ site.linkedin_url }})
with any additional questions or concerns.

Thanks for reading!
