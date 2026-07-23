---
name: council
description: Run a question or idea through the full LLM Council — independent first opinions, adversarial multi-hat critique (devil's advocate, optimist, feasibility, status quo, reframer), a member rebuttal round, on-demand research, user clarification checkpoints (agents can raise plain-English questions for the user to narrow scope or decide a fork), neutral chairman synthesis, and a member audit of that synthesis. Use when the user invokes /council, asks for "the council's" opinion, wants an idea stress-tested from multiple angles, or wants to avoid single-model yes-bias. Also handles `/council resume <runid>` for aborted runs.
---

# LLM Council

Runs `args` (the user's question or idea) through the full council process. All stages run via Agent tool calls to the subagents below — no API key, no external server, everything runs through Claude Code's own model-pinned subagents under the current subscription.

If `args` is empty, ask the user what question or idea to put to the council before doing anything else.

**This skill is deliberately heavy, always.** There's no lightweight mode and no auto-detection of "this is just a quick fact lookup" — every invocation runs the full roster (minus the two seats the orchestrator may skip as N/A — see Stage 2 seat tests), deliberates (members rebut the critique, then audit the Chairman's draft), and can pause for research after each of Stages 1–3. Worst case (max escalation, all advisors named as affected) can run 30+ Agent/SendMessage calls; typical runs are cheaper. That's intentional: `/council` is reserved for genuine decisions/ideas where independent opinions, adversarial critique, actual back-and-forth, and a checked final answer earn their cost. For quick factual questions, ask directly instead — the burden is on the caller to invoke `/council` only when it's warranted, not on the skill to guess. A per-run call tally is printed at the end (see After the run) so the cost is never invisible.

## Roster

| Agent | Model | Role |
|---|---|---|
| `council-opus` | Opus | Stage 1 first opinion, Stage 3 rebuttal, Stage 5 synthesis audit |
| `council-sonnet` | Sonnet | Stage 1 first opinion, Stage 3 rebuttal, Stage 5 synthesis audit |
| `council-devils-advocate` | Sonnet | Stage 2 — Black Hat, risk/weakness case (advisory) |
| `council-optimist` | Sonnet | Stage 2 — Yellow Hat, upside case (advisory, **conditional seat** — see Stage 2 seat tests) |
| `council-feasibility` | Sonnet | Stage 2 — practicality rating (advisory) |
| `council-status-quo` | Sonnet | Stage 2 — Status Quo Defender, switching-cost analysis (advisory, **conditional seat** — see Stage 2 seat tests) |
| `council-reframer` | Sonnet | Stage 2 — challenges the question's premise (advisory) |
| `council-researcher` | Opus | On-demand only — dispatched when any stage flags `NEEDS RESEARCH:` |
| `council-chairman` | Opus | Stage 4 draft synthesis + Stage 5 finalize — neutral, never gives a first opinion |

Roster is deliberately Opus + Sonnet only — Opus where judgment is highest (Chairman,
Researcher), Sonnet everywhere else for efficiency. No other model tiers are in the rotation.

**Known ceiling — read this before trusting a unanimous verdict.** Both first-opinion members are
Anthropic models: they share training and therefore share blind spots. The one failure mode this
setup *cannot* reliably catch is the case where both members are confidently wrong the *same way* —
Stage 3 rebuttal and Stage 5 audit reduce that risk but do not remove it, because a correlated
reviewer tends to share the error rather than spot it. The deliberation and audit machinery below
buys real rigor against *uncorrelated* mistakes and against a single overloaded synthesizer; it does
**not** manufacture the independence that only a different-vendor model would provide. Treat a
unanimous, high-confidence council answer as "two correlated draws agreed," not as independent
confirmation. (Adding a third-vendor member is the only structural fix and is deliberately out of
scope — see `CLAUDE.md` in this skill folder.)

**Single research channel:** only `council-researcher` has web access (`WebSearch`, `WebFetch`).
Every other agent in the roster obtains external/current facts exclusively by ending its output
with `NEEDS RESEARCH:` and letting the orchestrator dispatch a research pass — never by searching
inline. This is deliberate: it prevents multiple parallel agents from independently (and
redundantly) googling the same fact, and keeps external-fact-finding in one auditable place. Do
not add `WebSearch`/`WebFetch` back to other agents' `tools:` frontmatter without updating this
note.

**Single user channel:** some agents may end their output with a `NEEDS USER INPUT:` flag — a
question only the *user* can answer (unclear scope, a missing constraint, a values call). Like
research, this routes exclusively through the orchestrator, which batches flags at fixed
checkpoints and asks via `AskUserQuestion` — agents never assume they can address the user
directly. Who may flag is deliberately tiered: the **Stage 1 members, the Chairman, and the
Reframer** may flag whenever the answer would materially change their output; the **other four
hats** may flag only when the user's answer would *completely change the direction* of their
critique — not for refinements. Every flag must carry a stated `DEFAULT:` assumption so the run
can always proceed without blocking. See User-Input Handling below.

## Run Setup

Before dispatching anything:

- Generate a short **run ID** for this invocation: `<HHMMSS>-<slug>`, where `<slug>` is a short
  kebab-case slug derived from the question. Use it as the suffix on every named agent instance
  (e.g. `council-opus-<runid>`, `council-chairman-<runid>`) and as the persistence filename. A
  static suffix like `-1` would let a *second* `/council` invocation in the same session address a
  stale instance from an earlier run via `SendMessage`; the run ID prevents that collision.
- Initialize a **call counter** at 0. Increment it once for every Agent dispatch and every
  `SendMessage` you make during the run, tagged by stage, so the end-of-run tally (see After the
  run) is accurate. Track **user-input rounds** as a separate count in the same tally.
- **Context pack:** if the question references local files, a repo, or documents, read the
  referenced material **once, yourself, now** and assemble the relevant excerpts (with paths)
  into a single context pack. Append the *identical* pack to every dispatch that includes the
  question. Instruct every recipient to work from the pack only — not to explore the filesystem
  independently. Two members reading *different* files silently breaks the comparability of
  their positions, which the whole design depends on; agents keep `Read`/`Grep`/`Glob` only so
  they can open a path the pack itself cites. If the material is too large to pack sensibly,
  don't guess at a selection — ask the user to narrow scope before Stage 1 (counts as a
  user-input round).

