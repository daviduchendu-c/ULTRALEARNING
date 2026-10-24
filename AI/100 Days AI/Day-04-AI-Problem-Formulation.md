# Day 4: AI Problem Formulation

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 2 to 3 hours
**You will build:** a written formulation for each of 20 real problems (input, output, formulation, metric, baseline), and the formulation line for your two Day 3 briefs. There is also an optional script that frames one medical dataset three ways and shows that each framing gives a different verdict.
**You should know:** Day 2 (features, labels, supervised and unsupervised learning, reinforcement learning) and Day 3 (problem definition, baselines, metrics).
**You will install:** nothing for the written work. The optional script in section 6 needs Python 3.11 or newer, `numpy==2.2.6`, `pandas==2.2.3`, `scikit-learn==1.6.1` and `scipy==1.15.2`.

---

## 1. The big idea

Before you pick a model, you pick a **formulation**: the shape of the question the model answers. "Will this customer leave?" is classification. "How much will they spend?" is regression. "Which 50 customers should the retention team call first?" is ranking. These are three different models, trained on different labels, judged by different metrics, and they lead to different business actions, even when the data is the same.

Choosing the wrong shape is one of the costliest mistakes in ML, because everything downstream (labels, loss, metric, evaluation, deployment) is built on it. On Day 3 you wrote problem briefs. Today you learn the menu of formulations, so the "Prediction" line in those briefs is correct.

---

## 2. Background: where each formulation came from

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

## 3. The formulation finder

Use this flowchart on every new problem. It covers most cases. Section 4 covers the ones where it is ambiguous.

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

## 4. The 11 formulations

Each entry gives the intuition, the definition, one worked example with its metric and baseline, and the trap people fall into most often.

### 4.1 Classification

**Intuition.** Sorting mail into labeled trays.

