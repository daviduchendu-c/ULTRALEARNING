# Day 49: Datasets and DataLoaders

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3.5 hours
**Level:** intermediate. You can build and train a model (Day 48). Today the data stops being a variable in your notebook and becomes a component with a defined interface, so that the training code does not care where the examples come from.

**By the end of this day, you will have:**

- A custom `Dataset` class that reads examples from a CSV file and returns one labeled example at a time
- A transform that standardizes features using statistics from the training split only, and an augmentation that flips image-like tensors with a seeded random generator
- A `DataLoader` configured for batch size, shuffling, dropping the last incomplete batch and reproducible order
- Tests that check lengths, batch shapes, shuffle reproducibility and the rule that test data is never augmented

**What you need:**

- A computer with Python 3.11 or 3.12
- PyTorch 2.5.1 from Day 46, with the same environment
- NumPy 2.1.3 and pytest 8.3.3, as before
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 46: tensors, shapes and dtypes
- Day 48: a model that takes a batch of inputs and produces logits
- Day 45: why augmentation must preserve the label, and why the validation data is not augmented
- Day 13: sampling and randomness, at the level of a shuffled list

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Dataset abstraction](#dataset-abstraction)
- [Batches](#batches)
- [Shuffling](#shuffling)
- [Transforms](#transforms)
- [DataLoader](#dataloader)
- [Custom datasets](#custom-datasets)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a CSV dataset with transforms and a seeded loader](#project-a-csv-dataset-with-transforms-and-a-seeded-loader)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Up to now, your training code has held the whole dataset in a single tensor. That works for a few hundred points on a laptop. It fails when the data is a million images stored on disk, when it must be read from a database, or when each example needs its own preprocessing. The data pipeline is the part of a deep learning system that most often limits speed and correctness, and it is where many silent bugs live.

PyTorch separates the pipeline into two pieces. A `Dataset` answers one question: how many examples are there, and what is example number i? A `DataLoader` answers another: how do I group examples into batches, in what order, and how do I load them in parallel? Your model and training loop only see the batches. This separation means you can change where the data comes from without rewriting the training code.

On the job, the data loader is where teams spend a surprising share of their effort. Labels that are misaligned with inputs, normalization statistics computed on the whole dataset, and random seeds that are not set all produce models that look fine and fail later. The habits in this lesson, computing statistics only on training data and testing the loader directly, prevent most of those failures.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1998 | Yann LeCun and colleagues publish the MNIST database of handwritten digits, with a standard training and test split. | The first dataset that many students load, and the reference for the data pipelines in the early PyTorch tutorials. |
| 2009 | Jia Deng and colleagues publish ImageNet, a large labeled image collection that grows into the benchmark for image classification. | The scale of data that made loading and batching a serious engineering problem. |
| 2012 | Deep networks trained on GPUs show that large datasets and fast batch processing are required for progress. | Input pipelines become a focus of framework design. |
| 2017 | PyTorch releases `torch.utils.data` with `Dataset` and `DataLoader` as core abstractions. | The interface used in this lesson. |
| 2020s | Teams build streaming data pipelines that read from object storage and prepare batches in parallel workers. | The production form of the same idea, with the same separation of concerns. |

**Takeaway:** the data pipeline has grown from an array in memory to a streaming system, but the contract between the dataset and the batches has stayed the same.

---

## Glossary

- **Dataset**: an object that represents a collection of examples and provides the number of examples and access to each one by index.
- **Sample**: one example from a dataset, usually a pair of an input and a target.
- **Batch**: a group of samples stacked together into one tensor per field, used for one training step.
- **Collate function**: the function that combines a list of samples into one batch.
- **Transform**: a function applied to a sample, such as normalization or augmentation, when the sample is read.
- **Shuffling**: reordering the samples randomly, usually once per epoch, so batches differ between epochs.
- **Sampler**: the object that decides the order of indices passed to the dataset.
- **Worker**: a separate process that loads and transforms samples in parallel with the training loop.
- **drop_last**: an option that discards the final incomplete batch.
- **Standardization**: subtracting the mean and dividing by the standard deviation of each feature, using statistics from the training data.
- **Leakage**: the use of information from the test or validation data during training, which makes results look better than they are.
- **Seed**: a number that initializes a random generator, so that shuffles and augmentations can be reproduced.

---

## Dataset abstraction

Think of a library catalog. The catalog tells you how many books there are and gives you a shelf number for each. You do not need to know how the books are stored, only how to ask for one by its number. A `Dataset` is that catalog for training examples.

From first principles, a dataset class implements two methods. `__len__` returns the number of examples, and `__getitem__` takes an index and returns one sample. The data can live in memory, in a file, or behind a network call. The rest of PyTorch does not care, because it only uses these two methods. This design also allows lazy loading: a sample is read only when it is requested, so a dataset larger than memory is possible.

```python
import torch
from torch.utils.data import Dataset

class TinyDataset(Dataset):
    def __init__(self):
        self.X = torch.arange(10, dtype=torch.float32).reshape(5, 2)
        self.y = torch.tensor([0, 1, 0, 1, 1])

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

ds = TinyDataset()
print(len(ds), ds[2])       # 5 (tensor([4., 5.]), tensor(0))
```

The limitation of the abstraction is that `__getitem__` is called once per sample, so a slow implementation makes the whole pipeline slow. A dataset that reads a file on every call, for example, opens the file many thousands of times per epoch. Load what you can once, in `__init__`, and keep the per-item work small.

Do not return a Python list or a NumPy array from `__getitem__` and expect the default collate function to handle it. The default works with tensors, numbers and tuples of them. Convert to tensors inside the dataset, and you avoid a class of errors that otherwise show up only when the batch is formed.

In industry, datasets for large vision and language models are often sharded files on object storage, and a dataset class maps an index to a shard and an offset. The interface is the same as the one you write today.

---

## Batches

A batch is a stack of samples. The training step needs a tensor with the first dimension as the batch size, so the model can process many examples in one matrix operation. If each sample has shape (2,), a batch of 32 has shape (32, 2), and a batch of labels has shape (32, 1) or (32,), depending on the loss.

From first principles, the `DataLoader` calls `__getitem__` for each index in the batch, collects the returned tuples, and calls the collate function. The default collate function stacks the tensors in each field along a new first dimension. A batch of 32 samples, each returning a tensor of shape (2,) and a label of shape (), becomes a tensor of shape (32, 2) and a tensor of shape (32,). Knowing which shape the loss expects is the main job of the dataset author.

```python
import torch
from torch.utils.data import DataLoader

ds = TinyDataset()               # from the previous section
loader = DataLoader(ds, batch_size=2)
for xb, yb in loader:
    print(xb.shape, yb.shape)    # two batches of [2, 2] and [2], then a last batch of [1, 2] and [1]
```

The limitation of batching is that every sample in a batch must have the same shape, unless you write a custom collate function. Variable-length text and variable-size images need padding or a custom collation, which is a common source of runtime errors.

Confusing the label shape with the output shape of the model is a common error. A model that outputs (batch, 1) and a label of shape (batch,) make `BCEWithLogitsLoss` raise an error or broadcast into an incorrect pairing. Check the shapes of one batch before you train, and make the labels match the output exactly.

In the real market, a batch-shape bug often shows up only on the last batch of an epoch, when the size is smaller, so tests that use a single batch miss it. The `drop_last` option and a test over the whole loader catch that case.

---

## Shuffling

Consider a deck of cards that you always deal in the same order. Every hand is the same, and the player learns the pattern rather than the game. Shuffling the deck between rounds stops that pattern from forming. Shuffling the training data does the same for the model.

From first principles, if the data is sorted by class, each batch contains examples from one class, and the gradient points toward that class for many steps, which makes training unstable. Shuffling ensures that each batch is a random sample of the data. The `DataLoader` with `shuffle=True` creates a random permutation of the indices at the start of each epoch, using a random generator. Passing a seeded `torch.Generator` makes the permutation reproducible, which you need to debug a run or to compare two configurations fairly.

```python
import torch
from torch.utils.data import DataLoader

g = torch.Generator().manual_seed(0)
loader_a = DataLoader(TinyDataset(), batch_size=2, shuffle=True, generator=g)
g2 = torch.Generator().manual_seed(0)
loader_b = DataLoader(TinyDataset(), batch_size=2, shuffle=True, generator=g2)

print([yb.tolist() for _, yb in loader_a] == [yb.tolist() for _, yb in loader_b])   # True
```

The limitation of shuffling is that it does not remove all structure. Time series and sequential data must not be shuffled across the boundary between training and validation, because the model would see the future. For those data, you split by time first and shuffle only within the training part.

Do not shuffle the validation or test loader. The order does not change the accuracy, but it makes the per-batch metrics harder to compare across runs, and it hides a bug where the labels and inputs were misaligned. Use `shuffle=False` for evaluation.

In industry, shuffling bugs are a classic source of inflated metrics in time-based problems. When a random split leaks future information into training, the offline accuracy looks excellent and the production accuracy collapses.

---

## Transforms

A transform is a small function that changes a sample as it is read. Normalization makes features comparable in scale. Augmentation creates a varied version of the sample that keeps the same label. Because transforms run inside `__getitem__`, each epoch can see a different version of the same example.

From first principles, the mean and standard deviation for standardization must be computed on the training data only, and then applied unchanged to the validation and test data. Computing them over the whole dataset leaks information about the test set into the preprocessing, which is a form of leakage that Day 45 warned about. An augmentation such as a random horizontal flip is applied only to training samples, and it uses a random generator so that the same seed gives the same flips.

```python
import torch

class Standardize:
    def __init__(self, mean, std):
        self.mean = mean
        self.std = std

    def __call__(self, x):
        return (x - self.mean) / self.std

class RandomFlip:
    def __init__(self, p=0.5, generator=None):
        self.p = p
        self.generator = generator

    def __call__(self, x):
        if torch.rand(1, generator=self.generator).item() < self.p:
            return torch.flip(x, dims=[-1])
        return x
```

The limitation of transforms is that they add work to every sample read, so an expensive transform can slow training. Move slow preprocessing, such as decoding large images, into a precomputation step when possible. Augmentations also change the data distribution, and a model trained with heavy augmentation may do worse on clean test data if the test set differs.

Do not compute the standardization statistics on the full dataset before splitting. The fix is to split first, compute the statistics on the training split, and pass the same `Standardize` object to the validation and test datasets. The second mistake is applying augmentation to the test dataset, which makes the test numbers depend on random flips.

In the real market, a preprocessing pipeline is part of the model. When a model is deployed, the same standardization must run at inference, so the statistics are saved with the model and loaded with it.

---

## DataLoader

A `DataLoader` wraps a dataset and produces batches. It handles the batch size, the shuffle, the drop of the last incomplete batch and the optional parallel loading with worker processes. You create it once, and you iterate over it at each epoch.

From first principles, the loader asks the sampler for an order of indices, groups them into batches of the chosen size, calls the dataset for each index and collates the results. With `num_workers` greater than zero, it starts that many processes, and each prepares whole batches while the training loop runs on the previous ones. This overlap is what keeps the GPU busy. On Windows, worker processes start by importing the main module, so the training code must sit under an `if __name__ == "__main__":` guard, or the program will re-run itself in each worker and fail.

```python
from torch.utils.data import DataLoader

if __name__ == "__main__":
    loader = DataLoader(
        TinyDataset(),
        batch_size=2,
        shuffle=True,
        drop_last=False,
        num_workers=0,
    )
    for xb, yb in loader:
        pass
```

The limitation of workers is that they copy data and consume memory, and too many of them slow the machine. Start with `num_workers=0` to debug, then increase it when the loader is the bottleneck. A worker that runs a non-picklable transform, such as a lambda defined in the main module, fails on Windows, so use named classes for transforms.

Do not forget to set `shuffle=False` for validation and test, or set `drop_last=True` on the evaluation loader, which silently skips part of the test set and inflates or deflates the metric. In the real market, the evaluation loader should cover every example exactly once, and a test that counts the samples seen is a cheap guarantee.

---

## Custom datasets

Many real datasets arrive as files: CSV tables, folders of images or binary arrays. A custom dataset reads those files, checks them and presents them through the same two methods. The effort of writing it is small, and it is where you enforce the contract that the rest of the pipeline depends on.

From first principles, a custom dataset should load the file once, validate its shape and labels, convert to tensors with the right dtypes, and apply the transform in `__getitem__`. It should fail early and loudly when the data is malformed. A check such as "the label column contains only 0 and 1" catches many errors before training begins.

```python
import numpy as np
import torch
from torch.utils.data import Dataset

class CSVDataset(Dataset):
    def __init__(self, path, transform=None):
        table = np.loadtxt(path, delimiter=",", skiprows=1, dtype=np.float32)
        self.X = torch.from_numpy(table[:, :2].copy())
        self.y = torch.from_numpy(table[:, 2:3].copy())
        if not torch.isin(self.y, torch.tensor([0.0, 1.0])).all():
            raise ValueError("labels must be 0 or 1")
        self.transform = transform

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        x = self.X[idx]
        if self.transform is not None:
            x = self.transform(x)
        return x, self.y[idx]
```

The limitation of loading everything into memory in `__init__` is that it does not scale to datasets larger than the machine's memory. For those, read the file lazily in `__getitem__`, or use a format that supports random access, such as memory-mapped arrays or a database.

Do not trust that the file is clean. Missing values, labels encoded as strings and columns in a different order are common, and each one produces a different failure later. Validate the file at load time and raise an error with a clear message. In the real market, the data validation step at the start of a pipeline is often the most valuable line of code in the project.

---

## Hands-on exercises

1. **Dataset by hand.** Write a dataset of four samples where each sample is the pair (i, i squared) for i from 0 to 3. Print its length and the item at index 2.
2. **Last batch size.** For a dataset of 101 samples and batch size 16, list the sizes of all batches with `drop_last=False` and with `drop_last=True`. Check your answer by iterating over the loader.
3. **Shuffle reproducibility.** Create two loaders with the same seed and check that their batch orders match. Then change the seed and check that they differ.
4. **Leakage check.** Compute the mean and standard deviation of a random feature on the whole dataset and on a training split of 80 percent. Explain why the numbers differ and which one must be used for standardization.
5. **Flip test.** Apply the `RandomFlip` transform with p = 0.5 to a tensor of shape (1, 3) a hundred times with a seeded generator. Count how many flips happened and check that the count is close to 50.

---

## Project: a CSV dataset with transforms and a seeded loader

**Goal:** write the circle dataset to CSV files, load the training file with a custom dataset that validates its labels, standardize the features with training statistics, apply a seeded augmentation to the training data only, and test the loader's batches and order.

**Repo name:** `100-days-ai-day-49-datasets-and-dataloaders`

**Repo layout:**

```
100-days-ai-day-49-datasets-and-dataloaders/
├── README.md
├── requirements.txt
├── data/
│   ├── train.csv
│   └── test.csv
├── src/
│   └── data_lab/
│       ├── __init__.py
│       ├── make_files.py
│       ├── dataset.py
│       └── transforms.py
├── tests/
│   ├── test_dataset.py
│   └── test_loader.py
└── experiments/
    └── train_with_loader.py
```

**Starter code.** Create `requirements.txt` with PyTorch 2.5.1, NumPy 2.1.3 and pytest 8.3.3 pinned. Create `src/data_lab/make_files.py`, which generates 400 training points and 400 test points from the circle rule and writes them to `data/train.csv` and `data/test.csv` with a header row `x1,x2,label`:

```python
import numpy as np


def write_split(path, n, seed):
    rng = np.random.default_rng(seed)
    X = rng.uniform(-1.5, 1.5, size=(n, 2))
    y = (np.sum(X ** 2, axis=1) < 1.0).astype(np.float32)
    table = np.column_stack([X, y])
    np.savetxt(path, table, delimiter=",", header="x1,x2,label", comments="", fmt="%.6f")


if __name__ == "__main__":
    write_split("data/train.csv", 400, seed=0)
    write_split("data/test.csv", 400, seed=1)
```

Create `src/data_lab/dataset.py` with the `CSVDataset` class from the lesson, and `src/data_lab/transforms.py` with the `Standardize` and `RandomFlip` classes. Note that `RandomFlip` flips the last dimension, which is useful for image-like tensors. For the two-feature circle data it is a demonstration, and you should say so in the README.

**Step-by-step tasks:**

1. Create the repo, initialize Git, make a virtual environment and install the pinned packages. Run `python -m src.data_lab.make_files` from the repo root to create the CSV files.
2. Write `tests/test_dataset.py`. Check the length of the training dataset, the shape of one item, and that loading a CSV with a label of 2 raises a `ValueError`. Create the bad file in a temporary directory with pytest's `tmp_path` fixture.
3. Compute the mean and standard deviation of the training features in the experiment script, and apply `Standardize` to the training and test datasets with those statistics only.
4. Write `tests/test_loader.py`. Check that a loader with batch size 32 and `drop_last=True` produces only batches of size 32, that the number of batches is 12 for 400 samples, and that two loaders with the same generator seed produce the same first batch.
5. Check that the test loader has `shuffle=False`, that it contains exactly 400 samples across all batches, and that the augmentation is not applied to it.
6. Write `experiments/train_with_loader.py` that trains the Day 48 MLP for 50 epochs with the loader, prints the epoch loss and the test accuracy at the end, and saves the epoch losses to `results/epoch_losses.csv`.
7. Repeat the training with and without the random flip augmentation, using three seeds, and record the test accuracy for each in the README.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to create the CSV files and run the tests
- [ ] The statistics used for standardization, and the rule that they come from training data only
- [ ] The test accuracy table with and without augmentation, for three seeds
- [ ] A note that the flip is a demonstration on a two-feature dataset and would be meaningful on images
- [ ] A short description of what the loader test checks that a single-batch test would miss

**Done when:**

- The dataset and loader tests pass
- A malformed CSV raises a clear error in the test
- The test loader covers exactly 400 samples with no shuffling and no augmentation
- The statistics in the code come from the training split only, and the README says so
- The repo has at least five commits

**Evidence to commit to GitHub:** the source, the tests, the CSV files, the README with the accuracy table, the epoch losses file, and a `NOTES.md` that records one data problem you found while writing the loader and how the test or the validation caught it. The notes show that the pipeline was checked, not assumed.

**Why this project matters:** the dataset and the loader are the first code that touches your data, so their tests protect every experiment that follows. A pipeline with honest statistics and checked batches is the foundation of any result you report.

---

## Check your understanding

1. Why does a `DataLoader` need only `__len__` and `__getitem__` from a dataset, and what does that let you change without touching the training loop?
2. A model trains well on the training set and its test accuracy is much higher than expected. You find that the standardization statistics came from the whole dataset. Explain the mechanism that inflates the number, and say whether the effect is large or small in general.
3. Your sorted dataset is split into batches without shuffling, and the loss oscillates strongly across epochs. Explain why, and what change you would make.
4. Why must a validation loader keep `drop_last=False`?
5. On Windows, a training script that creates a `DataLoader` with `num_workers=2` fails with an error that repeats. What is the cause and the fix?
6. A custom dataset returns labels as Python integers in some rows and strings in others. Where should you catch this, and what would the error look like if you did not?

**Answers:**

1. The loader only needs to know the length and to index samples, so the training loop does not depend on where the data lives. You can switch from an in-memory tensor to a file on disk or a remote store by changing the dataset class, and the loop stays the same.
2. Statistics from the whole dataset include the test examples, so the standardization uses information about the data the model will be scored on. The effect depends on the sample size and the distribution, and it is usually small for large datasets with stable statistics, but it can be large when the test set is systematically different.
3. Each batch then contains examples from a single class or a narrow range, so the gradient points in one direction for many steps and changes direction between batches, which causes the oscillation. The fix is to shuffle the training data at the start of each epoch.
4. With `drop_last=True`, the final incomplete batch is discarded, so some test examples are never evaluated. The reported metric then covers only part of the test set, and the number is not comparable to other runs.
5. Each worker process imports the main module when it starts, and without a main guard the script creates new loaders and workers again, which repeats until the process fails. The fix is to put the training code under `if __name__ == "__main__":`.
6. The check belongs in the dataset's loading step, where the file is parsed, so that the error appears once with the row number. Without it, the failure appears later, inside the collate function or the loss, with a message about types that does not point to the file.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "MNIST database", for the dataset that most loader tutorials use
- Wikipedia, "ImageNet", for the large labeled image dataset and its role in deep learning
- Wikipedia, "Training, validation, and test data sets", for the split and why it matters
- Wikipedia, "Data augmentation", for the general idea and the label-preservation rule
- Wikipedia, "Data pre-processing", for normalization and standardization

**Tutorials**

- PyTorch Tutorials, "Datasets and DataLoaders", at pytorch.org/tutorials/beginner/basics/data_tutorial.html
- PyTorch documentation, "torch.utils.data", at pytorch.org/docs/stable, for the `Dataset`, `DataLoader` and `Sampler` APIs
- NumPy documentation, "numpy.loadtxt" and "numpy.savetxt", at numpy.org/doc/stable, for the CSV functions used in the project

**Watch**

- The PyTorch official channel on YouTube, for talks on data loading and performance
- Andrej Karpathy, "Neural Networks: Zero to Hero", the playlist on YouTube, for practical data handling in the context of training small models

**Practice**

- The PyTorch "Datasets and DataLoaders" tutorial, with the FashionMNIST example run on your machine. Change the batch size and the number of workers and time one epoch for each setting.

---

## Tomorrow

Day 50 puts everything together in a training loop. You will combine the model from Day 48, the loader from today, the optimizer and loss from Days 42 and 44, and a validation pass in eval mode with `no_grad`. You will also add checkpointing, so a run can be saved and resumed, and a metrics log. The loop you write tomorrow is the one that the final project on Day 51 depends on.
