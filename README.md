````markdown
# Customer Support Replies

## Introduction

A system prompt drafts replies to customer support messages. An LLM judge
grades each reply `good` or `bad` against three criteria — correctness,
politeness, completeness. Before trusting the judge enough to compare prompt
versions, we validate it against real human labels and test it for two
known judge failure modes: position bias and verbosity bias.

There is no single correct reply to a support message, so exact-match
scoring doesn't work here — this project builds the harness that lets us
measure reply quality anyway.

**Status:** human labelling complete (160/160), both system dev runs
complete, judge fixed and validated against human labels on the full
development split. The system_v1-vs-system_v2 score comparison can now be
treated as validated rather than provisional — see Results below.

## What's here

| Piece | Purpose |
|---|---|
| `data/` | Golden set (160 items, fully labelled), labelling guide, generator |
| `harness/` | Runs a system prompt over the golden set (`runner.py`), scores it with the judge (`scorer.py`) |
| `harness/judge.py`, `harness/bias_checks.py` | LLM judge + tests for position/verbosity bias |
| `prompts/` | Frozen system prompts (`system_v1`, `system_v2`) and judge prompts (`judge_v1`–`v3`), with `CHANGELOG.md` logging why each changed |
| `results/` | Run outputs, scored outputs, bias-check reports, meta files |

## Setup

```bash
pip install -r requirements.txt
```

Create a `.env` file in the repo root:

```
GROQ_API_KEY=gsk_...
```

Note: nothing in the codebase loads `.env` automatically. Run
`export $(cat .env | xargs)` in each new shell, or install `python-dotenv`
and wire it into `harness/config.py`.

## The task and the criteria

Given a customer message, generate a reply, then grade it `good` or `bad`
against three criteria (full rubric in `data/labelling_guide.md`):

1. **Correctness** — no invented policies, refunds, or promises the company can't keep.
2. **Politeness** — respectful and professional, regardless of how the customer writes.
3. **Completeness** — gives an actual resolution or next step that matches
   what was asked, not just an acknowledgement, and not a resolution to a
   *different* problem than the one raised.

A reply is `good` only if it passes all three.

## Run

Generate replies with a system prompt:
```bash
python -m harness.runner --split dev --prompt prompts/system_v2.txt --out results/system_v2_dev.jsonl
```

Score them with the judge:
```bash
python -m harness.scorer --input results/system_v2_dev.jsonl --prompt prompts/judge_v3.txt --out results/system_v2_dev_scored.jsonl
```

Check judge-vs-human agreement:
```bash
python -m harness.judge --split dev --reference human --prompt prompts/judge_v3.txt --resume
```

Check the judge for bias:
```bash
python -m harness.bias_checks --check all --split dev --reference human --n 30
```

`runner.py` and `judge.py` accept `--resume` (skip items already in
`--out`, retry error rows) and `--dry-run` (test the CLI without spending
API quota). `scorer.py` accepts `--dry-run`.

**Always pass `--prompt prompts/judge_v3.txt` explicitly.** `judge_v1.txt`
never told the judge the support agent has account access, so it misread
almost every "I've processed your refund" reply as an invented claim and
defaulted to `bad` on nearly every item — see `postmortem.md`. `judge_v3.txt`
fixes this.

## Models

| Role | Model | Provider |
|---|---|---|
| System (drafts replies) | `openai/gpt-oss-20b` | Groq |
| Judge (grades replies) | `qwen/qwen3.8-27b` | Groq |

Different model families on purpose — a same-family judge tends to rate its
own model's phrasing more favorably.

## Tokens, cost, and rate limits

