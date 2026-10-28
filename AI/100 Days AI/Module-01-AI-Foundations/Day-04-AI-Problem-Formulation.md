# Day 4: AI Problem Formulation

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 2 to 3 hours
**Level:** beginner. For learners who have finished Days 2 and 3 and want to choose the right type of model for a problem.

**By the end of this day, you will have:**

- A written formulation (input, output, formulation, metric, baseline) for each of 20 real problems
- A formulation line for each of your two Day 3 briefs
- A small project that frames one medical dataset three ways (regression, classification, ranking) and shows that each framing gives a different verdict

**What you need:**

- Pen and paper, or a text editor, for the written work
- For the project script only: Python 3.11 or newer, `numpy==2.2.6`, `pandas==2.2.3`, `scikit-learn==1.6.1` and `scipy==1.15.2` (installed in the project section)
- Your two problem briefs from Day 3

**Prerequisites:**

- Day 2: How Machines Learn (features, labels, supervised and unsupervised learning, reinforcement learning)
- Day 3: The AI/ML Lifecycle (problem definition, baselines, metrics)

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Where each formulation came from](#where-each-formulation-came-from)
- [The formulation finder](#the-formulation-finder)
- [Classification](#classification)
- [Regression](#regression)
- [Ranking](#ranking)
- [Clustering](#clustering)
- [Generation](#generation)
- [Detection](#detection)
- [Segmentation](#segmentation)
- [Forecasting](#forecasting)
- [Recommendation](#recommendation)
- [Anomaly detection](#anomaly-detection)
- [Control and decision-making](#control-and-decision-making)
- [One problem, many formulations](#one-problem-many-formulations)
- [Hands-on exercises](#hands-on-exercises)
- [Project: one dataset, three framings](#project-one-dataset-three-framings)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Before you pick a model, you pick a **formulation**: the shape of the question the model answers. "Will this customer leave?" is classification. "How much will they spend?" is regression. "Which 50 customers should the retention team call first?" is ranking. These are three different models, trained on different labels, judged by different metrics, and they lead to different business actions, even when the data is the same.

Choosing the wrong shape is one of the costliest mistakes in ML, because everything downstream (labels, loss, metric, evaluation, deployment) is built on it. On Day 3 you wrote problem briefs. Today you learn the menu of formulations, so the prediction line in those briefs is correct.

On the job, the formulation is the first thing a senior engineer checks in a project proposal. A team that agrees on the formulation in week one avoids rebuilding the pipeline in week six.

---

## Where each formulation came from

| Formulation | Early roots | Modern form |
|---|---|---|
| Regression | Least squares, Legendre (1805) and Gauss; Galton's "regression to the mean" (1880s) | Gradient-boosted trees and neural networks predicting prices, times, demand |
| Classification | Fisher's linear discriminant (1936), using the iris flower data | Spam filters, medical diagnosis, image classifiers |
| Clustering | k-means, described by Lloyd at Bell Labs (1957) and published later | Customer segmentation, grouping documents by embeddings |
| Forecasting | Box and Jenkins' ARIMA methods (1970) | Demand, energy and traffic forecasting with ML and deep learning |
| Ranking | Information retrieval in the 1960s to 1990s; learning to rank in the 2000s | Search engines, feeds, retrieval for LLM applications (Day 78) |
| Detection and segmentation | Viola-Jones face detection (2001) | Deep networks such as YOLO and Mask R-CNN (Days 58 and 59) |
| Generation | Statistical language models and Markov chains | LLMs and diffusion models |

Each formulation is an old, well-studied question with its own metrics and tools. When you recognize the shape, the known solutions come with it.

---

## The formulation finder

Run every new problem through this flowchart. It covers most cases. The concept sections that follow handle the ones where it is ambiguous.

```
What does the output look like?
│
├── A category ─────────────────────────────── CLASSIFICATION
│     └── only "normal vs weird", few or no examples of weird ── ANOMALY DETECTION
├── A number ──────────────────────────────────── REGRESSION
│     └── a future value of something measured over time ────── FORECASTING
├── An ordered list ───────────────────────────── RANKING
│     └── the list is personalized to one user ──────────────── RECOMMENDATION
├── Groups, with no labels given ───────────────── CLUSTERING
├── Things + where they are in an image or signal ─ DETECTION
│     └── a label for every pixel ───────────────────────────── SEGMENTATION
├── New content (text, image, audio, code) ─────── GENERATION
└── A sequence of actions that change the world ── CONTROL / DECISION-MAKING
```

**Reference image to find:** search "scikit-learn choosing the right estimator" for the official cheat-sheet map. It starts from the same question: is the output a category, a quantity, or neither?

---

### Classification

Think of sorting mail into labeled trays. Every letter goes into exactly one tray, and the only question is which one.

A model that does this has to produce a score for each tray. It usually outputs a probability per class, such as 0.92 for "spam" and 0.08 for "not spam." Then a rule turns the probabilities into a decision, and the usual rule is a threshold, 0.5 by default.

Classification maps an input to one category from a fixed set. **Binary** means two classes, **multi-class** means exactly one of many, and **multi-label** means several tags at once. [Wikipedia: Statistical classification](https://en.wikipedia.org/wiki/Statistical_classification)

Take email. The input is an email and the output is spam or not spam. The metric is precision and recall on spam, because spam is rare (class imbalance). The baseline is "never spam," which would score high accuracy and catch nothing.

The usual mistake is using accuracy on imbalanced data. If 1 email in 100 is spam, "never spam" is 99% accurate and useless. You spot it when a model with impressive accuracy has a recall near zero on the class you care about. The 0.5 threshold is also a business choice: a cancer screen should flag at a lower probability than a marketing-email filter. Multi-class and multi-label are different, too, since a news article can be both "sport" and "business."

Spam filtering, fraud scoring, ticket routing and medical triage are all classification jobs you will find in production.

---

### Regression

Guessing a parcel's weight by lifting it is regression. The answer is a quantity on a scale, and being close counts for something.

The model learns a function from inputs to a number, and training pushes the gap between the predicted number and the true one down. That gap is the loss, usually MSE or MAE (Day 2).

Regression maps an input to a continuous number. [Wikipedia: Regression analysis](https://en.wikipedia.org/wiki/Regression_analysis)

For house prices, the input is size, location and age, and the output is the sale price. The metric is mean absolute error in currency, because a manager can read "we are off by 12,000 on average." The baseline is the median price in the same area.

A frequent mistake is predicting a number when the decision needs a category, such as whether the house sells above asking. You spot it when nobody downstream uses the exact number, only whether it crosses a line. Prices also have a long tail, so a few mansions dominate MSE, and predicting the log of the price often works better.

Pricing, delivery-time estimates and insurance claim sizes are regression problems in most companies that sell something.

---

### Ranking

A librarian who puts the three most useful books on the desk first, instead of handing you every book in the building, is ranking.

The model does not need a perfect score for each item. It needs the right order. If item A is more relevant than item B, A must come first, and how far apart the scores are does not matter.

Given a query (or some context) and a set of candidates, ranking orders the candidates so the most relevant come first. [Wikipedia: Learning to rank](https://en.wikipedia.org/wiki/Learning_to_rank)

For a search, the input is a query plus 1,000 matching documents, and the output is an ordered list. The metric is precision@10, or NDCG (normalized discounted cumulative gain), which rewards putting the best items at the very top. The baseline is sorting by keyword match count.

A typical mistake is training a classifier and calling it ranking without checking the top of the list, which is where the value is. Another is measuring the whole list when users only see the top 5. You spot it when overall accuracy looks fine and users still complain that results are bad.

Search engines, job boards, feeds and the retrieval step in LLM applications all depend on ranking.

---

### Clustering

Sorting a pile of photos into groups when nobody tells you what the groups are is clustering.

With no labels, the only thing the algorithm can use is similarity. It puts examples that look alike in the same group and examples that look different in separate groups. That makes it unsupervised (Day 2).

Clustering partitions unlabeled examples into groups so that members of a group are similar. [Wikipedia: Cluster analysis](https://en.wikipedia.org/wiki/Cluster_analysis)

Say the input is each customer's purchase history and the output is 5 segments, which a marketer then names ("weekend bargain hunters," "monthly bulk buyers"). There is no ground truth, so you use internal measures such as the silhouette score, plus a check on whether the segments help the business.

The usual mistake is treating clusters as natural categories. The algorithm returns exactly as many groups as you ask for, so ask for 5 and you get 5 even if the data has 2. You spot it when the groups change a lot after a small change in settings or sample. If you have labels, classification is usually the better choice.

Customer segmentation, grouping support tickets and organizing documents by embeddings are the common market uses.

---

### Generation

A musician who has heard thousands of songs and writes a new one in the same style is generating.

A generative model learns what typical outputs look like, then produces a new one from an input such as a prompt, an image or a class. The output space is huge (any paragraph, any image), so there is rarely one correct answer to compare against.

Generation produces new content conditioned on an input.

Take a customer email as the input and a draft reply as the output. No single number works here. Teams use human ratings, rubric-based scoring by another model, and task checks (did the reply include the order number?). The baseline is a reply template.

The common mistake is using generation for a task that is classification or extraction. If you need "which department handles this email," a classifier is cheaper, faster and easier to test than an LLM writing an answer. You spot it when the expected output is always one of a short fixed list. Generated text can also be fluent and wrong (hallucination).

Drafting replies, summarizing documents, writing code and creating images are the products built on generation today.

---

### Detection

In "Where's Waldo," finding Waldo is not enough. You have to say Waldo is there and point to him. Detection is that.

The model must answer two questions for every target object: what is it, and where is it? The location, for images, is a bounding box, a rectangle drawn around the object.

Detection finds every instance of the target objects in an input and gives each one a location and a class. [Wikipedia: Object detection](https://en.wikipedia.org/wiki/Object_detection)

For a shelf photo, the output is a labeled box around each product. The metric is mean average precision (mAP), which checks both the class and how well each box overlaps the true box (intersection over union, IoU). The baseline is a pretrained general detector.

The typical mistake is using image classification ("this photo contains a person") when you need detection ("three people, here, here and here"). You spot it when the product needs counts or positions and your model only returns one label per photo. Every box in the training set has to be drawn by hand, which is expensive.

Retail shelf audits, driver-assistance systems and warehouse robots rely on detection.

---

### Segmentation

A coloring book where every region gets exactly one color is segmentation. No pixel is left blank.

The model assigns a class to every pixel of an image. That is finer than a bounding box, because the output follows the true outline of each object.

Semantic segmentation assigns a class to every pixel. Instance segmentation also separates individual objects of the same class. [Wikipedia: Image segmentation](https://en.wikipedia.org/wiki/Image_segmentation)

For a medical scan, the output is a mask marking every tumor pixel. The metric is the Dice coefficient or IoU between the predicted and true masks. The baseline is a threshold on pixel brightness.

The usual mistake is choosing segmentation when a bounding box would do, since masks cost far more to label. Pixel-average metrics also undervalue thin structures such as roads or blood vessels. You spot the second problem when a model scores high overall but visibly misses the thin parts.

Medical imaging, satellite mapping and photo editing tools use segmentation.

---

### Forecasting

Predicting tomorrow's weather from the past weeks of weather is forecasting.

It is regression on a time series, with one extra rule: time order matters. You predict future values from past values and from inputs you already know about the future, such as holidays or planned promotions.

Forecasting is regression on data ordered in time, where the target is a future value.

Take daily sales for the past two years as the input and sales for each of the next 14 days as the output. The metric is MAE or MAPE (mean absolute percentage error). The baseline is the same weekday last week (a seasonal naive forecast), which is surprisingly hard to beat.

The classic mistake is a random train/test split. If you train on Wednesday and test on Tuesday of the same week, the model has seen the future. You spot it when test scores are excellent and live performance is poor. Split by time: train on the past, test on the most recent period. The future can also break the pattern (a pandemic, a new competitor).

Retailers, energy companies and logistics teams forecast demand, load and traffic every day.

---

### Recommendation

A friend who knows your taste suggests a film. You trust them because they know you, not because the film is popular.

This is ranking made personal. For one user, the system ranks the items they have not interacted with by how likely they are to engage. It is often built from user-item interaction data (collaborative filtering) and item features.

Recommendation is personalized ranking of items for a single user.

For a streaming service, the input is a user's watch history and the output is the top 10 shows to display. The offline metric is recall@10 or NDCG on held-out interactions. The online metric is click or watch rate in an A/B test. The baseline is "most popular this week."

The well-known mistakes are the cold start problem (new users and new items have no history) and feedback loops (the system only learns about what it already showed, so it narrows over time). Optimizing clicks can favor clickbait over satisfaction. You spot a feedback loop when variety on the home page shrinks month after month.

Streaming, e-commerce, social feeds and news apps run on recommendation.

---

### Anomaly detection

A security guard who knows what a normal night sounds like notices the one strange noise. That is anomaly detection.

The model learns what normal looks like and scores how far a new input is from it. You use it when anomalies are so rare, or so varied, that you cannot collect enough labeled examples to train a classifier.

Anomaly detection flags inputs that differ from normal patterns. [Wikipedia: Anomaly detection](https://en.wikipedia.org/wiki/Anomaly_detection)

For a factory motor, the input is per-minute readings (temperature, vibration) and the output is an anomaly score. Above a threshold, the system raises an alert. The metric is the precision of the alerts and the share of known failures caught. The baseline is a fixed limit ("vibration above 3 mm/s").

The common mistake is assuming "unusual" means "bad." A holiday sale looks anomalous to a fraud model. You spot it when alerts keep firing on harmless events. If you have plenty of labeled fraud cases, plain classification usually works better.

Predictive maintenance, network security and payment monitoring use anomaly detection.

---

### Control and decision-making

Driving a car is control. Each steering choice changes what you see next, and the goal is arriving safely, not any single turn.

Here the model's outputs change its own future inputs. So the model has to be judged on the whole sequence of choices, not on each one alone. That is the setting of reinforcement learning (Day 2), planning and classical control.

Control and decision-making chooses a sequence of actions in an environment to maximize long-term reward.

Consider the state of a building's heating system and the weather forecast as the input, and heating settings every 15 minutes as the output. The metric is energy cost over a month while keeping the temperature in range. The baseline is a fixed thermostat schedule.

The typical mistake is using RL where a one-shot prediction would do. Choosing which ad to show is often classification plus a simple rule. RL needs a simulator or many safe trials, and learning directly on a live system can be dangerous. You spot it when the team cannot say how the model will be tested before it touches the real system. DeepMind has reported using RL to cut cooling energy in Google data centers.

Robotics, building energy management, pricing and logistics scheduling are the usual markets for control.

---

### One problem, many formulations

Two doctors can look at the same patient and ask different questions: "Is this person ill?" or "Who in this waiting room should I see first?" The patient is the same, and the work that follows is not.

The formulation is a **choice**, not a fact about the data. What fixes it is the decision the business has to make. Take customer churn:

| Question the business asks | Formulation | Label | Metric | Action |
|---|---|---|---|---|
| Will this customer cancel in the next 30 days? | Binary classification | cancelled yes/no | recall, precision | send an offer to flagged customers |
| When will this customer cancel? | Regression (or survival analysis) | days until cancel | MAE in days | time the offer |
| Which 200 customers should the team call this week? | Ranking | cancelled yes/no, used for ordering | precision@200 | call the top of the list |
| What kinds of customers do we have? | Clustering | none | usefulness to marketing | design a plan per segment |

The data can be identical. The right row depends on the decision in your Day 3 brief. A team with a fixed call capacity of 200 needs ranking. A classifier tuned to a 0.5 threshold might flag 900 people, and nobody can call 900 people.

Real systems often combine formulations. A self-driving stack uses detection, segmentation, forecasting (where will that pedestrian be in 2 seconds?) and control at once. An LLM chatbot uses ranking (retrieval) and generation. Formulate each component separately, and watch for the mistake of forcing the whole product into one label. You spot it when one metric is supposed to judge parts of the system that do different jobs.

In project meetings, the question "what decision does this output feed?" is how experienced engineers turn a vague request into one of these rows.

---

## Hands-on exercises

**Exercise 1: 20 real problems.** For each problem, write one line in this form:

`N: formulation | input | output | metric | baseline`

Do not look at the answer key until you have written all 20. Some problems have more than one defensible answer. If yours differs, write one sentence defending it.

1. Predict the selling price of a used car from its make, model, age and mileage.
2. Decide whether a credit card transaction is fraudulent, given thousands of labeled past fraud cases.
3. Order 500 job applicants so recruiters read the most promising CVs first.
4. Group 10,000 news articles by topic with no topic labels.
5. Write a product description from a list of product attributes.
6. Find and box every pedestrian in a dashcam frame.
7. Mark every pixel of road in a satellite image.
8. Predict hourly electricity demand for a city for the next 48 hours.
9. Suggest five songs to a user based on their listening history.
10. Flag unusual login behavior on a company network where no attacks have been labeled.
11. Teach a robot arm to pick up objects of different shapes.
12. Tag a photo with every object type it contains (dog, beach, umbrella), possibly several.
13. Estimate how many minutes until a food delivery arrives.
14. Decide which of 30 support teams should receive an incoming ticket.
15. Detect defective products on a factory conveyor belt camera, where defects are rare and vary in kind.
16. Translate customer emails from Spanish to English.
17. Predict next quarter's revenue for each of 200 stores.
18. Choose prices for airline seats over the weeks before a flight to maximize total revenue.
19. Read the handwritten digits of a postal code from an envelope scan.
20. Order the results for a query in an internal company document search.

**Answer key:**

| # | Formulation | Metric | Baseline | Note |
|---|---|---|---|---|
| 1 | Regression | MAE in currency | median price for that make and age | |
| 2 | Binary classification | precision and recall on fraud | "never fraud" plus existing rules | labels exist, so not anomaly detection |
| 3 | Ranking | precision@k for "invited to interview" | sort by years of experience | check for bias against groups of applicants before using |
| 4 | Clustering | topic coherence, human review | group by keywords in the title | |
| 5 | Generation | human ratings, attribute coverage check | a fill-in template | |
| 6 | Object detection | mAP at an IoU threshold | a pretrained detector | |
| 7 | Semantic segmentation | IoU on the "road" class | brightness and color threshold | |
| 8 | Forecasting | MAE or MAPE | same hour last week | split by time |
| 9 | Recommendation | recall@5, then A/B test | most popular songs | cold start for new users |
| 10 | Anomaly detection | alert precision, known incidents caught | fixed rules (logins at 3 a.m. from a new country) | no labels, so not classification |
| 11 | Control (reinforcement learning or planning) | grasp success rate | scripted grasp per object type | train in simulation first |
| 12 | Multi-label classification | per-label precision and recall, or F1 | most common tags | not multi-class: several labels at once |
| 13 | Regression | MAE in minutes | distance divided by average speed | |
| 14 | Multi-class classification | accuracy, plus per-team recall | keyword rules | exactly one team per ticket |
| 15 | Anomaly detection (or detection, if defects are labeled with boxes) | defects caught, false alarm rate | pixel difference from a reference image | |
| 16 | Generation (sequence-to-sequence) | BLEU or COMET scores, plus human review | an off-the-shelf translation API | |
| 17 | Forecasting | MAPE per store | last year's same quarter, adjusted for growth | |
| 18 | Control / decision-making | total revenue per flight | fixed price schedule by days-to-departure | each price changes future demand |
| 19 | Multi-class classification (10 classes, one per digit) | per-digit and whole-code accuracy | none simple; a pretrained OCR tool | the classic MNIST dataset is exactly this |
| 20 | Ranking | NDCG@10 or precision@10 | keyword search (BM25) | |

Score yourself. If you got fewer than 15 formulations right, reread the formulation finder section and redo the ones you missed. If you mixed up multi-class and multi-label (problems 12 and 14), reread the classification section.

Then write the formulation line for each of your two Day 3 briefs. Keep the 20 lines and the two briefs together. You will check them again on later days.

---

## Project: one dataset, three framings

This is the GitHub project for Day 4. It is optional, but it turns the one-problem-many-formulations idea into numbers you can see. You will frame one dataset as regression, classification and ranking, and score each framing with its own metric.

**Goal:** a Python script that trains on the scikit-learn diabetes dataset (442 patients with 10 measurements each: age, sex, body mass index, blood pressure and six blood serum values, already scaled). The target is a number that measures how far the disease progressed one year later.

### Repository layout

```
day-04-problem-formulation/
├── README.md              # your 20 formulation lines, your two Day 3 formulation lines, and your answers to the questions below
├── three_framings.py      # one dataset scored as regression, classification and ranking
└── requirements.txt       # pinned library versions
```

### Step 1: create the folder and environment

```bash
mkdir day-04-problem-formulation
cd day-04-problem-formulation
python -m venv .venv
```

Activate it:

- macOS or Linux: `source .venv/bin/activate`
- Windows (PowerShell): `.venv\Scripts\Activate.ps1`
- Windows (Command Prompt): `.venv\Scripts\activate.bat`

### Step 2: requirements

Create `requirements.txt`:

```text
numpy==2.2.6
pandas==2.2.3
scikit-learn==1.6.1
scipy==1.15.2
```

Install with `pip install -r requirements.txt`.

### Step 3: the script

Save this as `three_framings.py`:

```python
"""Frame the same dataset as regression, classification and ranking, and score each one."""

import numpy as np
from sklearn.datasets import load_diabetes
from sklearn.linear_model import LinearRegression, LogisticRegression
from sklearn.metrics import accuracy_score, mean_absolute_error, recall_score
from sklearn.model_selection import train_test_split
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

# 442 patients, 10 measurements each. The target is a number that measures
# how much the disease progressed one year later (roughly 25 to 346).
X, y = load_diabetes(return_X_y=True, as_frame=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.25, random_state=0)

# Framing 1: regression. Output is a number. Metric: mean absolute error.
reg = LinearRegression().fit(X_train, y_train)
pred = reg.predict(X_test)
print("REGRESSION  predict the progression score")
print(f"  mean absolute error: {mean_absolute_error(y_test, pred):.1f} points")

# Framing 2: classification. Output is a category: "high" if above the training median.
threshold = y_train.median()
clf = make_pipeline(StandardScaler(), LogisticRegression())
clf.fit(X_train, y_train > threshold)
labels = clf.predict(X_test)
print(f"\nCLASSIFICATION  is the score above {threshold:.0f}?")
print(f"  accuracy: {accuracy_score(y_test > threshold, labels):.2f}")
print(f"  recall on 'high': {recall_score(y_test > threshold, labels):.2f}")

# Framing 3: ranking. Output is an order. A clinic can only call 20 patients this week.
top_k = 20
order = np.argsort(pred)[::-1][:top_k]          # the 20 highest predicted scores
truly_severe = y_test.to_numpy() > y_train.quantile(0.75)
hits = truly_severe[order].sum()
print(f"\nRANKING  which {top_k} patients should the clinic call first?")
print(f"  precision@{top_k}: {hits}/{top_k} = {hits / top_k:.2f}")
print(f"  random order would give about {truly_severe.mean():.2f}")
```

The ranking framing reuses the regression model's predictions to order patients. A model trained for one formulation can serve another, but you have to measure it with the second formulation's metric.

### Step 4: predict before you run

Before you run it, answer two questions in your README. Which framing fits a clinic that can call exactly 20 patients a week? If you change `top_k` to 50, does precision@k go up or down? Then check your answers by running the script.

### Step 5: run and read

Run `python three_framings.py`. Expected output:

```text
REGRESSION  predict the progression score
  mean absolute error: 45.1 points

CLASSIFICATION  is the score above 139?
  accuracy: 0.74
  recall on 'high': 0.75

RANKING  which 20 patients should the clinic call first?
  precision@20: 8/20 = 0.40
  random order would give about 0.18
```

Read the three results as three answers to "is the model good?"

- As regression, the model is off by 45 points on average on a scale of about 25 to 346. That sounds poor.
- As classification, it puts 74% of patients on the correct side of the median. That sounds fine.
- As ranking, 8 of its top 20 picks are truly severe cases, more than twice the share that random calls would find. For a clinic with 20 calls a week, that is a useful tool.

The data and the model stay the same. The verdict depends on the clinic's decision.

### Done when

- Your 20 lines from Exercise 1 match the answer key, or you have written a defense for each place you differ.
- Your two Day 3 briefs each have a formulation line.
- You can name the formulation, metric and baseline for a new product feature in under a minute.
- `python three_framings.py` runs and prints the expected output above, and your README says which framing matches the clinic's decision and why.

---

## Check your understanding

1. A team frames "which products will a user buy?" as multi-class classification over 50,000 products. What problem will they hit, and what formulation fits better?
2. Why is a random train/test split wrong for forecasting but usually fine for classifying photos?
3. You have 2 million transactions and 40 labeled fraud cases. Classification or anomaly detection? What changes if you later have 40,000 labeled cases?
4. A product manager asks for "AI that writes the ticket category." Should you build generation or classification? Why?
5. When does ranking need a different metric from classification, even if the same model produces the scores?
6. Why is control (decision-making) harder to evaluate offline than regression?
7. A segmentation model scores 95% pixel accuracy on road images. Why might it still be bad?

**Answers (check after you try):**

1. Most products have very few examples, the output space is huge, and the user sees only a short list. Recommendation (personalized ranking) fits better, measured by recall@k.
2. Time-series rows are ordered, and future rows contain information that would not be available when predicting. Photos are usually independent of each other, so a random split does not leak the future. (Even with photos, watch for near-duplicates of the same scene across splits.)
3. With 40 cases, there is too little to learn what fraud looks like, so start with anomaly detection and rules. With 40,000 labeled cases, supervised classification usually wins, since it learns what fraud specifically looks like rather than just "unusual."
4. Classification. The set of categories is fixed and known, so a classifier is cheaper and faster, gives a probability, and is easy to test. Generation adds the risk of inventing categories that do not exist.
5. Whenever only the top of the list matters. A classifier can have good accuracy across all items while placing the few best ones at positions 40 to 60, which users never see.
6. The model's actions change the data it sees next. A log of past decisions only shows what happened after the old policy's actions, not what would have happened after different ones.
7. If roads cover only 5% of pixels, predicting "not road" everywhere gives 95% pixel accuracy. Use IoU or Dice on the road class instead.

---

## Sources to read and watch

**Read (reference):**
- [Wikipedia: Statistical classification](https://en.wikipedia.org/wiki/Statistical_classification)
- [Wikipedia: Regression analysis](https://en.wikipedia.org/wiki/Regression_analysis)
- [Wikipedia: Learning to rank](https://en.wikipedia.org/wiki/Learning_to_rank)
- [Wikipedia: Cluster analysis](https://en.wikipedia.org/wiki/Cluster_analysis)
- [Wikipedia: Object detection](https://en.wikipedia.org/wiki/Object_detection)
- [Wikipedia: Image segmentation](https://en.wikipedia.org/wiki/Image_segmentation)
- [Wikipedia: Recommender system](https://en.wikipedia.org/wiki/Recommender_system)
- [Wikipedia: Anomaly detection](https://en.wikipedia.org/wiki/Anomaly_detection)
- [Wikipedia: Multi-label classification](https://en.wikipedia.org/wiki/Multi-label_classification)
- Google for Developers: search "Introduction to Machine Learning Problem Framing" on developers.google.com. It is a short free course on this topic.
- [scikit-learn: Choosing the right estimator](https://scikit-learn.org/stable/machine_learning_map.html)
- Rob Hyndman and George Athanasopoulos, *Forecasting: Principles and Practice*, free online at [otexts.com/fpp3](https://otexts.com/fpp3/). Read chapter 1 for how forecasting problems are framed.

**Tutorials:**
- GeeksforGeeks: search "Classification vs Regression in Machine Learning" and "Types of Machine Learning" on geeksforgeeks.org.
- [Hugging Face Tasks](https://huggingface.co/tasks): a catalog of ML tasks (classification, detection, segmentation, generation and more), each with example inputs, outputs and metrics.

**Watch:**
- StatQuest with Josh Starmer: search "Machine Learning Fundamentals" videos on the [StatQuest channel](https://www.youtube.com/@statquest).
- Stanford CS231n (Convolutional Neural Networks for Visual Recognition): the lecture on "Detection and Segmentation" (search the title on YouTube), for a visual explanation of the difference between classification, detection and segmentation.

**Practice:**
- [Kaggle Competitions](https://www.kaggle.com/competitions): open five competitions and, for each, write down its formulation and its official evaluation metric. Each competition's "Evaluation" tab states the metric.

---

## Tomorrow

Day 5, **AI Development Environment**, sets up the toolkit for the rest of the series. Tomorrow you learn what `pip install -r requirements.txt` actually does, why pinned versions matter, and how NumPy, pandas, Matplotlib, scikit-learn, PyTorch, Jupyter and Hugging Face fit together. You will also learn what a GPU and CUDA are, and why random seeds decide whether someone else can reproduce the numbers in your README.

**Next:** [Day 5: AI Development Environment](Day-05-AI-Development-Environment.md)
