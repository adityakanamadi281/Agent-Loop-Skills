# standardMLAutoresearch

**A mono-agent ML research loop driven by analysis, not guesswork.**

Unlike [`karpathy`](../karpathy), which proposes changes and scores them blindly,
this loop treats every training run as an experiment to be *understood*. After each
run the agent performs a diagnostic pass — examining gradients, activations,
embeddings, error patterns, training dynamics, or whatever the results suggest —
then uses those findings to motivate the next change. Changes are hypotheses grounded
in evidence.

## What makes it different from karpathy

- **Analysis phase after every run** (step 6 of the loop). The agent chooses what
  to examine based on what the results suggest — not a fixed checklist. Findings are
  written to `iter<N>/analysis/`.
- **Hypothesis must cite the analysis** before any change is applied. No blind tries.
- **`results.tsv` has an `analysis_summary` column** — each row records the key
  finding that motivated the change.
- **Instrumentation is a valid iteration**: if the analysis reveals a useful quantity
  that isn't being logged, adding it counts as a loop step.

## Per-iteration sandbox layout

```
<sandbox_root>/
├── schema.yaml
├── results.tsv
└── iter<N>/
    ├── schema.yaml          # copy of root schema
    ├── code_snapshot/       # editable files before this iteration
    ├── run.log              # full training output
    ├── analysis/            # scripts written and run during the analysis phase
    └── results/             # outputs produced by analysis scripts
```

## Files

| File | Role |
| --- | --- |
| `SKILL.md` | The full loop program. |
| `schema.example.yaml` | Copy to `schema.yaml` and fill in, or answer interactively. |
