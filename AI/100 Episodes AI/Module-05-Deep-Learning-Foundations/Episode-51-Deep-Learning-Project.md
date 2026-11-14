# Day 51: Deep Learning Project

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 6 hours, spread over two sittings if you can. Training takes a few minutes per run on a CPU, and the write-up takes the rest.
**Level:** intermediate. This is the module project. It draws on every earlier day, and it asks you to make the decisions a team makes: the split, the architecture, the stopping rule, the single test evaluation and the report.

**By the end of this day, you will have:**

- A complete PyTorch project that trains a multi-layer perceptron on Fashion-MNIST, with a validation split, early stopping, a best checkpoint and a metric log
- A single, honest test-set evaluation, with per-class results and a confusion matrix
- Tests for the data split, the model contract and the checkpoint round trip
- A model card in the README that states what the model does, how it was trained, how it was evaluated and where it fails

**What you need:**

- A computer with Python 3.11 or 3.12. A GPU is optional. The project is sized for a CPU.
- PyTorch 2.5.1 and NumPy 2.1.3, as installed on Day 46
- torchvision 0.20.1, which is the release that pairs with PyTorch 2.5.1. Install it with `pip install torchvision==0.20.1 --index-url https://download.pytorch.org/whl/cpu` for a CPU build, or the matching CUDA index if your PyTorch build uses CUDA.
- pytest 8.3.3 and Matplotlib 3.9.2
- About 40 MB of disk space for the dataset, downloaded on first run
- Git and a free GitHub account

**Prerequisites:**

