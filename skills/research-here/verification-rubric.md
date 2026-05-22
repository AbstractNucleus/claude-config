# Verification Rubric & Failure Handling

Companion reference for `/research-here`. Used by the Verifier and Synthesizer agents.

## Verdict rubric

| Dimension | OK | WEAK | MISMATCH | DEAD | HALLUCINATED |
|---|---|---|---|---|---|
| URL resolves (HTTP 2xx/3xx) | ✅ | ✅ | ✅ | ❌ | ✅ |
| Page content loads (not login/paywall/JS-shell) | ✅ | ✅ | ✅ | ❌ | ✅ |
| Page topic matches citation context | ✅ | ✅ | ✅ |, | ❌ |
| Page content supports the claim | ✅ | partial / tangential | ❌ contradicts |, |, |
| Authority tag matches reality | ✅ | may be inflated |, |, |, |

## Authority ladder

- **primary**, official spec, maintainer docs, first-party paper, RFC, standards body
- **secondary**, reputable tech press, textbook, conference talk by a practitioner
- **tertiary**, blog post, forum answer, marketing page, tweet

## Publishable threshold

Every claim in the synthesis body must satisfy ONE of:
- ≥1 OK verdict from ≥1 source at `secondary` or better, OR
- ≥2 OK verdicts across any authority tier

Claims that don't meet the threshold move to Open questions or are dropped.

## Per-claim failure matrix

| Verifier status | Claim has OTHER OK sources | This is the ONLY source |
|---|---|---|
| OK | keep as-is | keep as-is |
| WEAK | keep; downgrade inline confidence (cite as `([weak source](URL))` in local mode) | move claim to Open questions OR trigger re-research (deep mode only) |
| MISMATCH | drop this citation; keep claim if threshold still met | drop the claim entirely |
| DEAD | keep claim, drop the URL, note in Gaps | trigger re-research (deep mode only); if still no source → drop claim |
| HALLUCINATED | drop citation + flag researcher in run log | drop the claim entirely + flag researcher in run log |

## Skeptic challenge handling (deep mode only)

| Severity | Re-research? | If RESOLVED | If STILL-WEAK | If CONFIRMED-COUNTER |
|---|---|---|---|---|
| high | yes (block synthesis until resolved) | keep strengthened claim | move to Open questions | retract claim + note counter |
| medium | yes, in parallel with synthesis scaffolding | merge new sources | demote to Open questions | retract |
| low | optional | keep | leave as-is with inline caveat | retract |

## Global abort conditions

Do NOT synthesize. Report to the user instead.

- More than 40% of URLs across all angles are DEAD or HALLUCINATED → probably a flaky researcher run. Suggest rerunning that angle with a narrower question.
- Skeptic returns `overall_assessment: insufficient-evidence` for every angle → topic may be too new, too niche, or too proprietary for web sources. Suggest narrowing or switching to a document-first approach.

## Graceful partial-failure behavior

Non-aborting degradation path:

1. If 1–2 angles fail verification hard (most URLs DEAD/HALLUCINATED/MISMATCH), synthesize from the surviving angles AND explicitly include a section `## Angles we could not verify` listing what was attempted and why it failed. Never silently drop.
2. If skeptic says `needs-rework` on an angle, run re-research, then synthesize regardless of re-research outcome, tag unresolved challenges in Open questions.
3. If a single high-value URL is DEAD but the claim has alternate OK sources, keep the claim, drop the dead URL, and note the dead link in the run's verification.json audit file. Future-you may want to find a replacement link.
