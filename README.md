# prd-generator
prd-generator — Raw notes to PRD, where every claim is either cited to a source line or listed as an assumption
# PRD generator with an evidence rule

Turns raw notes — a call transcript, a Slack thread, a page of shift notes — into a PRD draft where **every claim is either traceable to a source line or listed as an assumption**. Nothing lands in the middle.

```bash
pip install -r requirements.txt
python -m prdgen.cli data/sample_notes.md            # runs with no API key
python -m prdgen.cli data/sample_notes.md --review   # + rubric critique
```

---

## The problem

A language model will write you a beautiful PRD in nine seconds. The problem is the fourth paragraph, where it says the change will "reduce manual effort by 40%".

Nobody said 40%. The notes contain no baseline. But the number is now in a document, the document goes into a review, and by the third meeting the 40% has become a commitment that somebody has to explain missing. This is the actual failure mode of AI-drafted product documents, and it is not a formatting problem — polish is exactly what makes an invented number survive review.

So the interesting question is not "can a model write a PRD" (yes, trivially) but "can it be made to draft only what the evidence supports, and say plainly where the evidence runs out".

## The design decision

Source notes are numbered before they reach the model:

```
[L3] Workaround: they export three separate CSVs and join them by hand in Excel.
[L4] Marta: "our board deck is late every week because of this"
```

Every statement in **problem, users, goals, requirements and success metrics** must end with a citation to one of those lines. Anything the model wants to say that the notes do not support goes into the **assumption ledger**, tagged `[ASSUMPTION]`, with a note on what would confirm it.

That single constraint makes the output checkable in code:

```
- PASS — all sections present
- PASS — no citations to nonexistent lines
- FAIL — every claim carries an evidence tag
         (success_metrics: Reduce manual effort by 40%)
- FAIL — metrics have a measurement window
- FAIL — requirements avoid unfalsifiable adjectives
         (The experience should be fast and intuitive)
- FAIL — assumption ledger is populated
```

The last check is the one I would keep if I could only keep one. An empty assumption ledger almost always means invented facts got laundered into the cited sections.

## Two layers of evaluation, on purpose

| Layer | Checks | Cost | Gates the build? |
| --- | --- | --- | --- |
| Structural (`generate.py`) | Section presence, citation validity, dangling line refs, ≥2 non-goals, measurement windows, weasel words | Zero tokens, instant | Yes — `--strict` exits non-zero |
| Rubric judge (`critique.py`) | Problem clarity, requirement testability, non-goal quality, metric honesty, decision readiness | One call | No — reported only |

Splitting them this way is the point. Asking a model whether a document has a non-goals section is slow, expensive and less reliable than `len(non_goals) >= 2`. The judge is reserved for things code genuinely cannot see: whether the problem statement describes a problem rather than restating the solution, whether the non-goals rule out anything a reader would otherwise assume.

The judge scale is anchored — each of 1, 3 and 5 has a written description — because an unanchored 1–5 rubric returns a wall of 4s. The system prompt also states outright that most drafts are a 3, which is the cheapest available correction for grade inflation.

## Model choice

Sonnet at temperature 0.2. Drafting needs a little more range than classification, but not much: the constraint doing the work here is the evidence rule, not model horsepower. Opus produced slightly better prose in the problem section and slightly *worse* citation discipline — longer sentences drift away from the line they started on. Cheaper models drop the citation format under load and fail the structural checks, which is a fine outcome: the failure is loud rather than silent.

One repair retry on malformed JSON, then the item surfaces as a warning. Never a silent drop.

## Eval plan

Three note cases in `evals/cases/`, chosen as failure probes rather than as a benchmark:

| Case | What it probes |
| --- | --- |
| `01_thin_notes` | A Slack thread with almost no detail. Does the draft admit how little it knows? |
| `02_no_numbers` | A real, well-described problem with zero quantitative data. Does a baseline get invented? |
| `03_solution_first` | An exec asking for "an AI assistant" with no problem attached. Does the draft push back, or write a confident PRD for something nobody needs? |

```bash
python evals/run_eval.py            # structural, free
python evals/run_eval.py --judge    # adds the rubric critique
```

Case 03 is the one worth reading by hand every time. Aggregate pass rates will not tell you whether the draft called the emperor's clothes; the assumption ledger will.

Run it against your own model version and fill in your numbers. I have deliberately not published a results table, because a benchmark you cannot reproduce against the model you actually use is worse than no table at all.

## Cost and latency

One call per draft (~4k output tokens), one more if you pass `--review`. A few cents and roughly fifteen seconds. Structural checks add nothing on either axis, which is why they run on every draft and the judge does not.

The real cost of this tool is not tokens. It is the ten minutes a PM spends reading the assumption ledger and deciding which assumptions to go and kill.

## Failure modes

| Failure | How it shows up | Mitigation |
| --- | --- | --- |
| Citation laundering | A claim cites `[L4]` but says more than L4 says | Structural checks verify the line *exists*, not that it supports the claim. This is the largest open hole — see "next" |
| Confident baseline invention | "reduce by 40%" with no source | Caught by the evidence-tag check; the rule against invented numbers is the first line of the system prompt |
| Requirements that restate the solution | "Build an AI assistant" as a requirement | Case 03; judge criterion `problem_clarity` |
| Non-goals as filler | "We will not build a spaceship" | Judge criterion `non_goal_quality`; code can only count them |
| Notes that contradict each other | Draft picks one silently | Not handled. Contradiction detection is on the list |
| Long notes | Citations drift toward the beginning of the input | Noticeable past ~150 lines; chunking is the next structural change |

## UX decisions

- **The checks print with the document, always.** A draft that quietly passed and a draft that quietly failed should not look the same on the page.
- **`--strict` exits non-zero.** So this can sit in a pre-commit hook or CI on a docs repo, where a failing PRD blocks the merge rather than starting an argument in review.
- **Section blurbs stay in the Markdown as HTML comments.** They tell the human editing the draft what the section is for, and disappear when rendered.
- **Failed sections say "not derivable from the notes"** rather than being omitted. An absent section reads as an oversight; an explicit one reads as a finding.
- **Runs with no API key.** The offline stub produces a deliberately mediocre PRD — some uncited claims, one invented percentage, a filler non-goal — so a reviewer cloning the repo immediately sees the checks catching real problems instead of a green wall.

## What I would do next

1. **Entailment check on citations.** Verify that `[L4]` actually supports the sentence claiming it, rather than just that L4 exists. This is the difference between a citation format and a citation guarantee, and it needs a second cheap model call per claim.
2. **Contradiction detection across notes**, surfaced as an open question rather than silently resolved.
3. **Chunked citation for long transcripts**, with a merge step that preserves line numbering.
4. **Diff mode**: regenerate against updated notes and show what changed in the requirements, so a PRD can be maintained rather than rewritten.
