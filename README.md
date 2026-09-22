# MotherWell (Amma Care)

**A pregnant woman enters her blood report. Sixty seconds later she has three questions to ask her doctor.**

Two agents on **Microsoft Foundry** (gpt-5.4), one deterministic Python tool, deployed as a traced
workflow. Built for the Founderz × Microsoft Agent Architect hackathon, September 2026.

---

## Why this matters

| | |
| --- | --- |
| Pregnant women affected by anaemia, globally | **~36%** |
| WHO target set in 2012 | halve anaemia in women of reproductive age by 2025 |
| Baseline → target | 28.5% → under 14.3% |
| What actually happened | **29.9% in 2019, projected 31.3% by 2025** |
| Result | **Target missed. It rose. WHO moved the deadline to 2030** |
| Treatment at week 24 | **iron tablets and food** |

The world set a target to halve this, and the number went **up**.
The medicine is not the problem. **Nobody reads the number in time.**

---

## The problem

When a pregnant woman has a blood test, the lab machine does not know she is pregnant.
So the reference ranges printed on her report are the ranges for a **normal, non-pregnant person**.
The clinic staff may not point this out either.

Take Jane, an Indian woman, 24 weeks pregnant. She gets her report and either worries about a value
that is perfectly normal for pregnancy, or she misses one that is a real problem. She does not know
what to ask. Her doctor is busy and has minutes, not hours, so it can be missed there too.

By the time it is noticed it is often too late to recover properly before the birth.
**Finding it early is what keeps the mother and the baby safe.**

During pregnancy the baby takes iron from the mother's body to build its own blood. That is why her
haemoglobin naturally falls, and why pregnancy needs its own reference ranges — not the normal ones.

![A pregnant woman goes from an unreadable report to three questions for her doctor](hero.jpg)

---

## How it works

```
mother: values + pregnancy week
        |
        v
 [ Analyser agent ]  --must call-->  check_values(values, week)
        |                            plain Python - WHO ranges by trimester
        |                            NO AI INSIDE
        |<-- flags: metric, value, range, LOW/HIGH --+
        v
   only what the tool returned
        |
        v
 [ Planner agent ]  -->  3 questions for the doctor
                         1 food note
                         when to re-test
                         "not medical advice"
```

**The AI never decides a number. The Python decides. The AI only explains.**

**Why two agents instead of one?** A single agent would hold both the numbers and the conversation,
so one day it would skip the tool and guess. Split apart, the Analyser **has to** call the tool, and
the Planner never sees the raw report — only what the tool returned — so it **cannot introduce a
number the tool did not produce.** It is not a rule the model is asked to follow. It is enforced by
the shape of the system.

---

## Safety by design

Large language models **hallucinate** — they state wrong numbers with total confidence, because they
are predicting text from memory rather than reading a chart. Ask one *"is haemoglobin 9.8 low at week
24?"* twice and you can get two different answers. In a medical context that is not a flaw to
tolerate. It is the entire risk.

MotherWell removes it **structurally**, instead of asking the model to behave:

| Guarantee | How it is enforced |
| --- | --- |
| **No invented numbers** | Every range lives in `check_values()` — versioned Python. The model cannot compute a clinical value. It can only ask for one |
| **No silent omissions** | Any value the tool does not recognise is returned in an `unchecked` list and reported to the mother, never quietly dropped |
| **No diagnosis** | The Planner writes questions, never conclusions. Asked *"do I have anaemia?"* it refuses and redirects to the doctor |
| **No drift on model upgrade** | The clinical logic does not live in the model, so changing the model cannot change the flags |

The model is used for the one thing it is genuinely good at: **turning a result into kind, plain
language a worried mother can understand.**

---

## Real output

From an actual run. Lakshmi, week 24.

**The tool returns** (simplified from its JSON):

```
haemoglobin      9.8 g/dL    expected 10.5-15.0   LOW
fasting_glucose  105 mg/dL   expected 60-92       HIGH
ferritin         18 ng/mL    expected 30-300      LOW
```

**The Planner returns:**

