# Day 6, Part II: How AI Works and How to Use It

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 3 hours
**Level:** beginner to intermediate. For people who already use an AI chat tool and want to understand it, and who can run a short Python script.

**By the end of this day, you will have:**

- A working model of tokens, prediction, the context window and sampling, enough to predict where a model will fail
- A script that runs one task with two prompt versions and two temperatures over a 10-case evaluation set
- A README comparing the four combinations, with numbers, a cost estimate and a decision

**What you need:**

- Python 3.11 or newer
- `openai`, version 1.0 or newer, installed with pip. Run `pip show openai` afterwards and pin the version you got
- An OpenAI account and API key, needed only for the project. The API costs money, billed per token, so check the pricing page first
- A browser, for the Playground at platform.openai.com/playground

**Prerequisites:**

- Day 6, Part I: the end-to-end experiment with a train/test split, a baseline and metrics
- Basic Python: functions, loops, reading and writing a CSV, reading environment variables

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How AI works, at the level you need](#how-ai-works-at-the-level-you-need)
- [Tokens](#tokens)
- [Next-token prediction](#next-token-prediction)
- [The context window](#the-context-window)
- [Temperature and top-p](#temperature-and-top-p)
- [System and user messages](#system-and-user-messages)
- [How models learn to follow instructions](#how-models-learn-to-follow-instructions)
- [Why models hallucinate](#why-models-hallucinate)
- [Tool use](#tool-use)
- [Retrieval over your own documents](#retrieval-over-your-own-documents)
- [Evaluation](#evaluation)
- [OpenAI as a case study](#openai-as-a-case-study)
- [Model families](#model-families)
- [ChatGPT the app and the API](#chatgpt-the-app-and-the-api)
- [The Playground](#the-playground)
- [Privacy and data handling at work](#privacy-and-data-handling-at-work)
- [The average user: ask and accept](#the-average-user-ask-and-accept)
- [The power user: structure prompts and compare outputs](#the-power-user-structure-prompts-and-compare-outputs)
- [The practitioner: measure, decide, and know when not to use AI](#the-practitioner-measure-decide-and-know-when-not-to-use-ai)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: compare prompts and temperatures on a 10-case evaluation set](#project-compare-prompts-and-temperatures-on-a-10-case-evaluation-set)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Most people meet AI as a text box. You type, something answers, and the answer is often good enough that you stop asking how it was produced. That habit fails when the stakes rise: a wrong figure in a client report, an invented citation, a confidential contract pasted into an unapproved tool.

Day 6, Part I used a model you could read line by line. A large language model has billions of parameters, so you cannot read its reasoning, but you can learn the mechanisms that drive it and measure what it does. The path has three stages: the average user, the power user and the practitioner.

---

## How AI works, at the level you need

These are the mechanisms you meet every day. No linear algebra is needed.

### Tokens

A parcel depot sorts standard pieces, not whole addresses. A model cuts text into tokens, pieces learned from training data that can be a common word, part of a rare word or a punctuation mark. Cost and length limits are counted in tokens. For English, a rough rule is three quarters of a word per token, so a 1,500-word report is about 2,000 tokens.

A common mistake is estimating cost from word count, which fails with code and JSON. On the job, every cost estimate starts with a token count.

### Next-token prediction

Phone keyboards suggest the next word, and a language model does the same with far more context. At each step it scores every possible next token, turns the scores into probabilities, picks one and repeats. For "The capital of France is", it might give " Paris" a probability of 0.90 (an illustrative figure).

A common mistake is picturing a search engine that retrieves stored sentences. Spot it when the same fact returns in different words. On the job, check outputs rather than trusting their wording.

### The context window

Think of a desk of fixed size: everything the model must use has to be on it at once. The context window is that desk, measured in tokens. It holds the system message, earlier turns, pasted documents and the reply itself. Anything outside it is invisible for that call. A 100,000-token window (an illustrative figure) cannot hold a 60,000-token contract plus 50,000 tokens of history and still leave room to answer.

A common mistake is assuming a long chat remembers an early upload. On the job, long prompts cost more on every call.

### Temperature and top-p

A chef with a ranked menu usually orders the top dish but sometimes picks the second. Temperature sharpens or flattens the scores: low values make the top token dominant, high values give unlikely tokens more chances. Top-p samples only from the smallest group of top tokens whose probabilities add up to p. The docs advise changing one of the two, not both, and some models reject temperature, so check your model. At temperature 0, a coffee shop name requested three times usually comes back the same.

A common mistake is using high temperature for labels and wondering why they change. Spot it by repeating the same input. On the job, use low settings for classification and measure the rest.

### System and user messages

A contractor gets a standing brief and a specific request for today. The brief is the system message, and the request is the user message. An API call sends a list of messages with roles: system sets standing rules, user carries the input, and assistant holds earlier replies. Some newer models use a developer message for the same job, so check the docs. A system message saying "Reply with one word: billing, technical, account or other" should turn "I was charged twice" into "billing".

A common mistake is putting rules in a user message and expecting them to hold after forty turns. On the job, system messages live in version control.

### How models learn to follow instructions

A student who has read a whole library knows language but has never practiced answering. Pretraining teaches next-token prediction on huge amounts of text. That gives knowledge but not the habit of answering: asked "How do I boil an egg?", a pretrained model may continue with more questions. Post-training then fine-tunes it on good question-and-answer examples written by people, often followed by reinforcement learning from human feedback, where people or a reward model prefer the better answers.

A common mistake is assuming your chats retrain the model. When behavior changes, check the model id first.

### Why models hallucinate

A student who never leaves an exam question blank will sometimes write a confident wrong answer. The model is trained to produce likely continuations, not to verify claims, so a specific, fluent sentence can be the most likely one even when it is false. A hallucination is fluent, confident output that is false or unsupported by its sources. Ask for three papers with authors and years on a narrow topic, and some titles may not exist.

A common mistake is treating fluency as evidence. Spot it by checking every name, number and citation. The Wikipedia article "Hallucination (artificial intelligence)" covers the research.

### Tool use

A consultant sends a request to the accounts team rather than doing the sums in their head. Your program describes tools to the model, each with a name and arguments. The model replies with a structured request to call one, your code runs it and returns the result, and the model writes the answer. The model runs nothing itself. If the model asks for `get_order_status` with order 4417 and your code returns "shipped", the model can answer correctly.

A common mistake is trusting the arguments the model writes. On the job, validate every argument and confirm before anything changes data.

### Retrieval over your own documents

An open-book exam is easier when a librarian has pulled the right pages. Retrieval-augmented generation, or RAG, splits your documents into chunks and stores an embedding for each, a list of numbers that represents its meaning. For each question, the closest chunks are pasted into the prompt. A 300-page manual is about 200,000 tokens, while five retrieved 500-token chunks send about 2,500.

A common mistake is assuming the model read an upload in full. Spot it when answers cite the wrong section. On the job, search quality often matters more than the model.

### Evaluation

A driving test measures skill on routes you did not practice. A few impressive answers show a model can do something, not that it does so reliably. An evaluation set, or eval set, is a fixed list of cases with expected results, kept apart from the prompt you are writing. With ten cases, one answer moves the score by ten points, so 9 of 10 against 8 of 10 is not yet a trustworthy gap.

A common mistake is scoring a prompt on the examples inside its own system message, which the model has already seen. On the job, eval sets decide whether a change ships.

---

## OpenAI as a case study

OpenAI is one provider among several, used here because its products show the ideas clearly and its documentation is public. Names and screens change, so check platform.openai.com/docs before following any exact step.

### Model families

OpenAI offers general text models, models tuned for multi-step reasoning, and models for audio, images and embeddings. Names change every few months, so this text names none. Pick a current model from the models page. Start with the smallest model that could plausibly do the job, run your eval set, and move up only if it falls short.

A common mistake is defaulting to the newest or largest model. Spot it when the bill rises and the score does not. On the job, justify model choice with numbers.

### ChatGPT the app and the API

A restaurant gives you a finished meal with service. A commercial kitchen account gives you ingredients, and you pay for what you use. ChatGPT is a finished product with its own interface, saved conversations, settings and data controls. The API is a set of endpoints your code calls: you send messages, choose the model and temperature, and pay per token. The app may be shaped by saved settings you have forgotten, while the API returns only what your code asks for.

A common mistake is assuming the two behave the same. On the job, client work almost always uses the API, so its terms and billing matter.

### The Playground

The Playground, at platform.openai.com/playground, is a web console for the API. Useful features include the model selector, the system and user message boxes, temperature, max output tokens (a cap on reply length) and side-by-side comparison. The layout changes often, so follow the official docs for exact steps. Playground runs are billed like API usage, so check the pricing page.

A common mistake is assuming your code will use the settings you tuned in the Playground. Compare the settings panel with your code, model included.

### Privacy and data handling at work

This is a real concern. Anything you paste may be stored, logged or used for other purposes, depending on the product, plan and settings. Read OpenAI's data controls documentation, found through the platform docs and openai.com, before any client work. Until your organization approves a tool, keep client data, personal data, credentials and unreleased financials out of it.

A common mistake is assuming a paid plan or an API key means nothing is kept. On the job, a data review is part of every AI rollout.

---

## The average user: ask and accept

Most people type a question, accept the reply and move on. The weak spots are vague requests, replies accepted because they read well, and answers that change without anyone asking why. "Write a plan for my business" returns a plan that fits any business. Three habits fix most of this: say who the answer is for, ask for a checkable shape such as a list or a single label, and accept only what you can verify.

---

## The power user: structure prompts and compare outputs

A power user writes prompts as short specifications: role, task, input, constraints and output format. Compare "Summarize this email." with "You are helping a project manager who has not read the thread. Summarize the email below in three bullet points: decision, deadline, open question. If no deadline is stated, write 'none stated'. Use only the email." The second version removes choices the model would otherwise make for you.

To compare outputs, score both against a checklist written before you look, and run the same prompt twice to see how much it moves. Keep a log of prompt, settings, model and result. That log becomes your eval set.

---

## The practitioner: measure, decide, and know when not to use AI

A practitioner treats a prompt like code. Build an eval set of about ten cases that includes the awkward ones. Count tokens from the usage numbers, multiply by the pricing page figures, and report cost per case and per thousand. Compare against a simple baseline, such as the rule set from Day 1. If the baseline scores as well, use it.

Not using AI is often right: when a rule or lookup is exact and cheaper, when an error is costly and cannot be checked, when the data may not leave your environment, or when the eval shows no gain. A README that says "no AI here, and this is why" is strong work.

---

## Hands-on exercises (about 60 minutes)

**Exercise 1: Estimate tokens (about 10 minutes).** Pick three texts you wrote, of about 50, 300 and 600 words. Estimate each with the three-quarters rule. Check: a three-row table where each estimate equals the word count divided by 0.75, rounded to the nearest ten.

**Exercise 2: Temperature by hand (about 15 minutes).** In the Playground, set the system message to "Reply with a name only" and the user message to "Suggest a name for a coffee shop on a quiet street." Run three times at temperature 0, then three times at 1.0. Check: a six-row table of replies with their temperatures, and one sentence on whether the temperature 0 runs matched and the 1.0 runs varied.

**Exercise 3: A system message changes behavior (about 15 minutes).** Send "I want to cancel my subscription. What do I do?" under two system messages. First: "You are a terse assistant. Answer in one sentence." Second: "You are a support agent for Northwind Software, a fictional company. Ask one clarifying question before giving steps." Check: the first reply is one sentence and the second ends with a question mark. If either fails, adjust the system message and record the change.

**Exercise 4: Compare two prompts (about 20 minutes).** Paste a 200-word meeting note. Prompt A: "Summarize this." Prompt B: a role, three bullet points, a 60-word limit and "use only the note". Score each output: main decision stated, 60 words or fewer, no invented fact. Check: a table of both scores and one sentence on which you would ship.

---

## Project: compare prompts and temperatures on a 10-case evaluation set

The script makes 40 calls: ten support messages, two prompt versions, two temperatures. It writes every result to a CSV and prints totals. You then write a README comparing them.

### Repository layout

```
100-days-ai-day-06-part-ii-how-ai-works/
├── README.md
├── .gitignore
├── requirements.txt
├── data/
│   └── eval_set.csv
├── prompts.py
├── run_experiment.py
└── results/
    └── results.csv        # created by the script, then committed
```

### Step 1: create the repo and environment

```bash
mkdir 100-days-ai-day-06-part-ii-how-ai-works
cd 100-days-ai-day-06-part-ii-how-ai-works
mkdir data results
python -m venv .venv
```

Activate it with `.venv\Scripts\Activate.ps1` in PowerShell, or `source .venv/bin/activate` on macOS or Linux. Put `openai>=1.0` in `requirements.txt`, run `pip install -r requirements.txt`, then replace that line with `openai==` and the version `pip show openai` reports. Create `.gitignore` with `.venv/`, `.env` and `__pycache__/`.

Set your settings for this terminal session only. In PowerShell:

```powershell
$env:OPENAI_API_KEY = "<your key>"
$env:OPENAI_MODEL = "<a model id from the models page>"
$env:PRICE_IN_PER_M = "<input price per million tokens, from the pricing page>"
$env:PRICE_OUT_PER_M = "<output price per million tokens, from the pricing page>"
```

Never write the key into a file, the README or a commit. Before each commit, run `git grep -n "sk-"` and confirm it prints nothing.

### Step 2: the evaluation set

Create `data/eval_set.csv`:

```csv
case_id,text,expected
1,"I was charged twice for my March invoice.",billing
2,"The app crashes every time I open the camera screen.",technical
3,"How do I change the email address on my account?",account
4,"Can you send me a copy of my receipt from last year?",billing
5,"Your dashboard shows blank charts since the update.",technical
6,"I forgot my password and the reset email never arrives.",account
7,"Do you have any job openings in your support team?",other
8,"Please cancel my plan and refund the last payment.",billing
9,"Login says my account is locked after three tries.",account
10,"Who is the CEO of your company?",other
```

Do not edit these cases to suit a prompt.

### Step 3: the two prompt versions

Create `prompts.py`. Only the system message differs between versions.

```python
"""Two prompt versions. Only the system message differs."""

USER_TEMPLATE = "Message: {text}"

PROMPTS = {
    "v1_minimal": {
        "system": "Classify the customer message.",
        "user": USER_TEMPLATE,
    },
    "v2_structured": {
        "system": (
            "You classify customer support messages into exactly one label: "
            "billing, technical, account or other.\n"
            "billing: charges, invoices, receipts, refunds, plans or payments.\n"
            "technical: crashes, errors, bugs or broken features.\n"
            "account: login, password, email, profile or lockouts.\n"
            "other: anything else, including questions about the company.\n"
            "Reply with the label only, in lower case, with no punctuation."
        ),
        "user": USER_TEMPLATE,
    },
}
```

### Step 4: the experiment script

Create `run_experiment.py`:

```python
import csv
import os
import sys
from pathlib import Path

from openai import OpenAI

from prompts import PROMPTS

ROOT = Path(__file__).resolve().parent
EVAL_PATH = ROOT / "data" / "eval_set.csv"
RESULTS_PATH = ROOT / "results" / "results.csv"
TEMPERATURES = [0.0, 0.8]
FIELDS = ["prompt", "temperature", "case_id", "expected", "reply",
          "correct", "prompt_tokens", "completion_tokens", "error"]


def normalize(text):
    return text.strip().lower().rstrip(".")


def ask(client, model, system, user, temperature):
    response = client.chat.completions.create(
        model=model,
        temperature=temperature,
        messages=[{"role": "system", "content": system},
                  {"role": "user", "content": user}],
    )
    usage = response.usage
    return (response.choices[0].message.content or "",
            usage.prompt_tokens, usage.completion_tokens)


def run_calls(client, model, cases):
    rows = []
    for prompt_name, prompt in PROMPTS.items():
        for temperature in TEMPERATURES:
            for case in cases:
                user = prompt["user"].format(text=case["text"])
                error = ""
                try:
                    reply, p_tok, c_tok = ask(client, model, prompt["system"],
                                              user, temperature)
                except Exception as exc:  # e.g. a model that rejects temperature
                    reply, p_tok, c_tok = "", 0, 0
                    error = type(exc).__name__
                rows.append({
                    "prompt": prompt_name,
                    "temperature": temperature,
                    "case_id": case["case_id"],
                    "expected": case["expected"],
                    "reply": reply.strip(),
                    "correct": normalize(reply) == case["expected"],
                    "prompt_tokens": p_tok,
                    "completion_tokens": c_tok,
                    "error": error,
                })
    return rows


def summarize(rows):
    price_in = float(os.environ.get("PRICE_IN_PER_M", "0"))
    price_out = float(os.environ.get("PRICE_OUT_PER_M", "0"))
    print(f"{'prompt':<16}{'temp':>6}{'correct':>10}{'errors':>8}"
          f"{'in tok':>9}{'out tok':>9}{'est cost':>12}")
    for prompt_name in PROMPTS:
        for temperature in TEMPERATURES:
            group = [r for r in rows if r["prompt"] == prompt_name
                     and r["temperature"] == temperature]
            correct = sum(r["correct"] for r in group)
            errors = sum(1 for r in group if r["error"])
            p_tok = sum(r["prompt_tokens"] for r in group)
            c_tok = sum(r["completion_tokens"] for r in group)
            cost = (p_tok * price_in + c_tok * price_out) / 1_000_000
            print(f"{prompt_name:<16}{temperature:>6}{correct:>5}/{len(group):<4}"
                  f"{errors:>8}{p_tok:>9}{c_tok:>9}{cost:>12.6f}")


def main():
    if not os.environ.get("OPENAI_API_KEY"):
        sys.exit("OPENAI_API_KEY is not set. Set it in your shell, never in a file.")
    model = os.environ.get("OPENAI_MODEL")
    if not model:
        sys.exit("OPENAI_MODEL is not set. Copy a current model id from the models page.")

    with EVAL_PATH.open(newline="", encoding="utf-8") as f:
        cases = list(csv.DictReader(f))

    rows = run_calls(OpenAI(), model, cases)  # OpenAI() reads OPENAI_API_KEY

    RESULTS_PATH.parent.mkdir(exist_ok=True)
    with RESULTS_PATH.open("w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=FIELDS)
        writer.writeheader()
        writer.writerows(rows)

    summarize(rows)


if __name__ == "__main__":
    main()
```

Errors go into a column rather than stopping the run.

### Step 5: run it

```bash
python run_experiment.py
```

Your numbers depend on the model and the run, so there is no fixed output to match. Check that `results/results.csv` has 40 data rows. An error in every row usually means a wrong model id or an unsupported temperature. Fix the cause before going on.

### Step 6: write the README

Write `README.md` with the summary table at the top. Tick each item only when it is answered with numbers from your run.

- [ ] The model id tested, the run date, and the two price values used
- [ ] Which prompt scored higher, by how many cases, and whether ten cases is enough to trust that gap
- [ ] Whether temperature 0.8 changed any replies, with the case ids where they differed
- [ ] The cases that failed under both prompts, with one sentence on why
- [ ] Estimated cost per case, and for 1,000 messages of the same length
- [ ] Whether a simple keyword rule would score well here, and how you would check
- [ ] Your decision: which prompt you would ship, and what result would change your mind

### Done when

- `results/results.csv` is committed, with 40 rows (two prompts, two temperatures, ten cases) and no unexplained errors.
- `README.md` is committed, and every checklist item is answered with numbers from your run.
- `git grep -n "sk-"` prints nothing, and `.env` and `.venv/` are not committed.

---

## Check your understanding

1. Roughly how many tokens are in a 900-word English document, and why estimate before a long run?
2. A classifier gives different labels for the same input at temperature 1.0. Which setting do you change first, and what happens to the variety of replies?
3. A chat has run for 80 turns and the model ignores a rule from the first message. Name two likely causes and one fix.
4. A model requests a tool call with an order ID that is not in your database. What should your code do before running any tool that changes data?
5. Prompt B scores 9 of 10 and prompt A scores 8 of 10. Can you ship B?

**Answers (check after you try):**

1. About 1,200 tokens (900 divided by 0.75). Estimating shows whether the text fits the context window and what the call will cost.
2. Lower the temperature first, or change top-p but not both. A lower setting makes the top choice dominant, so the variety shrinks.
3. Later turns may have pushed the rule out of the window, or it sat in a user message with less weight. Move it into the system message and trim or summarize the history.
4. Validate the arguments, refuse IDs that do not exist, and return an error to the model rather than letting it guess. Confirm before anything changes data.
5. Not yet. With ten cases, one answer is a ten-point swing, which may be noise. Run a larger set and compare cost and failing cases first.

---

## Sources to read and watch

**Read**
- [OpenAI platform documentation](https://platform.openai.com/docs): start with the text generation guide and the models page.
- [Wikipedia: Large language model](https://en.wikipedia.org/wiki/Large_language_model)
- [Wikipedia: Hallucination (artificial intelligence)](https://en.wikipedia.org/wiki/Hallucination_(artificial_intelligence))
- [Wikipedia: Retrieval-augmented generation](https://en.wikipedia.org/wiki/Retrieval-augmented_generation)
- [OpenAI](https://openai.com): search the site for its data controls and privacy pages, and read them before any work use.

**Tutorials**
- The OpenAI Cookbook, a set of example notebooks. Search "openai-cookbook" on GitHub.

**Watch**
- Andrej Karpathy, "Intro to Large Language Models" (about an hour). Search the title on YouTube.
- 3Blue1Brown, "But what is a GPT?". Search the title on YouTube.

**Practice**
- The Playground at platform.openai.com/playground, using the four exercises above.

---

## Tomorrow

Tomorrow continues the module. Keep `data/eval_set.csv` and `results/results.csv` in the repo, since measuring against a fixed set carries forward.
