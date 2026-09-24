# MotherWell: changes

## 24 September 2026: after review feedback on the Founderz platform

| Point raised | What changed | Proof |
| --- | --- | --- |
| "Next check in two weeks" and the food note may act as clinical advice | The Planner never states a retest interval, a dose or a supplement. "Next check" asks her doctor when to repeat the tests. A number of weeks chosen by the model was itself a clinical number, which broke this project's own rule | `eval_agents.py` fails any output with "in 2 weeks" or "Next check: 4 weeks" |
| "The Planner never sees raw values", yet questions quote 9.8 g/dL | The contract is stated exactly in `PLANNER_FIELDS`: flags carry the flagged value; in-range values, the report and her name never cross. The deployed workflow sent the full tool output and the (synthetic) mother's name; it now sends only the flag view | `tests/test_contract.py` |
| A `+91` number cannot decide Hindi | Only her own choice or her device's language setting is certain. A phone code or region is a guess, and her document opens by inviting her to change it | `tests/test_contract.py` |
| Escalation safety, extreme values | Haemoglobin below 7.0 g/dL (WHO 2024: severe) is marked `urgent` by the tool; her document opens with "contact your doctor or health worker today" | E19, E20, unit tests |
| Values on a trimester threshold | A result that would differ one week earlier or later is listed as `week_sensitive`; one question asks the doctor to confirm the week | E21 to E23, unit test |
| Missing units, conflicting formats, data-entry errors, mixed-language reports | 11 new evaluation cases | E19 to E29 |
| How the ranges are versioned and audited | `RANGES_VERSION` in `amma_tool.py`, carried on every tool output. The history is the git history of that one file | unit test |
| The deploy script reused existing agents | `amma_deploy.py` publishes a new version of both agents, so the workflow runs the instructions in this repo | |

`python -m pytest -q tests`: 92 passed.

## 23 September 2026: safety fixes before submission

| Fix | Why it matters |
| --- | --- |
| `check_values` moved to `amma_tool.py`, with zero AI imports (a test proves it) | "No AI inside" is now enforced, not just claimed |
| Units handled: g/L, mmol/L, nmol/L and more are converted; an implausible number with no unit is never guessed | Before: 98 g/L haemoglobin was flagged HIGH and 5.8 mmol/L glucose LOW, the opposite of the truth |
| Text values such as "9.8 g/dL" or "9,8" are parsed; anything unusable goes to `unchecked` | Before: a text value crashed the tool |
| Pregnancy week validated (1 to 42) | Before: week 60 was treated as trimester 3 |
| Fasting glucose of exactly 92 mg/dL is HIGH | WHO 2013 gestational diabetes threshold is 92 or above |
| "glucose", "blood_sugar", "thyroid" no longer mapped | A random or after-meal sugar must not be judged against the fasting range |
| Tool call enforced in code: an Analyser answer without `check_values` is rejected, retried once, then fails closed | "The Analyser has no choice" is now literally true |
| Planner receives the tool's JSON flags, not the Analyser's prose | "The Planner cannot introduce a number the tool did not produce" is now literally true |
| Groundedness check prints any number in the Analyser text that the tool never produced | Implements the groundedness metric from the design |
| Her region overrides a phone country code when they disagree | Before: a Kerala mother with a +91 number got Hindi |
| Tool loop capped, bad tool arguments handled | No infinite loop, no crash |
| NFHS-5 wording: Bihar and Kerala are the extremes among large states | Nagaland and Arunachal Pradesh are lower than Kerala |

### New files
- `amma_tool.py` the deterministic tool (run it alone: `python amma_tool.py`)
- `tests/test_amma_tool.py` 59 tests: every range, boundaries, units, bad input, determinism
- `eval/eval_cases.jsonl` 18 synthetic evaluation cases, including a diagnosis request
- `eval_agents.py` agent level metrics: tool call rate, groundedness, disclaimer, no diagnosis, refusal
- `.github/workflows/tests.yml` runs the tests on every push, no Azure needed
- `.env.example`, `.gitignore`
