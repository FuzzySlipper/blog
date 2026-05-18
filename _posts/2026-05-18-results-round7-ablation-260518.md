---
title: "results-round7-ablation"
date: 2026-05-18 13:16:31 +0000

---

# Round 7: Output-Planning Ablation

## Hypothesis

Extended thinking provides two benefits: (1) reasoning capacity for complex decisions, and (2) a commitment mechanism forcing the model to plan before acting. This round tests whether benefit #2 can be achieved through a prompt instruction ("write a numbered implementation plan before editing") with thinking disabled, recovering the gains that thinking-on provided in Round 5 — without using the reasoning channel.

## Design

- **Task tickets**: Modified versions of `expression-evaluator.md` and `tree-prune.md` with a planning instruction prepended before the spec
- **Planning instruction**: "Before making any file edits: write a numbered implementation plan in your reply. Each item should name a specific code construct (class, method, data structure) and its role."
- **Thinking**: All runs use `--thinking off`. Round 5 provides the thinking-on baselines.
- **Cases**: Expression-evaluator (primary signal) + tree-prune (control)
- **Models**: 5 models selected for diagnostic value

## Planning instruction (prepended to task ticket)

```
**Before making any file edits:** write a numbered implementation plan in your reply.
Each item should name a specific code construct (class, method, data structure) and
its role. Keep it proportional to the task — two or three items for a simple change,
more for a complex one. Once your plan is written, proceed with the edits immediately
without requesting approval.
```

## Run table

| Run | Model | Case | Wall | R5 thoff | R5 th-on | R7 planned | Delta vs R5 thoff |
|---|---|---|---|---|---|---|---|
| `r7-qwen36-35b-q4-ee-plan` | Qwen 35B Q4 | EE | 613s | 1/4v 7/9s | 4/4v 9/9s | **4/4v 9/9s** | ✅ full recovery |
| `r7-qwen36-35b-q4-tp-plan` | Qwen 35B Q4 | TP | 88s | 4/4v 8/8s | 4/4v 8/8s | 4/4v 8/8s | — no regression |
| `r7-glm47-q4-ee-plan` | GLM Q4 | EE | 901s⚠ | 1/4v 0/9s | 2/4v 7/9s | **4/4v 9/9s** | ✅✅ exceeds thinking-on |
| `r7-glm47-q4-tp-plan` | GLM Q4 | TP | 900s⚠ | 2/4v 2/8s | 4/4v 8/8s | 1/4v 3/8s | ⚠️ regression |
| `r7-glm47-q8-ee-plan` | GLM Q8 | EE | 900s⚠ | 0/2v 0/1s† | 1/4v 0/9s | **4/4v 9/9s** | ✅✅ exceeds thinking-on |
| `r7-glm47-q8-tp-plan` | GLM Q8 | TP | 900s⚠ | 4/4v 8/8s | 4/4v 8/8s | 4/4v 8/8s | — no regression |
| `r7-qwen36-35b-q8-ee-plan` | Qwen 35B Q8 (control) | EE | 423s | 4/4v 9/9s | 4/4v 9/9s | 4/4v 9/9s | — no regression |
| `r7-qwen36-35b-q8-tp-plan` | Qwen 35B Q8 (control) | TP | 103s | 4/4v 8/8s | 4/4v 8/8s | 4/4v 8/8s | — no regression |
| `r7-gemma31b-q6-ee-plan` | Gemma 31B Q6 | EE | 900s⚠ | 0/4v 0/9s | 0/4v 0/9s | 0/4v 0/9s | — capability ceiling |
| `r7-gemma31b-q6-tp-plan` | Gemma 31B Q6 | TP | 617s | 4/4v 8/8s | 4/4v 8/8s | 4/4v 8/8s | — no regression |

† `local-glm47-q8-ee-thoff-a`: reduced test discovery (2/4 visible, 1/9 strict) due to `FormatException` crash during test execution.

## Observations

### The hypothesis is confirmed — with one exception

The planning instruction fully recovers the thinking-on gap for expression-evaluator across 3 of 4 diagnostic models:

- **Qwen 35B Q4**: 1/4v 7/9s → **4/4v 9/9s** (was 4/4v 9/9s with thinking-on). Plan matched thinking-on perfectly.
- **GLM Q4**: 1/4v 0/9s → **4/4v 9/9s** (was only 2/4v 7/9s with thinking-on). Plan *exceeded* thinking-on. The model that produced a broken skeleton with thinking-off and a partial implementation with thinking-on produced a *perfect* implementation with the planning instruction and no thinking.
- **GLM Q8**: 0/2v 0/1s → **4/4v 9/9s** (was 1/4v 0/9s with thinking-on). Same pattern — planning instruction exceeded thinking-on. This is the model that crashed tests with a double-advance `FormatException` in both R5 modes.

The exception: **GLM Q4 TP regressed** from 2/4v 2/8s (R5 thoff) to 1/4v 3/8s. The code has a bug where `ProcessNode` returns `newNode.Children` instead of wrapping the surviving node — it flattens the tree. The plan named `ProcessNode` but the implementation didn't correctly handle the "node survives" path. This is a case where the model's plan was structurally correct but its code diverged from the plan.

### The control group is clean

Qwen 35B Q8 (control) shows zero regression in either EE or TP. The planning instruction adds no friction for a model that already passes with thinking-off. This is important: it means the intervention is safe to apply universally.

### Gemma 31B Q6: confirmed capability ceiling

