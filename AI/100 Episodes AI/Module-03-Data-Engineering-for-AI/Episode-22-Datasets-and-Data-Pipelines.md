# Day 22: Datasets and Data Pipelines

*100 Days of AI Engineering, Module 3: Data Engineering for AI*

**Time:** about 4 hours
**Level:** intermediate. You have built cleaning and feature code in Days 19 to 21. Today the question changes from "is this transformation right?" to "will it stay right when the data changes, when it runs again, and when it runs in production?"

**By the end of this day, you will have:**

- A working understanding of dataset splits, data loaders, batch processing, shuffling, dataset versioning, reproducible preprocessing and training and serving consistency
- A small `datapipe` package that makes stable splits, shuffled mini-batches, a content-hashed dataset manifest and a shared feature function for training and serving
- A test suite that catches a split that moves between runs, a batch that skips rows and a feature that differs between training and serving

**What you need:**

- A computer with Python 3.10 or newer
- pandas 2.2 or newer, NumPy 2.0 or newer
- pytest 8.0 or newer (`pip install "pytest>=8.0"`)
- A code editor such as VS Code (free) or Jupyter Lab
- Git and a free GitHub account for the project section
- Optional for the reading section: PyTorch 2.x (`pip install torch`), free under a BSD-style license, to see a production data loader. The project does not require it.

**Prerequisites:**