- Day 39 to Day 42: the forward pass, activations, loss functions and the logits convention
- Day 43 and Day 47: what the backward pass computes, and why autograd can do it for you
- Day 44 and Day 45: optimizers, schedules, regularization and early stopping
- Day 46 to Day 50: tensors, modules, datasets, loaders and the training loop

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Choosing the problem and the data](#choosing-the-problem-and-the-data)
- [Splitting without leaking](#splitting-without-leaking)
- [Designing the model](#designing-the-model)
- [Training and stopping](#training-and-stopping)
- [Evaluating once](#evaluating-once)
- [Analyzing errors](#analyzing-errors)
- [Hands-on exercises](#hands-on-exercises)
- [Project: train, evaluate and document a neural network](#project-train-evaluate-and-document-a-neural-network)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

For fifty days, you have built the parts of a deep learning system. Today you assemble them into a project that someone else could run, check and build on. The shift is from writing code that works once to making a claim that others can verify. That claim has three parts: what the model does, how well it does it on data it has never seen, and where it fails.

The project uses Fashion-MNIST, a collection of 70,000 grayscale images of clothing items in ten classes, split into 60,000 training images and 10,000 test images. It is small enough to train on a laptop, and it is hard enough that a poor pipeline or a leaked test set produces numbers that look good and mean nothing. A multi-layer perceptron on Fashion-MNIST typically reaches the high 80s in accuracy, and the commonly reported range is close to that. If your model is far below it, the cause is almost always in the pipeline, not in the architecture.

On the job, a project like this one is what you hand over. A reviewer will read the split, the stopping rule, the evaluation procedure and the failure analysis before they trust the accuracy number. The discipline of this day is the discipline that makes a model shippable.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1998 | LeCun, Bottou, Bengio and Haffner describe the MNIST handwritten digit benchmark and the convolutional networks that solved it well. | The template for a small image benchmark with a fixed test set. |
| 2010s | Machine learning researchers begin to report results on MNIST-like benchmarks with standard splits, to make comparisons fair. | The norm of a fixed train and test split that this project follows. |
| 2017 | Han Xiao, Kashif Rasul and Roland Vollgraf release Fashion-MNIST as a drop-in replacement for MNIST, with the same format and harder classes. | The dataset used in this project, chosen because MNIST digits are too easy to separate models. |
| 2018 onward | Reproducibility checklists and model cards become common in research and industry, recording data, training and evaluation choices. | The model card you write in the README. |
| 2020s | Teams treat test-set discipline as a reporting requirement, and they keep a separate evaluation split for final numbers only. | The single test evaluation in this project. |

**Takeaway:** a benchmark is only useful if its split is respected. The rest of the project is built around that rule.

---

## Glossary

- **Fashion-MNIST**: a dataset of 28 by 28 grayscale images of ten clothing classes, with 60,000 training and 10,000 test images.
- **Class**: one of the labels, such as shirt, sneaker or bag.
- **Split**: the division of the data into training, validation and test sets.
- **Stratified split**: a split in which each class keeps roughly the same proportion in every part.
- **Model card**: a short document that states what a model is for, how it was trained, how it was evaluated and its known limits.
- **Confusion matrix**: a table that counts, for each true class, how often each class was predicted.
- **Per-class accuracy**: the accuracy computed separately for each class, which can reveal weak classes hidden by a good average.
- **Error analysis**: the study of the examples the model gets wrong, to find patterns in its failures.
- **Reproducibility**: the property that someone else can rerun the project and get the same results, within a stated tolerance.
- **Seed**: a number that fixes the random choices in a run, such as the split, the initialization and the shuffle.

---

## Choosing the problem and the data

Before you write any code, decide what the project is for. A model that classifies clothing images is useful for a catalog tagger, a sorting machine or a teaching example. The problem statement decides the metric, the cost of errors and the level of detail in the report.

From first principles, a classification problem needs an input format, a label set and a metric. Here the input is a 28 by 28 image, scaled to values between 0 and 1 and then standardized. The label set is the ten classes. The metric is accuracy, and you will also report per-class accuracy because a catalog that mislabels one class badly is not acceptable even if the average is high. The cost of errors is uneven in some applications, and you should say so in the model card even if you do not adjust for it.

The data comes from `torchvision.datasets.FashionMNIST`, which downloads the files on first use. Keep the downloaded files in a `data` folder that is listed in `.gitignore`, so the repository does not contain the dataset. Commit the code and the results, not the images.

The limitation of a benchmark is that it is not your production data. Fashion-MNIST images are centered, cropped and grayscale, and a real photo of a shirt on a model is very different. A model that scores well here does not show that it works on a shop's photos, and the model card must say so.

Do not pick the problem after you see the results. Write the problem statement first, commit it to the README, and then run the experiments. The order protects you from changing the question to fit the answer.

---

## Splitting without leaking

The split is the most important decision in the project. The training set is used to learn the weights, the validation set is used to make decisions, and the test set is used once to report a number. Each set has a different job, and mixing them makes the final number optimistic.

From first principles, the 60,000 training images should be divided into a training part and a validation part, for example 54,000 and 6,000. The split should be random and seeded, so that the same examples always go to the same part, and it should be stratified, so that each class appears in roughly the same proportion in both. The official 10,000 test images are kept aside, and no decision of any kind is made using them. The normalization statistics must be computed from the training part only, as Day 49 required.

```python
import torch
from torch.utils.data import random_split

def split_train_val(dataset, n_val=6000, seed=0):
    g = torch.Generator().manual_seed(seed)
    n_train = len(dataset) - n_val
    train_set, val_set = random_split(dataset, [n_train, n_val], generator=g)
    return train_set, val_set
```

The function above performs a random split, not a stratified one. For Fashion-MNIST, the classes are balanced in the full training set, so a random split is close to stratified in practice. If you adapt the project to an imbalanced dataset, write a stratified split and check the class counts in each part before training.

The limitation of a single split is that the validation accuracy has variance. A validation set of 6,000 images gives an accuracy estimate with a standard error of about 0.4 percentage points at an accuracy near 89 percent, which is enough to make early stopping noisy. Repeating the training with several seeds gives a clearer picture.

Computing the normalization mean and standard deviation on the full training set and then splitting is a small leak, and computing them on all 70,000 images, which includes the test set, is a clear one. The fix is to split first, compute the statistics on the training part, and apply them to the other parts. In the real market, a split that leaks is one of the most common reasons a model underperforms after launch.

---

## Designing the model

The architecture should be the simplest one that solves the problem well enough, and the design should be justified by a measurement, not by habit. For Fashion-MNIST, a multi-layer perceptron with two hidden layers is a reasonable baseline, and a convolutional network would do better, but convolution is the subject of Module 6, so this project uses a dense network and says so.

From first principles, the model flattens each 28 by 28 image into a vector of 784 numbers, passes it through a hidden layer of 256 ReLU units, a dropout layer, a hidden layer of 128 ReLU units, another dropout layer and a final layer of 10 logits. The loss is `nn.CrossEntropyLoss`, which takes the logits and the integer class labels. Dropout with a rate of 0.2 is a regularizer from Day 45, and the value is a starting point that you can tune on the validation set.

```python
import torch.nn as nn

class FashionMLP(nn.Module):
    def __init__(self, hidden=(256, 128), p_drop=0.2):
        super().__init__()
        layers = [nn.Flatten()]
        prev = 28 * 28
        for h in hidden:
            layers += [nn.Linear(prev, h), nn.ReLU(), nn.Dropout(p_drop)]
            prev = h
        layers.append(nn.Linear(prev, 10))
        self.net = nn.Sequential(*layers)

    def forward(self, x):
        return self.net(x)          # logits, shape (batch, 10)
```

The limitation of this architecture is that it ignores the two-dimensional structure of the image. Neighboring pixels are treated as unrelated inputs, so the model must learn spatial patterns from scratch, which needs more data and more capacity. That is the reason convolutional networks work better on images, and it is the first thing to say in the model card.

Do not tune the architecture until the test accuracy looks good. Change the design only in response to validation results, and record every change in a table, with the validation accuracy for each. A table of ten configurations with validation scores is far more credible than a single chosen model with no history.

---

## Training and stopping

Training uses the loop from Day 50, adapted for ten classes. The optimizer is Adam with a learning rate of 0.001, which is the default from the original paper. The batch size is 128, which is a common size that fits easily in memory. The loop runs for up to 60 epochs, with early stopping on the validation loss and a patience of 5 epochs. The best checkpoint is saved whenever the validation loss improves, and the latest state is saved every epoch for resuming.

From first principles, the stopping rule chooses the number of epochs from the validation curve, not from a fixed number chosen in advance. The best checkpoint is the model at the epoch with the lowest validation loss, which is the model the project will evaluate. The validation accuracy is also logged, and it can be used as the stopping criterion if you prefer, as long as you state the choice in the model card.

```python
import torch

@torch.no_grad()
def accuracy(model, loader, device):
    model.eval()
    correct, count = 0, 0
    for xb, yb in loader:
        xb, yb = xb.to(device), yb.to(device)
        preds = model(xb).argmax(dim=1)
        correct += (preds == yb).sum().item()
        count += len(yb)
    return correct / count
```

The limitation of early stopping on a validation loss is that the loss and the accuracy can disagree. A model can gain accuracy while its loss rises, because the probabilities become more confident and the mistakes become more expensive. State which quantity controls the stopping, and log both.

Do not pick the checkpoint with the best test accuracy after training for a fixed number of epochs. That choice uses the test set to select the model. Pick the checkpoint with the validation data only. In industry, a training run that is stopped by the test score is a finding that reviewers always flag.

---

## Evaluating once

The test evaluation is the final step, and it happens once. You load the best checkpoint, switch the model to eval mode, run the test loader under `no_grad` and record the number. You do not change the model after seeing the number, and if you do, you must report that the test set has been used and the reported number is optimistic.

From first principles, the test accuracy is an estimate of the accuracy on new data from the same distribution. Its standard error with 10,000 images is about 0.3 to 0.4 percentage points at these accuracies, so differences smaller than that between two models are not meaningful on a single run. Report the test accuracy with the standard error, or with the spread across several seeded runs.

The limitation of one test evaluation is that it reports one model. If you want to compare two architectures, you should train each several times with different seeds and compare the distributions, using the validation set for the selection and the test set for the final comparison only.

Do not evaluate the test set after every epoch to see the curve. The curve belongs on the validation set. The test set is a sealed envelope that you open once. In the real market, the teams that keep the test set sealed are the teams whose reported numbers survive contact with production.

---

## Analyzing errors

A single accuracy number hides the pattern of failures. The confusion matrix shows which classes are mistaken for which, and the per-class accuracy shows whether one class is much weaker than the rest. Fashion-MNIST has well-known confusions, such as between shirts, T-shirts and pullovers, because those items look alike at 28 by 28 pixels.

From first principles, the confusion matrix is a 10 by 10 table where row i and column j count how often a true class i is predicted as j. The diagonal holds the correct predictions. Off-diagonal entries show the systematic errors. Reading the matrix tells you whether the model's errors are random or structured, and whether they are the kind that a different architecture or more data would fix.

```python
import torch

def confusion_matrix(preds, labels, n_classes=10):
    cm = torch.zeros(n_classes, n_classes, dtype=torch.long)
    for t, p in zip(labels.view(-1), preds.view(-1)):
        cm[t, p] += 1
    return cm
```

The limitation of error analysis is that it is descriptive. It shows where the model fails, not why. To answer why, you need to look at the misclassified images and form a hypothesis, then test it with a change to the model or the data.

Do not fix the errors you see in the test set by adding examples from the test set to training, which is a form of leakage. If you want to improve a class, change the training data or the model, retrain, and evaluate on the validation set. In the real market, error analysis drives the next project, and the report should name the next step it suggests.

---

## Hands-on exercises

1. **Split check.** Create the split with seed 0 and count the number of examples of each class in the training part and in the validation part. Confirm that the counts are close to the proportions in the full training set.
2. **Normalization check.** Compute the mean and standard deviation of the pixel values for the training part only, and compare them with the commonly used values 0.2860 and 0.3530. Explain any difference in one sentence.
3. **Parameter count.** For the `FashionMLP` with the default hidden sizes, compute the number of parameters by hand, then check the sum with code. The first hidden layer has 784 times 256 weights plus 256 biases.
4. **Architecture table.** Train two configurations, one with hidden sizes (128,) and one with (256, 128), using the validation set only, with a fixed seed. Record their validation accuracy and parameter count in a table.
5. **Confusion reading.** Given a confusion matrix you produce, name the two classes with the most confusion between them and write one sentence on why those classes look alike.

---

## Project: train, evaluate and document a neural network

**Goal:** train a multi-layer perceptron on Fashion-MNIST with a validation split, early stopping and checkpoints, evaluate the best checkpoint once on the test set, and document the work in a model card that another engineer could use to decide whether to trust the result.

**Repo name:** `100-days-ai-day-51-deep-learning-project`

**Repo layout:**

```
100-days-ai-day-51-deep-learning-project/
├── README.md
├── MODEL_CARD.md
├── requirements.txt
├── .gitignore
├── src/
│   └── fashion_lab/
│       ├── __init__.py
│       ├── data.py
│       ├── model.py
│       ├── loop.py
│       └── evaluate.py
├── tests/
│   ├── test_data.py
│   ├── test_model.py
│   └── test_checkpoint.py
├── experiments/
│   ├── train.py
│   └── evaluate_test.py
├── checkpoints/
└── results/
    ├── metrics.csv
    ├── confusion_matrix.csv
    └── per_class_accuracy.md
```

**Starter code.** Create `requirements.txt` with PyTorch 2.5.1, torchvision 0.20.1, NumPy 2.1.3, pytest 8.3.3 and Matplotlib 3.9.2 pinned. Create `.gitignore` with `data/`, `checkpoints/` and `__pycache__/` so the dataset and the large files stay out of Git. Create `src/fashion_lab/data.py`:

```python
import torch
from torch.utils.data import random_split
from torchvision import datasets, transforms

MEAN, STD = 0.2860, 0.3530


def get_datasets(root="data"):
    to_tensor = transforms.Compose([
        transforms.ToTensor(),
        transforms.Normalize((MEAN,), (STD,)),
    ])
    train_full = datasets.FashionMNIST(root, train=True, download=True, transform=to_tensor)
    test = datasets.FashionMNIST(root, train=False, download=True, transform=to_tensor)
    return train_full, test


def split_train_val(train_full, n_val=6000, seed=0):
    g = torch.Generator().manual_seed(seed)
    return random_split(train_full, [len(train_full) - n_val, n_val], generator=g)
```

Create `src/fashion_lab/model.py` with `FashionMLP` from the lesson. Create `src/fashion_lab/loop.py` with the training and validation functions, adapted to `nn.CrossEntropyLoss` and to the argmax accuracy. Create `src/fashion_lab/evaluate.py` with the confusion matrix function and a per-class accuracy function.

**Step-by-step tasks:**

1. Create the repo, initialize Git, make a virtual environment, install the pinned packages and run `python -c "from fashion_lab.data import get_datasets; get_datasets()"` from the repo root to download the data.
2. Write `tests/test_data.py`. Use a small `TensorDataset` of 100 samples to test `split_train_val`, checking the sizes and that the two parts are disjoint. Write one test that the test set is never passed to the split function, by checking the function signature.
3. Write `tests/test_model.py`. Check that a batch of four images of shape (4, 1, 28, 28) produces logits of shape (4, 10), and that two calls in eval mode give identical outputs.
4. Write `tests/test_checkpoint.py`. Save the state dict of a small model and load it into a fresh model, and assert that the outputs match.
5. Write `experiments/train.py` that trains the model with a seed of 0, batch size 128, Adam at lr 0.001, up to 60 epochs, patience 5, and writes `results/metrics.csv` with one row per epoch. Save the best checkpoint to `checkpoints/best.pt`.
6. Run `experiments/train.py` and record the epoch of the best checkpoint and its validation accuracy in the README.
7. Write `experiments/evaluate_test.py` that loads `checkpoints/best.pt`, computes the test accuracy once, writes the confusion matrix to `results/confusion_matrix.csv` and the per-class accuracy to `results/per_class_accuracy.md`. Run it exactly once for the final number.
8. Repeat the training with seeds 1 and 2 using the same settings, and report the validation accuracy of each seed. Do not evaluate the test set for these extra seeds unless you state in the model card that you did so.
9. Fill in `MODEL_CARD.md` with the sections listed below.

**Model card sections:**

- Model details: architecture, parameter count, framework versions, seed
- Intended use and out-of-scope use
- Training data and split, including the normalization statistics and their source
- Training procedure: optimizer, learning rate, batch size, epochs, stopping rule and the epoch chosen
- Evaluation: validation accuracy per seed, test accuracy once, standard error, per-class results
- Error analysis: the two most confused class pairs and one hypothesis for each
- Limitations: the benchmark is not production data, a dense network ignores spatial structure, and the test set was used once
- Next steps: one change you would try next and how you would evaluate it without using the test set

**README checklist:**

- [ ] A one-line description and a link to the model card
- [ ] Setup steps with exact versions, including the torchvision install
- [ ] How to run the tests, the training and the test evaluation
- [ ] The table of validation accuracy for each seed
- [ ] The test accuracy, reported once, with the standard error
- [ ] The confusion matrix image or table, with one paragraph of reading
- [ ] Reproducibility notes: seed, hardware, time per epoch

**Done when:**

- All data, model and checkpoint tests pass
- The test evaluation was run exactly once for the reported number, and the README says so
- The metric log has one row per epoch with validation loss and accuracy
- The model card has all nine sections filled in with numbers from your runs
- The repo has at least six commits, and the dataset and checkpoints are not committed

**Evidence to commit to GitHub:** the source, the tests, the README, the model card, the metric log, the confusion matrix and per-class results, and a `NOTES.md` that records one decision you changed because of validation results and one place where you were tempted to look at the test set. The notes are the evidence of the discipline the project asks for.

**Why this project matters:** it is a complete piece of work with a clear claim, a clean evaluation and an honest account of its limits. A reviewer can check each part, and you can use the same structure for every model you build after this module.

---

## Check your understanding

1. Why must the test set be used once, and what happens to the reported number if you choose the checkpoint with it?
2. A colleague says that because the validation accuracy is 89 percent and the test accuracy is 87 percent, the model is overfitting to the validation set. What evidence would you need to support or reject that claim?
3. Explain why per-class accuracy can reveal a problem that the overall accuracy hides, using a case where one class is much weaker than the others.
4. You normalize the images with statistics from all 70,000 images before splitting. Explain the mechanism of the leak and say how you would verify whether it changed the results.
5. Your model's validation loss rises after epoch 12, but the validation accuracy keeps rising slowly until epoch 30. Which epoch do you choose for the best checkpoint, and what do you write in the model card?
6. A reviewer asks why you did not use a convolutional network. Give an honest answer that uses the project's scope and the limits of the dense model.

**Answers:**

1. The test set is the only estimate of performance on data the model has not influenced. If the checkpoint is chosen with it, the reported number is selected to be high and overstates the performance on new data. The procedure is to select with validation, evaluate the test set once, and report that single number.
2. Overfitting to the validation set means the selection process has tuned to the noise in those 6,000 images. Evidence would be a test accuracy that falls well below the validation accuracy after many rounds of tuning, or a validation accuracy that changes a lot across seeds. A gap of two points, with standard errors of about 0.4 points for validation and 0.3 for the test, is larger than sampling noise alone, so the check is different: compare the spread across seeds, and check whether the selection process made many choices against the validation set.
3. If one class has 60 percent accuracy and the others have 90 percent, the average can still be above 85 percent. A product that sells that class would fail its customers while the headline number looks good. Per-class results expose that weakness.
4. The normalization statistics include the test images, so the preprocessing carries a small amount of information about the test distribution. The effect on the result is usually small, but it violates the rule. To verify, compute the statistics both ways, retrain with each, and compare the validation and test accuracy across seeds.
5. Choose the checkpoint by the rule stated in the model card, which should be decided before training. If the stopping rule uses the loss, choose epoch 12. If it uses accuracy, choose epoch 30. The model card must name the rule and explain that the loss and accuracy disagreed, because the reader needs to know that the choice was made on a specific criterion.
6. A dense network is simpler to explain, and it is the right baseline to show what the pipeline can do. It ignores the spatial structure of images, so a convolutional model would likely do better, and that is the next step after this project, in Module 6. The limit is stated in the model card, and the project does not claim that the dense model is the best approach for images.

---

## Sources to read, watch and practice

**Read**

- Han Xiao, Kashif Rasul and Roland Vollgraf, "Fashion-MNIST: a Novel Image Dataset for Benchmarking Machine Learning Algorithms", arXiv 1708.07747
- The Fashion-MNIST repository by Zalando Research on GitHub, for the file format, the classes and the license
- Wikipedia, "MNIST database", for the dataset that Fashion-MNIST mirrors
- Margaret Mitchell and colleagues, "Model Cards for Model Reporting", arXiv 1810.03993, for the structure of the model card
- Wikipedia, "Confusion matrix", for the table and its reading

**Tutorials**

- torchvision documentation, "datasets.FashionMNIST" and "transforms", at pytorch.org/vision/stable, for the loading and preprocessing used in the project
- PyTorch Tutorials, "Quickstart", at pytorch.org/tutorials/beginner/basics/quickstart_tutorial.html, which trains a model on FashionMNIST
- scikit-learn documentation, "Model evaluation: quantifying the quality of predictions", at scikit-learn.org, for the confusion matrix and per-class metrics

**Watch**

- Andrej Karpathy, "A Recipe for Training Neural Networks", the written guide at karpathy.github.io, for the checklist to run through before you report a result
- The PyTorch official channel on YouTube, for talks on training and evaluating models

**Practice**

- The PyTorch Quickstart on FashionMNIST, extended with the split, the checkpoint and the model card in this project. Compare your numbers with the commonly reported range for a dense network.

---

## Tomorrow

Day 52 begins Module 6, Computer Vision, and it starts where this project stopped. An image is a grid of numbers, and you will learn what pixels, channels, resolution, color spaces and normalization mean in practice. The model you trained today treated each pixel as an unrelated input. Module 6 introduces the operations that use the image's structure, starting with the convolution you will implement by hand on Day 54.
