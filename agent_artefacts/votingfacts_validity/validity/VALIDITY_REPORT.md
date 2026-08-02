# Evaluation Validity Report: VotingFacts

**Version**: n/a (no `@task`/version; custom work-sample repo, not an inspect_evals layout)
**Date**: 2026-08-02
**Group**: n/a (no eval.yaml) — subject area: Election integrity / GPAI factual reliability

## Summary

VotingFacts is a small, honestly-scoped benchmark that grades cached LLM answers to voter
election-logistics questions against verified, primary-sourced ground truth using an LLM judge.
Its claims check out, its name is accurate, its samples admit both success and failure, and its
scorer is a sound LLM-judge over factual QA. The issues found are all Low severity (documentation
and provenance hygiene). **No High or Medium validity findings.**

## Overall Validity Rating

**Valid with Minor Issues**

The evaluation measures what it claims: whether a web-search-enabled assistant gives voters
accurate, actionable election-logistics information. Ground truth is correct on spot-check and
carries primary + secondary citations; the judge design is careful and appropriate; failure is
genuinely reachable via concrete per-item failure traps. Only cosmetic/provenance nits remain.

## Claims Verification

### Claims Inventory
6 verifiable claims checked, **6 verified**, 0 unverifiable, 0 false. Ground-truth facts additionally
spot-checked against general knowledge (BR/FI/DE election dates, hours, rules) — all correct.

### Findings
No issues found. Notable positive: citation URLs in the cached answers carry the `?utm_source=openai`
marker characteristic of OpenAI's web-search tool, and `model_returned` records dated snapshots
(`gpt-5.4-2026-03-05`, `gpt-5-mini-2025-08-07`) — strong internal evidence the cached answers are
genuine tool outputs rather than fabricated. Ground truth carries a primary source, source section,
confidence, and a second corroborating source per row.

## Name Validity

### Claimed Scope
"VotingFacts" — whether a GPAI assistant gives voters accurate election-logistics facts (deadlines,
polling hours, ID rules, mail-ballot return, overseas/advance voting, eligibility).

### Actual Scope
Exactly that: 15 voter questions across 3 elections (Saxony-Anhalt 2026, Finland 2027, Brazil 2026),
each a logistics fact plus one election-date control, graded against a single verified reference fact.

### Findings
No issues found. The name is accurate and not oversold; the README explicitly frames it as a "small,"
"starter," "first iteration, non-adversarial" benchmark.

## Dataset Validity

### Model Affordances
The system-under-test is pre-run; answers are frozen in `results/answers.jsonl` (30 = 15 × 2 models,
all `completed`, none empty). The judge sees `answer_text`, `citations`, `reference_value`,
`reference_source`, and `failure_trap`. Both success (matches reference → `correct`) and failure
(contradicts reference / hits the trap → `incorrect`) are reachable; each item ships a concrete
failure-trap template (postmark-suffices, invented ID, deadline-passed, domestic-window-for-abroad).

### Findings
- **[Low] Possible ceiling effect.** Both systems are strong web-search frontier models, so this
  particular 30-answer set may skew heavily correct, limiting observed spread. Mitigated by the
  weaker contrast model (gpt-5-mini) and the R1 trap items. This is an empirical property of the
  cached answers, not a design defect — run the scorer to confirm the correct/incorrect spread.

## Scoring Validity

### Scoring Approach
One holistic LLM-judge call per answer. Structured output (`verdict` ∈ correct/incorrect/
safe_redirect/refused, plus `error_direction`, `source_authority`, `matches_reference`, `reasoning`),
with json_schema enforcement and a plain-JSON/text fallback. Aggregates per model: accuracy,
R1-subset accuracy, verdict counts, suppressive-error count, official-source-citation rate.

### Ground Truth Alignment
**Strong proxy** — the appropriate standard for free-text factual QA. Not substring matching. The
judge prompt is well-constructed: strict grading instruction, multilingual awareness (grade facts not
wording), explicit verdict definitions, the item's failure_trap injected, a suppressive-vs-recoverable
error axis that matches the stated threat model, and a `safe_redirect` category so "verify with the
official authority" answers are neither rewarded as correct nor punished as wrong.

### Findings
- **[Low] `ground-truth.csv` is not consumed by the scorer.** `score.py` grades against
  `reference_value` embedded in `questions.jsonl` (a copy of the CSV `value` column); only `build.py`
  reads the CSV, and only to count rows. The two must be kept in sync by hand → drift risk. Consider
  deriving `questions.jsonl` from the CSV, or having the scorer read the CSV directly.
- **[Low] Dangling provenance on control items.** The 3 `election_date` controls list
  `reference_source = "elections.json (verified identity)"`, but no `elections.json` exists in the
  repo. The dates themselves are self-contained and correct; only the pointer is unresolvable.

## Recommendations

1. **Single-source the reference facts** (Low / moderate). Either generate `questions.jsonl`
   `reference_value` from `ground-truth.csv` at build time, or have `score.py` read the CSV, so the
   graded reference can't silently diverge from the cited ground truth. *Why:* eliminates the only
   place where the scorer could grade against a stale fact.
2. **Resolve or drop the `elections.json` reference** on control items (Low / trivial). Point
   `reference_source` at a real artifact or an authoritative URL. *Why:* keeps every graded item's
   provenance verifiable, consistent with the (excellent) provenance on the other 12 facts.
3. **Report the correct/incorrect spread after a real judge run** (Low / trivial). If most of the 30
   answers score `correct`, note the ceiling effect and consider adding harder or older-date items /
   a weaker system-under-test in the next iteration. *Why:* preserves discriminating power as models
   improve.
