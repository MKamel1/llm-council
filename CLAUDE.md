# CLAUDE.md - LLM Council (Claude Code native fork)

> The canonical, always-current copy of this document lives at
> `.claude/skills/council/CLAUDE.md` — `SKILL.md` references it from there. This root copy is kept
> in sync manually; if the two ever disagree, the skill-folder copy wins.

## What this is

This repo started as a clone of [karpathy/llm-council](https://github.com/karpathy/llm-council) —
a multi-stage LLM deliberation system originally built as a FastAPI backend + React frontend using
multiple providers via OpenRouter. That app has been **removed** and replaced with a
Claude-Code-native equivalent: `.claude/agents/council-*.md` + `.claude/skills/council/SKILL.md`,
run as `/council <question>` inside Claude Code.

**Why:** the user has a Claude Pro/Max subscription but no separate Anthropic API key or
OpenRouter account, and didn't want to pay for API access just to run this. Claude Code's Agent
tool can pin a subagent to a specific model (`model: opus` / `model: sonnet` in a subagent's
frontmatter) and run it headlessly, authenticated via the existing subscription login — no API
key, no OpenRouter, no per-token billing. That's the entire mechanism this fork relies on.

## Current architecture (5 stages + 2 flag channels)

Nine agents in `.claude/agents/`; full orchestration lives in `.claude/skills/council/SKILL.md`:

- **Stage 1 — First opinions:** `council-opus` + `council-sonnet`, parallel, blind to each other,
  each ending with a confidence rating.
- **Stage 2 — Multi-hat critique:** devil's advocate, optimist*, feasibility, status-quo*,
  reframer — parallel, blind to each other, all strictly advisory. (*Conditional seats: the
  orchestrator skips Status Quo when the question has no change dimension, and the Optimist when
  both first opinions are already affirmative on a purely factual question.)
- **Stage 3 — Member rebuttal:** the Stage 1 instances revise or defend their positions against
  the critique — this is the deliberation round, not blind review; instances persist via
  `SendMessage` so a member is revisiting its own prior position.
- **Stage 4 — Chairman draft:** `council-chairman` (a dedicated, opinion-free agent — never gives
  a first opinion, so it can't unconsciously favor its own earlier take) synthesizes, with a
  consensus-×-confidence readout that distinguishes agreement already present at Stage 1 from
  agreement that only appeared after rebuttal, bounded research escalation (capped at 2 cycles,
  targeted "bears on: <roles>" re-runs), and a contested-fork path that puts genuine values calls
  to the user instead of manufacturing a verdict.
- **Stage 5 — Member audit & finalize:** two *fresh* member spawns (not the Stage 1/3 instances)
  run a faithfulness-only audit of the draft — does it misrepresent an input, silently drop a
  preserved dissent, or misstate the consensus readout — then the Chairman finalizes.
- **Research channel:** only `council-researcher` (Opus, the sole agent with `WebSearch`/
  `WebFetch`) has web tools; every other agent ends output with `NEEDS RESEARCH:` and the
  orchestrator dispatches — one auditable path, no redundant parallel googling.
- **User-input channel:** members, Chairman, and Reframer may end output with
  `NEEDS USER INPUT:` (always paired with a `DEFAULT:` assumption so a run never blocks); the
  four other hats may only when the answer would completely change their critique's *direction*,
  not merely refine it. The orchestrator batches flags at stage checkpoints and asks via a single
  question per checkpoint, rewritten in plain English with trade-offs stated.
- **Run persistence & resume:** every run mirrors incrementally to `runs/<runid>.md`; a run that
  aborts after a retried failure can be continued with `/council resume <runid>` — the orchestrator
  reconstructs state from the file and uses fresh spawns (dead instances are never messaged).

This is loosely modeled on Edward de Bono's *Six Thinking Hats* — separate members embody separate
modes of judgment (facts, risk, upside, feasibility, inertia, premise) instead of one model trying
to hold all of them at once, which is what produces yes-bias in the first place.

## Deliberate constraints (read before "improving" the design)

- **Correlated-members ceiling.** Both first-opinion members are Anthropic models; they share
  training and therefore share blind spots. The deliberation (Stage 3) and audit (Stage 5)
  machinery reduces risk from *uncorrelated* mistakes and from a single overloaded synthesizer —
  it does **not** manufacture the independence only a different-vendor model would provide. A
  unanimous, high-confidence council answer is "two correlated draws agreed," not independent
  confirmation. Adding a third, different-vendor member is the only structural fix and is
  **deliberately out of scope** until non-Anthropic subagent execution exists (see Future below).
  This is also why there's no peer-review/ranking stage: ranking needs ≥3 independent answers;
  with 2 correlated ones, the design invests in deliberation instead. If a third-vendor member is
  ever added, a ranking stage becomes worthwhile again, slotting in after Stage 1.
- **Anonymization.** Model names never appear in any agent's context — members are labeled
  "Council Member A" / "Council Member B", hats are labeled by role only. No agent may speculate
  about which model wrote what. Only the orchestrator's user-facing terminal output names models.
- **Retry-once-then-abort failure handling.** Every agent runs on the same Claude subscription, so
  a *second consecutive* failure on one call is more likely systemic (rate limit, usage cap,
  outage) than an isolated fluke — continuing risks feeding the Chairman a silent gap. A single
  failure is common enough as a transient fluke that one retry absorbs it cheaply; discarding an
  entire multi-stage run over that would be wasteful. Aborts are now cheap to recover from, since
  runs persist to `runs/<runid>.md` and can be resumed rather than restarted from zero.
- **Heavy by design, always.** No lightweight mode, no auto-detection of "this is just a quick
  fact lookup" — every invocation runs the full roster (minus conditional-seat skips) and can
  pause for research or user input after each stage. The only dispatch-avoidance built in is the
  two Stage 2 seat tests, which skip calls that would return a known-empty answer ("N/A — nothing
  is changing here") — that's waste-trimming, not critique-lightening. A per-run call tally
  (Agent + SendMessage calls, by stage, plus skipped seats and user-input rounds) prints at the
  end of every run so the cost is never invisible.

## Removed from upstream

Deleted entirely (recoverable via `git log` — committed once, then removed in the commit that
added the agent/skill setup):

- `backend/` — FastAPI app: `config.py` (model list + chairman settings), `council.py` (stage
  prompting logic), `openrouter.py` (HTTP client), `storage.py`, `main.py`
- `frontend/` — React/Vite app with the Stage1/2/3 tabbed UI
- `main.py`, `start.sh`, `pyproject.toml`, `uv.lock`, `.python-version` — Python project plumbing

## Reconciling future upstream pulls

This repo lives at [MKamel1/llm-council](https://github.com/MKamel1/llm-council) as `origin` —
`git push`/`git pull` with no args target it directly. `karpathy/llm-council` is wired up as
`upstream` for pulling in future changes only (`git fetch upstream`), never for pushing. If
upstream changes land and get pulled/merged (this fork's history has diverged structurally, so
expect selective cherry-picking rather than a clean merge):

- **Prompt/logic changes to `backend/council.py`** (critique format, ranking math, chairman
  prompt) are the most likely thing worth porting — translate the *prompt wording and stage
  logic*, not the code, into `SKILL.md`'s stage instructions and the relevant agent files.
- **New models added to `backend/config.py`'s `COUNCIL_MODELS`** — decide if it's worth adding as
  a new subagent here (only meaningful for models Claude Code can actually run: Anthropic models
  via `model:` frontmatter, or self-hosted/local models once that path exists — see below).
- **Frontend UI changes** are not applicable — there's no web UI in this fork. If a visual tabbed
  view is wanted, that's the on-request "render as artifact" path in `SKILL.md`, not a frontend
  rebuild.
- If upstream restructures `backend/openrouter.py` (e.g. new provider support), that's irrelevant
  here — this fork doesn't use OpenRouter at all.

## Future: local models / other providers

The user's stated plan is to add local models (e.g. via Ollama) or other API providers later.
Tracked as [issue #2](https://github.com/MKamel1/llm-council/issues/2) — don't build until asked;
that issue is the placeholder for when this conversation happens. Two integration paths noted
there:

1. **Still Claude-Code-native:** if Claude Code ever supports pinning a subagent to a
   non-Anthropic/local model, extend the same `.claude/agents/council-<name>.md` pattern.
2. **Back to a real backend:** if local/other-provider models need to be called directly (the
   likelier path — Claude Code subagents can only run Claude models today), a lightweight script
   or small server alongside the skill could handle those specific council members while
   Opus/Sonnet stay on the subagent path. A hybrid, not a full revival of the OpenRouter backend.

**Also revisit then:**
- The retry-once-then-abort failure policy — once local models are in the mix, a single model's
  failure becomes a more plausible isolated event, and graceful degradation (continue with a
  noted gap, closer to upstream's original philosophy) may be worth reconsidering.
- The correlated-members ceiling and missing ranking stage described above — a local or
  alternate-provider model would be the first genuinely independent (non-Anthropic) voice, at
  which point a comparative ranking stage after Stage 1 becomes worthwhile.

Cross-run research caching is separately tracked as
[issue #1](https://github.com/MKamel1/llm-council/issues/1) — unblocked now that runs persist to
`runs/<runid>.md`, but not yet implemented.
