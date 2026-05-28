---
name: to-prd
description: Turn the current conversation context into a PRD and publish it to the project issue tracker. Use when the user invokes /to-prd, says "create a PRD", "PRD this", "write up a product brief", "draft a spec", or similar.
---

This skill takes the current conversation context and codebase understanding and produces a PRD. Don't re-interview the user — synthesize from context, then confirm the module breakdown and test scope in a single pass (see step 2).

If conversation context is thin (no meaningful problem description), stop and ask the user to describe the problem before drafting.

## Tracker detection

Before publishing, detect the project's issue tracker by inspecting the repo (`.github/`, `linear.toml`, `.jira/`, etc.). If unambiguous, proceed and tell the user which one. If ambiguous or none found, ask once. Fallback: write the PRD to a local `docs/prd-<slug>.md` file and tell the user.

## Process

1. Explore the repo to understand the current state of the codebase, if you haven't already. Use the project's domain glossary vocabulary throughout the PRD, and respect any recorded decisions in the area you're touching.

2. Sketch out the major modules you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

Present the proposed module breakdown and test scope to the user in a single confirmation message. Don't interview them step by step — show the whole sketch and let them redirect.

3. Write the PRD using the template below, then publish it to the project issue tracker. Apply the project's triage/intake label if one exists (e.g. `needs-triage`, `triage`, `inbox`); otherwise skip.

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list should be comprehensive within scope — cover all aspects of the feature that context supports. Don't invent stories beyond what the conversation has established.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>
