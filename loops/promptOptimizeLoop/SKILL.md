---
name: prompt-optimize-loop
description: >
  Use when the user has a system that takes a prompt as input and a way to score that system, and
  wants the prompt automatically improved to raise the score. Each iteration makes one targeted
  quality edit to the prompt — clarity, context, specificity, structure, examples — then runs the
  user's own evaluation command to measure the metric, and keeps the edit only if the metric
  improves; it loops until a target, plateau, or budget. The metric is supplied by the user (their
  eval command); the loop is metric-agnostic. Inspired by OpenEvolve / AlphaEvolve prompt evolution.
metadata:
  version: "0.1.0"
---

# Prompt Optimize Loop

An **evolutionary optimizer for a prompt** (OpenEvolve / AlphaEvolve-style). The artifact is a prompt
that feeds the user's system; the feedback signal is a **scalar metric produced by the user's own
evaluator**. You propose one quality-focused edit, measure the metric, keep it only if it improves,
and repeat — evolving the prompt toward higher scores.

You do **not** define the metric — the user does, because a prompt is only "better" relative to what
the system using it is measured on. By binding to the user's evaluator — task accuracy, an LLM-judge
score, a pass rate, a tool-call success rate, whatever they already track — the loop optimizes the
thing that actually matters. Your job is to make the prompt **clearer, better-grounded, more specific,
and better-structured** so that metric goes up.

---

## 1. Resolve bindings (setup — once)

If a `loop.run.yaml` exists, load it, confirm the values in one line, and skip to §2. Otherwise
resolve each binding, then write `loop.run.yaml` so re-runs are non-interactive.

**Detect host:** if `AskUserQuestion` is available you are in **Claude Code** — infer a likely value
and present it as the recommended option. Otherwise ask each as a quoted plain-text prompt.

- **`<prompt_file>`** — the prompt to optimize (the artifact the loop evolves).
- **`<eval_cmd>`** — **required.** The command that scores the current `<prompt_file>` by running the
  user's system and printing the metric. **Output convention:** a JSON object on the last line —
  `{"score": <number>, "feedback": "<optional notes/errors>", ...any extra metrics...}` — or a bare
  number. Higher is better unless `<objective>` says minimize. The loop treats this as a **black box
  and never edits it.**
  > If the model that executes the prompt is something `<eval_cmd>` calls internally, you just run
  > `<eval_cmd>`. If instead the prompt is executed by *you* (interactive development with no separate
  > inference endpoint), first run the current prompt over the user's eval inputs to produce outputs,
  > write them where `<eval_cmd>` reads, then run `<eval_cmd>` to score them.
- **`<objective>`** — `maximize` (default) or `minimize`, plus one line on what the metric measures, so
  edits are targeted rather than random.
- **`<target>`** — optional score at which to stop. **`<sandbox_root>`** (default `./sandbox/`),
  **`<budget>`** (default 10), **`<patience>`** (default 3 — plateau cutoff).

**Confirm and go.** Print the bindings; create nothing until the user confirms (skip only when
`loop.run.yaml` already existed). Then initialise the ledger (§3) and start.

---

## 2. The loop (evolve the prompt)

**Iteration 0 — baseline.** Run `<eval_cmd>` on the current prompt, record its `score` as the best,
and start a short **history**: `{iteration, edit, score, feedback}` per row. The `feedback` field from
`<eval_cmd>` is your richest signal — read it like AlphaEvolve's artifacts side-channel.

**Then, until stop (target, plateau, or budget):**

1. **Diagnose.** From the latest `score`, the `<eval_cmd>` feedback, and the recent history, name the
   prompt's single biggest current weakness.
2. **Make one targeted edit** from the prompt-quality toolkit — pick the operator that addresses that
   weakness:
   - **Clarity** — remove ambiguity, contradictions, and vague wording.
   - **Context** — supply missing domain knowledge, definitions, or background the task needs.
   - **Specificity** — make instructions concrete; pin down the output format; define what "good" is.
   - **Structure** — order the prompt into steps/sections; add a short checklist.
   - **Examples** — add one or two demonstrations of the desired input → output.
   - **Decomposition** — split a complex instruction into explicit ordered sub-steps.
   - **Guardrails** — state edge cases and what to avoid.

   One change per iteration, so its effect on the metric is attributable.
3. **Measure.** Run `<eval_cmd>` and read the new `score`.
4. **Keep or revert.** **Keep** if the metric improves (if the eval is stochastic, require a small
   margin so noise doesn't drive a keep); otherwise **revert** to the previous best prompt. Append
   `{edit, score, feedback}` to history either way.
5. **Escape local optima.** If the score has not improved for a couple of iterations, stop making tiny
   tweaks — branch from an earlier high-scoring variant, or try a bolder restructuring (a different
   decomposition, a fresh set of examples). Diversity beats grinding the same local hill.

**Stop** when `score` reaches `<target>`, when no iteration improved the best for `<patience>`
consecutive rounds, or at `<budget>` (every non-improving iteration counts toward patience; a keep
resets it). Restore the **best** prompt and report: the score trajectory, which edits moved the metric
(and which did not), and the final prompt.

---

## 3. Ledger

`<sandbox_root>/ledger.tsv`, tab-separated, never commas in the text. Header:
```
iter	score	status	edit
```
`status` ∈ {`baseline`, `keep`, `revert`}. Example (metric = task accuracy, maximize):
```
iter	score	status	edit
0	0.42	baseline	original prompt
1	0.61	keep	specificity: define each output label and the exact output format
2	0.61	revert	examples: add 3 few-shot demos — no metric gain
3	0.78	keep	context: add the domain rules the task assumes but never states
```
Report the **best** iteration, not the last.

---

## 4. Hard constraints
- **The metric is the user's.** Never edit `<eval_cmd>`, its data, or its scoring — that games the
  number instead of improving the prompt.
- **Optimize the prompt only, and preserve the task's intent.** Improve *how* the task is instructed,
  not *what* is being asked; do not tailor the prompt to exploit eval quirks that would break real use.
- **One edit per iteration**, and compare the *metric* (re-run the full eval), not a single sample.
- **Report the best variant, not the last.** The sandbox is self-contained — no `../` escapes.
- Do not pause the loop to ask whether to continue; run until target, plateau, or budget.