- Day 18: pandas filtering and reading and writing CSV
- Day 19: train, validation and test splits and the leakage rules
- Day 21: the feature pipeline and the fact that fitted transformers must be reused unchanged

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Dataset splits](#dataset-splits)
- [Data loaders](#data-loaders)
- [Batch processing](#batch-processing)
- [Shuffling](#shuffling)
- [Dataset versioning concepts](#dataset-versioning-concepts)
- [Reproducible preprocessing](#reproducible-preprocessing)
- [Training/serving consistency](#trainingserving-consistency)
- [Hands-on exercises](#hands-on-exercises)
- [Project: datapipe, a reproducible data pipeline](#project-datapipe-a-reproducible-data-pipeline)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Yesterday's pipeline ran once on one file. A real project runs the same steps many times. The data arrives every day, a colleague fixes a bug in a cleaning rule, a model is retrained next month, and a web service must compute the same features for one request at a time. Each of those runs can quietly produce different numbers. The split changes, the shuffle changes, the feature code drifts, and the model that was evaluated is not the model that was deployed.

This lesson is about making a pipeline repeatable. A repeatable pipeline gives the same answer for the same input, records which input it used, and computes the same features whether the code is running in a notebook, in a nightly batch job or inside a live API. These properties come from a few boring habits: deterministic splits, explicit random seeds, content hashes for data files, and one shared definition of each feature.

On the job, these habits decide whether a team can answer the question "why did the model change last Tuesday?" If the answer takes a day of archaeology, the team is not doing data engineering for AI. If the answer is one command that shows the file hash, the split and the feature version, the team can move quickly and safely.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1960s to 1970s | Database systems introduce versioned records and transaction logs, which give each change a place in a history. | The same idea of a recorded history underlies dataset versioning. |
| 1990s | Machine learning workflows start to separate training and evaluation data as a standard practice, and textbooks describe holdout and cross-validation methods. | The split is established as a core part of evaluation. |
| 2000s | Version control systems such as Subversion and Git become common in software engineering. | Teams learn to track code history, but data is hard to store in Git because files are large. |
| 2010s | Tools such as DVC, first released in 2017 as an open-source project under the Apache 2.0 license, extend version control to data and model files. | Data versioning becomes a practical part of ML workflows. |
| 2010s to today | Machine learning platforms and feature stores aim to serve the same feature definitions to training and to online prediction. | Training and serving consistency becomes an engineering requirement. |

**Takeaway:** repeatability is an old database and software idea that machine learning teams have had to relearn, because data is larger, less stable and less visible than code.

---

## Glossary

- **Dataset split**: a division of rows into subsets, usually training, validation and test, with a stated rule.
- **Holdout set**: a subset kept aside and not used for fitting, so that it can estimate performance on unseen data.
- **Data loader**: a component that reads a dataset and yields batches of examples in the form a model expects.
- **Batch**: a group of examples processed together in one step of training or inference.
- **Epoch**: one full pass through the training data.
- **Shuffling**: reordering the examples so that consecutive batches are not correlated with the original order.
- **Seed**: a number that fixes the sequence of random choices, so that the same seed gives the same shuffle or split.
- **Hash**: a fixed-length fingerprint of a value. The SHA-256 hash of a file changes if any byte of the file changes.
- **Manifest**: a small file that records the name, hash, size and schema of a dataset.
- **Dataset version**: an identifier for one specific state of a dataset, so that a result can be tied to the exact data that produced it.
- **Training/serving skew**: a difference between the features a model saw in training and the features it receives in production.
- **Feature store**: a shared system that defines features once and serves them to both training and online prediction.

---

### Dataset splits

A split divides the rows of a dataset into parts that play different roles. The training set is used to fit the model. The validation set is used to choose settings, such as the regularization strength. The test set is used once, at the end, to estimate how the final model performs on new data. Think of a student's practice exams, mock exam and final exam. Each serves a different purpose, and the final exam must not be practiced on.

From first principles, a split must satisfy two properties. It must be disjoint, so no row appears in two parts. It must be representative, so each part looks like the others and like the population you care about. A random split satisfies both for independent rows. Time series and grouped data need more care. If customers appear many times, a random split can put the same customer in train and test, which lets the model memorize the customer. A split by customer, or a split by time for data that changes over time, avoids that.

```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(0)
df = pd.DataFrame({"row": range(10)})
order = rng.permutation(len(df))

n_train = int(0.7 * len(df))
n_val = int(0.15 * len(df))
train = df.iloc[order[:n_train]]
val = df.iloc[order[n_train:n_train + n_val]]
test = df.iloc[order[n_train + n_val:]]

print(len(train), len(val), len(test))     # 7 1 2
```

The random split works, but it changes when the seed or the row order changes. If next month's file has a new row at the top, the permutation shifts and many rows move from train to test. A stable split assigns each row to a part based on its identity, not its position. The common mistake is to split by row position after sorting, which means the split depends on sort order. A practical habit is to hash a stable key, such as a customer ID, and assign the part from the hash. You will build that in the project.

---

### Data loaders

A data loader is the piece of code that turns a stored dataset into batches. It opens the file or the database, reads the rows, applies any per-example transformations, groups the rows into batches, and hands each batch to the model. Think of a kitchen runner who collects the ingredients for each order from the pantry and brings them to the chef in the right portions.

From first principles, a loader has four responsibilities: read data lazily so that the whole dataset does not have to fit in memory, transform each example consistently, group examples into batches of a chosen size, and optionally reorder them. Frameworks provide this as a class. PyTorch's `torch.utils.data.DataLoader` takes a dataset object and does all four, including parallel loading with worker processes. In the project you will write a small loader with NumPy so that the mechanics are visible.

```python
import numpy as np

X = np.arange(20).reshape(10, 2).astype(float)
y = np.arange(10) % 2

def batches(X, y, batch_size=4):
    for start in range(0, len(y), batch_size):
        yield X[start:start + batch_size], y[start:start + batch_size]

for xb, yb in batches(X, y):
    print(xb.shape, yb.shape)     # (4, 2) (4,), then (4, 2) (4,), then (2, 2) (2,)
```

The mistake to watch for is a loader that quietly keeps the last incomplete batch when the model expects a fixed size, or that drops it without telling you. Both can change the training run. The difference is small per step and large over many epochs. Data loader bugs often appear as a model that trains on fewer examples than the dataset contains, which shows up as a lower-than-expected row count in the training logs.

---

### Batch processing

Batch processing means handling many examples together, either for training or for scoring a large file. Batches make efficient use of hardware, because a matrix operation on 256 rows at once is much faster than 256 operations on one row each. The batch size is a trade-off. Larger batches use more memory and give smoother gradient estimates. Smaller batches use less memory and add noise that can help generalization.

From first principles, a batch loop has three parts: pick the rows for this batch, process them, and combine the results. For scoring, the combining step is usually writing predictions to a file or a database. For training, it is updating the model parameters. The loop must visit every row exactly once per pass, and the last batch may be shorter than the others.

```python
import numpy as np

def score_in_batches(X: np.ndarray, weights: np.ndarray, batch_size: int = 1000) -> np.ndarray:
    out = np.empty(len(X))
    for start in range(0, len(X), batch_size):
        end = min(start + batch_size, len(X))
        out[start:end] = X[start:end] @ weights
    return out

X = np.random.default_rng(0).normal(size=(2500, 3))
w = np.array([1.0, -2.0, 0.5])
print(score_in_batches(X, w).shape)                  # (2500,)
print(np.allclose(score_in_batches(X, w), X @ w))    # True
```

The check at the end is the most important line. A batched result must match the unbatched result. If it does not, the loop is skipping rows or writing to the wrong positions. The mistake is to pre-allocate an output array with the wrong length, or to forget `min` on the last slice. Nightly scoring jobs process millions of rows in batches, and a missing batch is one of the most common silent failures.

---

### Shuffling

Shuffling reorders the training examples so that each batch looks like a random sample of the whole dataset. Imagine a deck of cards sorted by suit. If you deal the cards in order, each hand contains only one suit, and a player learns a biased picture of the game. Shuffling mixes the deck.

From first principles, shuffling matters for training with stochastic gradient methods because the update from one batch should be a fair estimate of the update from the whole dataset. If the data is sorted by class, each batch is pure, and the model swings toward whichever class it has seen most recently. Shuffle once before training, and shuffle again at each epoch, using a seed for each epoch so that the order is reproducible.

```python
import numpy as np

labels = np.array([0] * 5 + [1] * 5)           # sorted by class: bad for batches
rng = np.random.default_rng([42, 0])           # seed and epoch index together
order = rng.permutation(len(labels))
print(labels[order][:4])                        # a mixed sample of the classes
```

Seeding with a list, `[42, 0]`, gives a different but reproducible order for each epoch. The common mistake is to shuffle the data before the split, which mixes the validation rows into the training order and can also leak information if the split is done later on the shuffled data. Split first, then shuffle only the training part. A second mistake is to shuffle time series data, which destroys the sequence the model needs to learn. Shuffling is one of the first things a training job does, and a missing shuffle is a common reason a model trains well for one month and badly for the next.

---

### Dataset versioning concepts

A dataset version is a fixed identifier for one exact state of the data. A model result means little unless you can say which data produced it. Think of a laboratory notebook where each experiment is labeled with the batch of reagents it used. Without the label, a surprising result cannot be checked.

From first principles, a version can be identified in three ways. The file name and a date are the weakest, because a file can be overwritten without changing its name. A content hash is strong, because any change to the bytes changes the hash, so two runs with the same hash saw the same data. A logical version, such as a schema version, records the structure of the data rather than the content. A good practice combines these: store the content hash, the row count, the columns and the date in a manifest next to the file.

```python
import hashlib
from pathlib import Path

def file_sha256(path: Path) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(1 << 16), b""):
            h.update(chunk)
    return h.hexdigest()
```

The tools for this are well established. DVC, the open-source Data Version Control project under the Apache 2.0 license, stores data files in a cache and keeps small pointer files in Git, so that a commit identifies the data it used. Git LFS, the Git Large File Storage extension, stores large files outside the normal Git history, with a pointer in the repository. The Hugging Face `datasets` library, open source under the Apache 2.0 license, records dataset versions and caches them locally. Each tool solves the same problem in a different way, and the concept is the same: a result must point to a hash.

The honest limitation is that a content hash depends on the exact bytes. Changing the line endings from Windows to Unix, or changing the order of rows, changes the hash even when the data is the same. Write files with a fixed format and fixed row order, and say so in the manifest.

---

### Reproducible preprocessing

Reproducible preprocessing means that the same input and the same code always produce the same output. It sounds trivial and it is often broken. Sources of irreproducibility include random seeds that are not set, dictionaries and sets whose order changes between runs, dates taken from the clock, floating-point sums computed in a different order, and fitted statistics that were not saved.

From first principles, a preprocessing step is reproducible when it is a pure function of its inputs, or when any state it learned is stored and loaded explicitly. A pure function such as "compute the log of spend and divide tickets by tenure" needs no stored state. A learned step such as "standardize with the training mean" needs the mean and standard deviation saved with the model. Saving those numbers in a JSON file next to the model, and loading them at prediction time, is the simplest version of this discipline.

```python
import json
from pathlib import Path
import numpy as np

def save_scaler(path: Path, mean: np.ndarray, std: np.ndarray) -> None:
    path.write_text(json.dumps({"mean": mean.tolist(), "std": std.tolist()}), encoding="utf-8")

def load_scaler(path: Path) -> tuple[np.ndarray, np.ndarray]:
    data = json.loads(path.read_text(encoding="utf-8"))
    return np.array(data["mean"]), np.array(data["std"])
```

The mistake is to recompute the scaler from whatever data is available at prediction time. A model served on a single request would then use the statistics of one customer, which is meaningless. A second mistake is to save the model but not the preprocessing, so the model is applied to features that have a different scale. Save every fitted object with the model, and load them together. The usual habit is to package the model and its preprocessing as one artifact with one version number.

---

### Training/serving consistency

Training/serving skew happens when the features computed for prediction differ from the features computed for training. The model is then given inputs it has never seen in the form it learned, and its predictions degrade without an error message. Think of a recipe written in grams for training and followed in cups in the kitchen. The dish comes out different, and nothing in the code says so.

From first principles, the cause is almost always two implementations of the same feature. The training code is written in a notebook or a batch job, and the serving code is rewritten in an API with a different library, a different default or a different order of operations. The fix is to write the feature logic once, in one function, and call it from both places. Then test that a row computed through the batch path and the same row computed through the serving path produce identical values.

```python
import numpy as np
import pandas as pd

def build_features(df: pd.DataFrame) -> pd.DataFrame:
    out = pd.DataFrame(index=df.index)
    out["log_spend"] = np.log1p(df["monthly_spend"].astype(float))
    out["tickets_per_month"] = (
        df["support_tickets"].astype(float) / df["tenure_months"].clip(lower=1)
    )
    return out

batch = pd.DataFrame({"monthly_spend": [20.0, 50.0], "support_tickets": [2, 0],
                      "tenure_months": [4, 10]})
single = pd.DataFrame([{"monthly_spend": 20.0, "support_tickets": 2, "tenure_months": 4}])
print(np.allclose(build_features(batch).iloc[[0]].to_numpy(),
                  build_features(single).to_numpy()))   # True
```

The common mistake is for the serving path to receive integers where training saw floats, or a missing field where training filled a default. Both change the result. A second source of skew is time: a feature that uses "the last 30 days" computed at training time must be computed at the same point in time at serving. Feature stores solve this by recording the timestamp of each feature value. The honest limitation is that a shared function does not protect you from a change in the source data. It protects you from a change in the code only. Monitor the distribution of features in production, as you learned in Day 20, to catch the rest.

---

## Hands-on exercises

Work through these in a notebook first, then move them into the project.

1. **Position-based split.** Create a table of 1000 rows with a customer ID column. Split it 70/15/15 by row position after a random shuffle. Add one new row at the top, and count how many rows change part. Explain why this is a problem for a monthly retraining job.
2. **Hash split.** Assign each customer to a part using `hash_bucket` on the customer ID. Add the same new row and recount. Explain why the number of rows that change part is near zero.
3. **Batch equality.** Write a batched scoring function and compare its output with an unbatched computation for batch sizes 1, 7, 100 and 10000. Confirm that all four match, and explain why size 1 is slow.
4. **Epoch shuffle.** Create a sorted label array. Produce the first batch of five epochs with a seed per epoch. Count how many distinct classes appear in each first batch. Explain what the sorted order would have produced.
5. **Manifest change.** Write a CSV, compute its hash, change one value, and compute the hash again. Then change only the line endings and compute it a third time. Explain both results.
6. **Skew experiment.** Compute features for the same ten customers through a batch path and a single-row path, and print the difference. Introduce a deliberate bug in one path, such as a different default for missing tenure, and show that the test catches it.
7. **Saved preprocessing.** Fit a scaler on a training array, save its mean and standard deviation to a JSON file, load them in a new process, and transform a test array. Confirm that the results match the in-memory scaler.

---

## Project: datapipe, a reproducible data pipeline

### Goal

Build a small package that does five things reproducibly: writes a dataset in a fixed format, creates a manifest with its content hash, splits the rows by a stable key, yields shuffled mini-batches with a seed per epoch, and computes features with one function that both training and serving call. The tests check each property, including the ones that break most often.

Suggested repository name: `100-days-ai-day-22-datapipe`

### Repo layout

```text
100-days-ai-day-22-datapipe/
├── README.md
├── datapipe/
│   ├── __init__.py
│   ├── generate.py
│   ├── versioning.py
│   ├── split.py
│   ├── batches.py
│   └── features.py
├── tests/
│   └── test_datapipe.py
├── data/
│   └── .gitkeep
├── run_pipeline.py
├── requirements.txt
└── .gitignore
```

### Starter code

`requirements.txt`:

```text
pandas>=2.2
numpy>=2.0
pytest>=8.0
```

`datapipe/__init__.py`: leave it empty.

`datapipe/generate.py`: copy the `make_subscriptions` function from your Day 21 repository, or use this short version, which keeps the columns the features need.

```python
import numpy as np
import pandas as pd


def make_subscriptions(n: int = 1000, seed: int = 3) -> pd.DataFrame:
    rng = np.random.default_rng(seed)
    tenure = rng.integers(1, 72, size=n)
    plan = rng.choice(["basic", "plus", "pro"], p=[0.5, 0.35, 0.15], size=n)
    spend = np.round(rng.uniform(5.0, 60.0, size=n), 2)
    tickets = rng.poisson(lam=1.5, size=n)
    churned = (rng.random(n) < 0.1 + 0.02 * tickets).astype(int)
    return pd.DataFrame({
        "customer_id": [f"s{i:05d}" for i in range(n)],
        "plan": plan,
        "tenure_months": tenure,
        "monthly_spend": spend,
        "support_tickets": tickets,
        "churned": churned,
    })


def write_csv(df: pd.DataFrame, path) -> None:
    # Fixed line endings and no index keep the file bytes, and so the hash, stable.
    df.to_csv(path, index=False, lineterminator="\n")
```

`datapipe/versioning.py`:

```python
import hashlib
import json
from pathlib import Path
import pandas as pd


def file_sha256(path: Path) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(1 << 16), b""):
            h.update(chunk)
    return h.hexdigest()


def write_manifest(csv_path: Path) -> dict:
    df = pd.read_csv(csv_path)
    manifest = {
        "file": csv_path.name,
        "sha256": file_sha256(csv_path),
        "rows": int(len(df)),
        "columns": list(df.columns),
    }
    csv_path.with_suffix(".manifest.json").write_text(
        json.dumps(manifest, indent=2), encoding="utf-8"
    )
    return manifest
```

`datapipe/split.py`:

```python
import hashlib
import pandas as pd


def hash_bucket(key: str, buckets: int = 100) -> int:
    digest = hashlib.sha256(key.encode("utf-8")).hexdigest()
    return int(digest, 16) % buckets


def split_by_id(df: pd.DataFrame, key: str = "customer_id",
                train_pct: int = 70, val_pct: int = 15):
    buckets = df[key].astype(str).map(hash_bucket)
    train = df[buckets < train_pct]
    val = df[(buckets >= train_pct) & (buckets < train_pct + val_pct)]
    test = df[buckets >= train_pct + val_pct]
    return train, val, test
```

`datapipe/batches.py`:

```python
from typing import Iterator
import numpy as np


def minibatches(X: np.ndarray, y: np.ndarray, batch_size: int,
                seed: int, epoch: int, shuffle: bool = True,
                drop_last: bool = False) -> Iterator[tuple[np.ndarray, np.ndarray]]:
    n = len(y)
    rng = np.random.default_rng([seed, epoch])
    order = rng.permutation(n) if shuffle else np.arange(n)
    for start in range(0, n, batch_size):
        idx = order[start:start + batch_size]
        if drop_last and len(idx) < batch_size:
            break
        yield X[idx], y[idx]
```

`datapipe/features.py`:

```python
import numpy as np
import pandas as pd

PLAN_LEVELS = ["basic", "plus", "pro"]


def build_features(df: pd.DataFrame) -> pd.DataFrame:
    """The one feature definition. Training and serving both call this function."""
    out = pd.DataFrame(index=df.index)
    out["log_spend"] = np.log1p(df["monthly_spend"].astype(float))
    out["tenure_months"] = df["tenure_months"].astype(float)
    out["support_tickets"] = df["support_tickets"].astype(float)
    out["tickets_per_month"] = out["support_tickets"] / out["tenure_months"].clip(lower=1)
    for level in PLAN_LEVELS:
        out[f"plan_{level}"] = (df["plan"] == level).astype(float)
    return out


def predict_features(record: dict) -> pd.DataFrame:
    """Serving path: one request, same function as training."""
    return build_features(pd.DataFrame([record]))
```

`run_pipeline.py`:

```python
from pathlib import Path
from datapipe.generate import make_subscriptions, write_csv
from datapipe.versioning import write_manifest
from datapipe.split import split_by_id
from datapipe.batches import minibatches
from datapipe.features import build_features

DATA = Path("data")

if __name__ == "__main__":
    DATA.mkdir(exist_ok=True)
    raw = make_subscriptions()
    write_csv(raw, DATA / "subscriptions.csv")
    manifest = write_manifest(DATA / "subscriptions.csv")
    print("sha256:", manifest["sha256"])
    print("rows:", manifest["rows"])

    train, val, test = split_by_id(raw)
    print("split sizes:", len(train), len(val), len(test))

    X = build_features(train).to_numpy()
    y = train["churned"].to_numpy()
    n_batches = sum(1 for _ in minibatches(X, y, batch_size=64, seed=0, epoch=0))
    print("batches in epoch 0:", n_batches)
```

`tests/test_datapipe.py`:

```python
import numpy as np
import pandas as pd
from pathlib import Path
from datapipe.generate import make_subscriptions, write_csv
from datapipe.versioning import write_manifest
from datapipe.split import split_by_id
from datapipe.batches import minibatches
from datapipe.features import build_features, predict_features


def test_split_is_disjoint_and_complete():
    df = make_subscriptions()
    train, val, test = split_by_id(df)
    ids = [set(train["customer_id"]), set(val["customer_id"]), set(test["customer_id"])]
    assert ids[0].isdisjoint(ids[1]) and ids[0].isdisjoint(ids[2]) and ids[1].isdisjoint(ids[2])
    assert len(train) + len(val) + len(test) == len(df)


def test_split_is_stable_when_new_rows_arrive():
    df = make_subscriptions(n=500)
    before = split_by_id(df)[2]["customer_id"]
    extra = pd.DataFrame([{"customer_id": "zz-new", "plan": "basic", "tenure_months": 1,
                           "monthly_spend": 9.0, "support_tickets": 0, "churned": 0}])
    after = split_by_id(pd.concat([extra, df], ignore_index=True))[2]["customer_id"]
    assert set(before).issubset(set(after))


def test_batches_cover_every_row_once():
    X = np.arange(100).reshape(50, 2).astype(float)
    y = np.arange(50)
    seen = np.concatenate([yb for _, yb in minibatches(X, y, batch_size=8, seed=1, epoch=0)])
    assert sorted(seen.tolist()) == list(range(50))


def test_epoch_order_is_reproducible_and_differs_across_epochs():
    X = np.zeros((40, 1))
    y = np.arange(40)
    a = [yb.tolist() for _, yb in minibatches(X, y, 10, seed=5, epoch=0)]
    b = [yb.tolist() for _, yb in minibatches(X, y, 10, seed=5, epoch=0)]
    c = [yb.tolist() for _, yb in minibatches(X, y, 10, seed=5, epoch=1)]
    assert a == b
    assert a != c


def test_training_and_serving_features_match():
    df = make_subscriptions(n=50)
    batch = build_features(df)
    row = df.iloc[[7]].to_dict(orient="records")[0]
    single = predict_features(row)
    assert np.allclose(batch.iloc[[7]].to_numpy(), single.to_numpy())


def test_manifest_changes_when_data_changes(tmp_path: Path):
    df = make_subscriptions(n=20)
    path = tmp_path / "subs.csv"
    write_csv(df, path)
    first = write_manifest(path)["sha256"]
    df.loc[0, "monthly_spend"] = df.loc[0, "monthly_spend"] + 1.0
    write_csv(df, path)
    second = write_manifest(path)["sha256"]
    assert first != second
```

### Step-by-step tasks

1. Create the repository, the environment, and the install. Create the files as shown.
2. Run `pytest -q`. All six tests should pass.
3. Run `python run_pipeline.py`. Copy the printed hash, row count, split sizes and batch count into your README.
4. Run `python run_pipeline.py` a second time and confirm that the hash is identical. Then open the CSV in an editor and check that the line endings are `\n`.
5. Break the split on purpose. Replace `split_by_id` with a version that uses `np.random.permutation` on row positions, run the stability test, and record that it fails. Restore the hash version.
6. Break the feature path on purpose. Change `predict_features` to fill a missing tenure with 1 instead of using the same clip as training. Run the consistency test on a record with tenure zero and record the failure. Restore the code.
7. Write the README with the sections in the checklist.

### README checklist

- [ ] Install and run instructions with the versions you tested
- [ ] The manifest output: hash, row count and columns
- [ ] The split sizes and the reason the split is by ID, not by row position
- [ ] The batch count and the batch size
- [ ] The result of the broken-split and broken-feature experiments
- [ ] One paragraph on what the hash does and does not protect against
- [ ] One limitation, for example that the synthetic generator is run again rather than read from a stored file

### Done when

- `pytest -q` passes all six tests
- Running `run_pipeline.py` twice produces the same hash and the same split sizes
- The split test fails when the split is made by row position, and passes with the hash split
- The consistency test fails when the serving path uses a different default, and passes with the shared function
- The manifest records the SHA-256 hash, the row count and the columns
- Every number in the README matches the printed output

### Evidence to commit to GitHub

Commit the package, the tests, the runner, the README and the generated manifest file, but not the large CSV. Commit the manifest so the reader can check the hash of a fresh run against the committed one. The history should show the two deliberate breaks as separate commits that were reverted, so that the reader can see the tests catching the problems. The reader should be able to show a green `pytest -q` run, the matching hash across two runs, and the README result of the broken experiments.

---

## Check your understanding

1. A monthly retraining job splits by row position after sorting by signup date. Explain what changes between two months and why that can make the model's test score unreliable.
2. The data is daily sales in time order. You shuffle the full dataset once, then take the last 20 percent as the test set. Describe the problem and how to reorder the steps.
3. A nightly job processes 10 million rows in batches of 1000, and the output file has 9,999,000 rows. Name two bugs that could produce that count, and describe the test you would write to catch both.
4. Two runs of a pipeline produce different content hashes, although the data looks identical when you open both files. List three things that could cause this and how you would check each.
5. A deployed model has a much worse error rate than in validation. Your first check finds that a feature is computed in the API by a different function from the training code. Explain the mechanism of the failure and what would have prevented it.
6. Your team wants to store a 4 gigabyte dataset in Git. Explain what happens to repository size and history if you do, and name two alternatives with the concept each one provides.

Work each answer out in writing before you read the answers below.

**Answers**

1. Adding a row at the top shifts every row position, so many customers move between train and test on the next run. The test set changes from month to month, so the scores cannot be compared, and a customer can appear in both parts. Split by a hash of a stable ID, so each customer always lands in the same part.
2. Shuffling destroys the time order. The "last 20 percent" then mixes early and late days, so the training rows include days that come after some test rows. The model learns from the future. Split by time first, keeping the order, then shuffle only the training part within each epoch.
3. Ten million rows in batches of 1,000 is 10,000 batches, and exactly 1,000 rows are missing, which means one full batch was skipped. Two bugs fit: an off-by-one in the loop range that stops one batch early, or an exception caught around a batch that writes nothing and lets the loop continue. The test is to compare the set of input IDs with the set of output IDs, and to assert that every batch writes its own range.
4. The difference is usually in bytes that do not change the data: CRLF versus LF line endings, a trailing newline, a byte-order mark, an index column that was written by accident, or floats formatted with different precision. Compare the two files with a byte-level diff, then compare the parsed tables with `equals`. Fix the write options so the bytes are stable.
5. The API computes the feature differently from the training code, for example with a different default for missing tenure, a different clip, or a different order of operations. The model then receives values it never learned from, and its error rate rises without any exception. Prevent this with one shared feature function and a test that compares the two paths on the same record.
6. Git stores every version of a 4 gigabyte file, so each change adds a large object, clones become slow, and GitHub rejects individual files above 100 MiB. Use DVC or Git LFS, which keep a small pointer file in Git and the large file in separate storage, identified by its hash. Alternatively, store versioned files in an object store and record their paths and hashes in a manifest.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, article "Training, validation, and test sets", for the roles of each split.
- Wikipedia, article "Stochastic gradient descent", for why batches and shuffling matter in training.
- Wikipedia, article "SHA-2", for the family of hash functions used in the manifest.
- Wikipedia, article "Version control", for the history and the concept of a recorded state.
- Wikipedia, article "Feature store", for the purpose of shared feature definitions.
- PyTorch documentation, "torch.utils.data" section, covering `Dataset` and `DataLoader`. Search "PyTorch torch.utils.data" on pytorch.org.
- scikit-learn documentation, User Guide section "Cross-validation: evaluating estimator performance", which covers group-aware and time-aware splits. Search the title on scikit-learn.org.
- DVC documentation, "Get Started" and "Data Versioning", on dvc.org, for the tool that stores data files with Git pointers.

**Tutorials**

- GeeksforGeeks, "Train Test Split in Python" and the articles on batch processing and data loaders.
- W3Schools, "Python File Handling" in the Python section of w3schools.com, for reading binary files in chunks.

**Watch**

- Full Stack Deep Learning, lectures on data management and on deployment, on YouTube. They cover training and serving skew and dataset versioning in practical terms.
- Search YouTube for "training serving skew" to find a talk from a company that describes its own incident.
- Search YouTube for "DVC data version control tutorial" for a walkthrough of the tool in a real project.

**Practice**

- The PyTorch tutorial "Datasets and DataLoaders" on pytorch.org, which runs in a browser notebook through Google Colab if you prefer not to install anything.

---

## Tomorrow

Tomorrow is the data project. Everything from Days 17 to 22 comes together on one messy dataset. You will clean it, explore it, engineer the features, split it reproducibly and write down every assumption you made. Day 23 is where you practice the discipline of this module on a dataset you did not design, so the stable split and the shared feature function you built today will be the first things you reach for.
