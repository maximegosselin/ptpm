> Imagine a web application where multiple people can collaborate simultaneously on designing a diagram.
> The application allows drawing shapes, moving them, connecting them, and writing text. What are the
> potential problems, and what are some solution approaches to address them?

This is a classic real-time collaboration problem with several interesting dimensions. Here are the main
challenges and solution approaches.

---

## Consistency of shared state

**The central problem:** two users modify the same element at the same time. Who wins?

**Approaches:**

- **Operational Transformation (OT)** - the historical approach (Google Docs). Complex to implement correctly.
- **CRDTs (Conflict-free Replicated Data Types)** - data structures that merge automatically without conflicts.
  Better suited to tree-like structures such as a diagram. Libraries: Yjs, Automerge.
- **Optimistic locking** - each operation carries a version number. The server rejects conflicting operations
  and the client rebases.
- **Temporary ownership** - as soon as a user selects a shape, it is "locked" for others (the simplest
  approach, but blocking).

---

## Latency and perception of fluidity

**The problem:** network operations introduce a visible delay.

**Approaches:**

- **Optimistic UI** - apply the operation locally immediately, then reconcile with the server response.
- **Local prediction** - display the movement in progress without waiting for confirmation.
- **Throttling/debouncing** - don't send every pixel of movement, but at intervals (e.g. 16ms for 60fps).

---

## Data model and operation granularity

**The problem:** how to represent changes in a way that makes them applicable in any order?

**Approaches:**

- **Immutable event log** - store operations (`MoveShape`, `ResizeShape`, `AddConnector`) rather than the
  final state. Each operation is idempotent and replayable. (You know this well.)
- **Fine granularity** - separate `position`, `style`, `text` as independent properties to reduce conflicts.
- **Vector clocks / Lamport timestamps** - ordering events without a centralized clock.

---

## Network transport

**The problem:** what mechanism to use for real-time communication?

**Approaches:**

- **WebSockets** - persistent bidirectional connection. Standard for this type of app.
- **WebRTC (data channels)** - peer-to-peer, reduces latency and server load, but complicates connection
  management.
- **Server-Sent Events** - unidirectional (server to client), sufficient if writes go through classic HTTP.

---

## Reconnection and offline state

**The problem:** a user loses their connection for 30 seconds.

**Approaches:**

- **Delta sync** - on reconnection, the client sends its last known `sequence_id` and receives only the
  missing operations.
- **Snapshot + log** - keep a periodic snapshot + the operation log since the snapshot to quickly reconstruct
  the state.
- **Local queue** - operations made offline are queued and submitted on reconnection (with conflict risk to
  handle).

---

## Presence and awareness of others

**The problem:** knowing who is there and what they are doing, without it being intrusive.

**Approaches:**

- **Other users' cursors** - display position in real time with a tolerance delay.
- **Awareness channels** - separate the "presence" channel (lightweight, ephemeral) from the "data" channel
  (persisted).
- **Visual locks** - color shapes selected by someone else.

---

## Server scalability

**The problem:** multiple active rooms at the same time, on multiple server instances.

**Approaches:**

- **Isolated rooms** - each diagram is an independent room with its own state.
- **Shared pub/sub** - Redis pub/sub or similar to synchronize server instances.
- **Sticky sessions** - all clients in a room go to the same server (simpler, less resilient).

---

## Complexity summary

| Approach             | Complexity  | Robustness |
|----------------------|-------------|------------|
| Optimistic lock      | Low         | Low        |
| Temporary ownership  | Low         | Medium     |
| OT                   | Very high   | High       |
| CRDT (Yjs)           | Medium      | High       |
| Event log + rebase   | Medium      | High       |

For a first prototype, **temporary ownership + WebSockets + event log** is a good starting point. For a
serious production app, **Yjs with WebSocket providers** is probably the shortest path to something robust.

What level of complexity / use case are you targeting?

