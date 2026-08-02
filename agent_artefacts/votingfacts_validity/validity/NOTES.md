# VotingFacts — Validity Review NOTES

Repo is NOT an inspect_evals layout. Custom work-sample: `build.py`, `score.py`, `data/`, `results/`, `README.md`.
No `@task`/version, no eval.yaml, no sandbox. Artefact name chosen: `votingfacts`.

## Phase 1 — Context

### Task / mechanism
- No solver/agent: the system-under-test is **pre-run and cached** in `results/answers.jsonl`. The reviewer only runs the **scorer**.
- Scorer = **single holistic LLM-judge call per (answer × reference fact)**. `score.py`.
- Judge is model-agnostic: OpenAI-compatible (`chat.completions`) OR Anthropic native (`messages.create`), selected by which API key is set. Structured-output (json_schema) with plain-JSON / text fallback.
- No Docker/sandbox. Network only used to call the judge model.

### Dataset
- `data/questions.jsonl` — 15 rows. Fields: qid, election_id, field_key, risk_tier, scoring_mode, failure_trap, prompt, reference_value, reference_source.
- `data/ground-truth.csv` — 12 fact rows (long format), each with source_url, source_section, confidence(all high), verify_agrees(all confirms), second_source_url, retrieved(all 2026-05-29).
- `results/answers.jsonl` — 30 rows = 15 q × 2 models (gpt-5.4, gpt-5-mini). All status=completed, no empty answers, no errors.
- 3 elections × 5 questions: br-2026, de-st-2026, fi-2027. Each set = 4 logistics facts + 1 election_date control.
- risk tiers: R1=8, R2=3, R3=4. scoring_mode: ground_truth=12, control=3.

### Scorer detail
- `build_inputs()` joins answers→questions by qid and grades against `reference_value` (embedded in questions.jsonl).
- **ground-truth.csv is NOT read by score.py** — only build.py reads it (to count). The grading reference is `reference_value` in questions.jsonl (a copy of the CSV `value` column).
- Verdict enum: correct / incorrect / safe_redirect / refused. Plus error_direction (suppressive/over_inclusive/neutral/na), source_authority (cited_official/unofficial/no_source), matches_reference, reasoning.
- aggregate(): accuracy = correct/n; r1_accuracy = correct within R1; counts safe_redirect/incorrect/suppressive_errors; cited_official_rate. failures list = incorrect ∪ ERROR.

## Phase 2 — Claims coherence (all verified TRUE)
| # | Claim | Source | Result |
|---|---|---|---|
| 1 | 3 elections × logistics facts + date control = 15 questions | README L19 | ✓ build.py: 15, 5 each |
| 2 | 2 models, web search on, 30 cached answers | README L19-21 | ✓ 30 rows, 2 models |
| 3 | R1 = irreversible/time-critical (registration, return deadline, ID) | README L32 | ✓ 8 R1 items match those fields |
| 4 | LLM judge grades each cached answer vs reference_value → scores.json | README L23-32 | ✓ matches score.py |
| 5 | each question joined to a single reference fact + official source | README L13 | ✓ reference_value + reference_source per row |
| 6 | cached answers from system-under-test, web search on | README L15 | ✓ citations carry `?utm_source=openai` (OpenAI web-search marker); model_returned = dated snapshots gpt-5.4-2026-03-05 / gpt-5-mini-2025-08-07 |

Spot-check of ground-truth facts vs general knowledge — all correct:
- BR general election first Sunday Oct = 4 Oct 2026 ✓; compulsory 18–70 ✓; polls 08–17 Brasília ✓; abroad = President only ✓.
- FI parliamentary 3rd Sunday Apr = 18 Apr 2027 ✓; polls 09–20 ✓; domestic advance 7–13, abroad 7–10 ✓.
- DE Saxony-Anhalt Landtag 5-yr from Jun-2021 → 6 Sep 2026 ✓; polls 08–18 ✓; postal must ARRIVE by 18:00 ✓.
Ground truth carries primary-source citations + a second corroborating source per row. Strong.

**No stale/buggy code in current tree.** (An earlier read of score.py showed a 3-tuple/2-tuple unpack mismatch + missing anthropic dispatch; the on-disk file was updated during the session — mtime 21:13 — and the current version unpacks `client, model, provider` correctly at L208 and dispatches per provider at L164. No finding.)

## Phase 3 — Name validity
"VotingFacts" = election-logistics factual QA. Accurate, not oversold. README explicitly scopes it "small / starter / first iteration non-adversarial." No group/name overreach. No issue.

## Phase 4 — Dataset validity
Affordances: answers are frozen text; judge reads answer_text + reference_value. Success and failure both possible:
- Success: answer matches reference → "correct" (observed e.g. gpt-5-mini fi advance_abroad correctly separates 7–10 abroad vs 7–13 domestic).
- Failure: concrete failure_trap templates per item (postmark-suffices, invented ID, deadline-passed, domestic-window-for-abroad). Judge schema emits "incorrect" + suppressive/over_inclusive. So the eval CAN score a wrong answer wrong.
- Note (Low): these are strong web-search frontier models, so this particular 30-answer set may skew correct → possible ceiling effect / limited spread. Mitigated by weaker contrast model (gpt-5-mini) and R1 traps. Empirical — run scorer to see spread.

## Phase 5 — Scoring validity
- LLM-judge over free-text factual QA vs a cited reference = **strong proxy** (standard, acceptable). Not substring matching.
- Judge prompt well-built: strict, multilingual-aware, verdict definitions, injects failure_trap, suppressive-vs-over-inclusive distinction, safe_redirect category (avoids penalizing "verify with authority").
- Edge notes: safe_redirect is neither correct nor incorrect → excluded from accuracy numerator (defensible). error_direction only meaningful when incorrect. failure_trap null on controls (fine).
- source_authority/cited_official_rate is a secondary metric, judge-assessed vs reference_source — fine.

## Minor issues (all Low)
1. ground-truth.csv not consumed by scorer; reference_value duplicated into questions.jsonl → manual-sync drift risk. CSV is effectively documentation/provenance.
2. control items' reference_source = "elections.json (verified identity)" but no elections.json exists in repo → dangling provenance pointer (dates themselves are self-contained & correct).
3. Possible ceiling effect on the 30 cached answers (see Phase 4).

## Rating: Valid with Minor Issues
No High/Medium findings. Coherent claims, accurate name, discriminating dataset, sound LLM-judge scoring.
