---
name: council
description: Run a question or idea through the full LLM Council — independent first opinions, anonymized peer review, multi-hat critique (devil's advocate, optimist, feasibility), on-demand research, and neutral chairman synthesis. Use when the user invokes /council, asks for "the council's" opinion, wants an idea stress-tested from multiple angles, or wants to avoid single-model yes-bias.
---

# LLM Council

Runs `args` (the user's question or idea) through the full council process. All stages run via Agent tool calls to the subagents below — no API key, no external server, everything runs through Claude Code's own model-pinned subagents under the current subscription.

If `args` is empty, ask the user what question or idea to put to the council before doing anything else.

## Roster

| Agent | Model | Role |
|---|---|---|
| `council-opus` | Opus | Stage 1 first opinion, Stage 2 peer review |
| `council-sonnet` | Sonnet | Stage 1 first opinion, Stage 2 peer review |
| `council-devils-advocate` | Sonnet | Stage 3 — Black Hat, risk/weakness case (advisory) |
| `council-optimist` | Sonnet | Stage 3 — Yellow Hat, upside case (advisory) |
| `council-feasibility` | Sonnet | Stage 3 — practicality rating (advisory) |
| `council-researcher` | Opus | On-demand only — dispatched when any stage flags `NEEDS RESEARCH:` |
| `council-chairman` | Opus | Stage 4 — neutral synthesis only, never gives a first opinion |

## Stage 1 — First Opinions

Dispatch the question to both `council-opus` and `council-sonnet`, single message, two parallel Agent tool calls, `run_in_background: false` for both (nothing can proceed until both return). Each gets only the raw question — no mention of the other model, no hint that this is a council.

Print both answers under `## Stage 1: First Opinions`, labeled by model name.

## Stage 2 — Peer Review

Anonymize the two Stage 1 answers as **Response A** and **Response B** (randomize which model gets which letter each run). Send each agent the *other* agent's anonymized response and ask it to evaluate accuracy and insight, and state which it finds stronger and why. Two parallel Agent calls, `run_in_background: false`.

Print both raw evaluations under `## Stage 2: Peer Review`, then de-anonymize for the reader so they can follow along — the agents themselves never saw the real identities.

## Stage 3 — Multi-Hat Critique

Send `council-devils-advocate`, `council-optimist`, and `council-feasibility` the same package: the original question plus both Stage 1 answers, labeled by model this time (no need to anonymize — these three are evaluating the idea, not ranking model quality). Three parallel Agent calls, `run_in_background: false`, each blind to the other two's output so they stay independent.

Print all three under `## Stage 3: Multi-Hat Critique`, subheaded by role (Devil's Advocate / Optimist / Feasibility).

**Research escape hatch:** if any Stage 1, 2, or 3 output ends with a line starting exactly `NEEDS RESEARCH:`, collect those questions and dispatch `council-researcher` — one Agent call per distinct question, in parallel if there's more than one — before moving to Stage 4. Print results under `## Research`. If nothing flagged it, skip this stage entirely; don't invoke the researcher speculatively.

## Stage 4 — Chairman Synthesis

Call `council-chairman` with everything gathered so far — the original question, both first opinions (labeled by model), both peer reviews, all three hats' output, and any research briefs — **assembled in randomized order** within the prompt so the chairman can't anchor on position. Attribute each section to its role (it needs to know a claim is "the devil's advocate's" to weigh it correctly) without implying a fixed reading order run-to-run.

Print the result under `## Stage 4: Final Answer`.

## After the run

Don't render an artifact by default — terminal output above is the deliverable. Only build an HTML artifact (tabbed view of all stages, mirroring the original web UI's stage tabs) if the user explicitly asks to see it as an artifact, or asks for a shareable view once the discussion has wound down. Load the `artifact-design` skill first when that happens.

## Notes

- This replaced the original OpenRouter-based FastAPI/React app from upstream `karpathy/llm-council`. See `CLAUDE.md` for why, and what to do if upstream changes need reconciling.
- Devil's Advocate, Optimist, and Feasibility are strictly advisory — they rate and flag, they never block or veto. Only the Chairman produces a final verdict, and it isn't bound by majority sentiment.
- To add a further hat (e.g. a "user advocate" or a Green Hat creative-alternatives member), add a new `.claude/agents/council-<name>.md` following the pattern of the Stage 3 members, then wire it into Stage 3 above.
