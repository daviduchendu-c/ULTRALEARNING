# Day 3: The AI/ML Lifecycle

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 2 to 3 hours
**Level:** beginner. Built for anyone who finished Day 2 and wants to see how a model fits into a real project.

**By the end of this day, you will have:**

- A map of the 13 lifecycle stages, what each one produces and how each one breaks
- A filled-in project brief for a delivery case and a second brief for a case of your own
- A leakage check that produces an 87% accurate model from pure noise, and an explanation of why that result is fake

**What you need:**

- Nothing to install for most of the day
- For the leakage check: Python 3.11 or newer, `numpy==2.2.6`, `scikit-learn==1.6.1` and `scipy==1.15.2` (installed in the project section)
- A text editor for the project briefs

**Prerequisites:**

- Day 2: How Machines Learn (features, labels, loss, generalization, train vs test)
- Basic Python, for the leakage check

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [A short history, so the vocabulary makes sense](#a-short-history-so-the-vocabulary-makes-sense)
- [The lifecycle in one picture](#the-lifecycle-in-one-picture)
- [Problem definition](#problem-definition)
- [Data collection](#data-collection)
- [Exploration](#exploration)
- [Cleaning](#cleaning)
- [Features](#features)
- [Baseline](#baseline)
- [Training](#training)
- [Validation](#validation)
- [Testing](#testing)
- [Evaluation](#evaluation)
- [Deployment](#deployment)
- [Monitoring](#monitoring)
- [Iteration](#iteration)
- [Data leakage](#data-leakage)
- [Training, validation and test sets](#training-validation-and-test-sets)
- [The work around training](#the-work-around-training)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: your briefs and a leakage check](#project-your-briefs-and-a-leakage-check)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

On Day 2 you wrote a full training loop in about ten lines. In a real job those ten lines are the smallest part of the work. Before them, someone has to decide what problem is worth solving, find and clean the data, and decide what "good" means. After them, someone has to prove the model works on data it has never seen, ship it, watch it in production and fix it when the world changes. That whole path is the **ML lifecycle**.

Many failed AI projects have a sound algorithm and still fail, because a stage was skipped. Common examples are no baseline, a test set that leaked into training, and no monitoring after launch. Today you learn the stages, what each one produces, and how each one breaks. This is the map for every project in the remaining 97 days.

On the job, the lifecycle is how you answer "where is the project stuck?" When a manager asks why a promising model is not live, the answer is almost always a stage other than training.

---

## A short history, so the vocabulary makes sense

| Period | What happened | Why it matters today |
|---|---|---|
| 1996 | Fayyad, Piatetsky-Shapiro and Smyth describe the "knowledge discovery in databases" (KDD) process: selection, preprocessing, transformation, data mining, interpretation. | First widely cited step-by-step process for learning from data. |
| Late 1990s | A group of companies publishes [CRISP-DM](https://en.wikipedia.org/wiki/Cross-industry_standard_process_for_data_mining): business understanding, data understanding, data preparation, modeling, evaluation, deployment, drawn as a loop. | Still a widely used process model in data science teams. Its key idea is that you go round more than once. |
| 2015 | Google engineers publish "Hidden Technical Debt in Machine Learning Systems" (Sculley et al., NeurIPS). Its famous figure shows the ML code as a small box surrounded by data collection, feature extraction, serving and monitoring. | The clearest statement that the model is the small part. |
| Around 2018 onward | [MLOps](https://en.wikipedia.org/wiki/MLOps) emerges as a discipline: version data and models, automate training and deployment, monitor in production. | The lifecycle becomes software engineering. Module 10 covers it. |
| 2023 onward | LLM applications add new stages: prompt and retrieval design, evaluation sets for generated text, and guardrails. | The same lifecycle, with different tools (Modules 8 and 9). |

**Takeaway:** the process has been written down for almost thirty years, and each version keeps the same shape: define, prepare, model, check, ship, watch, repeat.

---

## The lifecycle in one picture

```mermaid
flowchart LR
    A[1 Problem definition] --> B[2 Data collection]
    B --> C[3 Exploration]
    C --> D[4 Cleaning]
    D --> E[5 Features]
    E --> F[6 Baseline]
    F --> G[7 Training]
    G --> H[8 Validation]
    H -->|try another model or setting| G
    H --> I[9 Testing]
    I --> J[10 Evaluation]
    J --> K[11 Deployment]
    K --> L[12 Monitoring]
    L --> M[13 Iteration]
    M -->|new data, new features, new problem| A
```

If Mermaid does not render where you read this, here is the same loop in text:

```
problem → data → explore → clean → features → baseline → train ⇄ validate
   ▲                                                              │
   │                                                              ▼
iterate ◄── monitor ◄── deploy ◄── evaluate ◄────────────── test (once)
```

Two loops matter. The small loop between training and validation runs dozens of times in a week. The big loop from monitoring back to problem definition runs over months, and it holds most of the long-term work.

**Reference image to find:** search "CRISP-DM diagram" and "Hidden Technical Debt in Machine Learning Systems figure 1". Put them side by side. The first shows the stages. The second shows how little of the system is the model.

We follow one case through every stage. A food delivery company wants to reduce the number of customers who get a late order without warning. The product manager asks, "Can we use AI for late deliveries?" Each stage below shows what that stage does to this case.

---

### Problem definition

A builder asks what a house is for before pouring the foundation. A family home and a warehouse need different ones.

A model is a function from inputs to outputs. Before you fit anything, you have to fix which output, for whom, at what moment, and what counts as good. Without those four, nobody can judge a number the model produces.

So problem definition is a written statement of the decision the model supports, the exact prediction (input, output, and when it is made), the metric, the minimum acceptable performance, and what happens when the model is wrong.

"Can we use AI for late deliveries?" becomes: *At the moment an order is placed, predict whether it will arrive more than 15 minutes after the promised time. If yes, show a revised time to the customer. Success: at least 70% of truly late orders are flagged (recall), and no more than 1 in 3 flags is a false alarm (precision of at least 0.67). A false alarm costs a slightly worse customer impression. A missed late order costs a support ticket.*

The usual mistakes are starting with a model instead of a decision, choosing a metric nobody in the business cares about, and leaving out the moment of prediction. That last omission leads straight to leakage, covered in the data leakage section. You can spot it when two people on the team describe the project differently.

Clients rarely arrive with a precise prediction, so turning "use AI for X" into a brief like the one above is often the first paid deliverable.

---

### Data collection

You cannot cook a dish if the ingredients are not in the kitchen.

A model can only learn from examples it has seen, so the examples and their labels have to exist, cover the conditions you will face, and be legal to use.

Data collection means finding, gathering or creating the examples and labels the model needs, and checking that you are allowed to use them.

For late deliveries you need past orders (restaurant, time of day, distance, items), courier data, weather, and the label (actual arrival minus promised time, where more than 15 minutes means late). Six months of orders gives about 2 million rows.

A frequent mistake is collecting data that does not cover the conditions you will face, such as six months of summer data for a winter launch. Another is ignoring privacy law: customer addresses are personal data under rules such as the EU's GDPR. Your Day 2 "and" problem was also a collection failure. Thirty hand-written rows did not cover enough ways of complaining. Check the date range and the groups in your data against the conditions of launch.

Teams hire data engineers and buy datasets because collection is slow, and the legal review often takes longer than the modeling.

---

### Exploration

Walk around a used car before you buy it.

Every later stage assumes things about the data: how many rows, how balanced the labels are, what a normal value looks like. Looking first is the cheapest way to find out which of those assumptions is false.

Exploratory data analysis (EDA) means counting rows, checking label balance, and looking at distributions, missing values and relationships before any modeling.

In the delivery data you find that 8% of orders are late. Late orders cluster on Friday evenings. You also find 3,000 orders with a negative delivery time, which is impossible.

The usual mistake is skipping EDA because the deadline is close. The 8% balance means a model that predicts "never late" is already 92% accurate, so you need that number before you celebrate a 93% model. If you cannot say what fraction of rows has each label, you skipped it. EDA notebooks are a standard part of data science interviews and take-home tests (Day 20).

---

### Cleaning

Washing and peeling vegetables before cooking.

Models treat every number as true. A wrong, duplicated or missing value does not announce itself, so a person has to find it and decide what to do about it.

Cleaning is fixing or removing data that is wrong, missing, duplicated or inconsistent, and recording every decision you make.

The 3,000 negative delivery times came from a timezone bug in one region. You fix the timezone instead of deleting the rows, because deleting them would remove an entire region from the data. You log the fix in a cleaning notes file.

One mistake is silently dropping rows with missing values, which can remove a whole group of customers. Another is filling gaps with a statistic computed on all rows before splitting, which is a small leak (see the data leakage section). Count rows per group before and after cleaning to catch the first one. Data teams often say most project time goes into cleaning and preparing data. Day 19 covers it.

---

### Features

Choosing which facts to tell a friend so they can guess how late your pizza will be: distance, rain, how busy the restaurant is.

A model sees only the columns you give it, and in production it sees only what exists at the moment of the request. A usable feature is one you can compute at that moment.

Feature engineering is turning cleaned data into model inputs that are available **at the moment the prediction is made**.

Good features for the delivery case are distance in km, hour of day, day of week, the restaurant's average prep time last week, and rain yes or no. A tempting bad feature is "courier pickup delay." It predicts lateness very well, but it only exists after the order is placed and the courier arrives. At prediction time it does not exist yet. That is leakage by time.

Any feature computed with information from after the prediction moment is a mistake, and so is any feature that is a disguised copy of the label. A feature that makes the model look too good to be true is the warning sign. Feature stores such as Feast exist so that training and production compute features the same way.

---

### Baseline

Before buying an expensive fitness tracker, count your steps with a cheap pedometer. If the expensive one is not clearly better, keep your money.

A score means nothing alone. "92% accurate" can be impressive or worthless depending on what a trivial method scores, so you need that trivial score first.

A baseline is the simplest reasonable predictor. Common choices are "always predict the majority class," "predict the average," or a one-line business rule. It is the bar every model must beat.

For the delivery case, baseline 1, "never late," gives 92% accuracy and 0% recall. Baseline 2, "flag every Friday-evening order over 5 km," catches 40% of late orders. The model must beat both.

The mistake is having no baseline, so a 92% accurate model looks impressive. You can spot it when nobody can say what a rule of thumb scores. Google's *Rules of Machine Learning* begins with "Don't be afraid to launch a product without machine learning." Sometimes the rule is good enough to ship, and a baseline is how you find that out.

---

### Training

The Day 2 loop: guess, measure, adjust.

Training fits the model's parameters to examples. The fit is only meaningful if the model has seen nothing but the training rows, because later stages judge it on rows it has not seen.

Training is fitting model parameters on the training set only.

In code that is `model.fit(X_train, y_train)`, here with a gradient-boosted tree model on 1.4 million training rows.

The mistake is starting with the most complex model. Google's Rule #4 says to "keep the first model simple and get the infrastructure right." A simple model that runs end to end teaches you more than a complex one stuck in a notebook, so a model that took weeks to build and has no path to production is the symptom. This is the stage most tutorials show, and it takes the smallest share of time on most real projects.

---

### Validation

Practice exams. You can take them as often as you like to decide how to study.

You need to compare models and settings, which means scoring them. If you score them on the training rows, you reward memorizing. So you hold out some rows and use them only to compare.

Validation is measuring candidate models on a held-out validation set (or with cross-validation) to compare them and choose hyperparameters.

You try three tree depths. Depth 6 gets 0.74 recall on validation, depth 12 gets 0.71. You pick depth 6.

The mistake is using the test set as the validation set. Every time you look at a score and change something, that data shapes your choices. Do it a hundred times on the test set and the test score stops being an honest estimate. You spot it when the "final" score was seen many times before the final report. Hyperparameter search tools such as Optuna and scikit-learn's `GridSearchCV` automate this loop. Day 36 covers it.

---

### Testing

The final exam. You sit it once.

Because validation scores were used to make choices, they are slightly flattering. One last measurement on rows that influenced no choice is the only honest estimate.

Testing is one measurement of the chosen model on a test set that played no part in any decision.

The depth-6 model gets 0.72 recall and 0.70 precision on the test set. That is slightly lower than validation, which is normal.

The classic mistake is running the test, not liking the result, changing the model and testing again. The test set is now a second validation set, and you have no honest estimate. If you must iterate, collect a fresh test set. Regulated industries such as credit and medical devices require documented, held-out test results before a model can be used.

---

### Evaluation

A car that passes the emissions test still has to be comfortable to drive.

A metric is a summary. The business goal from problem definition is bigger than any one number, so you check the model against the goal itself, including the places where an average hides trouble.

Evaluation is judging the model against the business goal from problem definition, with the metric as one input. That includes error analysis, performance by subgroup, cost of errors, speed and fairness.

Overall recall is 0.72, but for orders from new restaurants it is 0.35, because those restaurants have no prep-time history. You note this as a known limitation and propose a fallback rule for new restaurants.

The mistake is reporting one overall number. A model can be good on average and terrible for one city, one language or one group of customers, and you only see it if you slice the results. Evaluation reports that show subgroup performance are increasingly something clients and regulators ask to see.

---

### Deployment

Moving from a test kitchen to a real restaurant with customers waiting.

A model that lives in a notebook helps nobody. It has to run where the decision happens, fast enough and with the same inputs it had in training.

Deployment is putting the model where it makes real predictions: an API the app calls, a nightly batch job, or a model on a phone.

The delivery model is saved to a file, wrapped in a small web service, and called by the ordering app with a 50 ms time limit. It goes live for 5% of customers first, which is a canary release.

Two mistakes are common: computing features differently in production than in training (training/serving skew, from Day 2: How Machines Learn), and having no way to roll back to the previous model. If production accuracy is far below test accuracy on day one, suspect skew. Common tools include FastAPI, Docker and cloud ML services such as AWS SageMaker, Google Vertex AI and Azure ML, which are paid beyond their free tiers. Day 92 covers serving.

---

### Monitoring

A doctor does not stop checking on a patient after surgery.

The world keeps changing after launch, and the model does not. Monitoring tracks inputs, predictions, latency and, when labels arrive, real performance, with alerts when something shifts.

Two kinds of change matter. **Data drift** is when the inputs change over time (new customer types, new devices). **Concept drift** is when the relationship between inputs and the label changes. What counted as fraud last year may differ this year. [Wikipedia](https://en.wikipedia.org/wiki/Concept_drift) has a good overview.

Three months after launch, the delivery model's recall drops to 0.55. The input monitor shows that average distance jumped, because the company started serving a new suburb. That is data drift. Separately, a new courier app changed how pickups work, so the old patterns mean something different. That is concept drift.

The mistake is monitoring only whether the service is up. A model can answer quickly, in the right format, and still be wrong. Labels often arrive late (you only learn an order was late after it arrives), so you also need input monitoring to catch trouble early. Evidently is open source, and WhyLabs and Arize are commercial options. Day 93 covers monitoring.

---

### Iteration

A recipe gets better each time you cook it and write down what went wrong.

The first version is a hypothesis. Monitoring and evaluation tell you where it fails, and the fix may sit in any earlier stage.

Iteration is using what monitoring and evaluation taught you to go back to an earlier stage: collect new data, add features, retrain, or redefine the problem.

For the delivery case you add suburb data and a "new restaurant" flag, retrain, validate, test on fresh data and redeploy. You also learn that customers mostly care about orders more than 30 minutes late, so the problem definition changes too.

The mistake is retraining on a schedule without checking whether the new model beats the current one. Mature teams automate retraining and compare each new model against the production model before swapping them. Most of a model's working life is spent in this loop, which is why teams budget for maintenance and not only for the first build.

---

### Data leakage

A student who sees the exam questions the night before scores well. Their score is excellent and says nothing about what they know.

A model is judged on rows it has not seen. Leakage is any path by which information from those rows, or from the future, reaches training. Then the score measures access to the answer instead of skill.

Put simply, nothing from the future or from the test set may reach training. The table shows where it usually gets in.

| Where it gets in | Example | What prevents it |
|---|---|---|
| A feature from after the prediction moment | "courier pickup delay" for a prediction made at order time | Write down the prediction moment and check every feature against it |
| Preprocessing fitted on all rows | Scaling or filling gaps with statistics that include test rows | Fit preprocessing on training rows only, inside a pipeline |
| Feature selection on all rows | Picking the 20 "best" columns before cross-validation | Put selection inside the pipeline so each fold refits it |
| The test set used for choices | Picking the model with the best test score | Touch the test set once |

Leakage is not always easy to see. The leakage check in the project section shows a model that looks 87% accurate on labels that are coin flips.

The mistake is believing a very good score. Results that look too good should send you hunting for leakage before anyone celebrates. In industry, leakage is one of the most common reasons a model looks great in a notebook and fails in production.

---

### Training, validation and test sets

Think of a course: homework you practice on, quizzes you use to adjust how you study, and one final exam.

Each role needs rows that the others have not touched, so one dataset is cut into three parts.

The **training set** fits the parameters. The **validation set** compares models and chooses settings. The **test set** is touched once, at the end, to estimate real-world performance. A common split is 60%, 20% and 20%. [Wikipedia](https://en.wikipedia.org/wiki/Training,_validation,_and_test_data_sets) describes it in more detail.

With the delivery data of 2 million rows, a 60/20/20 split gives 1.2 million rows to train, 400,000 to validate and 400,000 to test. Picking the best of many models on the validation set makes the validation score optimistic. Each model's score includes some luck. The maximum picks the luckiest model as well as the best one, so the chosen score is biased upward. The test set exists to give a number that has not been through that selection.

A common mistake is splitting randomly when the data has a time order. For a model that predicts the future, train on earlier rows and test on later ones, or the model gets to learn from days it should be predicting. You can spot it when the split ignores the date column.

Every job that involves a model asks for a held-out score, and interviewers often ask how you split your data.

---

### The work around training

Count the stages above. Training is one of thirteen. Think of a restaurant where cooking is one hour of a ten-hour shift. Ordering, prep and cleanup take the rest.

Training is a small slice because everything that makes it work (the right data, the right target, a way to check it) has to be built around it. Here is a typical split of effort for a first version of a project like the delivery case:

| Work | Rough share of time |
|---|---|
| Problem definition, meetings, agreeing on the metric | 10% |
| Data collection, exploration, cleaning, features | 45% |
| Baseline, training, validation, testing | 15% |
| Evaluation and error analysis | 10% |
| Deployment, monitoring setup, documentation | 20% |

These numbers are not from a study. They are a typical shape, and they vary a lot by team. The pattern holds across teams. If you only practice `fit`, you are practicing a small slice of the job.

Real projects are messier than any diagram. You will jump from evaluation back to data collection, or discover in deployment that the problem definition was wrong. Treat the diagram as a checklist of things you must not forget, not as a strict order.

Job listings for AI engineers mention data pipelines, evaluation and deployment far more often than model architecture, which fits this picture.

---

## Hands-on exercises (about 60 minutes)

**Exercise 1: name the stage that failed (about 25 minutes).** For each story, write `Story number: stage, because <one sentence>`.

1. A churn model scores 0.99 accuracy. One of its features is "account closed date."
2. A face-unlock model works well in the office and badly for users with darker skin, who were barely in the training photos.
3. A demand forecast was excellent for a year, then badly wrong after a new competitor opened next door.
4. A team spent three months building a deep network. A rule written by the operations manager in an afternoon turned out to be just as good.
5. A team tried 200 model settings and reported the best score on the test set as their final result.
6. A model was trained with prices in dollars; the production system sends prices in cents.
7. After launch, nobody noticed for six months that the model was predicting "no fraud" for every transaction.

**Answer key:** 1 features (leakage: the feature is only known after churn). 2 data collection (unrepresentative data), caught too late because evaluation did not check subgroups. 3 monitoring (concept or data drift), followed by iteration. 4 baseline (there was none). 5 testing (the test set was used as a validation set). 6 deployment (training/serving skew). 7 monitoring (only uptime was watched, not predictions).

**Exercise 2: write a project brief (about 35 minutes).** Write the brief for the delivery case first, using the stage sections above as your source. Then write one for a case of your own, such as (a) a gym that wants to predict which members will cancel next month, or (b) a city that wants to predict which streetlights will fail this week. Use this structure:

```markdown
# Project brief: <name>

## Decision
What will someone do differently because of this prediction?

## Prediction
- Input: ...
- Output: ...
- Moment of prediction: ...

## Success metric
- Metric: ...
- Minimum acceptable value: ...
- Baseline to beat: ...

## Cost of errors
- False positive: ...
- False negative: ...

## Data
- Sources: ...
- Label definition: ...
- Features NOT allowed (unknown at prediction time): ...

## Risks
- Drift: ...
- Groups that could be served worse: ...
```

Then list three features you would want for your own case, and mark any that would not be available at prediction time. Keep the brief. Day 4 uses it.

---

## Project: your briefs and a leakage check

This day has no product to build. The project collects your two briefs in one repository and adds one short script that shows leakage with your own eyes. The script is optional, but it is the quickest way to feel why the test set must stay untouched.

**Goal:** a small repository that holds your briefs and a script that produces an 87% accurate model from labels that are coin flips, then shows the honest 50%.

### Repository layout

```
day-03-ml-lifecycle/
├── README.md              # which step let the selection cheat, and your k=200 prediction
├── briefs/
│   ├── delivery.md        # the brief for the delivery case
│   └── my-case.md         # the brief for your own case
├── leakage_demo.py        # leaky vs honest cross-validation on pure noise
└── requirements.txt       # the three libraries the script needs
```

### Step 1: create the folder and a virtual environment

```bash
mkdir day-03-ml-lifecycle
cd day-03-ml-lifecycle
mkdir briefs
python -m venv .venv
```

Activate it:

- macOS or Linux: `source .venv/bin/activate`
- Windows (PowerShell): `.venv\Scripts\Activate.ps1`
- Windows (Command Prompt): `.venv\Scripts\activate.bat`

Your prompt should now start with `(.venv)`.

### Step 2: add your briefs

Save the two briefs from Exercise 2 as `briefs/delivery.md` and `briefs/my-case.md`.

### Step 3: write the leakage script (`leakage_demo.py`)

Create `leakage_demo.py` with this code:

```python
"""Show how one lifecycle mistake (peeking at test data) produces a fake result.

The labels are random coin flips, so no model can honestly beat about 50% accuracy.
"""

import numpy as np
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import StratifiedKFold, cross_val_score
from sklearn.pipeline import make_pipeline

rng = np.random.default_rng(seed=0)
X = rng.normal(size=(100, 5000))   # 100 rows, 5,000 columns of pure noise
y = rng.integers(0, 2, size=100)   # random labels: there is nothing to learn

folds = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)

# WRONG: choose the 20 "best" columns using all 100 rows, then cross-validate.
# The selection step has already looked at the rows that will later be used for testing.
X_selected = SelectKBest(f_classif, k=20).fit_transform(X, y)
leaky = cross_val_score(LogisticRegression(max_iter=1000), X_selected, y, cv=folds)

# RIGHT: put selection inside the pipeline, so it is refit on the training folds only.
pipeline = make_pipeline(SelectKBest(f_classif, k=20), LogisticRegression(max_iter=1000))
honest = cross_val_score(pipeline, X, y, cv=folds)

print(f"leaky accuracy:  {leaky.mean():.2f}")
print(f"honest accuracy: {honest.mean():.2f}")
```

### Step 4: requirements and run

Create `requirements.txt`:

```text
numpy==2.2.6
scikit-learn==1.6.1
scipy==1.15.2
```

Install and run:

```bash
pip install -r requirements.txt
python leakage_demo.py
```

Expected output:

```text
leaky accuracy:  0.87
honest accuracy: 0.50
```

With 5,000 random columns and only 100 rows, some columns line up with the random labels by pure chance. `SelectKBest` found the 20 luckiest columns using all 100 rows, including the rows each fold later tests on. So every test fold was judged with columns chosen because they matched that fold's labels. The model learned nothing. The selection step cheated for it. Inside a pipeline, selection is repeated on the training folds only, and the honest answer appears: a coin flip, 0.50.

The same mistake shows up in real research with many measurements per patient and few patients, such as gene expression studies. scikit-learn's "Common pitfalls" page warns about it directly. Before you run it again, predict what happens to the leaky score if you change `k` to 200. Then run it and check.

### Step 5: write the README

In `README.md`, write full sentences for each of these:

1. Which step of `leakage_demo.py` let the selection cheat, and why?
2. What you predicted for `k=200`, and what happened.
3. Which stage of the lifecycle your own brief is weakest on.

### Done when

- Your Exercise 1 answers each name a stage and give a reason.
- Your brief for the delivery case has every section filled in, and your own brief has every section filled in too.
- If you ran the leakage script, the output matches the expected output above (or differs only because of library versions, which you have noted), and you can say which step let the selection cheat.
- You can explain, without notes, why the test set is used once and what the validation set is for.

---

## Check your understanding

1. Your model beats the "most frequent class" baseline by 2 points of accuracy. Is that good news? What else do you need to know?
2. Why is it a problem to fill missing values with the average of the whole dataset before splitting?
3. A feature is extremely predictive in training and useless in production. Give two possible causes.
4. Labels for loan default arrive 12 months after the loan is made. How does that change your monitoring plan?
5. Why does picking the best of many models on the validation set make the validation score optimistic?
6. When might the right outcome of the lifecycle be "do not use ML"?

**Answers (check after you try):**

1. Not necessarily. You need the class balance (if 95% of rows are one class, 2 points may be most of the possible gain or almost nothing), the metric that matters for the business (often recall or precision, not accuracy), and how a simple rule does.
2. The average includes test rows, so information about the test set enters the training data. It is a small leak, and the habit causes bigger ones.
3. It is not available at prediction time (leakage by time), or it is computed differently in production (training/serving skew). Drift since training is a third possibility.
4. You cannot wait a year to learn the model is broken, so you monitor inputs and prediction distributions now (data drift, share of loans flagged), and measure performance when labels arrive.
5. Each model's score includes some luck. Picking the maximum picks the model with the best luck as well as the best skill, so the chosen score is biased upward. The test set exists to measure without that selection.
6. When a simple rule meets the success metric, when there is not enough representative data, when errors are too costly to accept a statistical system, or when nobody will act on the prediction.

---

## Sources to read and watch

**Read (reference):**
- [Wikipedia: Cross-industry standard process for data mining](https://en.wikipedia.org/wiki/Cross-industry_standard_process_for_data_mining)
- [Wikipedia: Training, validation, and test data sets](https://en.wikipedia.org/wiki/Training,_validation,_and_test_data_sets)
- [Wikipedia: Leakage (machine learning)](https://en.wikipedia.org/wiki/Leakage_(machine_learning))
- [Wikipedia: Concept drift](https://en.wikipedia.org/wiki/Concept_drift)
- [Wikipedia: MLOps](https://en.wikipedia.org/wiki/MLOps)
- [Google: Rules of Machine Learning](https://developers.google.com/machine-learning/guides/rules-of-ml) by Martin Zinkevich. Read rules 1 to 4 today.
- "Hidden Technical Debt in Machine Learning Systems," Sculley et al., NeurIPS 2015. Search the title on the NeurIPS proceedings site (papers.nips.cc). Look at Figure 1 and read the introduction.
- [scikit-learn: Common pitfalls and recommended practices](https://scikit-learn.org/stable/common_pitfalls.html), especially the section on data leakage.

**Tutorials:**
- GeeksforGeeks: search "Machine Learning Lifecycle" and "Data Leakage in Machine Learning" on geeksforgeeks.org.
- [Made With ML](https://madewithml.com) by Goku Mohandas: a free course that walks through the full lifecycle, from design to monitoring, with code.

**Watch:**
- DeepLearning.AI: "Machine Learning in Production" on Coursera, taught by Andrew Ng (free to audit). Week 1 covers the project lifecycle and is the best single video match for today.
- [Full Stack Deep Learning](https://fullstackdeeplearning.com): the lecture on "ML projects" or "Lifecycle" from any recent course year.

**Practice:**
- [Kaggle Learn: Intermediate Machine Learning](https://www.kaggle.com/learn/intermediate-machine-learning): do the "Data Leakage" lesson and its exercise.

**Note on links:** open each one once. If a page has moved, search its title; the content is what matters.

---

## Tomorrow

Day 4, **AI Problem Formulation**, zooms in on the problem definition stage. Today you saw that a vague request ("use AI for late deliveries") has to become a precise prediction before anything else can happen. Tomorrow you learn the menu of shapes that prediction can take: classification, regression, ranking, clustering, generation, detection, segmentation, forecasting, recommendation, anomaly detection and control. You will map 20 real problems to the right shape, and see that the same dataset scored three different ways gives three very different answers to "is the model good?" Bring your two project briefs. You will check whether you chose the right formulation for each.
