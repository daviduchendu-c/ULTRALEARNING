# Day 50: Training Loops

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 4 hours
**Level:** intermediate. You have every part of a training run from Days 42 to 49. Today you connect them into one loop, add a validation pass, save checkpoints you can resume from, and log metrics you can read later.

**By the end of this day, you will have:**

- A training loop with the full order of operations: train mode, forward pass, loss, zeroed gradients, backward pass, optimizer step, and metric logging
- A validation pass in eval mode inside `torch.no_grad`, with loss and accuracy computed over the whole validation set
- Checkpoints that save the model, the optimizer, the epoch, the best validation score and the data generator state
- A test that resumes a run from a checkpoint and checks that it matches an uninterrupted run

**What you need:**

- A computer with Python 3.11 or 3.12
- PyTorch 2.5.1 from Day 46, with the same environment
- NumPy 2.1.3, pytest 8.3.3 and Matplotlib 3.9.2, as before
- The `make_files.py`, `dataset.py` and `transforms.py` modules from Day 49
- The `ConfigurableMLP` from Day 48
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 42: the loss functions and their input conventions
- Day 44: optimizers and learning-rate schedules
- Day 45: validation sets, early stopping and why the test set is used once
- Day 47: the zero, forward, loss, backward, step order and no-grad evaluation
- Day 48 and Day 49: models, datasets and loaders

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Train mode](#train-mode)
- [Forward](#forward)
- [Loss](#loss)
- [Zero gradients](#zero-gradients)
- [Backward](#backward)
- [Optimizer step](#optimizer-step)
- [Validation](#validation)
- [Checkpointing](#checkpointing)
- [Metrics](#metrics)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a complete training run with checkpoints](#project-a-complete-training-run-with-checkpoints)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every model you have trained so far ran inside a loop you wrote quickly. The loop is where the decisions meet: how often to evaluate, which numbers to keep, when to stop and what to save if the machine crashes at epoch 37. A good loop is short, and it does the same thing every time. A bad loop works once and then fails in a way nobody can reproduce.

The standard loop has two nested parts. The outer part runs over epochs, and each epoch runs once over the training batches and once over the validation batches. The inner training part follows a fixed order: put the model in train mode, move the batch to the device, compute the logits, compute the loss, clear the gradients, run the backward pass, take an optimizer step and record the loss. The validation part puts the model in eval mode, disables gradients and computes the metrics. Each step has a reason, and removing or reordering one of them is a common source of bugs.

On the job, the training loop is the code that a team trusts most, because it produces the numbers they report. A reviewer checks the order of operations, the evaluation mode, the checkpoint contents and the logged metrics. A loop that can be resumed, reproduced and inspected is worth more than one that is fast but opaque.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1950s | Herbert Robbins and Sutton Monro describe iterative stochastic updates, which become the core of the training loop. | The idea that a loop of small updates can find a good solution. |
| 1986 | Rumelhart, Hinton and Williams train multi-layer networks by repeating passes over the training examples, with weights updated after each example or batch. | The repeated-pass structure that the epoch loop still follows. |
| 1990s | Neural network practitioners adopt early stopping and validation sets to decide when to stop training. | The validation pass of today's loop. |
| 2010s | Deep learning frameworks provide built-in checkpoint and resume utilities, and the training loop becomes a standard template. | The loop pattern in this lesson. |
| 2020s | Large models are trained on clusters for weeks, and checkpointing becomes a reliability requirement. | The reason resuming from a checkpoint must be tested, not assumed. |

**Takeaway:** the loop has barely changed in form, but the cost of getting it wrong has grown with the size of the runs.

---

## Glossary

- **Epoch**: one full pass over the training data.
- **Iteration or step**: one batch update of the parameters.
- **Train mode**: the state of a module in which dropout is active and batch normalization uses batch statistics.
- **Eval mode**: the state in which dropout is off and batch normalization uses running statistics.
- **Validation pass**: an evaluation over held-out data, used for decisions and never for updating the weights.
- **Checkpoint**: a saved snapshot of the training state, from which a run can continue.
- **Resume**: continuing a run from a checkpoint, with the same model, optimizer and data order as if it had not stopped.
- **Metric**: a number that summarizes performance, such as accuracy, loss or F1 score.
- **Best checkpoint**: the checkpoint with the best validation score so far, kept separately from the latest one.
- **Determinism**: the property that the same inputs and seeds produce the same outputs. It is achievable on the CPU and is harder on the GPU.
- **Generator state**: the internal state of a random number generator, saved so that the data order can continue exactly.

---

## Train mode

Think of a pilot who switches the aircraft to a specific mode before takeoff. The mode changes how the instruments behave, and the pilot must switch back before landing. In PyTorch, `model.train()` and `model.eval()` are those mode switches.

From first principles, the mode affects layers that behave differently in training and inference, such as dropout and batch normalization, which you met on Day 45. The mode does not change whether gradients are tracked. That is controlled by `torch.no_grad`. A loop should call `model.train()` at the start of each training epoch, because the validation pass switches the model to eval mode, and a forgotten switch back leaves dropout off during training.

```python
import torch.nn as nn

def train_epoch_setup(model):
    model.train()          # dropout active, batch norm uses batch statistics
    return model
```

The limitation of the mode switch is that it is global to the module, so a model with a frozen pretrained part and a trainable head needs care. Freezing a layer with `requires_grad` does not change its mode, so a frozen batch normalization layer still updates its running statistics in train mode, which can be unexpected.

Do not evaluate in train mode. The accuracy then changes between runs because dropout is random, and the batch normalization uses the statistics of each validation batch, so the numbers depend on how the data was split into batches. In the real market, this mistake produces validation numbers that look noisy and never match the offline evaluation, and the fix is always the same: call `eval()` before validation.

---

## Forward

The forward step runs the batch through the model and produces logits. It is the same call you have used since Day 48, with the batch moved to the device first. Moving the data inside the loop, rather than before it, keeps the loader's memory use low, because only one batch sits on the device at a time.

From first principles, each batch is a tensor of shape (batch, features), and the model returns logits of shape (batch, outputs). The loop does not need to know the architecture. It needs only the contract that the loss function expects, which is that the outputs are logits and the labels have the same shape.

```python
def forward_step(model, xb, device):
    xb = xb.to(device)
    return model(xb)
```

The limitation of this simple structure is that it does not handle models with several inputs or several outputs, such as a model that returns a dictionary. For those, the loop needs a small adapter that maps the batch to the model's arguments and the outputs to the loss. The adapter is worth writing once, and it is the place to test the contract.

Do not move the model to a device after creating the optimizer, which you saw on Day 48, or move the labels but not the inputs. Check both on the first batch, and let the device error tell you which tensor is in the wrong place.

---

## Loss

The loss compares the logits with the labels, using the loss function chosen for the task. For the circle task, that is `BCEWithLogitsLoss`, which expects logits and labels of the same shape, as Day 48 described. The loss is a scalar tensor, and it is the value that the backward pass starts from.

From first principles, the loss for the batch is an average over the examples in the batch. An average over batches of different sizes is not the same as an average over examples, so the epoch loss should be a weighted average, weighting each batch's loss by its size. The loop in this lesson does that, which keeps the reported number comparable across batch sizes.

```python
def batch_loss(loss_fn, logits, yb):
    return loss_fn(logits, yb)
```

The limitation of the batch loss is that it is noisy, and one batch can look very different from the next. Looking at the epoch average, or at a moving average over recent batches, gives a steadier picture.

Do not log the loss from the first batch of each epoch and compare it with the last batch of the previous epoch, since that compares unrelated numbers. Log the weighted epoch average, and log the step loss separately if you need detail.

---

## Zero gradients

Before each backward pass, the gradients from the previous step must be cleared. Day 47 explained the reason: gradients accumulate in `.grad`, so the loop must reset them. The reset is one call on the optimizer, and it belongs before the forward pass, so that the order matches the description in the lesson.

From first principles, `optimizer.zero_grad()` sets each parameter's gradient to zero, or to `None` in recent versions of PyTorch, which saves memory and is treated as zero by the optimizer. Calling it before the forward pass is equivalent to calling it after the optimizer step, and both are correct. The one order that is wrong is calling it between the backward pass and the step, which erases the gradients before they are used.

```python
def clear_gradients(optimizer):
    optimizer.zero_grad()
```

The limitation of clearing gradients is that it is easy to forget when a loop is rewritten. A loop that skips it still runs, and its loss curve becomes erratic. The test in the project checks that a single step on a known batch matches a manual computation, which catches the missing reset.

Do not zero the gradients inside the batch loop of a gradient accumulation scheme without adjusting the loss scale. If you accumulate over four micro-batches, zero only once every four, and divide each loss by four. In industry, a loop with gradient accumulation is one of the easiest places to introduce a subtle scale bug.

---

## Backward

The backward step computes the gradients of the loss with respect to the parameters, using the graph built during the forward pass. It is a single call, `loss.backward()`, and it fills in each parameter's `.grad` as Day 47 explained.

From first principles, the backward pass frees the graph by default, so the loss cannot be used for a second backward pass unless the graph is retained. In a normal loop, that is the behavior you want, because the next batch builds a new graph. The backward step is also the place where a training run most often fails with a NaN, when a gradient becomes infinite. A check on the loss value after each step catches it early.

```python
def backward_step(loss):
    loss.backward()
```

The limitation of checking for NaN is that it is a symptom, and the cause may be several steps earlier. Logging the gradient norm, as Day 47 suggested, shows the growth before the failure.

Do not call `backward` on a loss that has been moved to the CPU as a Python float, which breaks the graph. Keep the loss as a tensor until the logging step. Calling `.item()` after the backward and optimizer step is safe.

---

## Optimizer step

The optimizer step applies the update to every parameter, using the gradients just computed and the state the optimizer keeps. It is the only place in the loop where the parameters change, and that makes it a good place to log the learning rate and the gradient norm.

From first principles, `optimizer.step()` reads each parameter's `.grad`, computes the update rule for that optimizer, and writes the new values. If a learning-rate scheduler is used, `scheduler.step()` is called once per epoch or once per batch, depending on how the scheduler was designed, and the order relative to the optimizer step matters. PyTorch warns when the order is wrong, and you should read the warning rather than ignore it.

```python
def optimizer_step(optimizer, scheduler=None):
    optimizer.step()
    if scheduler is not None:
        scheduler.step()
```

The limitation of the optimizer step is that it cannot undo a bad update. If a batch produces an enormous gradient, the parameters move far in one step, and the next validation loss shows the damage. Clipping gradients before the step, as you did on Day 47, limits that risk.

Do not call `optimizer.step()` inside the validation loop, which updates the weights on validation data and makes the validation metric meaningless. The step belongs only in the training part of the loop. In the real market, this mistake is a form of leakage, and it is one that reviewers look for when they read a loop.

---

## Validation

The validation pass measures how the model performs on held-out data after each epoch. It runs in eval mode, under `torch.no_grad`, and it does not update any parameter. It produces the metrics you use for decisions: the best checkpoint, early stopping and the choice of hyperparameters.

From first principles, the validation loss and accuracy are computed over the whole validation set, using the same weighted average as the training loss. The validation set must not be shuffled, so the numbers are stable across runs, and it must cover each example exactly once. The loop uses the validation numbers to decide whether to save a checkpoint and whether to continue training.

```python
import torch

@torch.no_grad()
def evaluate(model, loader, loss_fn, device):
    model.eval()
    total_loss, correct, count = 0.0, 0, 0
    for xb, yb in loader:
        xb, yb = xb.to(device), yb.to(device)
        logits = model(xb)
        total_loss += loss_fn(logits, yb).item() * len(xb)
        preds = (torch.sigmoid(logits) > 0.5).float()
        correct += (preds == yb).sum().item()
        count += len(xb)
    return total_loss / count, correct / count
```

The limitation of a single validation number is that it hides the variance of the estimate. With a small validation set, the accuracy can change by several points from one split to another, and early stopping decisions made on noise are unreliable. Use a validation set large enough for the decisions you make, and report the spread across seeds when it matters.

Do not use the test set for the decisions the loop makes. Each time the loop selects a checkpoint on the test set, the test numbers become optimistic. Keep a separate test set for the final report, and check it once. In the real market, teams that use the test set for early stopping report accuracies that drop when the model reaches production.

---

## Checkpointing

A checkpoint is a snapshot of everything needed to continue training exactly. For a model, that means the parameters, the optimizer state, the epoch number, the best validation score and the state of the random generator that controls the data order. Saving only the model's weights is enough for inference, but not for resuming.

From first principles, the optimizer's state includes the moment estimates for Adam, which are not part of the model's state dict, so they must be saved separately. The generator state is a tensor returned by `get_state`, and restoring it with `set_state` makes the next shuffle identical to the one in the uninterrupted run. The checkpoint is a dictionary saved with `torch.save`, and it is loaded with `weights_only=True`, which limits what can be unpickled and protects against malicious files.

```python
import torch

def save_checkpoint(path, model, optimizer, epoch, best_val, generator, history):
    torch.save({
        "epoch": epoch,
        "model": model.state_dict(),
        "optimizer": optimizer.state_dict(),
        "best_val": best_val,
        "generator": generator.get_state(),
        "history": history,
    }, path)

def load_checkpoint(path, model, optimizer, generator):
    ckpt = torch.load(path, weights_only=True)
    model.load_state_dict(ckpt["model"])
    optimizer.load_state_dict(ckpt["optimizer"])
    generator.set_state(ckpt["generator"])
    return ckpt["epoch"], ckpt["best_val"], ckpt["history"]
```

The limitation of checkpoints is that they must match the code that loads them. If you rename a layer or change its size, the load fails or loads incorrectly, and a checkpoint from an older version may not resume. Store the configuration with the checkpoint, and check it on load. The GPU adds another limit: restoring the generator on a GPU run does not guarantee identical results, because some GPU operations are not deterministic.

Do not save the model object with `torch.save(model)` rather than its state dict. That approach stores the Python class by reference, so the file breaks when the code moves. Save the state dict, and keep the class in the code. Do not overwrite the best checkpoint with the latest one. Keep two files: the latest, for resuming, and the best, for reporting.

---

## Metrics

Metrics are the numbers that summarize a run. The loop logs them after each epoch to a file that can be read later, so that you can plot the curves and compare runs without rerunning them. A metric log is a table with one row per epoch and one column per quantity.

From first principles, the epoch metrics should include the training loss, the validation loss, the validation accuracy, the learning rate and the time taken. Logging the gradient norm, using the value returned by `clip_grad_norm_`, adds a stability signal. Writing the log to a CSV after each epoch means a crash loses at most one epoch of records.

```python
import csv
from pathlib import Path

def append_metrics(path, row):
    path = Path(path)
    new_file = not path.exists()
    with path.open("a", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=list(row.keys()))
        if new_file:
            writer.writeheader()
        writer.writerow(row)
```

The limitation of a single accuracy metric is that it hides the balance between classes and the calibration of the probabilities. For the circle task, the classes are roughly balanced, so accuracy is fine. For a fraud task, you would log precision, recall and a calibration measure as well.

Do not log only the final metrics. Without the curves, you cannot tell whether the model overfitted, stopped improving early or diverged in the middle of training. In the real market, the metric log is the first artifact a reviewer opens, and it is the record that makes a result reproducible.

---

## Hands-on exercises

1. **Order check.** Write the six steps of the training loop in order from memory, and then check them against the function in the project. Mark each step with the reason it must come where it does.
2. **Eval mode effect.** Train a model with dropout for one epoch, then evaluate it twice in train mode and twice in eval mode on the same batch. Report whether the two train-mode numbers differ and whether the two eval-mode numbers differ.
3. **Missing zero.** Remove the `zero_grad` call from a short training loop and run five epochs. Compare the loss curve with the original. Describe the difference.
4. **Resume by hand.** Save a checkpoint after two epochs, load it into a fresh model and optimizer, and run one more epoch. Compare the parameters with an uninterrupted three-epoch run.
5. **Log reading.** Write a short script that reads the metric CSV and prints the epoch with the lowest validation loss and the epoch where the training loss first dropped below the validation loss by more than 0.05.

---

## Project: a complete training run with checkpoints

**Goal:** assemble the full training loop from the lesson, with validation, early stopping, a best checkpoint, a latest checkpoint for resuming, and a metric log. Prove that a run resumed from a checkpoint matches an uninterrupted run.

**Repo name:** `100-days-ai-day-50-training-loops`

**Repo layout:**

```
100-days-ai-day-50-training-loops/
├── README.md
├── requirements.txt
├── data/
│   ├── train.csv
│   ├── val.csv
│   └── test.csv
├── src/
│   └── train_lab/
│       ├── __init__.py
│       ├── data.py
│       ├── models.py
│       ├── loop.py
│       └── checkpoints.py
├── tests/
│   ├── test_step_order.py
│   └── test_resume.py
├── experiments/
│   └── run_training.py
├── checkpoints/
└── results/
    └── metrics.csv
```

**Starter code.** Create `requirements.txt` with PyTorch 2.5.1, NumPy 2.1.3, pytest 8.3.3 and Matplotlib 3.9.2 pinned. Copy `make_files.py`, `dataset.py`, `transforms.py` and `models.py` from Days 48 and 49 into `src/train_lab/`. Extend `make_files.py` to also write `data/val.csv` with 200 points from seed 2, and keep `data/test.csv` for the final report only. Use a model without dropout, since the resume test needs exact equality, and say so in the README.

Create `src/train_lab/loop.py` with the training and evaluation functions from the lesson, and add a `fit` function that runs the epochs. The `fit` function must take the model, the loaders, the optimizer, the loss, the device, the number of epochs, the patience, the checkpoint paths, the metric log path and the generator.

Create `src/train_lab/checkpoints.py` with `save_checkpoint` and `load_checkpoint` from the lesson.

**Step-by-step tasks:**

1. Create the repo, initialize Git, make a virtual environment, install the pinned packages and generate the three CSV files.
2. Write `tests/test_step_order.py`. Create a tiny model with a fixed seed, compute one training step by hand with the explicit order, and assert that the parameters after `loop.train_one_epoch` on one batch equal the manual result. This test catches a missing zero or a wrong order.
3. Implement `fit` with early stopping on the validation loss, keep the best model in a separate file `checkpoints/best.pt`, and save the latest state to `checkpoints/latest.pt` at the end of every epoch.
4. Write `tests/test_resume.py`. Train an uninterrupted run for eight epochs with a seed and save the final parameters. Then train a second run for four epochs, save the latest checkpoint, create a fresh model and optimizer, load the checkpoint, and train for four more epochs. Assert that the final parameters match the uninterrupted run with `torch.allclose` at a tolerance of 1e-6.
5. Write `experiments/run_training.py` that runs `fit` for up to 200 epochs with patience 20, Adam at lr 0.01, batch size 32, and writes `results/metrics.csv`.
6. After training, evaluate the best checkpoint on the test set once, and record the test accuracy in the README, with the epoch number of the best checkpoint.
7. Plot the training and validation loss from the metric log and save the figure to `results/loss_curves.png`.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions and the CSV generation command
- [ ] How to run the tests and the training experiment
- [ ] The statement that the model has no dropout, and why
- [ ] The epoch of the best checkpoint, and the validation and test accuracy at that epoch
- [ ] The loss curves image, with one paragraph on whether the model overfitted
- [ ] The resume test result, with the largest parameter difference
- [ ] A list of the places in the loop where a mistake would be silent

**Done when:**

- The step order test passes, and it fails if you move `zero_grad` after `backward`
- The resume test passes with a tolerance of 1e-6
- The best checkpoint is separate from the latest checkpoint, and the test accuracy is reported once
- The metric log has one row per epoch with the columns from the lesson
- The repo has at least five commits

**Evidence to commit to GitHub:** the source, the tests, the README with the epoch and accuracy table, the loss plot, the metric CSV, and a `NOTES.md` that records the order in which you wrote the loop and one mistake you made while writing it. The notes show how the loop was checked, which matters more than the final number.

**Why this project matters:** a loop that resumes correctly and logs its metrics is the core of every training job you will run on a laptop or a cluster. The resume test is the proof that the checkpoint contains what the run needs.

---

## Check your understanding

1. Why must `model.eval()` be called before validation, and why must `model.train()` be called again at the start of the next training epoch?
2. Explain why the epoch loss should be a weighted average over examples and not a simple average over batches.
3. A loop calls `optimizer.step()` inside the validation function by mistake. Describe what the validation accuracy would show and why it would look plausible.
4. Why does a checkpoint need the optimizer state, and what goes wrong if you resume with a fresh Adam optimizer and only the model weights?
5. Your resume test fails with a tiny difference in one parameter, even though the logic looks right. Name two causes and one check you would make first.
6. You want to report a single test accuracy for a model that was selected using validation accuracy. Describe the procedure that keeps the reported number honest.

**Answers:**

1. Dropout and batch normalization behave differently in eval mode, so validation must use eval mode to measure the model as it will be used. The next training epoch must return to train mode, because otherwise dropout stays off and the training no longer regularizes the model.
2. Batches can have different sizes, and the last batch is often smaller. A simple average over batches gives the last small batch the same weight as a full batch, which distorts the epoch loss. The weighted average gives each example the same weight.
3. The validation step would update the weights on the validation data. The validation accuracy would improve more than the model would generalize, because the model would learn from the examples it is measured on. It looks plausible because the metric rises smoothly, and the test accuracy later disappoints.
4. Adam keeps running estimates of the first and second moments of the gradients for each parameter. A fresh optimizer starts those estimates at zero, so its first updates differ from the uninterrupted run, and the training path changes. Loading the optimizer state restores the estimates.
5. A common cause is a difference in the random generator state, for example if the generator was not saved or was saved after the epoch rather than before. Another is a nondeterministic operation, which is less likely on the CPU. The first check is to compare the data order in the two runs, by printing the first batch indices after the resume.
6. Use the validation set to select the model and its hyperparameters, and keep the test set untouched during that process. Evaluate the chosen checkpoint on the test set once, and report that number with the procedure used. If the test set was used more than once, say so and report the number as optimistic, or collect a fresh test set.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Stochastic gradient descent", for the iterative update that the loop repeats
- Wikipedia, "Early stopping", for the stopping rule used in `fit`
- PyTorch documentation, "Saving and Loading Models" and the `torch.save` and `torch.load` pages, at pytorch.org/docs/stable, for the `weights_only` argument and what is stored
- PyTorch documentation, "torch.optim" and "torch.optim.lr_scheduler", at pytorch.org/docs/stable, for the order of `step` calls

**Tutorials**

- PyTorch Tutorials, "Quickstart", at pytorch.org/tutorials/beginner/basics/quickstart_tutorial.html, which includes a complete loop with train and test functions
- PyTorch Tutorials, "Optimizing Model Parameters", at pytorch.org/tutorials/beginner/basics/optimization_tutorial.html
- PyTorch Tutorials, "Saving and Loading Models", at pytorch.org/tutorials/beginner/basics/saveloadrun_tutorial.html

**Watch**

- Andrej Karpathy, "A Recipe for Training Neural Networks", the written guide at karpathy.github.io, with its checklist for debugging a training run
- PyTorch official channel on YouTube, for talks on training and checkpointing

**Practice**

- The PyTorch Quickstart tutorial, run on your machine. Add a checkpoint save and a resume to it, and check the resume with a test like the one in this project.

---

## Tomorrow

Day 51 is the module project. You will take the loop from today, a dataset of your choice and a model you design, train it, evaluate it once on a test set, and write up the result as a project others can reproduce. The checklists from Days 39 to 50 become the review you run on your own work, and the final report is the evidence that the module built real skill.
