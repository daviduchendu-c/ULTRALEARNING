# Day 5: AI Development Environment

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 4 hours (installing PyTorch can take a while on a slow connection)
**You will build:** a reusable starter repo with a pinned environment, an environment check script that reports your library versions and compute device, a short tour of the core libraries, a Jupyter notebook, and a reproducibility test that shows your results can be repeated exactly.
**You should know:** Days 1 to 4, how to open a terminal, and how to create and activate a virtual environment (Days 1, 2 and 4 covered it).
**You will install:** Python 3.12 (3.11 or newer works), NumPy, pandas, Matplotlib, scikit-learn, PyTorch and JupyterLab, all at pinned versions. Optionally, Hugging Face `transformers`. Everything today is free.

---

## 1. The big idea

An ML result is only as trustworthy as the environment that produced it. If your README says "test accuracy 0.97" and a colleague gets 0.94 on their laptop, nobody knows which number is real. Treat the environment as part of the code. The Python version, every library version, the hardware (CPU or GPU) and the random seeds get written down and checked in.

Today you set up the toolkit for the rest of the series and make your results repeatable. Anyone who clones your repo should be able to run one install command and one run command, and get the numbers you reported.

---

## 2. A short history of the Python AI stack

| Year | Tool | Why it matters today |
|---|---|---|
| 2003 | [Matplotlib](https://en.wikipedia.org/wiki/Matplotlib) (John Hunter) | The default plotting library. Most other Python plotting tools build on it or copy its interface. |
| 2006 | [NumPy](https://en.wikipedia.org/wiki/NumPy) 1.0 (Travis Oliphant, merging earlier Numeric and Numarray) | Fast arrays. Every library below is built on NumPy arrays or works like them. |
| 2007 | [scikit-learn](https://en.wikipedia.org/wiki/Scikit-learn) starts as a Google Summer of Code project | Classical ML with one consistent `fit` / `predict` interface. |
| 2007 | NVIDIA releases [CUDA](https://en.wikipedia.org/wiki/CUDA) | Lets general programs run on GPUs, which later made deep learning practical. |
| 2008 | [pandas](https://en.wikipedia.org/wiki/Pandas_(software)) (Wes McKinney) | Tables with named columns, the standard tool for loading and cleaning data. |
| 2014 | [Project Jupyter](https://en.wikipedia.org/wiki/Project_Jupyter) spins out of IPython | Notebooks that mix code, output, plots and notes. |
| 2016 | [PyTorch](https://en.wikipedia.org/wiki/PyTorch) released by Facebook AI Research | Now the most widely used deep learning framework in research and much of industry. |
| 2018 onward | [Hugging Face](https://en.wikipedia.org/wiki/Hugging_Face) `transformers` library and Model Hub | Download and run thousands of pretrained models in a few lines. |

---

## 3. The stack and the tools

Python is the dominant language for AI work. Python itself is slow, but the heavy math runs in compiled C, C++ and CUDA code inside the libraries, and Python glues them together.

```
┌──────────────────────────────────────────────────────────────────┐
│  Your code: scripts (.py) and notebooks (.ipynb in JupyterLab)    │
├───────────────┬──────────────────┬───────────────────────────────┤
│ scikit-learn  │ PyTorch          │ Hugging Face (transformers,    │
│ classical ML  │ deep learning    │ datasets, Hub): pretrained     │
│               │                  │ models, built on PyTorch       │
├───────────────┴───────┬──────────┴───────────────────────────────┤
│ pandas (tables)       │ Matplotlib (plots)                        │
├───────────────────────┴──────────────────────────────────────────┤
│ NumPy (arrays) and SciPy (scientific routines)                    │
├──────────────────────────────────────────────────────────────────┤
│ Python 3.12 inside a virtual environment (.venv)                  │
├──────────────────────────────────────────────────────────────────┤
│ Hardware: CPU always; GPU via CUDA (NVIDIA) or MPS (Apple)        │
└──────────────────────────────────────────────────────────────────┘
```

Read it bottom up. Each layer uses the ones below it, so when something breaks, the cause is usually the lowest layer that is wrong. A wrong Python version breaks everything above it.

| Job | Tool |
|---|---|
| Load a CSV, clean columns, group and count | pandas |
| Math on whole arrays of numbers | NumPy |
| Plot a distribution or a loss curve | Matplotlib |
| Train a logistic regression, random forest or k-means | scikit-learn |
| Build and train a neural network | PyTorch |
| Use a pretrained language or vision model | Hugging Face `transformers` |
| Explore data interactively and show results | Jupyter |

**NumPy** provides the `ndarray`, a fixed-type grid of numbers, and fast operations on whole arrays (vectorization) without Python loops.

```python
import numpy as np

prices = np.array([100.0, 250.0, 80.0])
print(prices * 1.2)          # [120. 300.  96.]  every element at once
a = np.array([[1.0, 2.0], [3.0, 4.0]])
print(a.shape, a.mean(), a.sum(axis=0))   # (2, 2) 2.5 [4. 6.]
```

`axis=0` collapses the rows, so the sum runs down each column. Print `shape` first when something goes wrong. The most common NumPy error is a shape mismatch, such as a `(100,)` array where a function expects `(100, 1)`. That is why Day 2's `gradient_descent.py` used `x_train.reshape(-1, 1)`. Day 17 covers NumPy in depth.

**pandas** provides the `DataFrame`, a table with named, typed columns, plus tools to load, filter, join, group and summarize data.

```python
import pandas as pd

df = pd.DataFrame({"city": ["Lagos", "Accra", "Lagos"], "sales": [120, 80, 200]})
print(df.groupby("city")["sales"].sum())
# Accra     80
# Lagos    320
```

Two traps. Chained assignment such as `df[df.sales > 100]["city"] = "X"` may change a temporary copy. Use `df.loc[df.sales > 100, "city"] = "X"` instead. Also check `df.dtypes` after reading a CSV, because numbers stored as text will not add up. Day 18 covers pandas in depth.

**Matplotlib** draws static plots: lines, scatter plots, histograms and images. Give every plot axis labels and units. On a server with no display, save with `fig.savefig` instead of calling `plt.show()`, which is why today's code uses the `Agg` backend.

**scikit-learn** implements classical ML, preprocessing and evaluation behind one interface: `fit(X, y)` to train, `predict(X)` to infer, `transform(X)` for preprocessing, and `Pipeline` to chain them. Swapping a logistic regression for a random forest changes one word. The common mistake is fitting a scaler on all the data before splitting, which is the leak from Day 3. Put preprocessing inside a `Pipeline` so it is fit on training rows only.

**PyTorch** provides tensors (arrays that can live on a CPU or a GPU), automatic differentiation (it computes the gradients you wrote by hand on Day 2), and the building blocks of neural networks.

```python
import torch

x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
print(x @ x)        # matrix product: tensor([[ 7., 10.], [15., 22.]])
device = "cuda" if torch.cuda.is_available() else "cpu"
x = x.to(device)    # move the tensor to the GPU if there is one
```

Copy the install command from the selector at [pytorch.org/get-started/locally](https://pytorch.org/get-started/locally/) rather than typing your own, because builds differ by hardware. The GPU install is covered in section 6 and the project. A common error is `RuntimeError: Expected all tensors to be on the same device`. Move the model and the data to the same device. Module 5 teaches PyTorch properly.

**Hugging Face** runs the **Hub**, which holds hundreds of thousands of public models and datasets, and maintains libraries: `transformers` (pretrained models for text, vision and audio), `datasets` (loading and processing data), and `tokenizers` (fast text splitting). Most Hub models run on PyTorch. Public models are free to download, but check each model's license before you use it in a product. Pin the model name, and ideally its revision, so a different model does not download later. Expect the first run to download hundreds of megabytes into a cache folder (`~/.cache/huggingface` by default).

---

## 4. Virtual environments and pinned requirements

**Intuition.** Each project gets its own toolbox. If one project needs an old wrench, that does not force every other project to use it.

**Definition.** A **virtual environment** is a folder (here `.venv`) with its own Python interpreter and installed packages. Activating it makes `python` and `pip` point to that folder. **pip** is Python's installer, and it downloads **packages** from PyPI (the Python Package Index). A **requirements.txt** file lists what a project needs. **Pinning** writes an exact version (`pandas==2.2.3`) instead of a range (`pandas>=2.0`). A **dependency** is a package that another package needs, such as NumPy and SciPy for scikit-learn. [Python docs](https://docs.python.org/3/library/venv.html)

```bash
python -m venv .venv                 # create
source .venv/bin/activate            # activate (macOS/Linux)
.venv\Scripts\Activate.ps1           # activate (Windows PowerShell)
python -m pip install -r requirements.txt
python -m pip freeze > requirements-lock.txt   # record every installed package, including dependencies
```

`pip freeze` lists dependencies you never asked for, such as `scipy`, `joblib` and `threadpoolctl`. Keep `requirements.txt` short and readable. Commit the frozen lock file next to it for exact rebuilds.

**Common mistakes.** Installing into the system Python with no environment, then wondering why another project broke. Forgetting to activate, so `pip install` goes somewhere else. Check with `python -c "import sys; print(sys.prefix)"`, which should end in `.venv`. Committing `.venv` to Git. It is large and specific to one machine, so add it to `.gitignore`. On Windows PowerShell, activation can fail with an execution policy error. Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once, or use the Command Prompt activation from Day 1.

**Alternatives you will see at work:** `uv` (a fast, free installer from Astral), Poetry, and conda. Conda is popular for scientific packages. Anaconda's default channel has commercial licensing terms for larger organizations, while Miniforge with conda-forge is free. This series uses `venv` and `pip`, which come with Python.

---

## 5. Jupyter notebooks

**Intuition.** A lab notebook where every experiment, its result and your notes sit on the same page.

**Definition.** A notebook is a file of cells. **Code cells** run in a **kernel**, a live Python process that keeps variables in memory between cells, and their output (text, tables, plots) is saved under the cell. **Markdown cells** hold notes. JupyterLab is the browser-based editor, and VS Code also opens `.ipynb` files.

**Example.** Start `jupyter lab` from your activated environment. In a cell, run `import pandas as pd`, then in the next cell run `pd.__version__`. The version printed comes from the environment you launched Jupyter from.

**Common mistakes.** Running cells out of order. If you define `x` in cell 5, then delete cell 5, `x` still exists in the kernel, and the notebook fails for anyone who runs it top to bottom. Before you commit, use **Kernel → Restart Kernel and Run All Cells**. Another trap is a kernel that uses a different Python from your environment, so imports fail. Check with `import sys; sys.executable`.

**Limitation.** Notebooks are good for exploration and poor for production code. They are hard to test, hard to review in Git diffs and easy to run out of order. The habit in this series: explore in notebooks, then move the code you keep into `.py` files.

---

## 6. Hardware: CPU, GPU and CUDA

**Intuition.** A CPU is a few expert chefs who can each cook anything. A GPU is a thousand line cooks who can each only chop, but all chop at once. Deep learning is mostly chopping: multiplying large grids of numbers.

**Definition.** A CPU has a small number of complex cores, built for varied, sequential work. A GPU has thousands of simpler cores, built to run the same operation on many numbers in parallel. [Wikipedia](https://en.wikipedia.org/wiki/Graphics_processing_unit) Neural network training and inference are dominated by matrix multiplications, which parallelize well. A **device** is where a PyTorch tensor lives and computes: `cpu`, `cuda` (NVIDIA GPU) or `mps` (Apple silicon GPU).

**CUDA** is NVIDIA's parallel computing platform: a driver, a toolkit and libraries (cuBLAS for matrix math, cuDNN for neural network operations). A PyTorch CUDA build bundles the CUDA libraries it needs, so you only need a recent NVIDIA driver. CUDA works only on NVIDIA GPUs. AMD's equivalent is ROCm. Apple's is Metal, which PyTorch uses through the `mps` device.

**Example.** Multiplying two 4,096 by 4,096 matrices takes about 137 billion floating-point operations, or about 68.7 billion multiply-adds with each one counted as two operations. A modern GPU does this many times faster than a laptop CPU. The scikit-learn models in this module run on the CPU, so a GPU gives them no benefit.

`torch.cuda.is_available()` returns `True` only when three things line up: an NVIDIA GPU, a working driver, and a CUDA build of PyTorch. `torch.version.cuda` prints the CUDA version the build uses, or `None` for a CPU build. `nvidia-smi` shows the GPU, the driver version and current memory use.

**Common mistakes.**
- Buying or renting a GPU before you need one. The early days of this series run on a laptop CPU. From Module 5 on, Google Colab's free tier and Kaggle Notebooks give limited free GPU time. Cloud GPUs are paid by the hour.
- Installing the full CUDA Toolkit separately and assuming PyTorch will use it. pip builds bring their own libraries.
- A driver older than the CUDA version the PyTorch build needs. `is_available()` then returns `False`.

On a job post, "CUDA" usually means comfort with GPU setup and debugging. Writing CUDA kernels by hand is a specialist skill.

---

## 7. Reproducibility and random seeds

**Intuition.** A recipe is only useful if someone else can follow it and get the same cake. A shuffled deck is repeatable if you write down exactly how you shuffled.

**Definition.** A result is reproducible when the same code, data and environment give the same numbers. That needs pinned library versions, a recorded Python version, fixed random seeds, a fixed data version, and a written record of how to run everything.

A **random seed** is the starting number for a pseudo-random number generator. Computers make "random" numbers with a deterministic algorithm, so the same seed gives the same sequence. [Wikipedia](https://en.wikipedia.org/wiki/Random_seed) ML uses randomness in train/test splits, weight initialization, shuffling, random forests and dropout. Each library has its own generator: Python's `random`, NumPy, PyTorch, and the `random_state` argument in scikit-learn.

In the project, two runs with seed 42 give identical accuracy and an identical PyTorch number. Runs with no seed differ each time. `random_state=42` in scikit-learn and `torch.manual_seed(42)` are separate settings, so you need both.

**Common mistakes.**
- Seeding only one library.
- Choosing the seed that gives the best score ("seed hacking"). If your result changes a lot between seeds, report the mean and spread over several seeds. On Day 1, one seed gave 0.22 and another gave 0.89.
- Results that depend on unrecorded state: notebook cells run out of order, a file edited by hand, or a dataset downloaded as "latest."
- Expecting bit-for-bit identical numbers across machines. Different CPUs, operating systems, math libraries and GPU kernels can change floating-point results slightly. Some GPU operations are nondeterministic by default. PyTorch's reproducibility notes explain `torch.use_deterministic_algorithms(True)`, at some speed cost. Aim for identical results on one machine and close results across machines.

In regulated work such as finance or health, you may need to reproduce a model's results months later. Day 89 covers reproducible pipelines.

---

## 8. Hands-on exercises (about 60 minutes)

**Exercise 1: break and fix an environment (about 20 minutes).**

1. In a scratch folder, create a venv, activate it, and run `pip install pandas==2.2.3`.
2. Run `python -c "import sklearn"`. Read the error (`ModuleNotFoundError`). Explain in one sentence why it happened.
3. Deactivate the environment (`deactivate`) and run `python -c "import sys; print(sys.prefix)"`. Activate it again and rerun. Write down both paths and what changed.
4. Run `pip freeze`. Count the packages listed, although you installed only one. Name two of them and guess why pandas needs them.

**Exercise 2: notebook hygiene (about 20 minutes).**

1. Start `jupyter lab` from an activated environment that has JupyterLab installed (you will have one after step 2 of the project).
2. In cell 1, write `x = 10`. In cell 2, write `print(x * 2)`. Run both.
3. Delete cell 1. Run cell 2 again. It still prints 20. Explain why.
4. Choose Kernel → Restart Kernel and Run All Cells. Read the error. This is why you restart and run all before committing a notebook.

**Exercise 3: where does my compute go? (about 20 minutes).** Run this and time it:

```python
import time
import numpy as np

n = 1_000_000
values = np.random.default_rng(0).random(n)

start = time.perf_counter()
total = 0.0
for v in values:
    total += v * v
loop_s = time.perf_counter() - start

start = time.perf_counter()
total_np = float(np.dot(values, values))
numpy_s = time.perf_counter() - start

print(f"python loop: {loop_s:.3f} s, numpy: {numpy_s:.5f} s, speedup: {loop_s / numpy_s:.0f}x")
```

Record the speedup on your machine. It varies a lot by machine. One Windows laptop measured about 800 times. Write one sentence explaining why the NumPy version is faster.

---

## 9. Project: an AI environment starter repo

**Goal:** a repo you can copy at the start of every later project: pinned requirements, a `.gitignore`, an environment check, a library tour, a notebook and a reproducibility test.

Suggested repo name: `100-days-ai-engineering-day-05-ai-environment`.

### Repository layout

```
day-05-ai-environment/
├── README.md                  # setup steps and the Environment section from env_check.py
├── .gitignore
├── requirements.txt           # pinned direct dependencies
├── requirements-lock.txt      # output of pip freeze
├── requirements-optional.txt  # Hugging Face, only if you want the optional step
├── env_check.py               # versions and compute device
├── tour.py                    # NumPy, pandas, Matplotlib, scikit-learn in one task
├── reproducibility.py         # seeds on vs seeds off
├── hf_demo.py                 # optional: a pretrained model from the Hub
├── notebooks/
│   └── 01_tour.ipynb          # the tour, rebuilt as a notebook
└── iris_petals.png            # created by tour.py
```

### Step 1: install Python 3.12 and create the environment

Install Python 3.12 from [python.org/downloads](https://www.python.org/downloads/). On Windows, tick "Add python.exe to PATH" in the installer. Check with `python --version`. If you have several Pythons, use `py -3.12 -m venv .venv` on Windows, or `python3.12 -m venv .venv` on macOS and Linux.

```bash
mkdir day-05-ai-environment
cd day-05-ai-environment
mkdir notebooks
python -m venv .venv
```

Activate it (as on Day 1) and upgrade pip: `python -m pip install --upgrade pip`.

### Step 2: pinned requirements

Create `requirements.txt`:

```text
numpy==2.2.6
pandas==2.2.3
scipy==1.15.2
scikit-learn==1.6.1
matplotlib==3.10.0
torch==2.6.0
jupyterlab==4.3.5
ipykernel==6.29.5
```

Create `requirements-optional.txt`:

```text
transformers==4.48.3
```

Install and record the full environment:

```bash
python -m pip install -r requirements.txt
python -m pip freeze > requirements-lock.txt
```

If you have an NVIDIA GPU, install PyTorch with the CUDA command from the PyTorch selector **before** the rest, for example `python -m pip install torch==2.6.0 --index-url https://download.pytorch.org/whl/cu124`. pip will then see that `torch==2.6.0` is already satisfied. The download is large: several hundred megabytes for CPU builds, a few gigabytes for CUDA builds.

**Untested:** the CUDA command above has not been run on a GPU machine. Check it against the selector on pytorch.org before you use it.

Create `.gitignore`:

```text
.venv/
__pycache__/
.ipynb_checkpoints/
```

### Step 3: environment check (`env_check.py`)

```python
"""Print what this environment contains and which compute device PyTorch can use."""

import platform
import sys
from importlib.metadata import PackageNotFoundError, version

PACKAGES = ["numpy", "pandas", "matplotlib", "scikit-learn", "torch", "jupyterlab"]


def pick_device():
    import torch

    if torch.cuda.is_available():
        return "cuda", torch.cuda.get_device_name(0)
    if torch.backends.mps.is_available():
        return "mps", "Apple GPU"
    return "cpu", platform.processor() or "CPU"


def main():
    print(f"python   {sys.version.split()[0]}  ({sys.executable})")
    if sys.version_info < (3, 11):
        print("WARNING: this series expects Python 3.11 or newer")

    for name in PACKAGES:
        try:
            print(f"{name:<12} {version(name)}")
        except PackageNotFoundError:
            print(f"{name:<12} NOT INSTALLED")

    import torch

    device, detail = pick_device()
    print(f"\ndevice   {device} ({detail})")
    print(f"torch built with CUDA: {torch.version.cuda or 'no (CPU-only build)'}")

    # A tiny computation on the chosen device proves the install works end to end.
    a = torch.ones(3, 3, device=device)
    print(f"sum of a 3x3 matrix of ones on {device}: {a.sum().item()}")


if __name__ == "__main__":
    main()
```

Run `python env_check.py`. Expected output on a Windows laptop without an NVIDIA GPU (your paths and processor name will differ):

```text
python   3.12.x  (C:\...\day-05-ai-environment\.venv\Scripts\python.exe)
numpy        2.2.6
pandas       2.2.3
matplotlib   3.10.0
scikit-learn 1.6.1
torch        2.6.0
jupyterlab   4.3.5

device   cpu (Intel64 Family 6 Model 142 Stepping 12, GenuineIntel)
torch built with CUDA: no (CPU-only build)
sum of a 3x3 matrix of ones on cpu: 9.0
```

**Untested GPU output.** On a machine with a working NVIDIA setup, the device line should read `cuda (<your GPU name>)`, and the CUDA line should show a version such as `12.4`. This has not been run on a GPU machine, so check it against your own output. If you have an NVIDIA GPU and still see `cpu`, you probably installed the CPU build. Reinstall with the CUDA command. If the interpreter path does not end in `.venv\Scripts\python.exe` (Windows) or `.venv/bin/python` (macOS, Linux), your environment is not active.

### Step 4: the library tour (`tour.py`)

```python
"""A ten-minute tour: one small task through NumPy, pandas, Matplotlib and scikit-learn."""

import matplotlib

matplotlib.use("Agg")  # save plots to files; remove this line inside Jupyter
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
from sklearn.datasets import load_iris
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split

# NumPy: fast arrays and math on whole arrays at once.
a = np.array([[1.0, 2.0], [3.0, 4.0]])
print("NumPy  shape", a.shape, "mean", a.mean(), "column sums", a.sum(axis=0))

# pandas: tables with named columns.
iris = load_iris(as_frame=True)
df = iris.frame
df["species"] = df["target"].map(dict(enumerate(iris.target_names)))
print("\npandas  rows:", len(df))
print(df.groupby("species")["petal length (cm)"].mean().round(2).to_string())

# Matplotlib: plots.
fig, ax = plt.subplots(figsize=(5, 4))
for name, group in df.groupby("species"):
    ax.scatter(group["petal length (cm)"], group["petal width (cm)"], label=name, s=15)
ax.set_xlabel("petal length (cm)")
ax.set_ylabel("petal width (cm)")
ax.legend()
fig.savefig("iris_petals.png", dpi=120, bbox_inches="tight")
print("\nMatplotlib  saved iris_petals.png")

# scikit-learn: models with the same fit / predict interface.
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.3, stratify=iris.target, random_state=42
)
model = LogisticRegression(max_iter=1000).fit(X_train, y_train)
print(f"\nscikit-learn  test accuracy: {model.score(X_test, y_test):.2f}")
```

Expected output:

```text
NumPy  shape (2, 2) mean 2.5 column sums [4. 6.]

pandas  rows: 150
species
setosa        1.46
versicolor    4.26
virginica     5.55

Matplotlib  saved iris_petals.png

scikit-learn  test accuracy: 0.93
```

Open `iris_petals.png`. One species sits far from the other two, and the other two overlap a little. The overlap explains the 0.93. The mistakes happen there.

### Step 5: the same tour as a notebook

Run `jupyter lab`, create `notebooks/01_tour.ipynb`, and rebuild `tour.py` as cells: one Markdown cell explaining each library, then its code cell. Delete the `matplotlib.use("Agg")` line and add `plt.show()` after the plot so it appears inline. Finish with **Restart Kernel and Run All Cells**, check that every cell ran without errors, and save. The notebook runs from the `notebooks/` folder, so the saved image goes there unless you change the path.

### Step 6: the reproducibility test (`reproducibility.py`)

```python
"""Show that fixed seeds give identical results and missing seeds do not."""

import random

import numpy as np
import torch
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split


def set_seed(seed: int) -> None:
    """Seed every random number generator this project uses."""
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)


def run(seed: int | None) -> tuple[float, float]:
    """Split, train a random forest, and return its test accuracy and one torch number."""
    if seed is None:
        # Behave like a fresh program that never set a seed: take randomness from the OS.
        random.seed()
        np.random.seed()
        torch.seed()
    else:
        set_seed(seed)
    X, y = load_iris(return_X_y=True)
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=seed)
    model = RandomForestClassifier(n_estimators=20, max_depth=2, random_state=seed)
    model.fit(X_train, y_train)
    weight = torch.randn(1).item()  # stands in for a randomly initialized network weight
    return round(model.score(X_test, y_test), 4), round(weight, 4)


if __name__ == "__main__":
    print("seed 42, run 1:", run(42))
    print("seed 42, run 2:", run(42))
    print("seed 7:        ", run(7))
    print("no seed, run 1:", run(None))
    print("no seed, run 2:", run(None))
```

Expected output. The first three lines should match exactly. The two "no seed" lines will differ on every run, and from the values below:

```text
seed 42, run 1: (1.0, 0.3367)
seed 42, run 2: (1.0, 0.3367)
seed 7:         (0.9111, -0.1468)
no seed, run 1: (1.0, -0.4443)
no seed, run 2: (0.8889, -0.4112)
```

Run the script three times. Seeded lines never change, and unseeded lines change every time. Seed 42 and seed 7 give accuracies of 1.0 and 0.91 from the same code. With only 45 test rows, the split alone moves the score by 9 points.

### Step 7 (optional): a pretrained model from the Hub (`hf_demo.py`)

Install with `python -m pip install -r requirements-optional.txt`, then create `hf_demo.py`:

```python
"""Optional: run a small pretrained model from the Hugging Face Hub (downloads about 270 MB once)."""

from transformers import pipeline

MODEL = "distilbert/distilbert-base-uncased-finetuned-sst-2-english"

classifier = pipeline("sentiment-analysis", model=MODEL)
for text in ["This course is clear and useful.", "The install took forever and then failed."]:
    result = classifier(text)[0]
    print(f"{result['label']:<8} {result['score']:.3f}  {text}")
```

Expected output (after some download progress lines and a `Device set to use cpu` message):

```text
POSITIVE 1.000  This course is clear and useful.
NEGATIVE 1.000  The install took forever and then failed.
```

A score of 1.000 is rounded. The model is very confident on clear sentences like these. Try a sarcastic one ("Great, another error message.") and see how it does. You may also see a note suggesting `hf_xet` for faster downloads. It is optional.

### Step 8: prove the environment with an old result

If you did the optional Day 4 check, copy `three_framings.py` into this repo, run it, and confirm it prints the same numbers as on Day 4. If it does, your new environment reproduces an old result. Note that in the README.

### Step 9: write the README

Include:

1. **Setup:** the exact commands to create the environment and install, for your operating system.
2. **Environment:** paste the full output of `env_check.py`.
3. **Results:** the outputs of `tour.py` and `reproducibility.py`, and the Day 4 rerun if you did it.
4. **Answers:**
   - What is the difference between `requirements.txt` and `requirements-lock.txt` in this repo, and when would you use each?
   - Your Exercise 3 speedup, and why NumPy is faster.
   - Why did seed 42 and seed 7 give different accuracies, and what would an honest report of this model's accuracy look like? (Hint: run seeds 0 to 9 and report the mean and the range.)
   - Does your machine have a usable GPU? How did you check?

### Done when

- A fresh clone can be set up with the commands in your README, and `python env_check.py` runs without errors.
- `requirements.txt` pins every direct dependency with `==`, and `requirements-lock.txt` is committed.
- `.venv/` is not in the repo.
- `reproducibility.py` gives identical seeded lines on every run.
- `notebooks/01_tour.ipynb` runs top to bottom after a kernel restart.
- The README has all four sections, including the seed experiment over seeds 0 to 9.

---

## 10. Check your understanding

1. Your colleague's code fails with `ModuleNotFoundError: No module named 'sklearn'`, but they swear they installed it. Give the two most likely causes.
2. Why pin `scikit-learn==1.6.1` instead of writing `scikit-learn>=1.6`?
3. A notebook works on your machine and fails for a reviewer with `NameError: name 'df' is not defined`. What happened, and what habit prevents it?
4. `torch.cuda.is_available()` returns `False` on a laptop with an NVIDIA GPU. List three things to check.
5. Why does a GPU speed up neural network training but not a scikit-learn random forest?
6. Every seed is fixed, but your result differs in the third decimal place on a colleague's machine. Is something broken?
7. You report the best accuracy out of 20 seeds. Why is that misleading, and what should you report instead?

**Answers (check after you try):**

1. The package was installed into a different environment than the one running the code (the wrong venv, or none activated). Also note that the import name is `sklearn` but the install name is `scikit-learn`. Check `sys.executable` and `pip list` in the same terminal.
2. A range lets pip install a newer version later, whose defaults or behavior may differ, so your results may change or your code may break. An exact pin rebuilds the same environment.
3. A cell that defined `df` was deleted or edited after it ran, and the kernel still held the old variable. Restart the kernel and run all cells before committing.
4. A CPU-only PyTorch build is installed (`torch.version.cuda` is `None`). The NVIDIA driver is missing or too old (`nvidia-smi` fails or shows an old version). The active environment is not the one where the CUDA build was installed.
5. Neural network training is dominated by large matrix multiplications that GPUs run in parallel, and PyTorch has GPU implementations of them. scikit-learn's random forest runs on the CPU. Tree building involves many branching decisions, which GPUs are not designed for.
6. Probably not. Different CPUs, operating systems, math library builds and GPU kernels can change floating-point results slightly. Exact repeatability is expected on the same setup, and small differences across setups are normal. Large differences mean a real mismatch, such as different versions or data.
7. The maximum of 20 noisy results is biased upward, the same as picking the best validation score on Day 3. Report the mean and the spread (standard deviation or range) across seeds.

---

## 11. Sources to read and watch

**Read (reference):**
- [Python docs: venv, creation of virtual environments](https://docs.python.org/3/library/venv.html) and the tutorial page [Virtual Environments and Packages](https://docs.python.org/3/tutorial/venv.html)
- [NumPy: the absolute basics for beginners](https://numpy.org/doc/stable/user/absolute_beginners.html)
- [pandas: 10 minutes to pandas](https://pandas.pydata.org/docs/user_guide/10min.html)
- Matplotlib: the "Quick start guide" in the official documentation at matplotlib.org (search the title on the site)
- [scikit-learn: Getting started](https://scikit-learn.org/stable/getting_started.html)
- [PyTorch: Get started locally](https://pytorch.org/get-started/locally/) (the install selector)
- [PyTorch: Reproducibility notes](https://pytorch.org/docs/stable/notes/randomness.html)
- [scikit-learn: Common pitfalls](https://scikit-learn.org/stable/common_pitfalls.html), the section "Controlling randomness"
- [JupyterLab documentation](https://jupyterlab.readthedocs.io/)
- [Wikipedia: CUDA](https://en.wikipedia.org/wiki/CUDA), [Wikipedia: Graphics processing unit](https://en.wikipedia.org/wiki/Graphics_processing_unit), [Wikipedia: Pseudorandom number generator](https://en.wikipedia.org/wiki/Pseudorandom_number_generator)

**Tutorials:**
- [W3Schools: NumPy Tutorial](https://www.w3schools.com/python/numpy/default.asp) and [W3Schools: Pandas Tutorial](https://www.w3schools.com/python/pandas/default.asp)
- GeeksforGeeks: search "Python Virtual Environment" and "Getting started with Jupyter Notebook" on geeksforgeeks.org.
- [PyTorch: Learn the Basics](https://pytorch.org/tutorials/beginner/basics/intro.html), the official beginner tutorial. Read only the "Tensors" page today.
- Hugging Face: the free "LLM Course" (formerly the NLP course) at huggingface.co/learn. Read the chapter 1 page on what Transformers can do.

**Watch:**
- [CS50's Introduction to Programming with Python](https://cs50.harvard.edu/python/): the lecture on "Libraries" covers packages and pip.
- The official PyTorch YouTube channel: search "PyTorch" for an introductory video on tensors.
- NVIDIA's official YouTube channel: search "What is CUDA" for a short explainer.

**Practice:**
- [Google Colab](https://colab.research.google.com): open a new notebook, run `import torch; torch.cuda.is_available()`, then switch the runtime type to a GPU (Runtime → Change runtime type) and run it again. Free GPU time is limited and not guaranteed.

---

## 12. Tomorrow

Day 6, **First End-to-End AI Experiment**, puts the module together in one project. Using the environment you built today, you will take a real medical dataset through every stage of the Day 3 lifecycle: load it, inspect it, preprocess it, split it, beat a baseline, measure it with metrics chosen by the Day 4 formulation, save predictions, and read the model's mistakes one by one. Copy today's `requirements.txt`, `.gitignore` and `env_check.py` into the new repo. That is what a starter repo is for.
