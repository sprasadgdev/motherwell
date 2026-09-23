# MotherWell: review fixes (September 2026)

## What changed and why
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

## New files
- `amma_tool.py` the deterministic tool (run it alone: `python amma_tool.py`)
- `tests/test_amma_tool.py` 59 tests: every range, boundaries, units, bad input, determinism
- `eval/eval_cases.jsonl` 18 synthetic evaluation cases, including a diagnosis request
- `eval_agents.py` agent level metrics: tool call rate, groundedness, disclaimer, no diagnosis, refusal
- `.github/workflows/tests.yml` runs the tests on every push, no Azure needed
- `.env.example`, `.gitignore`

## Commit
```
pip freeze | grep -iE "azure-ai-projects|azure-identity|openai|python-dotenv" > requirements.txt
python amma_tool.py                  # tool alone
python -m pytest -q tests            # 59 passed
python amma_agents.py                # full run, check it still works with your project
python eval_agents.py                # optional: writes eval/eval_results.json
git add -A && git commit -m "Safety fixes: units, enforced tool call, tests, evaluation set, CI" && git push
```
`amma_deploy.py` needs no change: it still imports everything it used from `amma_agents.py`.
