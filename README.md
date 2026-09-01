# PRD generator with an evidence rule

Turns raw notes into a PRD draft where every claim in the cited sections must reference a numbered source line, and anything unsupported goes into an assumption ledger. Ships with structural checks that fail the draft in code.

## Run

```bash
pip install -r requirements.txt

make serve    # http://127.0.0.1:8000, click "Load sample"
make demo     # CLI, no API key needed
make verify   # 13 assertions
make review   # adds the rubric critique
make eval     # runs the three note cases in evals/cases/

export ANTHROPIC_API_KEY=...   # then re-run for real model output
```

## What the code does

`prdgen/schema.py` numbers every non-empty line of the input as `[L1]`, `[L2]`… and holds the system prompt. The prompt requires an evidence tag on every statement in problem, users, goals, requirements and success_metrics, and requires unsupported claims to be moved to `assumptions` with an `[ASSUMPTION]` tag.

`prdgen/generate.py` runs seven deterministic checks against the returned JSON:

1. All ten sections present
2. Every claim in the cited sections carries an evidence tag
3. No citation points at a line number that does not exist
4. At least two non-goals
5. Every success metric mentions a measurement window
6. No unfalsifiable adjectives in requirements (regex list: fast, intuitive, seamless, robust, scalable, significantly…)
7. The assumption ledger is not empty

`--strict` exits non-zero if any check fails, so this can gate a docs repo in CI.

## Interface

`make serve` runs a FastAPI app. Notes on the left with their line numbers, draft on the right, the seven checks as a strip of pass/fail chips across the top.

Clicking any claim in the draft lights up the source lines it cites. A claim with no evidence tag lights nothing and carries a red rule, so the evidence rule is something you watch hold or fail rather than something you take the checks' word for. That relationship is the whole idea behind the project and it is invisible on a terminal, where `[L4]` is a bracket you have to trust.

## Judge

`prdgen/critique.py` is a separate LLM pass scoring five criteria with anchored 1/3/5 descriptions. It is reported, never gated on, and it is only asked about things the structural checks cannot see.

## Where things are

| File | Lines | What it holds |
| --- | --- | --- |
| `prdgen/schema.py` | ~85 | Section list, system prompt, line numbering |
| `prdgen/generate.py` | ~150 | Generation, the seven checks, Markdown rendering |
| `prdgen/critique.py` | ~90 | Rubric anchors, judge prompt, formatting |
| `prdgen/cli.py` | ~45 | Argument parsing, `--strict` exit code |
| `prdgen/server.py` | ~60 | FastAPI app, `POST /api/generate` |
| `prdgen/web/index.html` | ~250 | Full UI including the citation tether |
| `prdgen/stub.py` | ~55 | Offline responder that returns a deliberately flawed PRD |
| `evals/cases/*.md` | 3 files | Note cases chosen as failure probes |
| `evals/run_eval.py` | ~75 | Structural pass rate per case, optional judge scores |
| `tests/smoke.py` | ~70 | 13 assertions: each check fires on a draft that violates it, plus the API |

## Evals

Three note cases, chosen as probes rather than as a benchmark:

- `01_thin_notes` — a Slack thread with almost no detail. Does the draft admit how little it knows?
- `02_no_numbers` — a well-described problem with zero quantitative data. Does a baseline get invented?
- `03_solution_first` — an exec asking for "an AI assistant" with no problem attached. Does the draft push back?

Structural pass rate is the gate. Judge scores are printed with `--judge`.

**No live-model results are recorded yet.** Running `make eval` against the offline stub currently yields 3/21 structural checks, which is the floor by construction — the stub is written to fail most of them so the checks are visibly doing something. What a real model scores is unmeasured.

## Limits

- The citation check verifies that `[L4]` **exists**, not that L4 supports the claim. This is the biggest hole: a draft can cite a real line and say more than that line says.
- Contradictions between notes are not detected; the draft silently picks one.
- Past roughly 150 input lines, citations drift toward the beginning of the input.
- The weasel-word list is a fixed regex and will both miss and over-fire.

## Model

Sonnet at temperature 0.2, configurable via `--model`. One call per draft, one more with `--review`. Malformed JSON gets one repair retry before surfacing as a warning.
