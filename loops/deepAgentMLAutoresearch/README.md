# deepAgentMLAutoresearch

**Analysis-first ML autoresearch, grounded in the scientific literature.**

[`standardMLAutoresearch`](../standardMLAutoresearch) plus a literature-review phase on
every iteration. After each training run the agent analyses what happened, turns its
findings into questions at two levels of abstraction, searches the literature, grades
the evidence, and implements only what prior work actually supports — pulling in known
techniques and baselines instead of reinventing them.

## How it works

```
run → analyse → ask (2 levels) → search → grade evidence → implement keepers → repeat
```

1. **Run & analyse** — train, then diagnose (gradients, activations, embeddings, errors,
   loss dynamics…) to find the empirical anchor for the next change.
2. **Question** — Level 1 (architecture fit, prior approaches, does the literature show
   success?) and Level 2 (init, weight-decay, attention/cache, schedule — micro-opts).
3. **Search** — research subagents run a triage-before-read funnel over the literature.
4. **Grade** — an evidence gate keeps only findings that are well-supported, applicable
   to *this* setup, and implementable within the editable files + budget.
5. **Implement** — Level-1 keepers first, then Level-2 refinements; each change carries
   a verification todo so the next run tells you whether it actually helped.

Every change is grounded in both your own diagnostics and a citation. A literature
ledger (`corpus.tsv`) tracks which papers were kept, implemented, and whether they
helped — so the loop learns which sources pay off.

## The literature toolchain

All access goes through `lit_search.py` — **standard-library only, Python ≥3.9, no
installs, no MCP setup required**. It works out of the box on free sources and gains
capability as you add optional keys.

| Source | Role | Cost |
|---|---|---|
| **Semantic Scholar** | semantic relevance search, full-text snippet search, citation graph | free (key recommended) |
| **OpenAlex** | citation/recency/venue-filtered discovery; cross-check | free |
| **arXiv** | full-text reading (HTML → LaTeX → PDF) | free |
| **Perplexity Sonar** (OpenRouter) | high-level synthesis with citations | paid (optional) |
| **bgpt.pro** | structured experimental results / limitations | free 50, then paid (optional) |

If a source is unavailable, the loop degrades gracefully — down to the host's built-in
web search — and keeps running.

## Setup

The loop asks you everything interactively (metric, run command, editable files,
sandbox, iteration strategy, budget, domain, and how deep to search). For API keys it
ensures a shared global key file exists and asks you to fill in the ones you want:

```bash
# the loop runs this for you during setup
python lit_search.py keys --init   # ensures keys.env at the project root
```

Keys live in one `keys.env` at your **project root**, shared by every skill in the
project (and **gitignored** — it sits inside the repo). You paste keys into it yourself,
so they never enter the conversation; the loop only ever checks presence (booleans). **A
free Semantic Scholar key is recommended** (the keyless shared pool is unreliable); get
one at <https://www.semanticscholar.org/product/api#api-key-form>. Everything else is
optional. See [`../../docs/api-keys.md`](../../docs/api-keys.md) for the convention.

See [`schema.example.yaml`](schema.example.yaml) for the resolved bindings, and
[`SKILL.md`](SKILL.md) for the full loop program.

## Best for

Problems where the relevant methods are already published and worth finding —
established architectures, known training recipes, well-studied failure modes. If the
problem is novel enough that little prior work applies, a non-grounded loop like
[`highTemperatureMLAutoresearch`](../highTemperatureMLAutoresearch) may explore faster.
