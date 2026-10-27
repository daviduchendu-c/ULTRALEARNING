# Day 2: How Machines Learn

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 3 to 4 hours
**Level:** beginner to intermediate. For learners who finished Day 1 and can run Python scripts. No calculus is needed.

**By the end of this day, you will have:**

- A printout of the numbers your Day 1 learned bot actually learned, and an explanation of why its strongest complaint word is "and"
- A gradient descent loop written by hand that finds the same parameters as scikit-learn
- A measurement of overfitting: training error against test error as a model gets more flexible
- A working picture of the learning loop and of the four ways a model gets a learning signal

**What you need:**

- Python 3.11 or newer
- A virtual environment (as on Day 1)
- `numpy==2.2.6`, `pandas==2.2.3`, `scikit-learn==1.6.1` and `scipy==1.15.2` (pinned in the project section)
- Your Day 1 `data/messages.csv`
- Paper or a whiteboard tool for the loop diagram

**Prerequisites:**

- Day 1: What Is Artificial Intelligence? (training vs inference, models and parameters)
- Basic Python
- How to activate a virtual environment

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [A short history](#a-short-history)
- [The learning loop in one picture](#the-learning-loop-in-one-picture)
- [Data](#data)
- [Features](#features)
- [Labels](#labels)
- [Models](#models)
- [Parameters and hyperparameters](#parameters-and-hyperparameters)
- [Training](#training)
- [Loss](#loss)
- [Optimization](#optimization)
- [Generalization](#generalization)
- [Inference](#inference)
- [Four ways to get a learning signal](#four-ways-to-get-a-learning-signal)
- [Supervised learning](#supervised-learning)
- [Unsupervised learning](#unsupervised-learning)
- [Self-supervised learning](#self-supervised-learning)
- [Reinforcement learning](#reinforcement-learning)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: open the black box and write the loop yourself](#project-open-the-black-box-and-write-the-loop-yourself)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Every learning system in this series, from a spam filter to a large language model, runs one loop. Take **data**, turn it into **features**, let a **model** with adjustable **parameters** make a guess, measure the error with a **loss**, nudge the parameters to shrink the loss (**optimization**), and repeat. When training stops, the model runs on new inputs (**inference**), and you hope it does as well there as it did in training (**generalization**).

Most bugs in ML systems sit in one of these pieces: wrong data, a feature that leaks the answer, a loss that measures the wrong thing, an optimizer that never converged, or a model that memorized instead of learning. If you can name the piece, you can start debugging. On the job, that naming skill is what separates someone who can fix a failing model from someone who retries it with different settings.

Tom Mitchell's textbook *Machine Learning* (1997) gives the classic definition. A program learns from experience E on task T, measured by performance P, if its performance on T improves with E. Today you make E, T and P concrete.

---

## A short history

| Year | What happened | Why it matters today |
|---|---|---|
| 1847 | Augustin-Louis Cauchy describes the method of steepest descent. | The ancestor of [gradient descent](https://en.wikipedia.org/wiki/Gradient_descent), which still trains almost every neural network. |
| 1958 | Frank Rosenblatt publishes the [perceptron](https://en.wikipedia.org/wiki/Perceptron), a model whose weights are adjusted from examples. | The first widely known "learn the parameters from data" machine. |
| 1986 | Rumelhart, Hinton and Williams popularize [backpropagation](https://en.wikipedia.org/wiki/Backpropagation) for multi-layer networks. | How gradients are computed in deep learning (Day 43). |
| 2016 | DeepMind's AlphaGo beats Lee Sedol at Go. | Reinforcement learning plus deep learning at world-class level. |
| 2018 onward | BERT and GPT-style models are pretrained on raw text with no human labels. | Self-supervised learning becomes the main way to train large models. |

**Takeaway:** the loop of guess, measure and adjust has stayed the same for decades. What changed is the size of the data, the models and the compute.

---

## The learning loop in one picture

Copy this into your notebook. Each box maps to a concept section below.

```
                 ┌────────────────────────────────────────────┐
                 │                 TRAINING                   │
                 │                                            │
  raw data ──► features (X) ──► model(parameters) ──► prediction
                 │                                       │    │
                 │              labels (y) ──────────────┤    │
                 │                                       ▼    │
                 │                                     loss   │
                 │                                       │    │
                 │       optimizer adjusts parameters ◄──┘    │
                 │       (repeat many times)                  │
                 └────────────────────────────────────────────┘

                 ┌────────────────────────────────────────────┐
                 │                 INFERENCE                  │
  new input ──► same features ──► model(frozen parameters) ──► prediction
                 └────────────────────────────────────────────┘
```

The labels enter only at the loss. The model never sees them directly. Inference runs the **same** feature steps as training. If the two differ, the model gets inputs it was never trained on. That mismatch is called training/serving skew, and it is a common production bug.

**Reference image to find:** search "machine learning training loop diagram" and compare a few versions with this one.

---

### Data

A student who studies only past exam papers can only be as good as those papers. A model is the same: whatever it knows came from the examples it was shown.

Each example (also called a sample or a row) is one unit, such as one email or one apartment. Collect enough of them and the pattern starts to show. Collect them from the wrong place and the pattern you see is not the one you will meet later. So the useful test of a dataset is not only how many rows it has but how well they cover the situations the model will face.

Put formally, data is the set of recorded examples the system learns from. It must be relevant, large enough to show the pattern, and **representative** of the inputs the model will see later.

Your Day 1 `messages.csv` has 30 rows, 10 per label. Every message is in English, and every complaint is about a delivery or a product. A model trained on it has never seen a complaint about a rude employee, so it has no way to recognize one.

The usual mistakes are collecting from one source (one city, one season) and deploying everywhere, counting rows instead of asking what situations they cover, and leaving in duplicate rows, which make the model look better than it is. Spot them by asking where each row came from and by checking for repeats before you train. Module 3 covers data work.

In the market, teams often spend more time collecting, cleaning and checking data than choosing a model.

---

### Features

A doctor looks at temperature, blood pressure, age and symptoms. Those measurements are what the doctor reasons from, and the model needs the same kind of thing: a list of measurable facts about each case.

A model can only do arithmetic, so every fact must become a number. All the numbers for one example form a row, and all the rows together form a table with one column per fact.

A [feature](https://en.wikipedia.org/wiki/Feature_(machine_learning)) is one numeric property of an example that the model can use. All the features together form the **feature matrix**, `X`, with one row per example and one column per feature.

On Day 1, `CountVectorizer` turned "Hi there!" into a row with a 1 in the "hi" column, a 1 in the "there" column, and 0 everywhere else. Thirty messages with 106 distinct words make a 30 by 106 matrix. The project prints that shape.

```
            hi  there  how  do  reset  password  ...
Hi there!    1     1    0    0     0         0
How do I...  0     0    1    1     1         1
```

A feature that contains the answer, such as a "refund_issued" column when you predict complaints, is **data leakage**. It gives great test scores and useless predictions in production. Spot it by asking of every column whether you would know its value at the moment you need the prediction. Mixed scales also cause trouble: square meters next to cents can make some algorithms misbehave until you rescale. Day 21 covers feature engineering.

In industry, much of the practical skill in classical ML is deciding which columns to build and which to leave out.

---

### Labels

A flashcard has a question on the front and the answer on the back. The label is the back of the card: the correct answer the model is trying to reproduce.

Without a correct answer, there is nothing to measure a guess against. A label can be a category ("complaint") or a number (1,050 per month), and a person or a process has to supply it for every training example.

A label (also called the target, written `y`) is the correct output for an example.

In `messages.csv`, the `label` column is `y`. In the rent data you build in the project, `rent` is `y` and `size_m2` is the only feature.

People write labels, so labels contain errors. Is "Can you stop calling me?" a question or a complaint? Day 1 labeled it a complaint, and someone else might not. If labelers disagree on 10% of rows, no model can be reliably more than about 90% "correct" by those labels. Check a sample of labels by hand before you blame the model.

Labeling is a line item in real project budgets, often done by annotation teams or by domain experts such as doctors for medical images.

---

### Models

A model is a machine with dials. The shape of the machine is fixed, and the dial settings are learned.

Choosing the type of model decides which settings are even possible. A straight line can only tilt and shift. A decision tree can only split and branch. A neural network can bend in far more ways.

Formally, a model is a family of functions `f(x; θ)` that maps features `x` to a prediction. `θ` (theta) stands for the parameters. Choosing the model type (a straight line, a decision tree, a neural network) chooses the family. Training picks one member of that family.

For rent, `rent = w * size + b`. The family is all straight lines. The dials are `w` (rent per square meter) and `b` (base rent). `w = 12, b = 300` is one line. `w = 8, b = 500` is another.

The mistake is choosing a family that cannot express the pattern. A straight line cannot fit a curve, however long you train it. That is **underfitting**, and you can spot it when the error is high on both the training data and the test data. The opposite mistake, a family so flexible that it fits the noise, is overfitting (see the generalization section).

In practice, picking the model family is one of the first decisions on any project, and a simple family is often the first thing a team tries.

---

### Parameters and hyperparameters

Parameters are the dial positions. Training turns them, and nobody sets them by hand. Hyperparameters are the settings on the outside of the machine, such as how fast you turn the dials. You pick those before the run.

Everything the model "knows" lives in its parameters. Everything about how the search for them is run lives in the hyperparameters.

Parameters are the numbers inside the model that training adjusts to reduce the loss. **Hyperparameters** are settings you choose before training, such as the learning rate, the number of steps or a tree's depth. Training does not change them.

The linear rent model has 2 parameters. The Day 1 Naive Bayes model stored one number per (label, word) pair, plus one per label: 3 × 106 + 3 = 321 numbers. GPT-style language models have billions.

More parameters are not automatically better. A model with far more parameters than training examples can memorize every example and still fail on new ones. Parameter count also sets cost: a 7-billion-parameter model stored as 16-bit floats (2 bytes each) needs about 14 GB for its weights alone. You can spot memorization when training error is far lower than test error.

When a vendor quotes a parameter count, translate it into memory and cost per request. That is how engineers size hardware and choose between models.

---

### Training

Learning to throw darts goes like this: throw, see where it landed, adjust your arm, throw again.

A model learns the same way. It guesses, the guess is compared with the right answer, and the parameters shift slightly so the next guess is better. Nothing more mysterious than that happens, only many times over.

Training repeats three steps: predict on the training data, measure the loss, and update the parameters to reduce it. One full pass over the training data is an **epoch**.

In scikit-learn, `model.fit(X_train, y_train)` runs a loop like the one in the learning loop picture. In the project you write that loop yourself, in about ten lines.

The mistake is measuring on the same data you trained on. The score is high and meaningless, because the model has seen every answer. You spot it when someone reports one number and never mentions a held-out set. Always hold out data the model never trains on.

On a team, training is the slow, expensive, occasional step, and it usually runs in pipelines that someone watches and reruns on a schedule.

---

### Loss

A golf score is one number, and lower is better. A loss does the same job for a model: it squeezes all the mistakes into a single number the optimizer can chase.

The model needs one number to improve against, because "a bit wrong on this row and very wrong on that one" gives it no clear direction. The loss function decides how each kind of mistake is counted.

A loss function compares predictions with the true labels and returns one number for the total error. Two common ones:

- **Mean squared error (MSE)** for numbers: the average of `(prediction - truth)²`.
- **Cross-entropy (log loss)** for categories: large when the model is confidently wrong, small when it is confidently right.

Take three apartments:

| True rent | Predicted rent | Error | Squared error |
|---|---|---|---|
| 900 | 1,000 | +100 | 10,000 |
| 1,200 | 1,100 | -100 | 10,000 |
| 1,500 | 1,500 | 0 | 0 |

MSE = (10,000 + 10,000 + 0) / 3 ≈ 6,667. The mean absolute error (MAE) for the same rows is (100 + 100 + 0) / 3 ≈ 66.7. Squaring punishes big misses. One miss of 300 contributes 90,000, nine times the contribution of a miss of 100.

A loss that does not match the business goal gives a misleading result. A fraud model can reach 99.9% accuracy by answering "not fraud" every time if only 0.1% of transactions are fraud. The score looks good and the model catches nothing. You spot it by asking what the model would score if it always gave the most common answer. Day 6 shows the same trap with a baseline. Day 42 covers loss functions in depth.

Teams choose their loss and their reported metric to match what the business cares about, such as cost of a missed fraud case against cost of a false alarm.

---

### Optimization

You are on a foggy hill at night and want the valley floor. You cannot see the valley, but you can feel the slope under your feet. Step downhill, feel again, repeat.

The loss is the height of the ground, and the parameters are your position. You cannot see the whole landscape, but the local slope tells you which way is down. Many small downhill steps get you to a low point.

Optimization searches for parameter values that make the loss small. **Gradient descent** computes the **gradient** (the slope of the loss with respect to each parameter) and steps the opposite way. The **learning rate** sets how big each step is.

```
new_parameter = old_parameter - learning_rate * gradient
```

Model: `rent = w * size`. One training row: size 50, rent 600. The right answer is `w = 12`. The loss is `L(w) = (50w - 600)²`, with gradient `dL/dw = 100 * (50w - 600)`. Start at `w = 0` with learning rate 0.0001.

| Step | w | Prediction 50w | Gradient | New w |
|---|---|---|---|---|
| 1 | 0 | 0 | -60,000 | 0 + 6 = 6 |
| 2 | 6 | 300 | -30,000 | 6 + 3 = 9 |
| 3 | 9 | 450 | -15,000 | 9 + 1.5 = 10.5 |

Each step halves the distance to 12, so after about 20 steps `w` is within 0.00002 of 12. You do not need calculus to follow this. Days 11 and 12 explain where the gradient formula comes from.

A learning rate that is too large overshoots. At 0.0004, starting from `w = 0`, `w` jumps to 24, then back to 0, forever. Anything larger flies off toward infinity. A learning rate that is too small wastes hours of compute. When a training loss rises or turns into `nan`, suspect the learning rate first.

```
loss
 │\                      too large: bounces or explodes
 │ \        ╱╲  ╱╲
 │  \      ╱  ╲╱  ╲
 │   \___  good: falls, then flattens
 │       \______________
 │ too small: falls very slowly ..........
 └──────────────────────────────────── steps
```

Real training scripts use refinements of this one update rule, such as SGD, Adam and AdamW. Day 44 covers them. Every large model you hear about is trained with a descendant of this update rule.

---

### Generalization

One student memorizes last year's answers. Another understands the method. On last year's paper they tie. On this year's paper, only one passes.

A model faces the same split. Doing well on examples it has seen proves little, since it may have memorized them. What counts is performance on new data from the same source, so you hold some data back and test on it.

Generalization is how well a model performs on data from the same source that it was not trained on. You estimate it with a held-out **test set**. The gap between training and test error shows whether the model memorized. **Overfitting** means the model fits the training data well and new data badly, because it has memorized noise. **Underfitting** means the model is too simple to capture the pattern, so it does badly everywhere.

In the project you fit three models of increasing flexibility to only 15 rent examples:

| Model flexibility (polynomial degree) | Train MSE | Test MSE |
|---|---|---|
| 1 (straight line) | 4,971 | 7,135 |
| 3 | 3,273 | 109,780 |
| 10 | 1,289 | 9,160,342,488 |

As flexibility rises, training error falls and test error explodes. The degree-10 curve passes close to all 15 training points and swings wildly between and beyond them. That is overfitting in one table.

The common mistakes are reporting the training score as the result, tuning settings against the test set until the test set leaks into your choices (Day 3 adds a separate validation set), and assuming the future looks like the test set. If the world changes, even a well-generalized model degrades. You spot the first two when test results look too good and then collapse in production.

In the market, this gap between test results and live results is the reason teams monitor deployed models after launch.

---

### Inference

Training is studying. Inference is sitting the exam, over and over, for every user.

Once training ends, the parameters are frozen. Using the model means feeding it a new input and running the same arithmetic it always runs. No learning happens at that point.

Inference runs the trained model, with its parameters frozen, on new inputs to produce predictions.

After training, the rent for a 65 m² apartment is one multiplication and one addition: 292.0 + 12.03 × 65 ≈ 1,074. For a large language model, each generated word takes billions of multiplications.

The mistakes are forgetting the preprocessing from training, such as rescaling with the training mean and standard deviation, and feeding inputs outside the training range. The rent model has never seen a 400 m² apartment, so its prediction there is a guess dressed up as a number. You spot both when predictions in production look odd for inputs that differ from the training data.

Inference cost per request decides whether an AI feature pays for itself, which is why serving gets its own day (Day 92).

---

## Four ways to get a learning signal

The loop in the learning loop section needs something to compare predictions against. The source of that signal defines the type of learning.

```
Where does the learning signal come from?
├── Human-provided labels ............ supervised learning
├── No labels, only structure ......... unsupervised learning
├── Labels made from the data itself .. self-supervised learning
└── Rewards from acting in a world .... reinforcement learning
```

---

### Supervised learning

A teacher marks every practice answer right or wrong, and you learn from the marks.

The training data contains inputs and the correct label for each. The model learns the mapping between them. If the label is a category, the task is **classification**. If it is a number, the task is **regression**. [Wikipedia](https://en.wikipedia.org/wiki/Supervised_learning)

Supervised learning means learning a function from labeled input-output pairs. Day 1's message classifier is classification, and today's rent model is regression.

The limit is that labels are slow and costly. A medical image may need a specialist's time. Most deployed business ML, such as credit scoring, churn prediction and demand forecasting, is supervised.

---

### Unsupervised learning

You dump a box of mixed buttons on a table and sort them into piles by color and size. Nobody names the piles for you.

With no labels, the algorithm can only look at how the examples relate to each other. It can find groups of similar items (**clustering**) or a smaller set of features that keeps most of the information (**dimensionality reduction**). [Wikipedia](https://en.wikipedia.org/wiki/Unsupervised_learning)

Unsupervised learning finds structure in unlabeled data.

```python
from sklearn.cluster import KMeans

points = [[1, 1], [1.5, 2], [1, 1.5], [8, 8], [8.5, 9], [9, 8]]
groups = KMeans(n_clusters=2, n_init=10, random_state=0).fit_predict(points)
print(groups)  # [1 1 1 0 0 0]
```

The algorithm found two groups. The numbers 0 and 1 are arbitrary names for them. Swap the names and the grouping stays the same. (On some Windows 11 machines this prints a harmless warning about counting CPU cores. You can ignore it.)

There is no single right answer to check against, so a useful grouping is hard to tell from a meaningless one. You asked for 2 groups, and the algorithm would happily find 5. Spot the problem by asking whether the groups mean anything to someone who knows the domain. Customer segmentation and grouping similar support tickets are common unsupervised jobs (Day 32).

---

### Self-supervised learning

Cover a word in a sentence with your thumb and guess it. The text tells you whether you were right, so nobody had to write labels.

The labels are created automatically from the raw data, usually by hiding part of an input and asking the model to predict it. The training is technically supervised, but no person writes labels, so it scales to enormous datasets. [Wikipedia](https://en.wikipedia.org/wiki/Self-supervised_learning)

Self-supervised learning is learning from labels generated out of the data itself. Here is one sentence turned into five training examples for next-word prediction:

```python
words = "the cat sat on the mat".split()
pairs = [(" ".join(words[:i]), words[i]) for i in range(1, len(words))]
for context, target in pairs:
    print(f"{context!r:>22} -> {target!r}")
```

```text
                 'the' -> 'cat'
             'the cat' -> 'sat'
         'the cat sat' -> 'on'
      'the cat sat on' -> 'the'
  'the cat sat on the' -> 'mat'
```

Run this over billions of sentences and you have the pretraining task of a GPT-style language model (Day 73).

The model learns every pattern in the raw data, including errors and biases. The pretext task (guess the next word) is not the job you want done, so pretrained models are usually fine-tuned afterwards (Day 74), and you spot the gap when a pretrained model gets the style right but the task wrong. Most large language models, and many image and speech models, start with self-supervised pretraining.

---

### Reinforcement learning

Training a dog with treats. Nobody shows the dog the right sequence of moves. It tries things and gets rewarded when the outcome is good.

An **agent** takes **actions** in an **environment**, receives **rewards**, and learns a **policy** (a way of choosing actions) that maximizes total reward over time. Unlike supervised learning, the agent is never told the right action, only how good the result was. Its own choices also decide which data it sees next. [Wikipedia](https://en.wikipedia.org/wiki/Reinforcement_learning)

Reinforcement learning is learning a policy from rewards received while acting. Here is a small case. Machine A pays out 30% of the time, machine B 60%. The agent does not know this.

```python
import numpy as np

rng = np.random.default_rng(0)
win_prob = [0.3, 0.6]          # hidden from the agent
estimates = [0.0, 0.0]         # what the agent believes
pulls = [0, 0]

for step in range(500):
    # Explore 10% of the time, otherwise pick the machine that looks best so far.
    arm = int(rng.integers(2)) if rng.random() < 0.1 else int(np.argmax(estimates))
    reward = 1.0 if rng.random() < win_prob[arm] else 0.0
    pulls[arm] += 1
    estimates[arm] += (reward - estimates[arm]) / pulls[arm]   # running average

print("pulls:", pulls)
print("estimates:", [round(e, 2) for e in estimates])
```

```text
pulls: [25, 475]
estimates: [0.24, 0.59]
```

The agent pulls machine B 95% of the time (475 of 500). The code also shows the trade-off between **exploration** (trying the other machine in case it is better) and **exploitation** (using what you already know). Without the 10% exploration, an early unlucky streak could lock the agent onto the worse machine forever.

RL needs many trials. In the real world, trials can be slow, expensive or dangerous (a robot falling over), so robotics RL usually trains in simulation first (Day 98). You spot a problem when the agent keeps repeating one action and never tests the others. RL also shows up in game AI, ad bidding and the preference-tuning stage of chat models (RLHF, Day 75).

---

## Hands-on exercises (about 60 minutes)

**Exercise 1: name the pieces (about 20 minutes).** For each system, write one line each for: data, features, label (or "none"), model output, and learning type.

1. A phone keyboard that suggests the next word.
2. An email spam filter.
3. A streaming service grouping users with similar viewing habits, with no predefined groups.
4. A robot vacuum that learns which cleaning routes finish fastest.
5. A bank predicting the probability a loan will default.
6. A photo app that learns image features by predicting which of two crops came from the same photo.

**Answer key:** 1 self-supervised (the next word in your own typing is the label). 2 supervised classification (labels from users clicking "spam"). 3 unsupervised clustering. 4 reinforcement learning (reward is time saved or area covered). 5 supervised (classification of default, often reported as a probability). 6 self-supervised (the label "same photo" comes from how the crops were made).

**Exercise 2: gradient descent by hand (about 20 minutes).** Use the one-row example from the optimization section (size 50, rent 600, start `w = 0`).

1. With learning rate 0.0001, compute steps 4 and 5 on paper. Confirm `w` goes to 11.25 and then about 11.63.
2. With learning rate 0.0002, compute step 1. What happens? (Answer: `w` jumps straight to 12 in one step. For this one-row problem, 0.0002 is exactly the "perfect" step size. Real problems never line up this neatly.)
3. With learning rate 0.0005, compute steps 1 and 2 and describe the behavior in one sentence.

**Exercise 3: MSE vs MAE (about 20 minutes).** Predictions `[100, 200, 300, 400]`, truths `[110, 190, 300, 600]`. Compute MSE and MAE by hand, then check with:

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error
truth, pred = [110, 190, 300, 600], [100, 200, 300, 400]
print(mean_squared_error(truth, pred), mean_absolute_error(truth, pred))
```

You should get MSE 10,050 and MAE 55. Write one sentence on which single row dominates the MSE and why.

---

## Project: open the black box and write the loop yourself

This is the GitHub project for Day 2. You open up the model you built on Day 1, then write the training loop yourself so no part of it stays hidden.

**Goal:** see the learned parameters of your Day 1 bot, write gradient descent from scratch, confirm it matches scikit-learn, and measure overfitting.

Suggested repo name: `100-days-ai-engineering-day-02-how-machines-learn`.

### Repository layout

```
day-02-how-machines-learn/
├── README.md               # your results and answers
├── requirements.txt        # pinned library versions
├── data/
│   ├── messages.csv        # copied from Day 1
│   └── rent.csv            # generated by make_data.py
├── inspect_bot.py          # prints what the Day 1 model learned
├── make_data.py            # creates the synthetic rent dataset
├── gradient_descent.py     # hand-written training loop
└── generalization.py       # train vs test error as flexibility grows
```

### Step 1: set up

```bash
mkdir day-02-how-machines-learn
cd day-02-how-machines-learn
mkdir data
python -m venv .venv
```

Activate the environment as on Day 1 (`source .venv/bin/activate` on macOS or Linux, `.venv\Scripts\Activate.ps1` in PowerShell). Copy `data/messages.csv` from your Day 1 folder into `data/`.

Create `requirements.txt`:

```text
numpy==2.2.6
pandas==2.2.3
scikit-learn==1.6.1
scipy==1.15.2
```

Run `pip install -r requirements.txt`. Pinning exact versions (`==`) means anyone who installs from this file gets the same libraries you used, so your numbers can be reproduced. Day 5 covers this in more detail.

### Step 2: inspect the Day 1 model (`inspect_bot.py`)

```python
"""Open up the Day 1 learned bot and print what it learned."""

import numpy as np
import pandas as pd
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.naive_bayes import MultinomialNB


def main():
    df = pd.read_csv("data/messages.csv")

    # Features: one column per word, holding how often that word appears in the message.
    vectorizer = CountVectorizer(lowercase=True)
    X = vectorizer.fit_transform(df["text"])
    y = df["label"]
    print(f"feature matrix: {X.shape[0]} rows x {X.shape[1]} word columns")

    # Training: Naive Bayes estimates one probability per (label, word) pair.
    model = MultinomialNB()
    model.fit(X, y)

    words = vectorizer.get_feature_names_out()
    log_probs = model.feature_log_prob_  # shape: (number of labels, number of words)
    n_params = log_probs.size + model.class_log_prior_.size
    print(f"learned numbers (parameters): {n_params}")

    for i, label in enumerate(model.classes_):
        # How much more likely is each word under this label than under the other labels?
        others = np.delete(log_probs, i, axis=0).mean(axis=0)
        strength = log_probs[i] - others
        top = np.argsort(strength)[::-1][:5]
        print(f"\n{label}: words that push a message toward this label")
        for j in top:
            print(f"  {words[j]:<10} {strength[j]:+.2f}")


if __name__ == "__main__":
    main()
```

Run `python inspect_bot.py`. Expected output:

```text
feature matrix: 30 rows x 106 word columns
learned numbers (parameters): 321

complaint: words that push a message toward this label
  and        +1.49
  this       +1.26
  have       +0.98
  am         +0.98
  want       +0.98

greeting: words that push a message toward this label
  there      +1.22
  well       +1.22
  team       +1.22
  are        +1.22
  hi         +1.22

question: words that push a message toward this label
  do         +1.39
  my         +0.84
  me         +0.76
  can        +0.76
  where      +0.70
```

The strongest complaint word is "and," not "broken" or "terrible." Complaints in this dataset tend to be long sentences joined with "and." That is a real pattern in these 30 rows and a useless one in the world. The model learns whatever correlates with the label, and 30 rows are too few to tell the difference. This is why 30 rows were not enough on Day 1. The five tied greeting words each appeared twice in the training messages, so the model cannot separate them.

### Step 3: generate data (`make_data.py`)

```python
"""Create a small synthetic dataset: apartment size (square meters) and monthly rent."""

import numpy as np
import pandas as pd

rng = np.random.default_rng(seed=0)
n = 100
size_m2 = rng.uniform(20, 120, size=n).round(1)
noise = rng.normal(0, 80, size=n)
rent = (300 + 12 * size_m2 + noise).round(0)

pd.DataFrame({"size_m2": size_m2, "rent": rent}).to_csv("data/rent.csv", index=False)
print("wrote data/rent.csv with", n, "rows")
```

Because you generated the data, you know the true rule: rent = 300 + 12 × size, plus random noise. That lets you check whether training recovers it. Real data never gives you this luxury.

### Step 4: write the training loop (`gradient_descent.py`)

```python
"""Train a one-feature linear model with gradient descent, written by hand."""

import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split


def mse(y_true, y_pred):
    """Mean squared error: the average of the squared mistakes."""
    return float(np.mean((y_true - y_pred) ** 2))


def main():
    df = pd.read_csv("data/rent.csv")
    x_train, x_test, y_train, y_test = train_test_split(
        df["size_m2"].to_numpy(), df["rent"].to_numpy(), test_size=0.2, random_state=42
    )

    # Rescale the feature so gradient descent behaves (mean 0, standard deviation 1).
    # The mean and std come from the training rows only.
    mean, std = x_train.mean(), x_train.std()
    xs_train = (x_train - mean) / std

    w, b = 0.0, 0.0          # the two parameters, starting at zero
    learning_rate = 0.02
    for step in range(301):
        y_pred = w * xs_train + b
        error = y_pred - y_train
        # Gradients of the MSE loss with respect to w and b.
        grad_w = 2 * np.mean(error * xs_train)
        grad_b = 2 * np.mean(error)
        # Optimization step: move each parameter a little against its gradient.
        w -= learning_rate * grad_w
        b -= learning_rate * grad_b
        if step % 50 == 0:
            print(f"step {step:>3}  train loss {mse(y_train, w * xs_train + b):>10.1f}")

    # Undo the rescaling so the parameters read in rent per square meter.
    slope = w / std
    intercept = b - w * mean / std
    print(f"\nhand-written GD: rent = {intercept:.1f} + {slope:.2f} * size_m2")
    print(f"test loss (MSE): {mse(y_test, slope * x_test + intercept):.1f}")

    reference = LinearRegression().fit(x_train.reshape(-1, 1), y_train)
    print(f"scikit-learn:    rent = {reference.intercept_:.1f} + {reference.coef_[0]:.2f} * size_m2")

    # Inference: use the trained parameters on an input the model never saw.
    print(f"\npredicted rent for 65 m2: {intercept + slope * 65:.0f}")


if __name__ == "__main__":
    main()
```

Sizes run from 20 to 120. Without rescaling, the gradient for `w` is about 70 times larger than the gradient for `b`. Any learning rate small enough to keep `w` stable then makes `b` crawl. Rescaling puts both on the same footing. The two "undo" lines convert the answer back to real units.

Run `python make_data.py`, then `python gradient_descent.py`. Expected output:

```text
wrote data/rent.csv with 100 rows
step   0  train loss  1368768.3
step  50  train loss    28859.5
step 100  train loss     6254.8
step 150  train loss     5873.5
step 200  train loss     5867.0
step 250  train loss     5866.9
step 300  train loss     5866.9

hand-written GD: rent = 292.0 + 12.03 * size_m2
test loss (MSE): 6717.3
scikit-learn:    rent = 292.0 + 12.03 * size_m2

predicted rent for 65 m2: 1074
```

The loss falls fast and then flattens, so the optimizer has converged. Your 10 lines found the same parameters as scikit-learn. The learned rule (292 + 12.03 × size) is close to the true rule (300 + 12 × size) but not equal, because the noise in 80 training rows pulls the estimate a little. The training loss of about 5,867 never reaches 0, because the noise (standard deviation 80, so variance about 6,400) cannot be predicted by any model.

### Step 5: measure generalization (`generalization.py`)

```python
"""Show the gap between training loss and test loss as a model gets more flexible."""

import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error
from sklearn.model_selection import train_test_split
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import PolynomialFeatures, StandardScaler


def main():
    df = pd.read_csv("data/rent.csv")
    # Keep only 15 training rows so a flexible model has room to memorize.
    x_train, x_test, y_train, y_test = train_test_split(
        df[["size_m2"]], df["rent"], train_size=15, random_state=1
    )

    print(f"{'degree':>6} {'train MSE':>12} {'test MSE':>12}")
    for degree in [1, 3, 10]:
        model = make_pipeline(
            StandardScaler(), PolynomialFeatures(degree=degree), LinearRegression()
        )
        model.fit(x_train, y_train)
        train_mse = mean_squared_error(y_train, model.predict(x_train))
        test_mse = mean_squared_error(y_test, model.predict(x_test))
        print(f"{degree:>6} {train_mse:>12.0f} {test_mse:>12.0f}")


if __name__ == "__main__":
    main()
```

`PolynomialFeatures(degree=3)` adds size² and size³ as extra features, so the line can bend. Expected output:

```text
degree    train MSE     test MSE
     1         4971         7135
     3         3273       109780
    10         1289   9160342488
```

### Step 6: write the README

Paste all three outputs, then answer in full sentences:

1. How many parameters did the Day 1 model learn, and where does the number 321 come from?
2. Why is "and" the top complaint word? Name one change to the data that would fix it.
3. Change `learning_rate` to 0.5, then to 1.1. Describe what the loss does in each case. Put it back to 0.02 afterwards.
4. How close did gradient descent get to the true rule (300 + 12 × size)? Why can it never match it exactly with this data?
5. In the generalization table, which degree would you ship, and why? Use the words "overfitting" and "test MSE" in your answer.
6. The degree-10 test MSE is in the billions. Pick one test row where it is wildly wrong (hint: print `x_test` and the predictions) and explain what happened.

### Done when

- All four scripts run after `pip install -r requirements.txt`, and the outputs match the expected outputs above (or differ only because of library versions, which you have noted).
- The README answers all six questions with your own numbers.
- Your hand-written gradient descent matches scikit-learn to two decimal places.
- You can explain, without notes, the difference between a parameter and a hyperparameter, using `w`, `b` and `learning_rate` as examples.

---

## Check your understanding

1. A model scores 99% on its training data and 61% on held-out data. What is the most likely problem, and what are two things you could try?
2. Why does the learning signal in self-supervised learning cost almost nothing, while supervised labels are expensive?
3. Your training loss suddenly becomes `nan` on step 40. What is the first hyperparameter you check, and why?
4. A colleague computes the mean and standard deviation for rescaling using all 100 rent rows before splitting. Why is that a (small) mistake?
5. Explain why a reinforcement learning agent must sometimes take an action it believes is worse.
6. Your spam filter's top spam word is "unsubscribe." Is that a good feature or a leak? What would you check?
7. Is choosing `n_clusters=2` in k-means training or a hyperparameter choice? What changes if you choose 5?

**Answers (check after you try):**

1. Overfitting. Get more training data, use a simpler model (or regularization, Day 45), and confirm the held-out set comes from the same source as the training set.
2. The labels are produced by a program from the raw data (hide a word, predict it), so no person has to read each example.
3. The learning rate. A rate that is too large makes the parameters overshoot and grow until the numbers overflow.
4. The test rows then influence the preprocessing, so a little information about the test set leaks into training. With 100 rows the effect is small, but the habit causes real leakage elsewhere (Day 3).
5. Its beliefs come only from the actions it has tried. Without exploration it can never discover that an option it undervalued is better.
6. It could be a real signal, since spam often contains it, but wanted newsletters may contain it too. Check how many non-spam emails contain it, and whether the word was added to emails after they were labeled.
7. A hyperparameter choice. Training then finds 5 groups whether or not 5 real groups exist. That is the main weakness of unsupervised learning.

---

## Sources to read and watch

**Read (reference):**
- [Wikipedia: Supervised learning](https://en.wikipedia.org/wiki/Supervised_learning)
- [Wikipedia: Unsupervised learning](https://en.wikipedia.org/wiki/Unsupervised_learning)
- [Wikipedia: Self-supervised learning](https://en.wikipedia.org/wiki/Self-supervised_learning)
- [Wikipedia: Reinforcement learning](https://en.wikipedia.org/wiki/Reinforcement_learning)
- [Wikipedia: Gradient descent](https://en.wikipedia.org/wiki/Gradient_descent)
- [Wikipedia: Loss function](https://en.wikipedia.org/wiki/Loss_function)
- [Wikipedia: Overfitting](https://en.wikipedia.org/wiki/Overfitting)
- [Google Machine Learning Crash Course](https://developers.google.com/machine-learning/crash-course): the "Linear regression" module, including its loss and gradient descent pages.
- [Stanford CS229: Machine Learning](https://cs229.stanford.edu/): the main lecture notes open with linear regression and gradient descent. Skim the first section. The notation will make sense after Module 2.
- [scikit-learn User Guide: Naive Bayes](https://scikit-learn.org/stable/modules/naive_bayes.html), to see what `feature_log_prob_` stores.

**Tutorials:**
- GeeksforGeeks: search "Gradient Descent in Linear Regression" and "Supervised and Unsupervised learning" on geeksforgeeks.org.
- [scikit-learn: Underfitting vs. Overfitting](https://scikit-learn.org/stable/auto_examples/model_selection/plot_underfitting_overfitting.html), the official example that inspired `generalization.py`.

**Watch:**
- [3Blue1Brown: Gradient descent, how neural networks learn](https://www.youtube.com/watch?v=IHZwWFHWa-w) (chapter 2 of the neural networks series, about 20 minutes).
- StatQuest with Josh Starmer: search "Gradient Descent, Step-by-Step" on the [StatQuest channel](https://www.youtube.com/@statquest).
- DeepLearning.AI: Andrew Ng's "Machine Learning Specialization" on Coursera, Course 1, week 1 videos on supervised vs unsupervised learning and gradient descent (free to audit).

**Practice:**
- [TensorFlow Playground](https://playground.tensorflow.org): change the learning rate and watch the training loss and test loss in the top right. Try a learning rate of 3 and see the loss explode, the same failure as question 3 in the README.

---

## Tomorrow

Day 3, **The AI/ML Lifecycle**, zooms out from the training loop to the project around it. Today you saw that `model.fit()` is about ten lines of logic. Tomorrow you see the stages before and after those lines: defining the problem, collecting and cleaning data, building a baseline, validating, testing, deploying and monitoring. You also see what goes wrong when a stage is skipped, starting with the leak from question 4 above. Keep today's repo. Its "and" problem is a data-collection failure, and you will name the stage it belongs to.