## Stage 1 — First Opinions

Dispatch the question to both `council-opus` and `council-sonnet`, single message, two parallel Agent tool calls, `run_in_background: false` for both (nothing can proceed until both return). Each gets only the raw question — no mention of the other model, no hint that this is a council. Name them `council-opus-<runid>` and `council-sonnet-<runid>` at dispatch; these instances are continued (via `SendMessage`) in Stage 3's rebuttal round, so the names must stay addressable through the run.

Each member ends its answer with a one-line **confidence rating** (High / Medium / Low + a short why) — this feeds the Chairman's consensus-×-confidence readout in Stage 4.

Print both answers under `## Stage 1: First Opinions`, labeled by model name.

Then run **Research Handling** and **User-Input Handling** for any flags raised in Stage 1 before proceeding to Stage 2.

## Stage 2 — Multi-Hat Critique

**Label convention:** `council-opus` is always “Council Member A” and `council-sonnet` is always “Council Member B”. These labels are for your orchestration bookkeeping only — never disclose actual agent or model names to any Stage 2+ agent.

**Seat tests (run these before dispatching — they exist to skip calls that would only buy a known-empty answer, not to lighten the critique):**

- **Status Quo seat:** dispatch only if the question has a change/switching dimension — something being changed, adopted, migrated, or replaced. For pure analysis ("which is correct?", "what does X mean?") the agent's own Step 0 would just return "N/A"; skip the dispatch and print `Status Quo: skipped — no change dimension, no switching cost to weigh.` The agent's Step 0 stays as the backstop for borderline cases you *do* dispatch.
- **Optimist seat:** dispatch only if (a) the question evaluates an idea, plan, or proposal (where a steelmanned FOR case is real signal), **or** (b) at least one Stage 1 answer is predominantly negative/skeptical (the counterweight case). If it's a purely factual question and both members already lean affirmative, the Optimist would largely duplicate them — skip and print `Optimist: skipped — both first opinions already affirmative on a factual question.`