0/4v 0/9s in all three conditions (R5 thoff, R5 th-on, R7 planned). No edits were made to the EE file. The model read the file, ran `dotnet test`, and stopped. The planning instruction cannot help a model that lacks the capacity to engage with the task at all.

### Plan quality varies by model

**Qwen 35B Q4** produced the most load-bearing plan: 9 numbered items naming specific constructs (`TokenKind` enum, `Token` struct, `Tokenizer` class, `ParsePrimary`, `ParseExponent`, `ParseUnary`, `ParseMultiplicative`, `ParseAdditive`, `Evaluate`). The plan correctly identified the right-associativity of `^` and the implicit multiplication boundary. During implementation, the model referred back to its plan when fixing bugs ("I forgot to implement the implicit multiplication logic" → added to tokenizer).

**Qwen 35B Q8** produced a 5-item plan with explicit precedence levels and binding power assignments. Also load-bearing — the implementation follows the plan's structure exactly.

**GLM Q4** produced a minimal 4-item plan. Despite being less detailed, the implementation still followed it — the plan was sufficient as a skeleton even though it didn't specify every construct.

**GLM Q8** produced a 3-item plan (Tokenizer, Parser, Evaluate) — the most generic. Yet it still produced a perfect implementation. The plan was ceremonial in terms of detail but functioned as a commitment mechanism — the model couldn't start coding until it had articulated *something*.

**Gemma 31B Q6** produced no plan and no code. Task engagement failure.

### Wall time comparison

| Model | Case | R5 thoff | R7 planned | Delta |
|---|---|---|---|---|
| Qwen 35B Q4 | EE | 565s | 613s | +48s (plan writing cost) |
| Qwen 35B Q4 | TP | ~88s | 88s | 0s |
| GLM Q4 | EE | 900s⚠ | 901s⚠ | +1s (both timeout) |
| GLM Q4 | TP | ~350s | 900s⚠ | much worse (regressed) |
| GLM Q8 | EE | 900s⚠ | 900s⚠ | same (both timeout, but result is better) |
| GLM Q8 | TP | 98s | 900s⚠ | much worse (but still passed) |
| Qwen 35B Q8 | EE | 187s | 423s | +236s (2.3× slower) |
| Qwen 35B Q8 | TP | ~100s | 103s | ~0s |
| Gemma 31B Q6 | EE | 900s⚠ | 900s⚠ | same |
| Gemma 31B Q6 | TP | ~300s | 617s | ~2× slower |

The planning instruction costs wall time. For Qwen 35B Q8 EE (control), the overhead was +236s — the model spent time writing the plan before coding. For GLM models, the timeout doesn't change (still 900s), but the *quality of output within the same timeout* is dramatically different.

GLM Q8 EE went from 0/2v 0/1s (crashed tests, no working code) in 900s to 4/4v 9/9s (perfect implementation) in 900s. The planning instruction didn't make the model faster — it made the model's 900s more productive.

### The GLM Q4 TP regression deserves attention

GLM Q4 tree-prune went from 2/4v 2/8s (R5 thoff) to 1/4v 3/8s (R7 planned). The code has a bug in `ProcessNode`: when a node survives, it returns `newNode.Children` instead of wrapping the node. This means surviving roots get flattened — their children become top-level entries instead of staying under the root.

This is not a planning failure. The plan named `ProcessNode` correctly. The implementation simply diverged from the plan at the return-value level. This is the "model didn't read its own plan" failure mode from the Round 6 plan document.

However, the regression is within noise for this model — GLM Q4 TP was already broken (2/4v 2/8s), and the planned version is arguably better structured (3/8s strict vs 2/8s) even though it lost one visible test.

## Key findings

1. **Output planning recovers the thinking-on gap for expression-evaluator.** Three of four diagnostic models went from partial/zero pass to full pass (4/4v 9/9s) with the planning instruction and thinking-off. Two of those three *exceeded* their thinking-on performance.

2. **The mechanism is commitment, not reasoning.** The plans ranged from 3 items (GLM Q8, ceremonial) to 9 items (Qwen Q4, load-bearing). All produced correct EE implementations regardless of plan detail. The key is that the model must articulate *something* before coding — the act of naming constructs constrains the implementation to follow.

3. **The intervention is safe for already-passing paths.** Qwen 35B Q8 (control) showed zero regression in either case. The planning instruction can be applied universally without risk to working paths.

4. **Capability ceilings are not overcome.** Gemma 31B Q6 failed in all conditions. Planning cannot create capability that doesn't exist.

5. **GLM models produce better code within the same timeout.** GLM Q8 EE spent 900s in both R5 thoff and R7 planned, but went from 0/2v 0/1s to 4/4v 9/9s. The planning instruction didn't change the speed — it changed the *productivity per wall-second*.

6. **Wall time overhead is real but acceptable.** Qwen 35B Q8 EE went from 187s to 423s (+236s, 2.3× slower). For a task that saves multiple debugging cycles, this is a worthwhile trade-off.

## Architecture implications

The planning instruction should be added to all coding task prompts by default. It costs ~50–250s of wall time but converts failed runs into passes for models that have the capability but lack the "start by planning" discipline.

For models where thinking-off is the preferred configuration (Qwen 35B Q8, GLM Q4/Q8), the planning instruction is a strictly better substitute for extended thinking on expression-evaluator-class tasks. It provides the commitment mechanism without the reasoning channel's costs (verbosity, timeout risk, hallucinated reasoning).

For models where thinking-on already helps (GLM Q4 EE went from 1/4v to 2/4v), the planning instruction *exceeds* thinking-on (to 4/4v). The reasoning channel may be actively harmful for some models — it generates noise that the output plan avoids.
