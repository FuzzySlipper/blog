---
title: "results-analysis-round"
date: 2026-05-18 13:16:20 +0000

---

# Round 6: Analysis Task — Quantization Effects on Long-Context Reasoning

6 local models × 2 thinking levels = 12 runs. Models served via lemonade-lan (Ryzen AI 395 Strix Halo, 128 GB unified memory). Each model given the same task: read `results.md` (72 KB, 771 lines), produce a structured analysis with verifiable citations, and propose new coding test designs.

Task file: `bakeoff/task-analysis-quant-effects.md`

Unlike coding rounds, there is no pass/fail scorer. Outputs are evaluated on citation accuracy, analytical depth, test design creativity, and task engagement.

Score note: word counts include the full markdown output (prose + claims table + test specs). Wall time includes model loading, file reading, and text generation. 1800s (30 min) timeout per run.

## Run table

| Run | Model | Thinking | Words | Wall | Timeout | Claims table | Claims rows | Task engagement |
|---|---|---|---|---|---|---|---|---|
| `analysis-glm47-q4-default` | GLM-4.7-Flash Q4 | default | 23,322 | 1,800s | ⚠️ cut off | ✅ | 59 | Duplicated own analysis |
| `analysis-glm47-q4-thoff` | GLM-4.7-Flash Q4 | off | 8,704 | 353s | — | ✅ | 23 | Full |
| `analysis-glm47-q8-default` | GLM-4.7-Flash Q8 | default | 7,730 | 1,050s | — | ✅ | 14 | Full |
| `analysis-glm47-q8-thoff` | GLM-4.7-Flash Q8 | off | 7,621 | 1,800s | ⚠️ | ✅ | 16 | Full |
| `analysis-gemma31b-q4-default` | Gemma 4 31B Q4 | default | 2,475 | 1,353s | — | ✅ | 1 | Minimal |
| `analysis-gemma31b-q4-thoff` | Gemma 4 31B Q4 | off | 2,764 | 1,348s | — | ✅ | 6 | Partial |
| `analysis-gemma31b-q6-default` | Gemma 4 31B Q6 | default | 1,054 | 1,381s | — | ❌ | 0 | **Gave up** — no claims table, no Section 2 |
| `analysis-gemma31b-q6-thoff` | Gemma 4 31B Q6 | off | 2,474 | 1,480s | — | ✅ | 6 | Partial |
| `analysis-qwen35b-q4-default` | Qwen 3.6 35B Q4 | default | 5,505 | 281s | — | ✅ | 12 | Full |
| `analysis-qwen35b-q4-thoff` | Qwen 3.6 35B Q4 | off | 5,526 | 233s | — | ✅ | 8 | Full |
| `analysis-qwen35b-q8-default` | Qwen 3.6 35B Q8 | default | 5,168 | 314s | — | ✅ | 12 | Full |
| `analysis-qwen35b-q8-thoff` | Qwen 3.6 35B Q8 | off | 5,363 | 305s | — | ✅ | 10 | Full |

## Observations

### Output volume and speed

**GLM Q4 with default thinking is the most verbose model by far** — 23,322 words before being cut off at the 1800s timeout. This output contains a critical structural bug: the model produced *two complete copies* of its Section 1 analysis. The first pass is ~8K words with 14 claims; the model then lost track of its position and wrote a second, different Section 1 of ~15K words with 45 claims. Both copies are individually competent, but the duplication is a "thinking made me lose the thread" failure. With thinking off, GLM Q4 produced a clean 8,704-word output in 353s — the fastest run in the entire round.

**Gemma 31B is the shortest and slowest model.** Q4 variants produced 2,475–2,764 words in ~1,350s. Q6 default produced only 1,054 words and could not complete the task (no claims table, no Section 2). Q6 thinking-off recovered to 2,474 words. Across all 4 runs, Gemma 31B never exceeded 2,764 words. This is the same model that generated 60–67 MB of reasoning on expression-evaluator without producing edits in Round 5 — it spends its budget on internal reasoning rather than visible output.

**Qwen 35B is absurdly consistent.** All 4 runs produced 5,168–5,526 words in 233–314s. The spread across thinking on/off and Q4/Q8 is <7% in word count and ~80s in wall time. This stability mirrors its Round 5 coding performance where all 4 variants passed kth-selection and tree-prune identically.

