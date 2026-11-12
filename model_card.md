# BugHound Mini Model Card (Reflection)

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a short Python snippet, propose a minimal fix, score the risk of that fix, and recommend automatic application or human review.
**Intended users:** Students and developers learning how to put an AI model inside a guarded coding workflow.

BugHound is a prototype, not a replacement for a test suite or code review. It only looks for a few simple patterns and does not execute the proposed code before making its recommendation.

## 2) How does it work?

The agent trace follows five stages:

1. **PLAN** records that the agent will scan the snippet and prepare a fix.
2. **ANALYZE** finds issues. With no client, it uses local heuristics for `print(`, bare `except:`, and `TODO`. With Gemini enabled, it sends the analyzer prompt to the model and accepts only a valid issue array with `type`, `severity`, and `msg` fields. Invalid output falls back to heuristics.
3. **ACT** proposes a fix. It uses simple replacements offline, or asks Gemini for a behavior-preserving rewrite. Empty or failed model output falls back to the heuristic fixer.
4. **TEST** runs the local risk assessor. This is a static risk calculation; it is not execution-based testing.
5. **REFLECT** turns the risk result into a decision. The current policy only allows auto-fix at a perfect score of 100; otherwise it recommends human review.

## 3) Inputs and outputs

I tested:

- `sample_code/cleanish.py`: already uses logging and has no detected issues. The original code was returned unchanged with score 100.
- `sample_code/mixed_issues.py`: detected a low-severity print issue, a high-severity bare `except:`, and a medium-severity TODO. The heuristic fix added logging and replaced the bare exception. Risk was 30/high, so auto-fix was refused.
- `sample_code/flaky_try_except.py`: detected the bare exception, proposed a broad `except Exception`, and scored 55/medium. It required review.
- A comments-only snippet: detected no issues, but the guardrail now scores it 70/medium and refuses auto-fix because there is no executable code.

Gemini analysis was attempted on `mixed_issues.py` and `flaky_try_except.py`. The returned analyzer text was not parseable as the required JSON array, so both runs logged a fallback to heuristics. Gemini’s empty fixer output also triggered the heuristic fixer fallback.

Outputs include detected issues, revised code, a unified diff, a risk score and reasons, an auto-fix boolean, and the complete agent trace.

## 4) Reliability and safety rules

### Severity deductions

High, medium, and low issues deduct 40, 20, and 5 points respectively. This makes a high-severity reliability finding dominate the decision. A false positive can block a harmless cleanup; a false negative can leave a real data-loss or exception-handling problem under-scored.

### Structural checks

The assessor deducts points when the revised code is much shorter, when return statements disappear, or when a bare exception is changed. These checks catch over-editing and likely behavior changes. They can produce false positives for intentional refactors, and they can miss semantic changes that preserve line count and the word `return`.

### Perfect-score auto-fix policy

Auto-fix is allowed only when the score is exactly 100. This is intentionally conservative: even a low-severity issue sends the change to review. The tradeoff is extra human work for safe formatting or logging changes.

### No-executable-code guardrail

Comments-only input loses 30 points and cannot auto-fix. Without this rule, returning the unchanged comments received a perfect score and looked like a successful automatic repair even though there was nothing to repair.

## 5) Observed failure modes

1. **Model format failure:** Gemini did not return the required JSON issue array during the two observed runs. The analyzer rejected the response and used heuristics. This preserved a usable result, but Gemini-specific findings were lost.
2. **Over-broad exception fix:** The heuristic fixer changes `except:` to `except Exception as e:`. That is safer than a bare catch in some cases, but it can still hide the specific exception the function should handle and does not prove behavior preservation.
3. **Unsafe confidence on comments-only input:** Before the new guardrail, a comments-only snippet had no issues, score 100, and an auto-fix recommendation. The new risk signal changes that to medium risk and human review.

## 6) Heuristic vs Gemini comparison

Heuristics were deterministic and consistently identified the three patterns they support. Gemini was more expressive in principle, but the observed responses did not satisfy the required output contract, so the reliability path fell back to the same heuristic findings. The risk scorer treated both paths identically once they produced the same normalized issue list. This shows why model integration needs an output contract and a tested fallback, not only a good prompt.

## 7) Human-in-the-loop decision

BugHound should refuse to auto-fix any change with a high-severity issue, a structural warning, malformed model output, or no executable input. The trigger belongs in `reliability/risk_assessor.py` because it should apply regardless of whether the proposal came from heuristics or Gemini. The UI should say: **“Human review required: the proposed change did not meet the auto-fix safety policy. Inspect the diff and run tests before applying it.”**

## 8) Improvement idea

Add an AST parse and compile check for the proposed Python before risk scoring. If parsing fails, return the original code, mark the risk high, and require review. This is a small change that would catch malformed rewrites that the current line-based checks can miss.