> Now read this article https://maximegosselin.com/posts/pull-the-plug-modeling/ and reconsider the question I
> asked you. What changes in the way you think?

Good question. Reading the article, I realize my previous answer was entirely in the technical register.
I jumped straight to CRDTs, WebSockets, and vector clocks without ever asking what the domain is really
trying to do.

Here is what PTPM changes in the way I think.

---

## What I didn't do

I treated the problem as a distributed engineering problem. But the real question should have been: **how do
people collaborate on a diagram in a world without electricity?**

The answer is obvious: they don't do it simultaneously. They pass the sheet around.

---

## What the paper world reveals

**A document belongs to someone.**
A sheet is on someone's table. For someone else to touch it, there is an explicit handover. This is not a
technical detail, it is the nature of collaborative work on a physical artifact.

What this implies for the domain: **ownership** is not a locking mechanism to implement. It is the reality
of the domain. The "lock" I presented as the simplest solution is actually the most faithful solution to
the domain.

**Corrections leave a trace.**
Nobody erases. You cross out, add, correct. The history is visible on the document itself.

What this implies: an immutable event log is not just a good technical practice. It is the natural behavior
of the domain. Modifying a shape "in place" without leaving a trace is a digital abstraction that moves away
from the real domain.

**People work asynchronously.**
In a paper world, two people do not modify the same sheet at the same time. The real question is not "how
to handle simultaneous conflicts" but "is simultaneity really what the domain requires?"

What this implies: I assumed real-time collaboration was the central constraint. Perhaps the real need is to
be able to see what others are doing, not necessarily to act on the same object at the same instant.

---

## What this changes concretely

My first answer framed the problem as: *"how to reconcile concurrent states?"*

After PTPM, the question becomes: *"how to transfer responsibility for an artifact between people, in an
explicit and traceable way?"*

These are two very different problems. The first leads toward OT, CRDTs, and considerable complexity. The
second leads toward an explicit handoff model, operations named according to the business vocabulary, and a
visible history, which is much simpler and probably more correct.

---

## What survives from my first answer

Latency and visual feedback (cursors, presence) remain relevant, because they don't exist in a paper world
but are a real added value of digital. This is where we can intentionally go beyond the physical domain,
not by technical reflex.

---

In summary: PTPM would have prevented me from drowning in accidental complexity before even understanding
the essential complexity.

> And by applying PTPM, how would you model several people collaborating together on the same diagram?

Let's do the exercise seriously.

---

## Pull the plug

We cut the electricity. We have a large table, paper, pencils, and several people who want to design a
diagram together.

---

## What we observe

**The sheet is at the center of the table.**
Everyone can see it. But only one person can write on it at a time, otherwise it's chaos. Naturally,
people organize themselves without being asked.

**Someone picks up the pencil.**
It's an explicit and visible act. Others see that the sheet is "taken." They wait, or they talk, or they
point to suggest.

**The pencil gets passed.**
"Here, it's your turn." The handover is intentional. Nobody grabs it. And when someone puts the pencil down
without passing it, others know it's available.

**Mistakes get crossed out.**
You don't tear up the sheet to start over. You cross out, annotate, add an arrow. The history of decisions
remains visible on the paper itself.

**People comment out loud.**
"Put the box more to the left." "No, connect it to this one instead." The diagram is a shared artifact, but
the conversation around it is just as real as the drawing.

**Nobody works in silence in their corner.**
If someone wants to propose a variant, they take another sheet and place it next to it. "Look, would this
be better?"

---

## What this reveals as domain events

- The pencil was picked up by someone
- A shape was drawn
- A shape was moved (crossed out and redrawn)
- A connection was drawn between two shapes
- Text was added
- A correction was made (with visible trace)
- The pencil was passed to someone else
- The pencil was set down (available)
- A variant was proposed on a separate sheet
- The variant was adopted (the main sheet is replaced) or rejected

---

## What this reveals as domain rules