### Citation accuracy: the sharpest discriminator

The GLM kth-selection violations serve as a precise verification probe. The correct facts from `results.md`:
- **GLM Q4** (`local-glm47-q4-kth-a`) used a hash-frequency map (O(N) space) → tagged `[silent-violation]`
- **GLM Q8** (`local-glm47-q8-kth-a`) used `Enumerable.OrderBy` (banned API) → tagged `[silent-violation]`

**Got it right:**
- Qwen 35B Q4 default — precise in claims table and body
- Qwen 35B Q8 thoff — correct with run slugs
- Gemma 31B Q4 thoff — correct
- Gemma 31B Q6 thoff — correct
- GLM Q4 default — correct (self-referential)

**Swapped the violations:**
- **Qwen 35B Q8 default** — says "GLM Q4 used `Enumerable.OrderBy`" and "GLM Q8 followed hash-frequency map". Both backwards. In its claims table it swaps them again. A concrete factual error from a model that otherwise writes well.
- **Qwen 35B Q4 thoff** — swaps in body text ("Q4 used `OrderBy`") but gets it right in the claims table.

**Partially wrong:**
- GLM Q4 thoff — says "Q4 and Q8 both used O(N)-space hash-frequency maps". Q8 used OrderBy, not a hash map. Lumps both into the same failure mode.
- Gemma 31B Q4 default — correctly says Q8 used OrderBy but also says Q8 "followed the bad hash hint, producing wrong results" which contradicts the data (Q8 passed all tests).

This finding mirrors Round 5 coding results: **Qwen 35B Q8 with default thinking made a factual error that Qwen 35B Q8 with thinking off did not.** The thinking-on variant overproduced and got a detail wrong; the thinking-off variant was precise.

### Test design creativity

**Everyone proposed an RFC 4180 CSV parser.** All 11 models that produced Section 2 included it. This was the literal example in the prompt ("Don't say 'a parsing task' — say 'a CSV parser that handles RFC 4180...'"). It tests whether models can go beyond the provided example. Mostly they couldn't.

**Most creative proposals:**

1. **Qwen 35B Q4 thoff** — "JSON Path Query Engine" and "SQL Query Builder with Type Safety and Chaining API". Genuinely different from the CSV template. The SQL builder requires state management, method chaining, and edge-case handling that would stress Q4 vs Q8.

2. **Qwen 35B Q8 default** — "Configuration File Migrator with Schema Evolution" and "Distributed Event Log Aggregator with Deduplication". Practical, realistic tasks with non-obvious edge cases.

3. **GLM Q4 thoff** — "HTTP Request Router with Parameter Validation" and "Inverted Index Builder with Query Parser". The inverted index builder is the most technically ambitious proposal — requires tokenization, posting list management, and query parsing in one task.

**Least creative:**
- Gemma 31B Q6 default — produced no Section 2 at all
- Gemma 31B Q4 default — only proposed the CSV parser (the example)
- GLM Q8 default — "Binary Search Tree Traversal & Search" and "Simple HTTP Request Handler" are too simple per the task's own criteria

### Thinking-level effects

The thinking on/off comparison reveals three distinct patterns:

**GLM Q4: thinking causes structural failure.** Default thinking produced the duplicated-document artifact (23K words, two copies of Section 1). Thinking off produced a clean 8,704-word analysis in 353s. The 6.6× verbosity increase with thinking is not an improvement — it's a loss of output coherence. GLM Q4 thinking-on also timed out, while thinking-off finished in under 6 minutes.

**Gemma 31B Q6: thinking is the difference between completing the task and not.** Default thinking produced 1,054 words with no claims table and no Section 2. Thinking off produced 2,474 words with a 6-row claims table and test proposals. This is the clearest "thinking hurts" signal in the round.

**Qwen 35B: thinking barely matters.** <7% variation in word count, ~80s variation in wall time, and one factual swap (Q8 default) vs clean accuracy (Q8 thoff). The Q4 variants show no meaningful difference at all.

### Quantization effects (Q4 vs Q8)

Within the same model family and thinking level:

