# CLAUDE.md — LLM Council (Claude Code native, deployed copy)

> This is the design-rationale document `SKILL.md` refers to. It lives next to the skill so the
> references never dangle. The copy at the root of the GitHub fork
> ([MKamel1/llm-council](https://github.com/MKamel1/llm-council), branch `master`) describes an
> **older 4-stage architecture** (cross-evaluation, three hats, no rebuttal/audit) and should be
> synced from this file when the repo is next touched.

## What this is

This project began as a clone of [karpathy/llm-council](https://github.com/karpathy/llm-council) —
a multi-stage LLM deliberation system originally built as a FastAPI backend + React frontend using
multiple providers via OpenRouter. The original app was removed and rebuilt Claude-Code-native:
`.claude/agents/council-*.md` + `.claude/skills/council/SKILL.md`, invoked as `/council <question>`.

**Rationale:** the user has a Claude Pro/Max subscription and no separate API key or OpenRouter
account. Claude Code's Agent tool can pin a subagent to a model (`model: opus` / `model: sonnet`
in frontmatter) and run it under the existing subscription login — no API keys, no per-token
charges. That mechanism is the foundation of this fork.

## Current architecture (5 stages + 2 flag channels)

Nine agents in `.claude/agents/`; full orchestration lives in `SKILL.md`:

- **Stage 1 — First opinions:** `council-opus` + `council-sonnet`, parallel, blind to each other,
  each ending with a confidence rating.
- **Stage 2 — Multi-hat critique:** devil's advocate, optimist*, feasibility, status-quo*,
  reframer — parallel, blind to each other, all strictly advisory. (*Conditional seats: the
  orchestrator skips Status Quo when the question has no change dimension, and the Optimist when
  both first opinions are already affirmative on a purely factual question.)
- **Stage 3 — Member rebuttal:** the Stage 1 instances revise or defend their positions against
  the critique (deliberation, not blind review — instances persist via `SendMessage`).
- **Stage 4 — Chairman draft:** `council-chairman` (never gives a first opinion) synthesizes,
  with a consensus-×-confidence readout that distinguishes Stage 1 convergence from
  post-rebuttal convergence, bounded research escalation (2 cycles, targeted "bears on:"
  re-runs), and a contested-fork path that puts genuine values calls to the user instead of
  manufacturing a verdict.
- **Stage 5 — Member audit & finalize:** two *fresh* member spawns run a faithfulness-only audit
  of the draft; the Chairman then finalizes.
- **Research channel:** only `council-researcher` (Opus) has web tools; every other agent flags
  `NEEDS RESEARCH:` and the orchestrator dispatches — one auditable path, no redundant googling.
- **User-input channel:** members, Chairman, and Reframer may flag `NEEDS USER INPUT:` (with a
  stated `DEFAULT:` so the run never blocks); the four other hats only when the answer would
  completely change their critique's direction. The orchestrator batches flags at stage
  checkpoints and asks via `AskUserQuestion`, in plain English with trade-offs explained.
- **Run persistence & resume:** every run mirrors to `runs/<runid>.md`; aborted runs resume via
  `/council resume <runid>` (fresh spawns, reconstructed context — dead instances are never
  messaged).

The design draws on Edward de Bono's *Six Thinking Hats*: distinct agents hold distinct analytical
stances rather than asking one model to balance competing viewpoints simultaneously, which tends
to produce optimistic bias.

## Deliberate constraints (read before "improving" the design)

- **Correlated-members ceiling.** Both first-opinion members are Anthropic models; they share
  training and therefore blind spots. The deliberation/audit machinery protects against
  *uncorrelated* mistakes and an overloaded synthesizer — it cannot manufacture cross-vendor
  independence. Adding a third, different-vendor member is the only structural fix and is
  **deliberately out of scope** until non-Anthropic execution exists (see local-models section).
  That's also why there is no peer-review/ranking stage: ranking needs ≥3 independent answers;
  with 2 correlated ones the design invests in deliberation (rebuttal + audit) instead. If a
  third-vendor member is ever added, a ranking stage becomes worthwhile again, after Stage 1.
- **Anonymization.** Model names never appear in any agent's context — members are "Council
  Member A/B", hats are labeled by role. Only the orchestrator's user-facing output names models.
- **Retry-once-then-abort failure handling.** Every agent runs on the same subscription, so a
  second consecutive failure on one call is more likely systemic (rate limit, cap, outage) than a
  fluke; continuing would feed the Chairman silent gaps. Aborts are cheap to recover now that
  runs persist and resume.
- **Heavy by design.** No lightweight mode. The only dispatch-avoidance is the two Stage 2 seat
  tests, which skip calls that would return a known-empty answer ("N/A — nothing is changing") —
  that's waste-trimming, not critique-lightening.

## Upstream reconciliation

The GitHub fork is `origin`; `karpathy/llm-council` is `upstream` (fetch only). The original
`backend/` (FastAPI, `config.py`, `council.py`, `openrouter.py`, `storage.py`) and `frontend/`
(React/Vite) were removed — they exist in `git log`. When pulling upstream changes:

- **Prompt/algorithmic refinements** in `backend/council.py` are the valuable candidates —
  translate the *prompt substance and procedural logic* into `SKILL.md` and agent prompts, never
  the source code itself.
- **Model additions** to `COUNCIL_MODELS` — only actionable for models Claude Code can pin
  (Anthropic via `model:` frontmatter, or local models once supported).
- **Frontend/UI changes** are inapplicable; the equivalent here is the on-request "export as
  artifact" flow in `SKILL.md`.
- **`openrouter.py` changes** are inapplicable — this fork bypasses OpenRouter entirely.

## Future: local models / alternate providers

Tracked as [issue #2](https://github.com/MKamel1/llm-council/issues/2). Two integration models:

1. **Stay Claude-Code-native** if subagent pinning ever supports non-Anthropic/local models —
   extend the `council-<name>.md` pattern unchanged.
2. **Lightweight helper** (script or microservice) managing just the non-Anthropic members while
   Opus/Sonnet stay subagents — a hybrid, not a restoration of the OpenRouter stack.

**Revisit then:** the retry-once-then-abort failure policy (an isolated local-model failure is
plausible, so graceful continuation with a noted absence may fit better), and the third-vendor
member + ranking stage described above.

Cross-run research caching is separately tracked as
[issue #1](https://github.com/MKamel1/llm-council/issues/1) — unblocked by run persistence, not
yet implemented.
