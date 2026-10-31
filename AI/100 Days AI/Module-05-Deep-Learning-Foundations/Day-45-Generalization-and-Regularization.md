# Day 45: Generalization and Regularization

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3.5 hours
**Level:** intermediate. You can train a small network (Days 43 and 44) and read a training curve. Today the question changes from "does the loss go down?" to "does the model work on data it has never seen?"

**By the end of this day, you will have:**

- A clear definition of overfitting, and a measured example in which a network memorizes a small training set
- Working weight decay, early stopping and input-noise augmentation applied to the network from Day 43
- Tested NumPy implementations of inverted dropout and batch normalization, with the behavior that differs between training and inference
- An experiment table that compares the generalization gap for each method and states which one helped on your data

**What you need:**

- A computer with Python 3.11 or 3.12
- NumPy 2.1.3 (`pip install numpy==2.1.3`), free and open source under the BSD license
- pytest 8.3.3 (`pip install pytest==8.3.3`), free and open source under the MIT license
- Matplotlib 3.9.2 (`pip install matplotlib==3.9.2`), for the training curves
- The `TwoLayerNet` class and the Adam optimizer from your Day 43 and Day 44 repositories
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 13: the idea of a sample and its variance
- Day 14: estimating a quantity from a sample and the idea of an estimate that varies from sample to sample
- Day 43: the backward pass of the two-layer network
- Day 44: the Adam optimizer and the training loop

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Overfitting](#overfitting)
- [Weight decay](#weight-decay)
- [Dropout](#dropout)
- [Early stopping](#early-stopping)
- [Data augmentation](#data-augmentation)
- [Batch normalization concept](#batch-normalization-concept)
- [Hands-on exercises](#hands-on-exercises)
- [Project: measuring what helps generalization](#project-measuring-what-helps-generalization)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A network that scores 100 percent on the examples it trained on has learned something, but you do not yet know what. It may have learned the rule that generalizes to new examples, or it may have memorized the training set, including its noise. Only a test on fresh data tells you which. The gap between training performance and test performance is the central measurement of this module, and most of the techniques in this lesson exist to shrink that gap.

The tools are simple in form. You can constrain the weights, make the network train on noisy versions of its inputs, randomly turn units off during training, stop early, or normalize activations so the optimizer sees a more stable problem. Each one trades a little training fit for a better fit to new data. None of them is free, and none helps every problem, so the only reliable way to choose is to measure on a validation set.

On the job, regularization decisions are made constantly. A team with a small labeled dataset needs strong augmentation and early stopping. A team with millions of examples may need almost none. Model reviews ask for the train and validation curves, not only the final accuracy, and a good engineer can read those curves to say whether the model is memorizing, underfitting or simply not yet trained.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1943 | Andrey Tikhonov publishes regularization for ill-posed problems, adding a penalty term to stabilize solutions. | The mathematical root of weight decay and of penalized learning in general. |
| 1991 | Anders Krogh and John Hertz study weight decay in neural networks and relate it to the complexity of the learned function. | The first careful treatment of penalty terms for network weights. |
| 2012 | Geoffrey Hinton and colleagues introduce dropout as a way to reduce co-adaptation of hidden units. | Dropout becomes a standard regularizer in large image networks. |
| 2015 | Sergey Ioffe and Christian Szegedy propose batch normalization, which normalizes activations within each mini-batch. | A standard component of many convolutional networks, and a source of much discussion about why it works. |
| 2017 | Chiyuan Zhang and colleagues show that large networks can fit random labels, so the capacity of a network alone does not explain its generalization. | A reminder that generalization is an empirical property that must be measured. |

**Takeaway:** the field started with penalties from mathematics and moved to randomized and normalizing techniques. The question of why deep networks generalize is still open in part, so measurement matters more than theory.

---

## Glossary

- **Training error**: the loss or error of the model on the data it was fitted to.
- **Test error**: the error on data held out from training, used to estimate performance on new data.
- **Validation set**: a held-out set used to choose hyperparameters and stopping points, separate from the final test set.
- **Overfitting**: a model that fits the training data well but performs noticeably worse on new data.
- **Underfitting**: a model that is too simple to fit even the training data.
- **Generalization gap**: the difference between test error and training error.
- **Regularization**: any technique that reduces overfitting, by constraining the model or adding information such as noise.
- **Weight decay**: a penalty proportional to the squared size of the weights, added to the loss.
- **Dropout**: a method that randomly sets a fraction of the activations to zero during training.
- **Early stopping**: halting training when the validation error stops improving.
- **Data augmentation**: creating new training examples by transforming existing ones in ways that keep their labels.
- **Batch normalization**: a layer that normalizes each feature across the examples in a mini-batch and then rescales it with learned parameters.
- **Running statistics**: averages of the batch mean and variance, kept during training and used at inference.

---

## Overfitting

Consider a student who memorizes the answers to last year's exam. The student scores perfectly on last year's paper and fails this year's, because the questions are different. The student learned the paper, not the subject. A model that overfits does the same thing with data.

From first principles, a model with more capacity can represent more functions, and it can reduce its training error by fitting the noise in the data as well as the signal. The test error is the average loss on new samples, and the expected test error is the sum of three parts: the irreducible noise, the bias from the model being too simple and the variance from the model depending too much on the particular training sample. Overfitting is the regime of high variance. You see it as a training curve that keeps falling while the validation curve rises or stalls.

The experiment in this project shows it directly. A two-layer network with 32 hidden units trained on 40 points of the circle dataset can reach near-zero training loss, while its accuracy on 400 held-out points stays well below that. The gap is the symptom to measure.

```python
import numpy as np

def generalization_gap(train_acc, test_acc):
    return train_acc - test_acc

print(generalization_gap(1.00, 0.82))   # 0.18 gap: a sign of overfitting
```

The limitation of the gap as a single number is that it hides the cause. A large gap can come from too little data, too much capacity, a train and test split that differs in distribution, or a bug such as a leak of test labels into training. You should look at the curves and at the data split before you decide what to change.

Do not tune the model on the test set. Each time you look at test accuracy and change something, you spend part of the test set, and the number you report becomes optimistic. Keep a validation set for decisions, and use the test set once at the end. In the real market, a team that reports a test score after many rounds of tuning on it is making a claim about its validation procedure, not about the model.

---

## Weight decay

Think of a rubber band attached to each weight, pulling it gently toward zero. Small weights feel little pull, and large weights feel a lot. Weight decay is that pull. It discourages the network from relying on large weights, which usually means a smoother function.

From first principles, weight decay adds a penalty lambda/2 times the sum of squared weights to the loss. The gradient of that penalty with respect to a weight w is lambda times w, so the update becomes w = w - lr (dL/dw + lambda w). Each step shrinks the weight by a factor of (1 - lr lambda) plus the data gradient. The bias terms are usually excluded, because shrinking a bias does not control the complexity of the function in the same way.

```python
import numpy as np

def add_weight_decay(grads, params, lam, skip=("b1", "b2")):
    for name in params:
        if name not in skip:
            grads[name] = grads[name] + lam * params[name]
    return grads
```

The limitation is that the best value of lambda depends on the scale of the loss and the data. Too small has no effect, and too large makes the model underfit. Weight decay also interacts with normalization layers: a network with batch normalization is insensitive to the scale of the weights before the layer, so the penalty acts differently there. Some optimizers, such as AdamW, apply the decay directly to the weights, which avoids mixing the penalty with the adaptive scaling.

Do not apply weight decay to every parameter by reflex. Decaying the normalization gains, or the biases, can hurt performance, and many recipes exclude them. In the real market, weight decay is a standard hyperparameter in every serious training recipe, and it is usually set by a short sweep rather than chosen once and forgotten.

---

## Dropout

Picture a sports team where any player might be absent from a given game. The team cannot rely on one star, so every player learns to contribute. Dropout trains a network under the same conditions: on each step, a random fraction of its hidden units is switched off, so no unit can depend on a particular partner being present.

From first principles, during training each activation is multiplied by a random mask that is 0 with probability p and 1 otherwise. To keep the expected value of each activation the same, the surviving activations are scaled up by 1 divided by the keep probability, which is called inverted dropout. At inference, the mask is not applied, so the network runs deterministically and the scaling is already in place. The training and inference paths therefore differ, and the code must know which mode it is in.

```python
import numpy as np

def dropout_forward(A, p, rng, training=True):
    if not training or p == 0.0:
        return A, None
    keep = 1.0 - p
    mask = (rng.random(A.shape) < keep) / keep
    return A * mask, mask

def dropout_backward(dA, mask):
    return dA if mask is None else dA * mask
```

The limitation is that dropout adds noise, so training takes longer, and the benefit is smaller for models that are already small or for convolutional layers where neighboring activations are correlated. Some studies find that dropout is less useful in batch-normalized networks, and modern transformers use dropout inconsistently.

Do not forget to switch to inference mode before evaluation. If dropout stays on during validation, the accuracy numbers are noisy and pessimistic, and the early stopping decisions are wrong. Frameworks handle this with an explicit train and eval mode. In your own code, pass the mode through, and test that the output is deterministic when dropout is off.

---

## Early stopping

Consider a runner in a long race who notices that the pace has started to fall off. Pushing harder will not help, and the runner stops before exhausting themselves. Early stopping is the rule that halts training when the validation error has stopped improving for a set number of epochs.

From first principles, after each epoch you compute the validation loss. You keep a copy of the parameters with the lowest validation loss so far. If the loss has not improved for a fixed number of epochs, called the patience, you stop and restore the best copy. The method limits the effective capacity of the training run, because the weights have not yet moved far enough to fit the noise.

```python
import copy

def train_with_early_stopping(model, optimizer, train_step, val_loss_fn, max_epochs, patience):
    best_loss, best_params, wait = float("inf"), None, 0
    for epoch in range(max_epochs):
        train_step(model, optimizer)
        val_loss = val_loss_fn(model)
        if val_loss < best_loss:
            best_loss, best_params, wait = val_loss, copy.deepcopy(model.params), 0
        else:
            wait += 1
            if wait >= patience:
                break
    if best_params is not None:
        model.params = best_params
    return best_loss
```

The limitation is that the validation set is used for the stopping decision, so it is no longer a clean test of performance. Hold out a separate test set for the final number. The patience value is also a choice, and a noisy validation curve can stop training too early. Use a patience long enough to see the trend, not a single bad epoch.

Do not restore the last parameters instead of the best ones, which throws away the benefit of stopping. Stopping on the training loss is also an error, because that loss never rises during normal training and never signals overfitting. In industry, early stopping is the cheapest regularizer and is almost always on, because it saves compute as well as improving the model.

---

## Data augmentation

Think of a photographer who takes a picture of a cat and then creates several versions by cropping, flipping and changing the brightness. The cat is still a cat in every version, so the model sees more variety without any new photos. Data augmentation makes that process systematic.

From first principles, augmentation creates new samples by applying transformations that should not change the label. For images, these might be flips, crops and color changes. For tabular or synthetic data, adding small random noise to the inputs is a simple form, and it encourages the model to predict similar outputs for nearby inputs. This smoothness is the reason augmentation helps: the model cannot rely on exact coordinates that happen to appear in the training set.

```python
import numpy as np

def jitter_augment(X, y, copies, sigma, rng):
    Xs = [X]
    ys = [y]
    for _ in range(copies):
        Xs.append(X + rng.normal(scale=sigma, size=X.shape))
        ys.append(y)
    return np.concatenate(Xs), np.concatenate(ys)
```

The limitation is that augmentation is only valid when the transformation preserves the label. A horizontal flip is fine for a cat and wrong for a handwritten digit 2 that becomes an illegible shape, and a rotation that turns a 6 into a 9 changes the answer. The noise scale is also a choice: too much noise blurs the classes into each other.

Do not apply augmentation to the validation and test sets. Those sets should show the model real data, not transformed data, unless the transformation is part of the deployment. In the real market, augmentation is one of the cheapest ways to get more data for vision and speech systems, and teams that work with medical images or manufacturing defects rely on it heavily, always checking that the label is preserved.

---

## Batch normalization concept

Consider a factory line where each station receives parts of a different size. Each station struggles to adjust, because its input keeps changing. If every station first rescales its input to a standard size and then applies its own fine adjustment, the line runs more smoothly. Batch normalization does this for the activations inside a network.

From first principles, for each feature in a mini-batch, batch normalization computes the mean mu and variance var over the examples, normalizes x_hat = (x - mu) / sqrt(var + eps), and then applies learned scale gamma and shift beta to get y = gamma x_hat + beta. During inference, it uses running averages of mu and var collected during training. The learned gamma and beta let the network undo the normalization if that is useful, so the layer does not reduce the model's expressive power.

```python
import numpy as np

def batchnorm_train(X, gamma, beta, running, momentum=0.1, eps=1e-5):
    mu = X.mean(axis=0)
    var = X.var(axis=0)
    X_hat = (X - mu) / np.sqrt(var + eps)
    running["mean"] = (1 - momentum) * running["mean"] + momentum * mu
    running["var"] = (1 - momentum) * running["var"] + momentum * var
    return gamma * X_hat + beta

def batchnorm_eval(X, gamma, beta, running, eps=1e-5):
    X_hat = (X - running["mean"]) / np.sqrt(running["var"] + eps)
    return gamma * X_hat + beta
```

The limitation is that batch normalization depends on the batch. With a batch of one, the variance is zero and the statistics are meaningless, and small batches give noisy estimates. Its regularizing effect is real but partly a side effect of batch noise, and it is not a substitute for other methods. Layer normalization and group normalization avoid the batch dependence, and transformers mostly use layer normalization.

Do not use training-mode batch statistics at inference, and do not forget to update the running averages during training. Either error gives predictions that depend on the batch composition, which is a serious bug in production. In industry, batch normalization is common in convolutional networks, and serving code must switch to inference mode, which many teams test explicitly.

---

## Hands-on exercises

1. **Gap reading.** You have training accuracy 0.97 and validation accuracy 0.71 after 100 epochs. Describe what the training curve and the validation curve probably look like, and name the first two changes you would test.
2. **Weight decay arithmetic.** A weight of 2.0 gets a data gradient of 0.5 with lr = 0.1 and lambda = 0.01. Compute the update by hand, with and without weight decay.
3. **Dropout mean check.** Using `dropout_forward` with p = 0.5 on an array of ones of length 100000, confirm that the mean of the output is close to one. Then check that the output in inference mode equals the input.
4. **Batch statistics.** Generate a batch of 64 examples with 3 features drawn from a normal distribution with mean 5 and standard deviation 3. Apply the training-mode batch normalization with gamma = 1 and beta = 0. Confirm the output has mean near zero and standard deviation near one per feature.
5. **Stopping point.** Given a list of twenty made-up validation losses that fall and then rise, find by hand the epoch that early stopping with patience 3 would restore.

---

## Project: measuring what helps generalization

**Goal:** create a small training set in which the network overfits, then apply weight decay, early stopping and augmentation one at a time and in combination. Measure the generalization gap for each setting on the same split, and write up which method helped on your data and by how much.

**Repo name:** `100-days-ai-day-45-generalization-and-regularization`

**Repo layout:**

```
100-days-ai-day-45-generalization-and-regularization/
├── README.md
├── requirements.txt
├── src/
│   └── regularize_lab/
│       ├── __init__.py
│       ├── data.py
│       ├── regularizers.py
│       ├── network.py
│       └── batchnorm.py
├── tests/
│   ├── test_regularizers.py
│   └── test_batchnorm.py
└── experiments/
    └── generalization_table.py
```

**Starter code.** Create `requirements.txt` with the pinned packages. Copy `network.py` from Day 43 into `src/regularize_lab/network.py`, and copy `optimizers.py` from Day 44 into `src/regularize_lab/`. Create `src/regularize_lab/data.py` so that it generates a circle dataset with the labeling rule from Day 44, with a function `make_circles(n, seed)` that returns X of shape (n, 2) and y of shape (n, 1). Then create the regularizer helpers below in `src/regularize_lab/regularizers.py`.

```python
import numpy as np


def add_weight_decay(grads, params, lam, skip=("b1", "b2")):
    for name in params:
        if name not in skip:
            grads[name] = grads[name] + lam * params[name]
    return grads


def dropout_forward(A, p, rng, training=True):
    if not training or p == 0.0:
        return A, None
    keep = 1.0 - p
    mask = (rng.random(A.shape) < keep) / keep
    return A * mask, mask


def dropout_backward(dA, mask):
    return dA if mask is None else dA * mask


def jitter_augment(X, y, copies, sigma, rng):
    Xs, ys = [X], [y]
    for _ in range(copies):
        Xs.append(X + rng.normal(scale=sigma, size=X.shape))
        ys.append(y)
    return np.concatenate(Xs), np.concatenate(ys)
```

Write `src/regularize_lab/batchnorm.py` with the two functions from the lesson, and write the tests in the next steps.

**Step-by-step tasks:**

1. Create the repo, initialize Git, create a virtual environment and install the pinned packages.
2. Write `tests/test_regularizers.py`. Check that `add_weight_decay` leaves bias gradients unchanged and adds lambda times the weight to the weight gradients. Check that inverted dropout keeps the mean near one over 100,000 values and that inference mode returns its input unchanged.
3. Write `tests/test_batchnorm.py`. Check that the training output has mean near zero and standard deviation near one per feature, and that the eval output uses the running statistics by comparing with a hand-computed value for one example.
4. Write `experiments/generalization_table.py`. Use 40 training points, 400 validation points for early stopping, and 400 test points, all from different seeds. Train for up to 2,000 epochs with Adam at lr = 0.01. Run six configurations: baseline, weight decay 0.001, weight decay 0.01, early stopping with patience 100, jitter augmentation with five copies and sigma 0.1, and the combination of early stopping and augmentation.
5. For each configuration, record the training accuracy, the test accuracy and the gap. Repeat each configuration with three seeds and report the mean and the spread.
6. Save the table as `results/generalization_table.md` and a plot of train and validation loss for the baseline and the early stopping run to `results/curves.png`.
7. Add dropout with p = 0.2 to the hidden layer as an optional extension. You must update the backward pass to multiply by the mask, and you must confirm with a gradient check that the update is still correct when the mask is fixed.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and the table experiment
- [ ] The results table with the mean and spread across three seeds for each configuration
- [ ] The curves plot, with one paragraph describing the baseline overfitting
- [ ] A statement of which method reduced the gap most, and whether the test accuracy also rose
- [ ] A note on the split: which sets were used for decisions and which for the final number

**Done when:**

- The regularizer and batch normalization tests pass
- The baseline shows a clear generalization gap on your data
- The table covers all six configurations with three seeds each, and every number comes from your run
- The README states which method you would use first, and cites the numbers that support it
- The repo has at least five commits

**Evidence to commit to GitHub:** the source, the tests, the README with the table, the curves plot, and a `NOTES.md` that records one configuration that did not help and what that tells you about your data. A negative result is useful evidence when it comes with the numbers.

**Why this project matters:** it turns regularization from a list of terms into a measurement. The discipline of separate validation and test sets, and of reporting spread across seeds, is what you will be asked for in a model review.

---

## Check your understanding

1. Explain why training loss can keep falling while validation loss rises, and what that pattern tells you about the model.
2. Weight decay shrinks weights every step. Why does a penalty on the squared weights tend to produce smoother functions than no penalty?
3. Dropout uses inverted scaling during training. What would go wrong at inference if you omitted the scaling and only removed the mask?
4. Early stopping restores the best parameters rather than the last. Explain what would happen to the test accuracy if you used the last parameters after a long patience period.
5. You add random noise to the inputs as augmentation, and the test accuracy falls. Name two causes and one check you would run.
6. A batch normalized network gives different predictions for the same input depending on the other examples in the batch at inference time. What is the bug, and how would you test for it?

**Answers:**

1. The model is fitting the particular training examples more closely as it trains, including their noise. The validation loss rises because the learned function no longer matches the broader pattern in new data. The pattern is the classic sign of overfitting, and it suggests stopping, adding regularization or adding data.
2. A large weight amplifies small input changes, so a function with large weights changes quickly. The penalty pushes weights toward zero, so the function changes more slowly and responds less to noise in the inputs.
3. Without the scaling, the expected activation at inference would be smaller than during training, by the keep probability. The next layer would receive systematically smaller inputs, and its predictions would be shifted. Inverted dropout moves the scaling to training so inference needs no change.
4. The last parameters may have moved into the region where the model fits the noise, so the test accuracy would be lower than at the best validation point. The best copy is the one that generalized best on the validation data, which is the quantity early stopping optimizes.
5. The noise may be too large, so the classes blur together, or the transformation may not preserve the label near the boundary. A check is to plot the augmented points on the data, or to try a smaller sigma and compare the validation accuracy with and without augmentation on the same seeds.
6. The bug is that the layer uses batch statistics at inference instead of the running statistics, or that the running averages were not updated during training. A test is to run the same example in a batch of one and in a batch of many, and confirm the output is identical in evaluation mode.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Overfitting", for the definition and the examples
- Wikipedia, "Regularization (mathematics)", for the penalty view of weight decay
- Wikipedia, "Dropout (neural networks)", for the method and its variants
- Wikipedia, "Batch normalization", for the algorithm and the training and inference modes
- Wikipedia, "Early stopping", for the stopping rule
- Wikipedia, "Data augmentation", for the general idea and its use in vision
- Nitish Srivastava, Geoffrey Hinton and colleagues, "Dropout: A Simple Way to Prevent Neural Networks from Overfitting", Journal of Machine Learning Research, 2014
- Sergey Ioffe and Christian Szegedy, "Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift", arXiv 1502.03167
- Chiyuan Zhang and colleagues, "Understanding deep learning requires rethinking generalization", arXiv 1611.03530

**Tutorials**

- Google, "Machine Learning Crash Course", at developers.google.com/machine-learning/crash-course, the sections on overfitting and regularization
- PyTorch documentation, "torch.nn" pages for Dropout and BatchNorm1d, at pytorch.org/docs/stable, for the train and eval mode behavior
- GeeksforGeeks, search for "Regularization in Machine Learning" for worked examples

**Watch**

- StatQuest with Josh Starmer, "Machine Learning Fundamentals: Bias and Variance", on YouTube
- Andrej Karpathy, "A Recipe for Training Neural Networks", the written guide at karpathy.github.io, which includes a section on overfitting

**Practice**

- Google's ML Crash Course exercises on overfitting, and the TensorFlow Playground with the noise and the number of hidden units changed, so you can watch the boundary overfit and then smooth out

---

## Tomorrow

Day 46 moves from NumPy to PyTorch. You will install PyTorch 2.5.1, learn how tensors store shapes, dtypes and devices, and move computation between the CPU and a GPU when one is available. Every concept from the last six days reappears as a tensor operation, and the model you built by hand will become a few lines of framework code. The regularizers from today will also map directly onto the PyTorch modules you meet later.
