# Collaborative Diagram Tool

This experiment presents the same problem to Claude twice: once without any modeling guidance, and once
after applying **Pull The Plug Modeling (PTPM)**. The two responses differ significantly.

The full exchange is available in French ([original-french.md](original-french.md)) and in English
([english-transcript.md](english-transcript.md)).

---

## The problem

> Imagine a web application where multiple people can collaborate simultaneously on designing a diagram.
> The application allows drawing shapes, moving them, connecting them, and writing text. What are the
> potential problems, and what are some solution approaches?

---

## First answer: pure engineering

Without PTPM, Claude responded with technical solutions: CRDTs, Operational Transformation,
WebSockets, vector clocks, optimistic UI, delta sync, presence channels. The response did not question
whether those problems were relevant in the first place.

---

## After pulling the plug

After being asked to read about PTPM and reconsider, Claude's response changed. Questions emerged that
were not raised in the first pass:

- How do people collaborate on a diagram in a world without electricity?
- Is simultaneity actually what the domain requires — or just an assumption?
- Does ownership belong to the domain model itself, rather than being a technical locking mechanism?
- How is the responsibility for an artifact transferred between people, explicitly and traceably?
- To modify the diagram, does the editor need to know what others are thinking at that moment?
- Is the pencil available?
- What does the sheet look like right now?
- How many people are actively collaborating on the same diagram at the same time, in a real scenario?
- Is this actually active collaboration — or consultation, or broadcast? Are those the same domain?

---

## Why PTPM was beneficial

**It reframed the problem.** The first answer treated the problem as a distributed systems challenge.
PTPM revealed it was a domain problem first: how do people share responsibility for a shared artifact?

**It exposed false assumptions.** Real-time simultaneous editing was never questioned — it was taken as
a given. PTPM surfaced it as an assumption worth challenging. In the physical world, two people don't
edit the same sheet at the same time. Why should the software require it?

**It reduced the solution space.** Once ownership was recognized as a domain concept rather than a
technical workaround, the need for CRDTs, OT, and conflict resolution was reduced. A shared sheet with
an explicit pencil holder is a simpler model.

**It exposed features that solve non-problems.** Real-time cursors and "X is currently editing..."
indicators are answers to "what are others thinking right now?" — a question the domain never asks. In
the paper world, you only need to know two things: is the pencil available, and what does the sheet look
like now?

**It revealed that scale changes the domain.** Hundreds of people around a single sheet is not
physically viable. At that scale, the model is no longer collaborative editing — it becomes display,
facilitation, and governance. PTPM prevented designing a large-scale solution for a problem that arises
at small scale.

**It avoided accidental complexity before understanding essential complexity.** As Claude put it:
*"PTPM would have prevented me from drowning in accidental complexity before even understanding the
essential complexity."*

---

## Conclusion

The resulting solution will still be technical — it is software, after all. But the technical decisions
will have been shaped by the right questions rather than by assumptions. That is the difference PTPM
makes.