**Definition.** Map an input to one category from a fixed set. The model usually outputs a probability per class, and a threshold (0.5 by default) turns that into a label. **Binary** means two classes, **multi-class** means exactly one of many, and **multi-label** means several tags at once. [Wikipedia](https://en.wikipedia.org/wiki/Statistical_classification)

**Worked example.** Input: an email. Output: spam or not spam. Metric: precision and recall on spam, because spam is rare (class imbalance). Baseline: "never spam."

**Trap.** Using accuracy on imbalanced data. The 0.5 threshold is also a business choice. A cancer screen should flag at a lower probability than a marketing-email filter. Multi-class and multi-label are different: a news article can be both "sport" and "business."

### 4.2 Regression

**Intuition.** Guessing a parcel's weight by lifting it.

**Definition.** Map an input to a continuous number. The training loss is usually MSE or MAE (Day 2). [Wikipedia](https://en.wikipedia.org/wiki/Regression_analysis)

**Worked example.** Input: house size, location and age. Output: sale price. Metric: mean absolute error in currency, because a manager can read "we are off by 12,000 on average." Baseline: the median price in the same area.

**Trap.** Predicting a number when the decision needs a category (does the house sell above asking, or not?). Prices also have a long tail, so a few mansions dominate MSE. Predicting the log of the price often works better.

### 4.3 Ranking

**Intuition.** A librarian who puts the three most useful books on the desk first, not every book in the building.

**Definition.** Given a query (or some context) and a set of candidates, order the candidates so the most relevant come first. The order matters, not the exact score. [Wikipedia](https://en.wikipedia.org/wiki/Learning_to_rank)

**Worked example.** Input: a search query and 1,000 matching documents. Output: an ordered list. Metric: precision@10, or NDCG (normalized discounted cumulative gain), which rewards putting the best items at the very top. Baseline: sort by keyword match count.

**Trap.** Training a classifier and calling it ranking without checking the top of the list, which is where the value is. Measuring the whole list when users only see the top 5.

### 4.4 Clustering

**Intuition.** Sorting a pile of photos into groups without anyone telling you the groups.

**Definition.** Partition unlabeled examples into groups so that members of a group are similar. It is unsupervised (Day 2). [Wikipedia](https://en.wikipedia.org/wiki/Cluster_analysis)

**Worked example.** Input: each customer's purchase history. Output: 5 segments, which a marketer then names ("weekend bargain hunters," "monthly bulk buyers"). There is no ground truth, so you use internal measures such as the silhouette score, plus a check on whether the segments help the business.

**Trap.** Treating clusters as natural categories. The algorithm returns exactly as many groups as you ask for. And if you have labels, classification is usually the better choice.

### 4.5 Generation

**Intuition.** A musician who has heard thousands of songs writes a new one in the same style.

**Definition.** Produce new content conditioned on an input: a prompt, an image or a class. The output space is huge (any paragraph, any image), so there is rarely one correct answer.

**Worked example.** Input: a customer email. Output: a draft reply. No single number works here. Teams use human ratings, rubric-based scoring by another model, and task checks (did the reply include the order number?). Baseline: a reply template.

**Trap.** Using generation for a task that is classification or extraction. If you need "which department handles this email," a classifier is cheaper, faster and easier to test than an LLM writing an answer. Generated text can also be fluent and wrong (hallucination).

### 4.6 Detection

**Intuition.** "Where's Waldo." You have to say Waldo is there and point to him.

**Definition.** Find every instance of the target objects in an input and give each one a location (in images, a bounding box) and a class. [Wikipedia](https://en.wikipedia.org/wiki/Object_detection)

**Worked example.** Input: a shelf photo. Output: a labeled box around each product. Metric: mean average precision (mAP), which checks both the class and how well each box overlaps the true box (intersection over union, IoU). Baseline: a pretrained general detector.

**Trap.** Using image classification ("this photo contains a person") when you need detection ("three people, here, here and here"). Every box has to be drawn by hand, which is expensive.

### 4.7 Segmentation

**Intuition.** A coloring book where every region gets exactly one color.

**Definition.** Assign a class to every pixel of an image (semantic segmentation). Instance segmentation also separates individual objects. [Wikipedia](https://en.wikipedia.org/wiki/Image_segmentation)

**Worked example.** Input: a medical scan. Output: a mask marking every tumor pixel. Metric: Dice coefficient or IoU between the predicted and true masks. Baseline: a threshold on pixel brightness.

**Trap.** Choosing segmentation when a bounding box would do. Masks cost far more to label. Pixel-average metrics also undervalue thin structures such as roads or blood vessels.

### 4.8 Forecasting

**Intuition.** Predicting tomorrow's weather from the past weeks of weather.

**Definition.** Regression on a time series. You predict future values from past values and from known future inputs (holidays, planned promotions). Time order matters.

**Worked example.** Input: daily sales for the past two years. Output: sales for each of the next 14 days. Metric: MAE or MAPE (mean absolute percentage error). Baseline: the same weekday last week (a seasonal naive forecast), which is surprisingly hard to beat.

**Trap.** A random train/test split. If you train on Wednesday and test on Tuesday of the same week, the model has seen the future. Split by time: train on the past, test on the most recent period. The future can also break the pattern (a pandemic, a new competitor).

### 4.9 Recommendation

**Intuition.** A friend who knows your taste suggests a film.

**Definition.** Personalized ranking. For one user, rank the items they have not interacted with by how likely they are to engage. It is often built from user-item interaction data (collaborative filtering) and item features.

**Worked example.** Input: a user's watch history. Output: the top 10 shows to display. Offline metric: recall@10 or NDCG on held-out interactions. Online metric: click or watch rate in an A/B test. Baseline: "most popular this week."

**Trap.** The cold start problem: new users and new items have no history. Feedback loops: the system only learns about what it already showed, so it narrows over time. Optimizing clicks can favor clickbait over satisfaction.

### 4.10 Anomaly detection

**Intuition.** A security guard who knows what a normal night sounds like and notices the one strange noise.

**Definition.** Flag inputs that differ from normal patterns. Use it when anomalies are so rare, or so varied, that you cannot collect enough labeled examples to train a classifier. [Wikipedia](https://en.wikipedia.org/wiki/Anomaly_detection)

**Worked example.** Input: per-minute readings from a factory motor (temperature, vibration). Output: an anomaly score. Above a threshold, the system raises an alert. Metric: precision of the alerts and the share of known failures caught. Baseline: a fixed limit ("vibration above 3 mm/s").

**Trap.** "Unusual" is not the same as "bad." A holiday sale looks anomalous to a fraud model. If you have plenty of labeled fraud cases, plain classification usually works better.

### 4.11 Control and decision-making

**Intuition.** Driving a car. Each steering choice changes what you see next, and the goal is arriving safely, not any single turn.

**Definition.** Choose a sequence of actions in an environment to maximize long-term reward. This is usually reinforcement learning (Day 2), planning or classical control. Unlike the other formulations, the model's outputs change its own future inputs.

**Worked example.** Input: the state of a building's heating system and the weather forecast. Output: heating settings every 15 minutes. Metric: energy cost over a month while keeping the temperature in range. Baseline: a fixed thermostat schedule.

**Trap.** Using RL where a one-shot prediction would do. Choosing which ad to show is often classification plus a simple rule. RL needs a simulator or many safe trials, and learning directly on a live system can be dangerous. DeepMind has reported using RL to cut cooling energy in Google data centers.

---

## 5. One problem, many formulations

The formulation is a **choice**, not a fact about the data. Take customer churn:

| Question the business asks | Formulation | Label | Metric | Action |
|---|---|---|---|---|
| Will this customer cancel in the next 30 days? | Binary classification | cancelled yes/no | recall, precision | send an offer to flagged customers |
| When will this customer cancel? | Regression (or survival analysis) | days until cancel | MAE in days | time the offer |
| Which 200 customers should the team call this week? | Ranking | cancelled yes/no, used for ordering | precision@200 | call the top of the list |
| What kinds of customers do we have? | Clustering | none | usefulness to marketing | design a plan per segment |

The data can be identical. The right row depends on the decision in your Day 3 brief. A team with a fixed call capacity of 200 needs ranking. A classifier tuned to a 0.5 threshold might flag 900 people, and nobody can call 900 people.

**Honest limitation:** real systems often combine formulations. A self-driving stack uses detection, segmentation, forecasting (where will that pedestrian be in 2 seconds?) and control at once. An LLM chatbot uses ranking (retrieval) and generation. Formulate each component separately.

---

## 6. Practical check (about 60 minutes)

**Exercise 1: 20 real problems (about 60 minutes).** For each problem, write one line in this form:

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

Score yourself. If you got fewer than 15 formulations right, reread section 3 and redo the ones you missed. If you mixed up multi-class and multi-label (problems 12 and 14), reread section 4.1.

Then write the formulation line for each of your two Day 3 briefs. Keep the 20 lines and the two briefs together. You will check them again on later days.

**Exercise 2 (optional): one dataset, three framings (about 30 minutes).** The diabetes dataset in scikit-learn has 442 patients with 10 measurements each (age, sex, body mass index, blood pressure and six blood serum values, already scaled). The target is a number that measures how far the disease progressed one year later. Save this as `three_framings.py` in any folder:

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

The data and the model stay the same. The verdict depends on the clinic's decision. Before you rerun it, answer two questions in writing. Which framing fits a clinic that can call exactly 20 patients a week? If you change `top_k` to 50, does precision@k go up or down? Then check your answers by running it.

**Done when**

- Your 20 lines match the answer key, or you have written a defense for each place you differ.
- Your two Day 3 briefs each have a formulation line.
- You can name the formulation, metric and baseline for a new product feature in under a minute.
- If you ran Exercise 2, you can say which framing matches the clinic's decision and why.

---

## 7. Check your understanding

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

## 8. Sources to read and watch

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

## 9. Tomorrow

Day 5, **AI Development Environment**, sets up the toolkit for the rest of the series. Tomorrow you learn what `pip install -r requirements.txt` actually does, why pinned versions matter, and how NumPy, pandas, Matplotlib, scikit-learn, PyTorch, Jupyter and Hugging Face fit together. You will also learn what a GPU and CUDA are, and why random seeds decide whether someone else can reproduce the numbers in your README.
