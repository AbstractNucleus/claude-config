---
name: grill-me
description: Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree before any implementation. Use when user wants to stress-test a plan, get grilled on their design, or mentions "grill me".
---

# Grill Me

Stress-test the user's plan branch-by-branch until every consequential decision is settled. One question at a time, with your recommended answer.

## How to ask

Use the `AskUserQuestion` tool. One question per turn. For each, include your recommended answer as the first option labeled "(Recommended)" and 2-3 alternatives so the user can quickly pick or redirect.

## What to grill on

Prioritize decisions where:
- Reasonable engineers would disagree
- The wrong choice is expensive to reverse (schema, public APIs, dependency choices, security model)

Skip:
- Defaults that don't matter
- Anything you can settle by reading code, docs, or the project's coding standards first

## Information ladder

Before asking, try in order:
1. Look at the code — does it already imply an answer?
2. Read project docs / `CLAUDE.md` / decision records
3. Then ask the user

## Termination

Stop when every branch of the decision tree has a settled answer. Then write a single message summarizing the resolved plan (decisions made + open follow-ups, if any). That summary is the deliverable; the user can hand it to another agent to implement.
