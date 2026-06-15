---
name: karpathy
description: >
  Use when the user wants a fully-autonomous, iterate-and-score research loop
  modelled on Karpathy's autoresearch program. One agent hacks code, runs it,
  and keeps changes that improve a user-defined scalar metric — looping forever
  until manually interrupted.
metadata:
  version: "0.1.0"
---

# Karpathy Autoresearch Loop

This is an autonomous research loop. You propose changes, run them, keep the
ones that improve the metric, discard the rest, and repeat — forever, until the
human interrupts you. You are the researcher. Do not pause to ask for permission
once the loop is running.

---

## 1. Resolve bindings (setup — do this once)

Work with the user to answer the following questions. Collect all answers before
proceeding; do not start the loop until every binding is confirmed.

### 1a. What metric should the loop minimize?

Ask the user:
> "What scalar metric should I minimize? (e.g. `val_loss`, `val_bpb`, `error_rate`)"

Confirm how to read it from the training output — grep pattern, JSON key, or last
stdout line. Record as **`<metric>`**.

### 1b. What Python environment should be used?

Ask the user:
> "What Python environment / uv project should I use to run training?
>  (e.g. `uv run train.py`, or a path to a virtualenv + script)"

Record as **`<run_cmd>`** — the exact shell command that launches one training run.

> **Remote sandboxes**: if the sandbox is on a remote host (SSH, cloud VM, container),
> note the connection details. All shell commands below are issued through whatever
> connection the user specifies. The loop itself runs on the user's local machine;
> only the `<run_cmd>` is dispatched remotely if needed.

### 1c. Which files are fair game to edit?

Ask the user:
> "Which files may I edit? List them. (Typical answer: just `train.py`, or a set of
>  model/config files.)"

Record as **`<editable_files>`**. Every other file is read-only.

### 1d. What is the training entrypoint?

Ask the user:
> "What command starts a single training run? Is it the same as your run command,
>  or a separate script?"

Confirm the full shell string. This becomes the body of step 4 of the loop.
Record as **`<entrypoint>`** (often the same as `<run_cmd>`).

### 1e. Where should the sandbox live?

Ask the user:
> "Where should I keep the sandbox (results log + per-iteration snapshots)?
>  Give me an absolute path. If training runs on a remote host, tell me whether
>  the sandbox should be local or remote."

Record as **`<sandbox_root>`**. Create the directory now if it does not exist.

### 1f. Iteration strategy — branches or same-branch snapshots?

Ask the user:
> "How should I track iterations?
>  (a) **Branch per iteration** (Karpathy original): each experiment gets a git commit
>      on a dedicated `autoresearch/<tag>` branch; good runs advance the branch,
>      bad runs are `git reset`'d.
>  (b) **Same-branch snapshots**: all work stays on the current branch; each iteration
>      gets a folder under the sandbox with a code snapshot."

Record as **`<iter_strategy>`** = `branches` or `snapshots`.

**If `branches`**: propose a run tag based on today's date (e.g. `jun14`). The branch
`autoresearch/<tag>` must not already exist. Create it: `git checkout -b autoresearch/<tag>`.

**If `snapshots`**: initialise the sandbox layout now:

```
<sandbox_root>/
├── results.tsv          # append-only log (TSV, see §3)
└── iter1/
    └── code_snapshot/   # copies of every <editable_file> as-of that iteration
```

Subsequent iterations add `iter2/`, `iter3/`, … alongside `results.tsv`.

### 1g. Gate on time or epochs?

Ask the user:
> "Should each training run be gated by **wall-clock time** (e.g. 5 minutes) or
>  by a fixed **number of epochs**?"

Record as **`<gate>`** = `time` or `epochs`.

**If `time`**:
- Ask: "How many minutes per run?"
- Record as **`<budget_minutes>`**.
- Before the first run, verify that `<entrypoint>` honours a time budget. If it does
  not already stop after `<budget_minutes>` minutes, inject a wrapper:

  ```python
  # auto-injected time-budget wrapper — written to <sandbox_root>/run_with_timeout.sh
  #!/usr/bin/env bash
  timeout $(( <budget_minutes> * 60 )) <entrypoint> "$@"
  ```

  Use `<sandbox_root>/run_with_timeout.sh` as the effective run command for this loop.
  Set a hard kill timeout at `2 × <budget_minutes>` minutes — if a run exceeds it,
  kill it and treat it as a crash.

**If `epochs`**:
- Ask: "How many epochs per run?"
- Record as **`<budget_epochs>`**.
- Verify that `<entrypoint>` (or a config it reads) accepts an epoch count. If needed,
  patch `<entrypoint>` (or the appropriate config file, if it is in `<editable_files>`)
  to cap training at `<budget_epochs>` epochs before the first run.

### 1h. Read in-scope files

Read every file in `<editable_files>` now for full context. Note the current metric
value (if any) so the baseline run has something to compare against.

