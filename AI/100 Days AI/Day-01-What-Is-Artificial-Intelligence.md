# Day 1: What Is Artificial Intelligence?

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 3 to 4 hours
**You will build:** an AI field map and a small project that runs the same message classifier two ways: with hand-written rules, then with a learned model. You will compare their mistakes.
**You should know:** basic Python (variables, functions, lists, reading a CSV file). No math is needed yet.
**You will install:** Python 3.10 or newer, `pandas` and `scikit-learn`. The setup steps are in section 8.

---

## 1. The big idea in one paragraph

Artificial intelligence (AI) is the field of building systems that do things we would call intelligent when a person does them: recognizing faces, understanding speech, translating text, playing games, planning routes, writing code. AI is not one technique. It is a family of approaches, and the most important split is between two of them: **hand-written rules** and **patterns learned from data**. Almost everything you will study in these 100 days lives on one side of that split or the other.

[Wikipedia: Artificial intelligence](https://en.wikipedia.org/wiki/Artificial_intelligence) gives the full history. Read the introduction and the section on "Goals" today.

---

## 2. A short history, so the vocabulary makes sense

| Period | What happened | Why it matters today |
|---|---|---|
| 1950 | Alan Turing asks "Can machines think?" and proposes the [Turing test](https://en.wikipedia.org/wiki/Turing_test). | Defined the question. Behavior, not internals, was the test. |
| 1956 | The [Dartmouth workshop](https://en.wikipedia.org/wiki/Dartmouth_workshop) coins the term "artificial intelligence." | The field gets its name. |
| 1960s–1980s | **Symbolic AI**: programs manipulate symbols and logic rules. [Expert systems](https://en.wikipedia.org/wiki/Expert_system) encode a human expert's rules (medical diagnosis, machine configuration). | Rules are still used in many real products. Good when rules are known and stable. |
| 1980s–1990s | "AI winters": funding dried up after rule-based systems failed to scale to messy real-world problems. | Lesson: hand-written rules break on messy inputs. |
| 1990s–2010s | **Statistical machine learning** grows: models learn patterns from data (spam filters, search ranking, recommendations). | The shift from "tell the machine the rules" to "show the machine examples." |
| 2012 onward | **Deep learning** (large neural networks trained on lots of data and compute) wins image recognition and then speech, translation and games. | The engine behind modern AI. |
| 2017 onward | The [transformer](https://en.wikipedia.org/wiki/Transformer_(deep_learning_architecture)) architecture arrives. | Basis of today's large language models (covered in Module 8). |
| 2022 onward | [Generative AI](https://en.wikipedia.org/wiki/Generative_artificial_intelligence) becomes mainstream: systems that produce text, images, audio and code. | What most people now mean by "AI." |

**Takeaway:** AI has moved from rules written by people, to patterns learned from data, to very large learned models. Knowing this arc helps you judge which tool fits which problem.

---

## 3. The vocabulary you must own by the end of today

Read each term, then say the definition out loud in your own words.

- **Artificial intelligence (AI):** any technique that makes a machine perform a task that seems to require intelligence.
- **Machine learning (ML):** a subset of AI where a system improves its performance on a task from data, instead of from explicitly coded rules. [Wikipedia](https://en.wikipedia.org/wiki/Machine_learning)
- **Deep learning (DL):** ML using neural networks with many layers. [Wikipedia](https://en.wikipedia.org/wiki/Deep_learning)
- **Generative AI:** models that produce new content (text, images, audio, code) rather than only labeling existing content. [Wikipedia](https://en.wikipedia.org/wiki/Generative_artificial_intelligence)
- **Narrow AI (weak AI):** AI built for one task or a small set of tasks. Every AI system in use today is narrow.
- **Artificial general intelligence (AGI):** a hypothetical system with broad, human-level ability across most tasks. It does not exist yet, and experts disagree on how close it is. [Wikipedia](https://en.wikipedia.org/wiki/Artificial_general_intelligence)
- **Symbolic vs statistical AI:** rules and logic written by people, versus behavior estimated from data.
- **Predictive vs generative systems:** predictive systems answer "what is this?" or "what comes next in a number sequence?" Generative systems answer "make me something that fits this description."
- **Model:** the learned or designed function that maps inputs to outputs. In ML it is a set of **parameters** (numbers) adjusted during training.
- **Parameters:** the numbers inside a model that training adjusts. Modern LLMs have billions.
- **Algorithm:** a precise, finite sequence of steps that solves a problem. The training procedure is an algorithm; the trained model is not.
- **Training:** the phase where parameters are adjusted using data.
- **Inference:** the phase where a trained model is used on new inputs. Training happens once (or occasionally); inference happens every time someone uses the system.
- **Data science vs ML engineering vs AI engineering vs research:** see section 5.

---

## 4. Concept deep dive: where AI, ML, DL and generative AI sit

Draw this in your notebook. It is the most important picture of the day.

```
Artificial Intelligence (any machine doing "intelligent" tasks)
├── Symbolic AI (rules, logic, search, expert systems)
├── Classical search and planning (A*, game trees)
└── Machine Learning (learns from data)
    ├── Classical ML (linear models, trees, SVMs, k-means ...)
    └── Deep Learning (neural networks with many layers)
        ├── Computer vision (images, video)
        ├── Natural language processing (text, speech)
        ├── Reinforcement learning (learning by acting and getting reward)
        └── Generative models (produce new content: LLMs, diffusion models)
```

Three ideas to notice:

1. **Every ML system is AI, but not every AI system is ML.** A chess engine built from hand-coded evaluation rules is AI and not ML.
2. **Deep learning is a subset of ML.** It is not a separate universe.
3. **Generative AI is a use of deep learning** (mostly), not a separate branch. A large language model is a deep network trained to predict the next token of text.

**Reference images to find:** search "AI vs ML vs deep learning diagram" on Google Images and compare three of them. NVIDIA and IBM both publish clear versions.

**Video (about 20 minutes):** [3Blue1Brown: But what is a neural network?](https://www.youtube.com/watch?v=aircAruvnKk). Watch it now. You will see the whole chapter 2 concepts (neurons, weights, layers) before we study them properly on Day 39.

---

## 5. Roles: how AI, ML, data science and research differ

| Role | Typical question | Typical output |
|---|---|---|
| Data scientist | "What does the data say, and what decision should it support?" | Analyses, dashboards, statistical models |
| ML engineer | "How do I train, evaluate and ship a model that works reliably?" | Trained models, pipelines, evaluation reports |
| AI engineer | "How do I build a product that uses AI models, retrieval, tools and evaluation?" | Applications, agents, LLM-powered features |
| AI researcher | "Can we create a new method that works better?" | Papers, new architectures, new training methods |

For this 100-day program, you are aiming at **ML and AI engineering**, with enough data science to be dangerous and enough research literacy to read papers.

---

## 6. Symbolic vs learned: why both still matter

Two systems can solve the same problem in very different ways.

**Problem:** decide if an email is spam.

- **Symbolic version:** "If the subject contains 'free money' or the sender is on a blocklist, mark as spam." You write the rules. You know exactly why each email was flagged. It breaks the moment spammers change their wording.
- **Learned version:** show the system 10,000 emails labeled spam or not spam. It finds its own patterns in the words, senders and links. It adapts when you retrain. But you cannot always say exactly why one email was flagged.

Good engineers know when to use each. A bank's fraud rule (a hard limit on transfers above a certain amount) and a fraud model (a classifier trained on past transactions) often work together.

---

## 7. Hands-on exercises (about 60 minutes)

**Exercise 1: Build your AI field map (about 25 minutes).**

1. Open a blank page, paper or whiteboard tool. Write the title "AI field map."
2. Without looking, draw the tree from section 4. Include all four leaves under "Deep Learning" and both branches under "Machine Learning."
3. Open section 4 and compare your drawing with the original. Write the names of anything you missed in a different color or with a star.
4. List five real products you use each week. Use these if you need ideas: a phone camera that finds faces, a maps app that suggests a route, a music app's recommendations, autocorrect, a voice assistant, a spam filter, a photo app that searches by "dog" or "beach."
5. Next to each product, write one line in this form: `Product: category, because <one reason>`. The category must be one of: symbolic, classical ML, deep learning, generative. If you are not sure, write `unsure, check later`. That is an honest answer, and you will revisit it on Day 2.

**Exercise 2: Problem sorting (about 35 minutes).**

Write the answer to each problem below in this form: `Problem number: rules, learning, or mix, because <one sentence>`. Do not look up anything. Use your best guess and the definitions from section 6.

1. Sort a list of names alphabetically.
2. Tell if a photo contains a cat.
3. Predict tomorrow's electricity demand for a city.
4. Answer questions about a PDF you upload.
5. Decide whether to approve a bank loan, given a hard legal rule that applicants must be 18 or older.
6. Convert a temperature from Celsius to Fahrenheit.
7. Flag a bank transfer above 10,000 dollars for review.
8. Detect fraudulent card payments from past transactions.
9. Suggest the next song in a playlist.
10. Check whether a password has at least eight characters and one digit.

**Answer key (check after you write your answers):** 1 rules. 2 learning. 3 learning. 4 mix (retrieval rules plus a language model). 5 mix (the age rule is a rule; the approval score is learned). 6 rules. 7 rules. 8 learning. 9 learning. 10 rules.

If you disagreed with three or more of these, reread section 6 before the project.

---

## 8. Project: a rule-based bot and a learned bot that solve the same problem

This is the GitHub project for Day 1. You are not building a product yet. You are feeling the difference between the two approaches with your own hands.

**Goal:** a Python program that classifies short messages as **greeting**, **question**, or **complaint**, first with hand-written rules, then with a small learned model. Then you compare where each one fails.

### Repository layout

```
day-01-what-is-ai/
├── README.md              # what you built, what you learned, where each approach failed
├── data/
│   └── messages.csv       # 30 labeled messages (you will copy them in step 2)
├── symbolic_bot.py        # rule-based classifier
├── learned_bot.py         # small learned classifier (bag of words + Naive Bayes)
├── compare.py             # runs both on every row and prints their mistakes
└── requirements.txt       # the two libraries this project needs
```

### Step 1: create the folder and a virtual environment

A virtual environment keeps this project's libraries separate from the rest of your computer. Run these commands in a terminal from the folder where you keep projects.

```bash
mkdir day-01-what-is-ai
cd day-01-what-is-ai
mkdir data
python -m venv .venv
```

Activate it:

- macOS or Linux: `source .venv/bin/activate`
- Windows (PowerShell): `.venv\Scripts\Activate.ps1`
- Windows (Command Prompt): `.venv\Scripts\activate.bat`

Your prompt should now start with `(.venv)`. Every command in this project assumes that you are inside the folder with the environment active.

### Step 2: write the dataset

Create `data/messages.csv` with exactly this content. It has 30 rows, 10 per label. Some rows are deliberately tricky, and the tricky rows are where the two approaches disagree.

```csv
text,label
Hi there!,greeting
Good morning team,greeting
Hello how are you doing today,greeting
Hey everyone,greeting
Good evening from the office,greeting
Hi hope you are well,greeting
Hello again,greeting
Morning! Hope the week is going well,greeting
Hey there long time no see,greeting
Greetings to the whole team,greeting
How do I reset my password?,question
When does the store open?,question
Where can I find my invoice?,question
Can you help me with my order?,question
What is the refund policy?,question
Is the service down right now?,question
Why was I charged twice?,question
Do you ship to Canada?,question
Could you tell me the opening hours.,question
Who do I contact about a damaged item?,question
This app keeps crashing and I am furious,complaint
Your delivery was three days late,complaint
The product arrived broken,complaint
This is the worst service I have ever used,complaint
Can you stop calling me?,complaint
I have been waiting two hours and nobody answered,complaint
My order was charged twice and I am angry,complaint
The screen is cracked and the box was damaged,complaint
Terrible experience I want my money back,complaint
I want to speak to a manager about this mess,complaint
```

The file has no commas inside the message text, so you do not need quotation marks. If you later write your own messages with commas, wrap the text in double quotes.

Check the count: `python -c "import pandas as pd; print(pd.read_csv('data/messages.csv')['label'].value_counts())"` should print 10 for each label.

### Step 3: the rule-based bot (`symbolic_bot.py`)

Create `symbolic_bot.py` with this code. Read the comments. Each rule is a decision a person made.

```python
"""Rule-based message classifier: humans write the rules."""

from __future__ import annotations

import re

GREETING_OPENERS = {"hi", "hello", "hey", "greetings", "morning", "good morning", "good evening"}
QUESTION_WORDS = {"how", "what", "when", "where", "why", "who", "can", "do", "does", "is", "are", "will"}
COMPLAINT_WORDS = {"crashing", "late", "broken", "furious", "angry", "terrible", "worst", "cracked", "damaged"}


def tokenize(text: str) -> list[str]:
    """Lowercase the text and split it into words. Punctuation is dropped."""
    return re.findall(r"[a-z']+", text.lower())


def classify(text: str) -> str:
    tokens = tokenize(text)
    if not tokens:
        return "unknown"

    # Rule 1: a question mark or a question word at the start means a question.
    if text.strip().endswith("?") or tokens[0] in QUESTION_WORDS:
        return "question"

    # Rule 2: any complaint word anywhere in the message means a complaint.
    if COMPLAINT_WORDS & set(tokens):
        return "complaint"

    # Rule 3: a greeting word in the first one or two words means a greeting.
    opener_two_words = " ".join(tokens[:2])
    if tokens[0] in GREETING_OPENERS or opener_two_words in GREETING_OPENERS:
        return "greeting"

    # No rule matched. The honest answer is "I don't know."
    return "unknown"


if __name__ == "__main__":
    for sample in ["Hi there!", "How do I reset my password?", "This app is broken"]:
        print(f"{sample!r} -> {classify(sample)}")
```

**Why the code matches whole words.** An earlier version used `text.startswith("do")`. That also matched "download" and "document," which are not questions. Splitting the text into words and checking each word fixes that. This is the kind of bug rule systems have: the rule is only as good as the author's idea of what the words mean.

### Step 4: the learned bot (`learned_bot.py`)

Create `learned_bot.py` with this code. Nobody writes rules about words here. The model counts the words in each training message and learns which counts go with which label.

```python
"""Learned message classifier: the model estimates patterns from examples."""

import pandas as pd
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.model_selection import StratifiedKFold, cross_val_score
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import make_pipeline


def build_model():
    """Bag of words (count each word) feeding a Naive Bayes classifier."""
    return make_pipeline(CountVectorizer(lowercase=True), MultinomialNB())


if __name__ == "__main__":
    df = pd.read_csv("data/messages.csv")
    model = build_model()

    # 5 rounds: train on 4 folds, test on the fifth, then average the five scores.
    folds = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    scores = cross_val_score(model, df["text"], df["label"], cv=folds)
    print("cross-validated accuracy:", round(scores.mean(), 2))

    model.fit(df["text"], df["label"])
    for text in ["Hey, what time is it?", "The package never arrived"]:
        print(f"{text!r} -> {model.predict([text])[0]}")
```

**Why cross-validation and not one split.** With 30 rows, a single train/test split leaves only 9 test rows. In one trial on this dataset, changing the random seed moved the score from 0.22 to 0.89. Five rounds, each testing on different rows, give a steadier number. You will study this properly on Days 28 and 63.

### Step 5: compare them (`compare.py`)

Create `compare.py` with this code.

```python
"""Run the rule-based and learned classifiers on every row and show where each one is wrong."""

import pandas as pd
from sklearn.model_selection import StratifiedKFold, cross_val_predict

import learned_bot
import symbolic_bot


def main():
    df = pd.read_csv("data/messages.csv")

    # The rules were written by hand, so they need no training split.
    df["symbolic"] = df["text"].apply(symbolic_bot.classify)

    # The learned model gets a prediction for each row from a model that never saw that row.
    # Without this, the score would measure memory, not learning.
    folds = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    df["learned"] = cross_val_predict(learned_bot.build_model(), df["text"], df["label"], cv=folds)

    print(f"rows: {len(df)}")
    print(f"symbolic accuracy: {(df['symbolic'] == df['label']).mean():.2f}")
    print(f"learned accuracy:  {(df['learned'] == df['label']).mean():.2f}")

    mistakes = df[(df["symbolic"] != df["label"]) | (df["learned"] != df["label"])]
    print("\nRows where at least one approach is wrong:")
    print(mistakes.to_string(index=False))


if __name__ == "__main__":
    main()
```

**Read the symbolic number carefully.** The rule lists were written after looking at these same 30 rows, so the symbolic score is optimistic. A rule written to fit known examples often scores well on them and worse on new ones. You can test this yourself in the README (question 4).

### Step 6: requirements and run

Create `requirements.txt`:

```text
pandas>=2.0
scikit-learn>=1.3
```

Install and run:

```bash
pip install -r requirements.txt
python symbolic_bot.py
python learned_bot.py
python compare.py
```

Expected output. Your numbers should match these. If they differ slightly, your library versions differ, which is fine. Note the difference in the README.

```text
'Hi there!' -> greeting
'How do I reset my password?' -> question
'This app is broken' -> complaint
cross-validated accuracy: 0.57
'Hey, what time is it?' -> greeting
'The package never arrived' -> complaint
rows: 30
symbolic accuracy: 0.87
learned accuracy:  0.57
```

`compare.py` then prints the table of rows where at least one approach was wrong. Expect about 15 rows. Look at the `symbolic` and `learned` columns side by side. Some mistakes are made by only one approach, and those are the most useful rows.

### Step 7: write the README

In `README.md`, answer these questions in full sentences. Paste the accuracy table from your run at the top.

1. Which approach was more accurate on your data, and by how much? Give both numbers.
2. Give two examples where the rules failed and explain why. Hint: look for rows where `symbolic` is `unknown` or wrong. The row "Could you tell me the opening hours." has no question mark, so Rule 1 misses it.
3. Give two examples where the learned model failed and explain why. Hint: look at the words it could have learned from. With 30 rows, a word that appears once in training tells it almost nothing.
4. Remove the rule words you added after reading the data (keep only the question-start words and greeting words), then rerun `compare.py`. Did the symbolic accuracy drop? What does that say about how much the rule lists were tuned to this data?
5. If you added 100 more labeled messages, which approach would improve more? Why?
6. Which approach could you explain to a non-technical manager? Which one would they trust more on a new kind of message they had never seen?

### Done when

- `python compare.py` runs after `pip install -r requirements.txt`, and the output matches the expected output above (or differs only because of library versions, which you have noted).
- `data/messages.csv` has 30 rows, 10 per label.
- The README answers all six questions, and includes at least two failure cases for each approach.
- You can explain, without notes, the difference between a rule and a parameter. A rule is a decision a person wrote down. A parameter is a number inside a learned model that training adjusted. Nobody wrote the parameters of `learned_bot.py`; the fitting step set them from the data.

---

## 9. Check your understanding

1. In one sentence, what is the difference between AI and machine learning?
2. Why is every deep-learning system also a machine-learning system?
3. What is a parameter, and what changes it?
4. Name two properties of symbolic AI that are still valuable today.
5. Why might a generative model be called a "predictive" model in one sense?
6. What is the difference between training and inference, and which one costs more per use?

**Answers (check after you try):**

1. AI is the broad goal of machines doing intelligent tasks. ML is one approach to AI, where behavior comes from data.
2. Deep learning is defined as machine learning with neural networks of many layers.
3. A parameter is a number inside a model. Training adjusts parameters.
4. Explainability (you can read the rules) and reliability on well-understood, stable problems.
5. A language model is trained to predict the next token of text, so in that sense it is a predictor. A generated response is produced by repeating that prediction one token at a time.
6. Training is the one-time or occasional learning phase. Inference runs every time the model is used, so it is the repeated cost at scale.

---

## 10. Sources to read and watch

**Read (reference):**
- [Wikipedia: Artificial intelligence](https://en.wikipedia.org/wiki/Artificial_intelligence)
- [Wikipedia: Machine learning](https://en.wikipedia.org/wiki/Machine_learning)
- [Wikipedia: Deep learning](https://en.wikipedia.org/wiki/Deep_learning)
- [Wikipedia: Symbolic artificial intelligence](https://en.wikipedia.org/wiki/Symbolic_artificial_intelligence)
- [Wikipedia: Expert system](https://en.wikipedia.org/wiki/Expert_system)
- [Google Machine Learning Crash Course](https://developers.google.com/machine-learning/crash-course) (start with "Framing" and "Introduction to ML")
- [Harvard CS50's Introduction to Artificial Intelligence with Python](https://cs50.harvard.edu/ai/): watch the first lecture today, for the overall shape of the field.

**Watch:**
- [3Blue1Brown: But what is a neural network?](https://www.youtube.com/watch?v=aircAruvnKk) (about 20 minutes)
- [StatQuest with Josh Starmer](https://www.youtube.com/@statquest): search "machine learning basics" and watch one video.

**Practice:**
- [scikit-learn User Guide](https://scikit-learn.org/stable/user_guide.html): read the "Getting started" page only, for today.

**Note on links:** open each one once. If a page has moved, search its title; the content is what matters.

---

## 11. Tomorrow

Day 2, **How Machines Learn**, introduces data, features, labels, loss and training in detail. Bring your field map, your answer key results from Exercise 2, and your `compare.py` output. You will revisit the learned bot and ask what exactly it learned: which words pushed a message toward each label.
