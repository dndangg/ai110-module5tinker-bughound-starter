# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

BugHound follows a 5-step loop:

1. PLAN: Log a plan to do a quick scan and fix proposal.
2. ANALYZE: Detect issues.
	- Heuristic mode: uses pattern rules (print, bare except, TODO).
	- Gemini mode: asks Gemini for JSON issues, then parses/validates the output.
	- If Gemini output is malformed, it falls back to heuristics.
3. ACT: Propose a fix.
	- Heuristic mode: rule-based rewrite (for example print -> logging, except: -> except Exception as e:).
	- Gemini mode: asks Gemini for full rewritten code.
	- Guardrail added in this activity: if analysis already fell back, force heuristic fixer for consistency/safety.
4. TEST: Score risk in `assess_risk` using severity and structural checks.
5. REFLECT: Decide whether auto-fix is allowed (`should_autofix`) or human review is required.

---

## 3) Inputs and outputs

**Inputs:**

- Tested snippets:
	- `sample_code/cleanish.py`
	- `sample_code/print_spam.py`
	- `sample_code/mixed_issues.py`
	- `sample_code/flaky_try_except.py`
	- Weird case: comments-only snippet (`# TODO only` / `# just comments`)
- Input shape:
	- Short functions
	- try/except blocks
	- print-based scripts
	- TODO comments
	- no executable code (comments only)

**Outputs:**

- Detected issue types included:
	- Code Quality (print statements)
	- Reliability (bare `except:`)
	- Maintainability (TODO comments)
- Proposed fixes included:
	- Replacing `print(` with `logging.info(` and adding `import logging`
	- Replacing `except:` with `except Exception as e:`
	- Leaving clean code unchanged
- Risk report examples:
	- `cleanish.py`: low risk, score 100, auto-fix true (no changes needed)
	- `mixed_issues.py`: high risk, score 30 before change; high risk, score 20 after added "many lines changed" signal; auto-fix false
	- `flaky_try_except.py`: medium risk, score 55, auto-fix false
	- comments-only weird case: low risk, score 80, auto-fix true

---

## 4) Reliability and safety rules

List at least **two** reliability rules currently used in `assess_risk`. For each:

- What does the rule check?
- Why might that check matter for safety or correctness?
- What is a false positive this rule could cause?
- What is a false negative this rule could miss?

Rule A: Return statement removal check
- Check: If original code has `return` and fixed code does not, subtract 30.
- Why it matters: Losing returns often changes function behavior and can silently break callers.
- False positive risk: Some valid refactors may move behavior without a literal `return`.
- False negative risk: A `return` can still exist but semantics can change (for example wrong value).

Rule B: Large edit footprint check (added in this activity)
- Check: If unified diff has more than 6 changed lines, subtract 10.
- Why it matters: Bigger edits are more likely to include unintended behavior changes.
- False positive risk: A safe rename/reformat across several lines may be penalized.
- False negative risk: A dangerous 1-2 line change may still pass this check.

Rule C: Bare except modified check
- Check: If `except:` is changed, subtract 5 and require review.
- Why it matters: Exception handling changes affect control flow and error visibility.
- False positive risk: Many such changes are actually improvements.
- False negative risk: It does not verify whether the new exception type is truly appropriate.

---

## 5) Observed failure modes

Provide at least **two** examples:

1. A time BugHound missed an issue it should have caught  
2. A time BugHound suggested a fix that felt risky, wrong, or unnecessary  

For each, include the snippet (or describe it) and what went wrong.

1. Missed issue example
- Snippet: `sample_code/flaky_try_except.py`.
- What happened: It flagged bare `except:` but did not flag fragile file handling (`open(...); read(); close()` instead of `with open(...)`).
- Why this matters: Resource-handling and exception-safe cleanup risks are still under-detected.

2. Risky/wrong fix path example
- Snippet: `sample_code/print_spam.py` with `MockClient` before Part 4 guardrail.
- What happened: Analyzer fell back to heuristics, but fixer still called LLM client and could return unrelated placeholder rewrite text.
- Why this matters: This is a format/consistency failure that can produce unusable fixes.
- Mitigation added: In workflow, if analyzer fallback is used, force heuristic fixer.

---

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- What did Gemini detect that heuristics did not?
- What did heuristics catch consistently?
- How did the proposed fixes differ?
- Did the risk scorer agree with your intuition?

- In these runs, Gemini did not contribute additional detections because analyzer output repeatedly failed JSON parsing and fell back.
- Heuristics consistently caught print statements, bare `except:`, and TODO comments.
- Proposed fixes were effectively heuristic in most runs due to analyzer/fixer fallbacks.
- Risk scorer mostly matched intuition:
	- clear multi-issue code was high risk and not auto-applied,
	- clean code stayed low risk,
	- weird comments-only case felt slightly overconfident (low risk with TODO-only signal).

---

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

- What trigger would you add?
- Where would you implement it (risk_assessor vs agent workflow vs UI)?
- What message should the tool show the user?

Scenario: Any run where Gemini analysis or fix output is malformed/fallback-triggered should refuse auto-fix.

- Trigger: If analyzer fallback occurred, set `should_autofix` to `False` regardless of score.
- Implementation location: Agent workflow (`bughound_agent.py`) plus optional enforcement in `risk_assessor.py`.
- User message: "Model output was unreliable in this run. BugHound used fallback logic and requires human review before applying changes."

---

## 8) Improvement idea

Propose one improvement that would make BugHound more reliable *without* making it dramatically more complex.

Examples:

- A better output format and parsing strategy
- A new guardrail rule + test
- A more careful “minimal diff” policy
- Better detection of changes that alter behavior

Write your idea clearly and briefly.

Add a strict two-part analyzer contract: JSON schema validation + confidence flag.

- Keep current JSON parsing, but require each issue to pass schema checks (`type`, `severity`, `msg`) and add a top-level `confidence` number.
- If schema fails or confidence is below a threshold, skip LLM fixing and use heuristic-only behavior.
- This is low complexity, keeps the current architecture, and directly reduces unsafe confidence from ambiguous model output.
