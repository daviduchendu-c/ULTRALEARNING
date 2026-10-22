# Day 1: What Is Artificial Intelligence?

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 3 to 4 hours
**You will build:** an AI field map and a small project that shows the two main ways machines are made to act intelligently.
**You should know:** basic Python (variables, functions, lists). No math is needed yet.

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

## 7. Hands-on exercise (about 60 minutes)

**Exercise 1: Build your AI field map.**

1. Open a blank page, paper or whiteboard tool.
2. Draw the tree from section 4 from memory. Then compare it with the original.
3. Add five real products you use each week (for example, a phone camera's face detection, a maps app's route suggestion, a music app's recommendations, an autocorrect, a voice assistant).
4. Next to each product, write one line: is this symbolic, classical ML, deep learning, or generative? If you are not sure, write "unsure, check later." That is an honest answer.

**Exercise 2: Problem sorting.** Write one sentence each for ten everyday problems, then say whether each is best solved by rules, by learning from data, or by a mix. Examples: "sort a list of names," "tell if a photo has a cat," "predict tomorrow's electricity demand," "answer questions about a PDF," "decide whether to approve a loan."

---

## 8. Project: a symbolic bot and a learned bot that solve the same problem

This is the GitHub project for Day 1. You are not building a product yet. You are feeling the difference between the two approaches with your own hands.

**Goal:** a Python script that classifies short messages as **greeting**, **question**, or **complaint**, first with hand-written rules, then with a tiny learned model. Compare where each one fails.

### Repository layout

```
day-01-what-is-ai/
├── README.md              # what you built, what you learned, where each approach failed
├── data/
│   └── messages.csv       # 30 short sentences you write yourself, with labels
├── symbolic_bot.py        # rule-based classifier
├── learned_bot.py         # tiny statistical classifier (bag of words + Naive Bayes)
├── compare.py             # runs both, prints accuracy and misclassified examples
└── requirements.txt       # scikit-learn, pandas
```

### Step 1: write the dataset

Create `data/messages.csv` with two columns, `text` and `label`. Write 30 rows, 10 per label. Make some of them tricky (for example, "Can you stop calling me?" is a question wrapped in a complaint).

```csv
text,label
Hi there!,greeting
Good morning team,greeting
How do I reset my password?,question
When does the store open?,question
This app keeps crashing and I am furious,complaint
Your delivery was three days late,complaint
...
```

### Step 2: the symbolic bot (`symbolic_bot.py`)

```python
"""Rule-based message classifier: humans write the rules."""

GREETING_WORDS = {"hi", "hello", "hey", "good morning", "good evening"}
QUESTION_STARTS = ("how", "what", "when", "where", "why", "who", "can", "do", "does", "is")
COMPLAINT_WORDS = {"crashing", "late", "broken", "furious", "angry", "terrible", "worst"}


def classify(text: str) -> str:
    lowered = text.lower().strip()
    words = set(lowered.replace("!", "").replace("?", "").split())

    if text.strip().endswith("?") or lowered.startswith(QUESTION_STARTS):
        return "question"
    if words & COMPLAINT_WORDS:
        return "complaint"
    if any(lowered.startswith(g) for g in GREETING_WORDS):
        return "greeting"
    return "unknown"


if __name__ == "__main__":
    for sample in ["Hi there!", "How do I reset my password?", "This app is broken"]:
        print(f"{sample!r} -> {classify(sample)}")
```

### Step 3: the learned bot (`learned_bot.py`)

This uses bag-of-words features and Naive Bayes. You will study both properly on Days 28, 63 and 64. For now, run it and observe.

```python
"""Learned message classifier: the model estimates patterns from examples."""

import pandas as pd
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.model_selection import train_test_split
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import make_pipeline


def train(csv_path: str = "data/messages.csv"):
    df = pd.read_csv(csv_path)
    X_train, X_test, y_train, y_test = train_test_split(
        df["text"], df["label"], test_size=0.3, random_state=42, stratify=df["label"]
    )
    model = make_pipeline(CountVectorizer(lowercase=True), MultinomialNB())
    model.fit(X_train, y_train)
    return model, X_test, y_test


if __name__ == "__main__":
    model, X_test, y_test = train()
    print("held-out accuracy:", round(model.score(X_test, y_test), 2))
    for text in ["Hey, what time is it?", "The package never arrived"]:
        print(f"{text!r} -> {model.predict([text])[0]}")
```

### Step 4: compare (`compare.py`)

Write a script that loads the CSV, runs `symbolic_bot.classify` on every row, runs the learned model on every row, and prints:

- accuracy for each approach on the full dataset
- every row that either approach gets wrong, side by side

Keep this short. The point is the table of mistakes, not the number.

### Step 5: write the README

In `README.md`, answer these five questions in full sentences:

1. Which approach was more accurate on your data, and by how much?
2. Give two examples where the rules failed and explain why.
3. Give two examples where the learned model failed and explain why.
4. If you added 100 more labeled messages, which approach would improve more? Why?
5. Which approach could you explain to a non-technical manager? Which one would they trust more?

### Done when

- The repo runs with `python compare.py` after `pip install -r requirements.txt`.
- The README has honest failure cases, not just the good results.
- You can explain, without notes, the difference between a rule and a parameter.

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
5. A language model predicts the next token. Generation is repeated prediction.
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

Day 2, **How Machines Learn**, introduces data, features, labels, loss and training in detail. Bring your field map and your `compare.py` results. You will revisit the learned bot and ask what exactly it learned.
