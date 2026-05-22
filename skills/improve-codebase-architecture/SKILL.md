---
name: improve-codebase-architecture
description: Find deepening opportunities in a codebase. Use when the user wants to improve architecture, find refactoring opportunities, consolidate tightly-coupled modules, or make a codebase more testable and AI-navigable. Will read `CONTEXT.md` and `docs/adr/` if they exist; creates `CONTEXT.md` lazily otherwise.
---

# Improve Codebase Architecture

Surface architectural friction and propose **deepening opportunities**, refactors that turn shallow modules into deep ones. The aim is testability and AI-navigability.

## Glossary

Use the vocabulary in [LANGUAGE.md](LANGUAGE.md) exactly in every suggestion — read it before producing candidates. Consistent language is the point; don't drift into "component," "service," "API," or "boundary."

Key principles (see [LANGUAGE.md](LANGUAGE.md) for definitions):

- **Deletion test**: imagine deleting the module. If complexity vanishes, it was a pass-through. If complexity reappears across N callers, it was earning its keep.
- **The interface is the test surface.**
- **One adapter = hypothetical seam. Two adapters = real seam.**

This skill is _informed_ by the project's domain model. The domain language gives names to good seams; ADRs record decisions the skill should not re-litigate.

## Process

### 1. Explore

Read the project's domain glossary and any ADRs in the area you're touching first.

Then dispatch an exploration sub-agent (use the Task/Agent tool with whatever exploration agent type is available, or use Grep/Glob/Read directly if no specialized agent exists) to walk the codebase. Don't follow rigid heuristics, explore organically and note where you experience friction:

- Where does understanding one concept require bouncing between many small modules?
- Where are modules **shallow**, interface nearly as complex as the implementation?
- Where have pure functions been extracted just for testability, but the real bugs hide in how they're called (no **locality**)?
- Where do tightly-coupled modules leak across their seams?
- Which parts of the codebase are untested, or hard to test through their current interface?

Apply the **deletion test** to anything you suspect is shallow: would deleting it concentrate complexity, or just move it? A "yes, concentrates" is the signal you want.

### 2. Present candidates

Present a numbered list of deepening opportunities. For each candidate:

- **Files**, which files/modules are involved
- **Problem**, why the current architecture is causing friction
- **Solution**, plain English description of what would change
- **Benefits**, explained in terms of locality and leverage, and also in how tests would improve

**Use CONTEXT.md vocabulary for the domain, and [LANGUAGE.md](LANGUAGE.md) vocabulary for the architecture.** If `CONTEXT.md` defines "Order," talk about "the Order intake module", not "the FooBarHandler," and not "the Order service."

**ADR conflicts**: if a candidate contradicts an existing ADR, only surface it when the friction is real enough to warrant revisiting the ADR. Mark it clearly (e.g. _"contradicts ADR-0007, but worth reopening because…"_). Don't list every theoretical refactor an ADR forbids.

Do NOT propose interfaces yet. Ask the user: "Which of these would you like to explore?"

### 3. Grilling loop

Once the user picks a candidate, drop into a grilling conversation. Walk the design tree with them, constraints, dependencies, the shape of the deepened module, what sits behind the seam, what tests survive.

**Exit when** the user picks a final design or explicitly asks you to start implementing. Don't loop indefinitely.

Side effects happen inline as decisions crystallize:

- **Naming a deepened module after a concept not in `CONTEXT.md`?** Add the term to `CONTEXT.md` (the project's domain glossary, a simple list of `Term, one-sentence definition`). Create the file lazily at the project root if it doesn't exist.
- **Sharpening a fuzzy term during the conversation?** Update `CONTEXT.md` right there.
- **User rejects the candidate with a load-bearing reason?** Offer an ADR, framed as: _"Want me to record this as an ADR so future architecture reviews don't re-suggest it?"_ Only offer when the reason would actually be needed by a future explorer to avoid re-suggesting the same thing, skip ephemeral reasons ("not worth it right now") and self-evident ones. Write it to `docs/adr/NNNN-short-title.md` with: Context, Decision, Consequences. Number sequentially.
- **Want to explore alternative interfaces for the deepened module?** See [INTERFACE-DESIGN.md](INTERFACE-DESIGN.md).