Send the remaining seats — always `council-devils-advocate`, `council-feasibility`, and `council-reframer`; plus `council-optimist` / `council-status-quo` when their seat test passes — the same package: the original question and both Stage 1 answers (post-Research-Handling, if revised) labeled “Council Member A” and “Council Member B”. **Never include actual agent or model names in any prompt passed to these advisors.** All dispatches in parallel Agent calls, `run_in_background: false`, each blind to the others’ output so they stay independent. Name each `council-<role>-<runid>` at dispatch — Stage 4 escalation continues these same instances via `SendMessage` rather than re-spawning them.

Print all dispatched seats under `## Stage 2: Multi-Hat Critique`, subheaded by role (Devil’s Advocate / Optimist / Feasibility / Status Quo / Reframer), with skipped seats noted by their skip line. Downstream packages (Stage 3, Chairman) include the skip note so no agent mistakes a skipped seat for a silent one.

Then run **Research Handling** and **User-Input Handling** for any flags raised in Stage 2 before proceeding to Stage 3.

## Stage 3 — Member Rebuttal

This is the council's deliberation round — the one place a member responds to another agent's reasoning instead of working in isolation. Continue each Stage 1 member instance via `SendMessage` (not a fresh spawn — the member is revisiting *its own* position, so its prior context should persist; there is no blind-review concern here, unlike a peer-review of someone else's answer). Send each member:

- all dispatched hats' critiques from Stage 2, labeled by role only (no model names), with the skip line for any seat the Stage 2 seat tests skipped,
- the *other* member's Stage 1 answer, anonymized as "the other Council Member's answer," and
- **which label is its own** — e.g. tell `council-opus-<runid>` "you are Council Member A; critiques referring to Council Member A concern your answer, Council Member B is the other member." Without this, the member has to infer its own identity by matching quoted text in the critiques against its own answer, which silently fails whenever a critique paraphrases rather than quotes — the member could end up rebutting objections aimed at the other member's answer while leaving its own unanswered. This discloses only the member's own bookkeeping label, not any model identity — the anonymization contract is unaffected.

Ask each member to **revise or explicitly defend** its position in light of the critique: engage the single strongest objection head-on, concede what genuinely lands, and hold ground (with reasons) where it doesn't — not merely restate the original answer. The member may update its confidence rating. If `SendMessage` to the instance fails, fall back to a fresh Agent call reconstructing its Stage 1 answer + the critique package in the prompt.

Print both revised positions under `## Stage 3: Member Rebuttal`, labeled by model name. These revised positions (not the Stage 1 originals) are what the Chairman synthesizes.

Then run **Research Handling** and **User-Input Handling** for any flags raised in Stage 3 before proceeding to Stage 4.

## Research Handling

This procedure runs at **three separate points** — after Stage 1, after Stage 2, and after Stage 3 — each time scoped to only the flags that stage just raised. It does not wait until the end to process everything at once: an answer that gets revised after research needs to be the version every *later* stage critiques or synthesizes, or the Chairman ends up working from inputs that describe a position no longer held.

At each of these points:

1. Collect any output from that stage ending with a line starting exactly `NEEDS RESEARCH:`.
2. Dedupe within this batch — if two flagged questions are essentially the same, research once and give the answer to both flaggers. (Dedupes only within a stage's batch and only within this run — see Notes on cross-run caching.)
3. Dispatch `council-researcher` — one Agent call per distinct question after deduping, in parallel if there's more than one, named `council-researcher-<runid>-<n>`.
4. For each flagging agent, `SendMessage` its own named instance the relevant research brief and ask it to revise its output in light of the answer. Use the **revised** output downstream. If `SendMessage` fails, fall back to a fresh Agent call with the agent's original output + the brief reconstructed.
5. Print results under `## Research` (the brief) and update that stage's printed output to the revised version.

If a stage raised no flags, skip this procedure for that stage — don't dispatch the researcher speculatively, and don't insert a pause nothing asked for.

## User-Input Handling

Runs at the same checkpoints as Research Handling (after Stages 1, 2, and 3 — plus the Chairman's draft, see Stage 4), scoped to the flags that stage just raised. A flag is any output ending with a block of the form:

```
NEEDS USER INPUT: <the question>
DEFAULT: <the assumption the agent will proceed on if the user has no preference>
TRADEOFF: <one sentence — what choosing differently would change>
```

At each checkpoint:

1. Collect that stage's flags. Dedupe within the batch — if two agents ask essentially the same thing, ask once and give the answer to both flaggers.
2. Ask via **one** `AskUserQuestion` call for the whole batch (max one user-input round per checkpoint). **Rewrite every question in plain, simplified English before asking** — no council jargon, no agent role names required to understand it — and put the trade-off in the option descriptions so the user sees what each choice costs. Make the agent's `DEFAULT:` the first option, marked "(Recommended)" only if you actually agree it's the sensible default.
3. If a batch has more than 4 distinct questions (the `AskUserQuestion` cap), ask the 4 most consequential and resolve the rest with their stated `DEFAULT:` assumptions, telling the user which defaults were auto-applied.
4. `SendMessage` each flagging agent's named instance the user's answer and ask it to revise. Use the revised output downstream, and update that stage's printed output. If `SendMessage` fails, fall back to a fresh Agent call reconstructing the original output + the answer.
5. Record each Q, the user's answer, and any auto-applied defaults under `## User Input` in the printed output and the run file. The user's answers are facts for the rest of the run — include them (labeled "User clarification") in every later stage's package.

**Who may flag** (enforced by the agents' own prompts; if an unauthorized or malformed flag appears anyway, treat the `DEFAULT:` as adopted and move on rather than bouncing it back):
- `council-opus` / `council-sonnet` (first opinion and rebuttal only — never the Stage 5 audit), the **Chairman** (draft phase only), and the **Reframer**: whenever the answer would materially change their output.
- The other four hats: only when the user's answer would completely change the direction of their critique.

Agents always answer under their stated `DEFAULT:` rather than blocking — so even if the user's reply is "no preference," the run has a complete output to continue from.

## Stage 4 — Chairman Draft Synthesis

Name the Chairman `council-chairman-<runid>` at its first dispatch. Call it with everything gathered so far — the original question, both **Stage 1 original** positions (labeled “Council Member A” / “Council Member B”, marked superseded), both **Stage 3 revised** positions (same labels), all dispatched hats’ Stage 2 output (labeled by role only, skip lines included for skipped seats), any research briefs, and any user clarifications. **Model names must not appear anywhere in the Chairman’s context.** The Stage 1 originals are provided alongside the revisions so the Chairman can see what each member changed under critique — conceded, answered, or silently dropped — not to be synthesized from directly.

The Chairman produces a **draft** synthesis (not yet the final answer — Stage 5 audits it). Its draft must include a one-line **consensus-×-confidence readout**: did the members converge or stay split, and how confident were they (drawing on their confidence ratings)? Per `council-chairman.md`, this readout must also state *when* any agreement formed — independently at Stage 1, or only after seeing each other's answers in Stage 3 — since those carry very different evidentiary weight. A split-but-confident result and a converged-but-low-confidence result are very different signals too, and the reader should see which one this is.

The Chairman may also conclude the question is a **genuine contested fork** — a values/judgment call the council cannot resolve for the user. In that case its draft ends with a `NEEDS USER INPUT:` block presenting the fork: the live positions, what separates them, and the trade-off, all in plain English. Run **User-Input Handling** for it (this is the Stage 4 checkpoint): ask the user to make the call, `SendMessage` the answer back to `council-chairman-<runid>`, and have it produce an updated draft with a real verdict reflecting the user's weighting — dissent still preserved per its own rules. If the user declines to choose, the draft falls back to presenting the fork as the answer. **One fork ask per run**; it does not count against the escalation cycle cap. The Chairman may use the same mechanism at the draft phase for a non-fork scope/values question when only the user can answer it — same checkpoint, same one-round cap. Stage 5 always audits the *post-fork-resolution* draft, never the superseded one.

**Simplified escalation (bounded, capped at 2 cycles total):** if the Chairman's draft ends with a `NEEDS RESEARCH:` escalation and a "bears on: <roles>" line (see `council-chairman.md` for when it should):

1. If this would be the 3rd cycle, do not escalate — tell the Chairman this is the final cycle and ask it to finalize its draft with an explicit caveat instead.
2. Otherwise: dispatch `council-researcher` for the flagged question → `SendMessage` the brief to only the instance(s) named in "bears on:" (they already hold their own context; send just the brief), asking each to revise or counter-argue; those not named carry their prior output forward unchanged. "Bears on:" may name Stage 2 advisor roles **and/or "Council Member A/B"** — the member case is reserved for when the research directly contradicts a member's stated factual premise (see `council-chairman.md`); route it to that member's Stage 1/3 instance (`council-opus-<runid>` / `council-sonnet-<runid>`, per your label bookkeeping) for one bounded revision of its Stage 3 position. If "bears on: none," skip straight to re-synthesis but still pass the Chairman the new brief — it's the only new input and must reach the Chairman explicitly. → `SendMessage` `council-chairman-<runid>` the updated output + brief for a new draft; increment the cycle counter. Continuing the same Chairman instance across cycles is safe — it has no first opinion of its own to anchor on, so accumulating deliberation context is exactly its job. If `SendMessage` to it fails, fall back to a fresh Agent call reconstructing the full package.

**Known, accepted trade-off:** members never see an advisor's escalation-revised critique — their Stage 3 rebuttals answered the pre-escalation version. The Chairman holds both the original and the revised advisor output and weighs the revision itself; re-opening the rebuttal round for every escalation would cascade cost for marginal benefit. The one exception is the narrow "bears on: Council Member A/B" patch above — a member gets a revision pass *only* when research contradicts its stated factual premise, because synthesizing from a position known to rest on a false fact is worse than the extra call. Don't widen that exception without reconsidering the cost trade explicitly.

Print each draft (and its escalation reasoning, if any) under `## Stage 4: Chairman Draft` so the user can see why a run did or didn't spend extra cycles.

## Stage 5 — Member Audit & Finalize

The Chairman's draft is the one output nothing has checked yet — so before it becomes the answer, the members audit it. Dispatch **two fresh** spawns of `council-opus` and `council-sonnet`, in parallel, `run_in_background: false` (fresh, **not** the deliberating Stage 1/3 instances — a fresh instance auditing the draft is far less prone to rubber-stamping a synthesis that happens to echo its own earlier view). Give each only: the original question, the full input package the Chairman saw (both revised positions + all hats + any research, still anonymized), and the Chairman's draft.

Ask a **faithfulness-only** rubric — not "do you agree with the conclusion," but:
- Does the draft accurately represent each input, or does it misstate one?
- Did it silently drop a load-bearing objection or a preserved dissent?
- Does the consensus-×-confidence readout match what the inputs actually show?

Print both audits under `## Stage 5: Member Audit`, labeled by model name.

Neither the `NEEDS RESEARCH:` nor the `NEEDS USER INPUT:` protocol applies at the audit — it's an internal-consistency check against inputs the council already has, not against the world or the user. If an audit ends with either flag anyway, do not dispatch the researcher or ask the user; pass the line through to the Chairman as an ordinary faithfulness comment.

Then `SendMessage` `council-chairman-<runid>` the two audits and ask it to **finalize**: fix any faithfulness flag that lands (or say why a flag is mistaken), then produce the final answer. If both audits are clean, it finalizes as-is with a one-line note that the audit passed. Print the result under `## Stage 5: Final Answer`.

**Known, accepted trade-off:** the finalize pass itself is the one output no one audits — auditing it would regress forever. The risk is bounded because finalize only patches specific flags on an already-audited draft; it is not a rewrite. Don't add a second audit round without reconsidering that cost trade explicitly.

## Failure Handling

If **any** Agent or `SendMessage` call anywhere in the run fails or errors — any stage, `council-researcher` dispatches, escalation cycles, or the Stage 5 finalize — **retry that single call once.** If the retry also fails, **stop the run** — don't proceed. Report to the user plainly which agent failed and why (if known), that it was retried, and that the council run was aborted rather than completed with a gap. Also report the run file path (`runs\<runid>.md`), which stages it preserves, and that `/council resume <runid>` can continue from there — an abort discards the failed call, not the run's completed work.

This applies uniformly to every call, not just the early stages: a Chairman failure at escalation or finalize is the single most expensive failure point in the run — the entire run's spend sits behind it — and deserves the same retry-then-abort protection as a Stage 1 call, not less.

This is deliberate, not an oversight: every agent runs on the same underlying Claude subscription, so a *second consecutive* failure on the same call is more likely a systemic issue (rate limit, usage cap, outage) than an isolated fluke — continuing past that risks feeding the Chairman a silent gap instead of a real signal worth investigating first. A single failure, though, is common enough as a transient fluke that discarding an entire multi-stage run over it is wasteful; one retry absorbs that case cheaply. Revisit the "stop after 2" threshold once local models are added (see `CLAUDE.md` in this skill folder), where a single model's failure becomes a more plausible isolated event and graceful degradation may make more sense.

## Run persistence

As the run proceeds, mirror everything you print to the terminal into a run file at the
**absolute** path `C:\Users\mmbka\.claude\skills\council\runs\<runid>.md` (the run ID from Run
Setup). Use the absolute path, never one relative to the session's working directory — `/council`
can be invoked from inside any project folder, and a relative path would scatter run files across
whichever project happened to be the cwd instead of collecting them in one place cross-run tooling
can grep. Write it incrementally (or at minimum, once at the end of the run) so a completed run is
never terminal-only.

This is the concrete prerequisite for cross-run research caching (tracked as issue #1, see Notes):
before dispatching `council-researcher` for a flagged `NEEDS RESEARCH:` question, a future version
of this skill can grep prior files in `runs/` for a question close enough to reuse instead of
re-researching from scratch. That caching logic isn't built yet — for now, just make sure the
files exist to build it on top of.

## Resume

If invoked as `/council resume <runid>` (or the user asks to resume an aborted run):

1. Read `runs\<runid>.md`. If it doesn't exist, list the files in `runs\` and ask the user which run they meant.
2. Determine the last fully completed stage from the file's printed sections, and reconstruct the run state from it: the question, any context pack, each stage's (revised) outputs, research briefs, and user answers.
3. Continue from the first incomplete stage. **Assume no instance from the aborted run is still alive** — every continuation uses the fresh-spawn fallback pattern already defined per stage (a fresh Agent call with the instance's own prior output + the new package reconstructed in the prompt), never `SendMessage` to a dead name. Generate a fresh run ID suffixed `-r` (e.g. `<oldrunid>-r`) for the new instances, but keep appending output to the *original* run file, noting where the resume began.
4. Continue the call counter from the tally recorded in the file (or count the file's logged calls if no tally was written); the final tally covers the whole run, resumed portion included.

A resumed run loses the dead instances' private context — that's the cost of the abort, and the fallback pattern is designed to absorb it. Don't re-run completed stages to "refresh" them; their outputs in the file are the record.

## After the run

Print a **cost tally** as the last thing in the run: the total number of Agent + `SendMessage`
calls from the counter, broken down by stage, plus skipped seats and user-input rounds (e.g.
`Stage 1: 2, Stage 2: 4 (Optimist skipped), Stage 3: 2, Research: 1, User input: 1 round,
Stage 4: 1, Stage 5: 3 → total 13`). For something that advertises itself as heavy, this is the
cheapest possible accountability and lets the user judge whether a given question was worth it.

Don't render an artifact by default — terminal output above is the deliverable. Only build an HTML artifact (tabbed view of all stages, mirroring the original web UI's stage tabs) if the user explicitly asks to see it as an artifact, or asks for a shareable view once the discussion has wound down. Load the `artifact-design` skill first when that happens.

## Notes

- This replaced the original OpenRouter-based FastAPI/React app from upstream `karpathy/llm-council`. See `CLAUDE.md` in this skill folder for why, and what to do if upstream changes need reconciling. (The copy at the root of the [MKamel1/llm-council](https://github.com/MKamel1/llm-council) fork describes an older 4-stage architecture and needs syncing from the local one.)
- All hats are strictly advisory — they rate and flag, they never block or veto. Members deliberate (Stage 3) and audit (Stage 5) but do not produce the verdict. Only the Chairman produces the final answer, and it isn't bound by majority sentiment.
- To add a further hat (e.g. a "user advocate" or a Green Hat creative-alternatives member), add a new `.claude/agents/council-<name>.md` following the pattern of the Stage 2 members, then wire it into Stage 2 above.
- Roster is currently Opus + Sonnet only (2 first-opinion members) by deliberate choice — see the Known ceiling note in Roster. There is no peer-review/ranking stage: with two correlated members, ranking (which needs ≥3 independent answers) buys little, so the design invests in *deliberation* (rebuttal + audit) instead. If a third, ideally different-vendor, member is ever added, a comparative peer-review/ranking stage becomes worthwhile again and should slot in after Stage 1.
- Tracked as [issue #1](https://github.com/MKamel1/llm-council/issues/1): cross-run research caching, now unblocked by the run-persistence files described above but not yet implemented.