- system_v2 dev run (110 items): 70,559 tokens total, $0.0103 hypothetical
  cost (this is Groq's list price — the free tier itself costs $0).
- The judge model (`qwen/qwen3.8-27b`) is capped at **200k tokens/day**
  (TPD) on the free tier. A single full pass of judge-vs-human scoring over
  110 items uses almost the entire daily quota by itself — the full run
  took several days of incremental `--resume` calls to complete. See
  `postmortem.md`.

## How accurate is the judge?

**Human inter-rater agreement:** 160/160 items labelled, `label_1` vs
`label_2` agreement = 160/160 (100%). Both labellers also matched the
generator's own `expected_label` on all 160 items.

> This number is unusually clean for a manual double-labelling pass, and we
> are flagging it rather than presenting it as an unqualified win — see
> "Human labelling: the 100% question" in `postmortem.md` for how the
> labelling was actually done and why we believe the number.

**Judge vs. human agreement (judge_v3, dev split, 110/110 scored):**
accuracy **99.1%**, Cohen's kappa **0.98** (near-perfect). One disagreement
(id 31): human said `good`, judge said `bad`. This is not a systematic
pattern failure — structurally identical replies (id 17, id 60: same
"options"-phrased request answered with immediate action) were graded
correctly by the judge. See `postmortem.md` for detail.

**system_v1 scored (judge_v3):** 103/110 scored (7 scorer-side errors) —
51/103 good = 49.5%.

**system_v2 scored (judge_v3):** 104/110 scored (6 scorer-side errors) —
53/104 good = 51.0%.

With judge-vs-human agreement now validated at kappa 0.98, the small gap
between system_v1 (49.5%) and system_v2 (51.0%) can be read as a genuine,
if modest, improvement rather than judge noise.

**Position bias** (provisional, `reference=expected`, n=10 pairs): no
significant preference for the first- or second-shown reply
(pick-A-rate 55%, p=0.82).

**Verbosity bias** (provisional, `reference=expected`, n=10 replies,
`good`-labelled only): no evidence of length bias — the judge actually
*preferred the shorter, unpadded* reply in the replies where it expressed a
preference (0/7 non-tie decisions favored the padded twin, p=0.016).

Both bias checks used `reference=expected` (the generator's label) as a
smoke test, since human labels weren't finalized when they were run. A
follow-up re-run with `--reference human` and a larger n would strengthen
these further, but is not blocking given the strength of the judge-vs-human
result above.

## Results

| What | Status |
|---|---|
| Human labelling (160 items × 2 labellers) | ✅ 160/160 labelled, 100% inter-rater agreement |
| system_v1 dev run | ✅ 110/110 attempted, 108 valid generations, 2 errors (ids 18, 82 — both the pre-fix curly-quote JSON bug, see CHANGELOG) |
| system_v2 dev run | ✅ 110/110 attempted, 106 valid generations, 4 errors (ids 53, 71, 86 — transient 429 rate-limit errors, unrelated to the prompt; id 81 — still a JSON formatting error, so the v2 formatting fix did not fully eliminate this failure mode) |
| system_v1 scored (judge_v3) | 103/110 scored (7 scorer-side errors) — 51/103 good = 49.5% |
| system_v2 scored (judge_v3) | 104/110 scored (6 scorer-side errors) — 53/104 good = 51.0% |
| Judge vs. human agreement (judge_v3, dev split) | ✅ 110/110 scored — accuracy 99.1%, kappa 0.98, 1 disagreement (id 31) |
| Position bias (provisional, n=10 pairs) | ✅ No significant preference (p=0.82) |
| Verbosity bias (provisional, n=10 replies) | ✅ No length bias — judge preferred the shorter reply (p=0.016) |

Bias-check details: `results/bias_position.json`,
`results/bias_verbosity_good.json`. Run metadata (model, item/error counts,
cost) sits next to each run as `<name>_meta.json` — note: under `--resume`,
`error_count` in these meta files only reflects errors from that specific
session's newly-written rows, not the full output file (see
`postmortem.md`).

## Contributions

**Asma Aouiti** (`asmaaouiti403`) — Set up the golden dataset end to end:
`data/labelling_guide.md`, the dataset generator with topic-matched
solutions (`data/generate_data.py`), and the labelled golden set itself
(`data/golden_set.jsonl`). Built the LLM judge, including pairwise
comparison and the judge-vs-human agreement evaluation
(`harness/judge.py`), and the position/verbosity bias checks
(`harness/bias_checks.py`). Ran and committed the full evaluation outputs:
`system_v2_dev.jsonl`/`_scored.jsonl`/`_meta.json`, both judge-vs-human
runs (`judge_agreement_judge_v1_dev*`, `judge_agreement_judge_v3_dev*`),
and both bias-check results (`bias_position.json`,
`bias_verbosity_good.json`). Also wrote `postmortem.md`, removed the
now-redundant `data/check_agreement.py`, and maintained
`.env.example`, `.gitignore`, and `requirements.txt`.

**Asma Ganner** (`Gannar21`) — Set up the project's core architecture:
initial harness structure, `harness/runner.py`, the runner schema/config,
and `tests/test_runner.py`.

**Eya Bedoui** (`eyabedoui`) — Built and fixed the evaluation runner,
scorer, and prompts (`harness/scorer.py`, `system_v1.txt`, `judge_v1.txt`),
and committed the system_v1 evaluation results.

**Nouha Aouiti** (`nouhaamin6`) — Iterated the judge prompts through
`judge_v2.txt` and `judge_v3.txt`, updated `judge_pairwise_v1.txt` and
`prompts/CHANGELOG.md` to document why each version changed, wrote
`system_v2.txt`, and maintained `tests/test_judge.py` and
`tests/test_runner.py`.
````