### 1i. Initialise results.tsv

Create `<sandbox_root>/results.tsv` with the header row only:

```
iter	<metric>	status	description
```

(Tab-separated. Do not use commas — they break in description text.)

For `branches` mode, add a `commit` column after `iter`:

```
commit	<metric>	status	description
```

### 1j. Confirm and go

Print a summary of all resolved bindings and ask the user to confirm. Once confirmed,
begin the experiment loop immediately.

---

## 2. The experiment loop

**NEVER STOP** once the loop has begun. Do not ask "should I continue?", do not pause
between iterations, do not wait for permission. The human may be asleep. Run until
manually interrupted.

Each iteration follows this sequence:

### Step 1 — Snapshot current state (snapshots mode only)

In `snapshots` mode, before touching any file, copy every file in `<editable_files>`
into `<sandbox_root>/iter<N>/code_snapshot/`, preserving relative paths.

In `branches` mode, note the current git commit hash (short, 7 chars).

### Step 2 — Propose and apply a change

The **first** run is always the unmodified baseline — run `<entrypoint>` without
changing anything.

For every subsequent run, propose one focused experimental idea:
- What to change and why.
- Which file(s) in `<editable_files>` you are editing.

Apply the change directly to the files. Keep it focused — one idea per iteration.

**Simplicity criterion**: all else being equal, simpler is better. A marginal gain
that adds ugly complexity is not worth keeping. Removing code and matching or beating
the baseline is a win.

In `branches` mode: `git commit -am "<short description of idea>"` after editing.

### Step 3 — Run the experiment

```bash
# time-gated
<sandbox_root>/run_with_timeout.sh > <sandbox_root>/iter<N>/run.log 2>&1

# epoch-gated (no wrapper needed)
<entrypoint> > <sandbox_root>/iter<N>/run.log 2>&1
```

Redirect everything. Do not let output flood your context.

Hard limits:
- **Time gate**: if the run exceeds `2 × <budget_minutes>` minutes, kill it → crash.
- **Epoch gate**: if the run does not terminate after `<budget_epochs>` epochs +
  reasonable overhead, kill it → crash.

### Step 4 — Read the result

Extract `<metric>` from the log using whatever grep/parse pattern was agreed in §1a.

If the output is empty → the run crashed. Read the last 50 lines of `run.log` and
attempt a fix. If the fix is trivial (typo, missing import), fix and re-run **once**.
If the idea is fundamentally broken, log it as `crash` and move on.

### Step 5 — Log to results.tsv

Append one row to `<sandbox_root>/results.tsv` (tab-separated):

**snapshots mode**:
```
<N>	<metric_value>	<status>	<short description>
```

**branches mode**:
```
<commit>	<metric_value>	<status>	<short description>
```

`<status>` is one of: `keep`, `discard`, `crash`.

Use `0.000000` for `<metric_value>` on crashes.

Do not commit `results.tsv` to git (leave it untracked).

### Step 6 — Keep or revert

**If `<metric>` improved** (lower than the current best):
- Status → `keep`. Update the current-best record.
- In `branches` mode: stay on this commit (the branch advances).
- In `snapshots` mode: the code files remain as-is.

**If `<metric>` did not improve** (equal or worse, or crash):
- Status → `discard` (or `crash`).
- In `branches` mode: `git reset --hard HEAD~1` to revert the commit.
- In `snapshots` mode: restore every `<editable_file>` from
  `<sandbox_root>/iter<N>/code_snapshot/` back to the working directory.

### Step 7 — Repeat

Go to Step 1 for iteration N+1. If you are running out of ideas:
- Re-read the in-scope files for angles you missed.
- Try combining two near-misses from `results.tsv`.
- Try a more radical architectural change.
- Recall that the simplicity criterion means *deleting* something and staying equal
  is a valid win.

Never stop. The loop runs until the human interrupts you, period.

---

## 3. results.tsv format

Tab-separated. Never use commas in descriptions.

**snapshots mode** (header + example rows):
```
iter	<metric>	status	description
1	0.997900	keep	baseline
2	0.993200	keep	increase LR to 0.04
3	1.005000	discard	switch to GeLU activation
4	0.000000	crash	double model width (OOM)
```

**branches mode**:
```
commit	<metric>	status	description
a1b2c3d	0.997900	keep	baseline
b2c3d4e	0.993200	keep	increase LR to 0.04
c3d4e5f	1.005000	discard	switch to GeLU activation
d4e5f6g	0.000000	crash	double model width (OOM)
```

---

## 4. Hard constraints (never violate)

- Only edit files in `<editable_files>`. Everything else is read-only.
- Do not install new packages or add dependencies not already present.
- Do not modify the evaluation harness — the metric is the ground truth.
- Do not pause the loop to ask the human for direction.
- Do not let training output flood your context — always redirect to a log file.
- The sandbox (`<sandbox_root>/`) must be fully self-contained; no `../` escapes.