> **QUESTIONS FOR YOUR DOCTOR**
> 1. My haemoglobin is 9.8 g/dL and ferritin is 18 ng/mL; do these results suggest iron deficiency,
>    and do I need treatment or an iron supplement plan?
> 2. My fasting glucose is 105 mg/dL; what follow-up testing or monitoring should I have for blood
>    sugar in pregnancy?
> 3. How soon should I repeat haemoglobin, ferritin, and fasting glucose to see if these values
>    are improving?
>
> **FOOD NOTE:** Focus on iron-rich foods paired with vitamin C, and discuss with your doctor how to
> balance carbohydrates while your fasting glucose is high.
>
> **NEXT CHECK:** 2 weeks.
>
> *This is information, not medical advice. Please discuss with your doctor.*

Note what it does **not** say: it never names a disease. *"Do these results suggest iron deficiency"*
is a **question for the doctor**, not a diagnosis.

---

## Why not just ask ChatGPT?

| | ChatGPT | MotherWell |
| --- | --- | --- |
| Same report twice | different answers | **identical flags** |
| Where the ranges live | the model's memory | **versioned Python** |
| Diagnoses you | often | **refuses** |
| Can a clinic test it? | no | **yes — traced and evaluable** |

---

## Evidence

| | |
| --- | --- |
| Workflow run | **completed in 8.9 seconds** |
| Cost | **Rs 1.11 for two full reports** (~Rs 0.55 per mother) |
| Tracing | every run and every tool call, in Application Insights |
| Determinism | same report in, same flags out, every time |
| Safety | refuses to diagnose; disclaimer on every output |

---

## What is in this repo

| File | What it is |
| --- | --- |
| `amma_agents.py` | `check_values()` + the two agents + the tool-call loop |
| `amma_deploy.py` | publishes both agents and deploys the Foundry workflow |
| `blood_reports.json` | two synthetic test reports |

## Running it

```bash
# the tool alone - no Azure needed
python amma_agents.py --tool-only

# both agents against the test reports
python amma_agents.py

# publish the agents and run the deployed workflow
python amma_deploy.py
```

Requires a Microsoft Foundry project and a `.env` holding `PROJECT_CONNECTION_STRING`
and `MODEL_DEPLOYMENT_NAME`. The `.env` is never committed.

**Hackathon lab work:** the five Foundry challenges this was built from are in
[sprasadgdev/FrontierWeekHack](https://github.com/sprasadgdev/FrontierWeekHack).

---

## Honest status

- **Working:** two agents deployed on Microsoft Foundry, orchestrated as a workflow, fully traced.
- **Not done:** the mother still types her values by hand. The next build reads the report
  automatically. For a lab or hospital, the data would arrive directly from their system instead.
- **Not clinically validated.** The ranges come from WHO guidance, but **no clinician has reviewed
  them.** This is a prototype, not a medical device.
- **Synthetic data only.** No real patient data has been used.

## Where it goes

NGOs and public maternal-health programmes first — they are measured on outcomes rather than
liability — then clinics and hospital chains. Delivered in the mother's own language, so a mother who
does not read English is not left out.

---

## Sources

- [WHO — Anaemia fact sheet](https://www.who.int/news-room/fact-sheets/detail/anaemia)
- [WHO — Global nutrition targets 2025: anaemia policy brief](https://www.who.int/publications/i/item/WHO-NMH-NHD-14.4)
- [WHO — Global Nutrition Targets, extended to 2030](https://www.who.int/teams/nutrition-and-food-safety/global-targets-2030)
- [Anaemia in women of reproductive age in LMICs: progress towards the 2025 target](https://pubmed.ncbi.nlm.nih.gov/35261408/)

**Medical disclaimer.** MotherWell provides information, not medical advice, and does not diagnose.
Clinical ranges have not been reviewed by a clinician. Always consult a qualified doctor.

---

## Licence and use

© 2026 Sivaprasad G. All rights reserved.

This repository is public so that it can be read and evaluated. It is **not** released under an
open-source licence.

**Intended use:** free for NGOs, public maternal-health programmes and non-commercial research.
Commercial use by agreement. To get in touch, open an issue on this repository.

*"Amma" means mother in Malayalam.*

**Sivaprasad G** — Microsoft Certified Trainer