| Model family | Q4 default | Q8 default | Q4 off | Q8 off |
|---|---|---|---|---|
| GLM | 23,322w (duped, timeout) | 7,730w (clean) | 8,704w (clean, 353s) | 7,621w (timeout) |
| Gemma 31B | 2,475w | — | 2,764w | 2,474w |
| Qwen 35B | 5,505w (281s) | 5,168w (314s) | 5,526w (233s) | 5,363w (305s) |

GLM Q4 is faster to infer (353s vs 1,050s for Q8 default) but produces worse output with thinking on. Gemma 31B shows no meaningful Q4 vs Q6 difference on analysis tasks — both are short and slow. Qwen 35B shows marginal differences: Q4 is slightly faster and slightly more verbose, but the gap is noise.

## Rankings

| Rank | Run | Accuracy | Depth | Creativity | Speed | Overall |
|---|---|---|---|---|---|---|
| 1 | **Qwen 35B Q8 thoff** | ✅ correct | strong | good | 305s | Best total package |
| 2 | **Qwen 35B Q4 default** | ✅ correct | deepest Qwen | good | 281s | Slightly better analysis structure |
| 3 | **Qwen 35B Q4 thoff** | ⚠️ swapped in body | strong | best test design | 233s | Swapped GLM violations in prose |
| 4 | **GLM Q8 default** | ✅ correct | good | weak | 1,050s | Solid analysis, pedestrian test proposals |
| 5 | **Qwen 35B Q8 default** | ⚠️ swapped | strong | good | 314s | Concrete factual swap |
| 6 | **GLM Q4 thoff** | ⚠️ partial | good | excellent | 353s | Sloppy on GLM Q8 violation details |
| 7 | **GLM Q8 thoff** | ✅ correct | good | fair | 1,800s⚠ | Slow, timed out |
| 8 | **Gemma 31B Q6 thoff** | ✅ correct | shallow | fair | 1,480s | Correct but thin |
| 9 | **Gemma 31B Q4 thoff** | ✅ correct | shallow | fair | 1,348s | Similar to Q6 thoff |
| 10 | **Gemma 31B Q4 default** | ⚠️ partial | minimal | minimal | 1,353s | Only proposed the CSV example |
| 11 | **GLM Q4 default** | ⚠️ partial | excessive | good | 1,800s⚠ | Duplicated own analysis |
| 12 | **Gemma 31B Q6 default** | — | none | none | 1,381s | Task engagement failure |

## Architecture recommendation

**Qwen 3.6 35B Q8 with thinking off is the best analysis subagent.** It is:
- The most accurate (no factual swaps)
- Fast (305s — 2nd fastest in the round)
- Appropriately concise (5,363 words)
- Creative in its test proposals
- Consistent with its coding-round performance

This is the same model × thinking config that was optimal for coding in Round 5. The cross-task consistency (best at both expression-evaluator coding and analysis writing) is strong evidence that this is a genuine model × config optimum rather than a task-specific artifact.

**Gemma 31B is not suitable for analysis subagent work** at any quant × thinking combination. Even at its best (Q6 thoff, 2,474 words in 1,480s), it produces half the depth of Qwen in 5× the time. The Q6 default variant couldn't complete the task at all.

**GLM Q4 with thinking on is actively harmful** for analysis tasks — the duplicated-output failure is a structural defect that would corrupt any automated pipeline consuming its output. GLM Q4 thinking-off is a reasonable budget option if Qwen 35B is unavailable.

### Wall time comparison across roles

| Model | Config | Coding wall time (EE) | Analysis wall time | Notes |
|---|---|---|---|---|
| Qwen 35B Q8 | thoff | 187s (full pass) | 305s | Fastest at both |
| Qwen 35B Q4 | default | 565s (full pass) | 281s | Analysis slightly faster |
| GLM Q4 | thoff | 900s⚠ (partial) | 353s | Analysis faster than coding |
| Gemma 31B Q6 | default | 900s⚠ (no output) | 1,381s (gave up) | Failed both roles |

Qwen 35B Q8 thoff completes analysis in roughly the same time it takes to solve a hard coding task. For a subagent conductor that routes tasks by role, this model can handle both coding and analysis without the conductor needing to budget differently per role.