1. **Only one person holds the pencil at a time.** This is not a technical constraint, it is the nature of
   the artifact.
2. **Everyone sees the sheet at all times.** Visibility is passive and universal.
3. **Suggestions come from conversation, not from the pencil.** You can influence the drawing without holding
   the pencil.
4. **Corrections do not destroy history.** You add, cross out, but the trace remains.
5. **A variant is a separate artifact**, not a modification of the original.
6. **Transfer of the pencil is explicit and voluntary.** It cannot be stolen.

---

## What this suggests as a model

The domain is not "real-time collaborative editing." The domain is:

> **A group of people building a shared artifact, passing the responsibility for writing explicitly, under
the watchful eye of all.**

This changes everything. The central model is not a concurrent canvas. It is an **artifact with an active
holder** surrounded by **observers who can suggest**.

The two distinct roles that emerge naturally:

- **The pencil holder** - they act on the diagram
- **The observers** - they see in real time, they suggest, they discuss

And the central mechanic is not conflict resolution. It is **responsibility transfer**.

---

This is where I would stop before touching anything technical. Does this model match what you envision as
the user experience?

> This naturally leads us to ask the following question: to make a modification to the diagram, do we need
> to know what others are thinking at that moment?

No. And it's an interesting trick question.

In the paper world, when you hold the pencil, you don't have access to others' thoughts. You see their
faces, you hear their voices if they speak. But their unexpressed intention is invisible to you, and that
doesn't stop you from drawing.

What matters is the **state of the sheet**, not the mental state of the other participants.

---

## What this reveals

The question of real-time cursors, which I had presented as a "presence" feature, is actually an attempt
to make visible something that doesn't exist in the domain: the unexpressed intention of others.

In the paper world, you don't need to know where others' eyes are looking to do your work. You need to
know:

1. **Is the pencil available?**
2. **What does the sheet look like now?**

That's it. The rest is noise.

---

## The concrete implication

Other users' cursors, "X is currently editing..." indicators, real-time highlighted zones... these are
solutions to a problem the domain doesn't pose.

The real visibility need is much simpler: **everyone sees the same sheet, updated when something changes.**

This brings the technical complexity down to a much more reasonable level. No ephemeral state
synchronization down to the millisecond. Just a shared sheet and a pencil that belongs to someone.

> The example of a few people around a table makes sense. But what if we were talking about hundreds of
> people instead? What would change?

Good question. Let's go back to the paper world, but with hundreds of people.

---

## What we observe

A hundred people around a single sheet is physically absurd. Nobody can see, nobody can get close. The
table model breaks down.

What naturally happens in a paper world when a group gets large:

**The sheet gets displayed.** It's pinned to the wall or projected with an overhead projector. It becomes
a reference visible to all, but which nobody touches directly.

**A secretary or facilitator holds the pencil.** They don't decide. They transcribe what the group decides
collectively. The pencil no longer belongs to an ordinary participant, it belongs to a role.

**Speech replaces the pencil.** You don't get up to modify the diagram. You raise your hand, you propose,
the group reacts, the facilitator draws if the proposal is accepted.

**Subgroups work in parallel.** The room is divided. Each group works on its own sheet, on a portion of
the problem. Then the sheets are put together.

---

## What this reveals as a fundamental change

At small scale, the model is: **one active holder, observers.**

At large scale, the model changes completely: **a displayed reference artifact, collective governance to
modify it, and decomposition into sub-artifacts.**

These are not two variants of the same model. They are two different domains.

---

## What this suggests

Before designing anything, the real question to ask is: **how many people actively collaborate on the same
diagram at the same time, in a real case?**

Because if the honest answer is "3 to 5", the passing-pencil model holds perfectly. If the answer is
"hundreds", we're probably no longer in active collaboration, we're in **consultation** or **broadcast**,
which is a very different domain with very different needs.

PTPM prevents us from building a large-scale solution to a problem that arises at small scale.
