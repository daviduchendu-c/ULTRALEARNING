# Day 46: PyTorch Tensors

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3.5 hours
**Level:** beginner to intermediate. You know NumPy arrays, shapes and broadcasting from Days 9 and 39. Today the same ideas move into PyTorch, and you add two things NumPy does not handle: the device a tensor lives on and the autograd flag that Day 47 explains.

**By the end of this day, you will have:**

- PyTorch 2.5.1 installed and verified in a clean virtual environment
- A working understanding of tensor shapes, dtypes and devices, and of the difference between `view`, `reshape` and in-place operations
- A tested PyTorch version of the Day 39 forward pass that matches your NumPy results to floating-point tolerance
- A device report from your own machine, and a short benchmark that shows when moving work to a GPU helps

**What you need:**

- A computer with Python 3.11 or 3.12
- PyTorch 2.5.1 (CPU build, or a CUDA build if you have an NVIDIA GPU). Install instructions are in the project section. PyTorch is free and open source under the BSD license.
- NumPy 2.1.3 (`pip install numpy==2.1.3`), for the comparison with your Day 39 code
- pytest 8.3.3 (`pip install pytest==8.3.3`)
- An NVIDIA GPU with a recent driver is optional. The lesson works on a CPU, and the GPU section says what changes.
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 9: matrix shapes and matrix multiplication
- Day 39: the forward pass of a dense layer in NumPy
- Day 45: the idea of train and inference modes, which appears again here

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Tensors](#tensors)
- [Shapes](#shapes)
- [Devices](#devices)
- [Dtypes](#dtypes)
- [Tensor operations](#tensor-operations)
- [CPU/GPU movement](#cpugpu-movement)
- [Hands-on exercises](#hands-on-exercises)
- [Project: tensor basics and the forward pass in PyTorch](#project-tensor-basics-and-the-forward-pass-in-pytorch)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every model you have written so far used NumPy arrays. NumPy is excellent for learning, but it has no notion of a GPU, no automatic differentiation and no standard way to describe a neural network. PyTorch provides the array type, called a tensor, and it adds the three things a deep learning library needs: a device to run on, an autograd system that records operations, and a library of layers and optimizers built on top.

The change looks small on the surface. The code `X @ W + b` is almost the same in both libraries. The differences are where bugs hide. A tensor created on the CPU and multiplied by a tensor on the GPU raises an error, a float64 tensor silently costs more memory and time than float32, and an in-place operation on a tensor that autograd needs produces a confusing error on Day 47. Learning to read a tensor's shape, dtype and device at a glance is the most useful habit you can form today.

On the job, most debugging of deep learning code starts with tensor metadata. Engineers print shapes and devices to find a broadcasting mistake, check dtypes when a model runs out of memory, and move tensors between devices when a data pipeline and a model disagree. The skill is routine, and it is also the first thing an interviewer will test.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1995 | The Python package Numeric, the ancestor of NumPy, is released by Jim Hugunin. | The array model that PyTorch borrowed for its tensor interface. |
| 2006 | NumPy is released by Travis Oliphant, consolidating the array library. | The standard in-memory array format that tensors interoperate with. |
| 2007 | NVIDIA releases CUDA, which lets general programs run on the GPU. | The reason deep learning moved from CPUs to GPUs. |
| 2016 | Torch, a Lua library, and TensorFlow are both widely used. Researchers begin to prefer dynamic graphs. | The motivation for PyTorch's eager execution. |
| 2017 | PyTorch is released by Facebook AI Research, with tensors that run on CPU and GPU and an autograd engine. | The framework and the tensor design used in this module. |

**Takeaway:** PyTorch's tensor is a NumPy-style array with a device attached. Most of the design choices you need today follow from that one fact.

---

## Glossary

- **Tensor**: a multi-dimensional array of numbers, the basic data type of PyTorch, which may live on a CPU or a GPU.
- **Shape**: the size of each dimension of a tensor, such as (32, 3, 224, 224) for a batch of 32 color images.
- **Rank or number of dimensions**: how many axes a tensor has. A vector has one, a matrix has two.
- **Dtype**: the numeric type of each element, such as float32, float16, int64 or bool.
- **Device**: the hardware where a tensor's memory lives, such as `cpu` or `cuda:0`.
- **Broadcasting**: the rule that lets operations combine tensors of different shapes by stretching the size-1 dimensions.
- **View**: a tensor that shares memory with another tensor but has a different shape or stride.
- **Contiguous**: a tensor whose elements are stored in memory in row-major order, without gaps.
- **In-place operation**: an operation that overwrites the memory of its input, named with a trailing underscore in PyTorch, such as `add_`.
- **Eager execution**: running operations immediately as Python reaches them, rather than building a static graph first.
- **Synchronization**: waiting until a GPU has finished its queued work, which matters when timing code.
- **Half precision**: the float16 or bfloat16 format, which uses 16 bits per number and is faster and smaller than float32 on suitable hardware.

---

## Tensors

Think of a tensor as a labeled box of numbers that knows its own shape and where it sits. A vector is a single row of boxes, a matrix is a grid, and a tensor is a stack of grids. The box can be moved to another room, which is the device, and the labels tell you how to index it, which is the shape.

From first principles, a PyTorch tensor is an object that holds a pointer to a block of memory, a shape, a stride and a dtype, and a device. The stride tells the library how many elements to skip to move one step along each dimension, which is how a view can reshape memory without copying it. You create tensors with `torch.tensor` from Python data, with `torch.zeros`, `torch.ones`, `torch.rand` or `torch.randn`, or with `torch.from_numpy` from a NumPy array.

```python
import numpy as np
import torch

a = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
b = torch.zeros(2, 3)
c = torch.from_numpy(np.arange(6, dtype=np.float32).reshape(2, 3))

print(a.shape, b.shape, c.shape)   # torch.Size([2, 2]) torch.Size([2, 3]) torch.Size([2, 3])
```

The limitation of `torch.from_numpy` is that the tensor shares memory with the array. Changing one changes the other, and that surprises people. Use `torch.tensor(array)` if you want an independent copy.

Do not treat a tensor as a Python list. A tensor is a fixed-shape block of memory, so operations are vectorized, and a Python loop over its elements is slow and usually wrong. In the real market, a training loop that spends most of its time in Python loops over tensor elements is one of the most common performance problems, and vectorizing the code often gives a large speedup.

---

## Shapes

Picture a shipping label that lists the number of boxes on each axis of a pallet. Reading the label before you stack the pallet prevents mistakes. Shapes do the same job for tensors, and most deep learning bugs are shape bugs.

From first principles, the shape is a tuple of integers, one per dimension. Matrix multiplication requires the inner dimensions to match: (32, 784) times (784, 128) gives (32, 128). Elementwise operations use broadcasting: a tensor of shape (128,) can be added to (32, 128), because the vector is stretched to match each row. Broadcasting aligns shapes from the right, and each pair of dimensions must be equal or one of them must be 1.

```python
import torch

X = torch.randn(32, 784)
W = torch.randn(784, 128)
b = torch.randn(128)

Z = X @ W + b                      # b broadcasts across the 32 rows
print(Z.shape)                     # torch.Size([32, 128])

x = torch.randn(4, 1)
y = torch.randn(1, 5)
print((x + y).shape)               # torch.Size([4, 5]) by broadcasting
```

The limitation of broadcasting is that it hides mistakes. A bias of shape (32, 1) added to a matrix of shape (32, 128) runs without error and gives a result with the wrong meaning. Assert the shapes you expect in critical places, and read the shape of every intermediate result when a model behaves strangely.

Confusing `view` with `reshape` is a common mistake. `view` returns a tensor that shares memory and requires the memory to be compatible with the new shape, so it can fail on a non-contiguous tensor. `reshape` does the same job and copies the data if it has to. Use `reshape` unless you need the memory sharing and you know the layout is compatible. In the real market, the error "view size is not compatible with input tensor's size and stride" is a common source of confusion, and calling `.contiguous()` before `.view()` is the usual fix.

---

## Devices

Consider a kitchen with two stations: a small counter and a large industrial stove. You can cook on either, but ingredients must be on the same station as the pan. A tensor and the operation that uses it must be on the same device. The CPU is the counter, and a GPU is the stove.

From first principles, PyTorch places each tensor on a device when it is created or moved. The device string `cpu` names the processor. The string `cuda` or `cuda:0` names the first NVIDIA GPU, which requires a CUDA-enabled build of PyTorch and a working driver. You check availability with `torch.cuda.is_available()`, and you choose a device once at the start of a program.

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
x = torch.randn(4, 3, device=device)
w = torch.randn(3, 2, device=device)
print(device, (x @ w).device)
```

The limitation is that the GPU is not always faster. For small tensors, the cost of launching work on the GPU and copying data outweighs the computation, so the CPU can win. For large matrix multiplications and convolutions, the GPU is far faster.

Do not mix devices in one operation, which raises a runtime error that names both devices. The fix is to move every input to the same device before the operation. A less obvious mistake is creating a new tensor inside a model's forward pass without specifying the device, which puts it on the CPU and triggers an error only when it meets a GPU tensor. In industry, teams run training on GPUs and serving on CPUs or specialized accelerators, so device handling is part of deployment, not only training.

---

## Dtypes

Think of a measuring cup that can be marked in different units. A cup in milliliters is precise and bulky, a cup in cups is compact and rougher. The dtype is the unit for numbers in a tensor. float32 uses 32 bits and is the usual default. float16 and bfloat16 use 16 bits, which halves the memory and can be much faster on modern GPUs, at the cost of precision. int64 holds integer indices, and bool holds masks.

From first principles, PyTorch infers the dtype from the Python values when you create a tensor, and it uses float32 as the default for floating-point data. Operations between dtypes follow promotion rules, so mixing float32 with int64 gives float32. The loss function and the labels have dtype requirements too: cross-entropy expects int64 class indices, and binary cross-entropy expects floating-point labels of the same shape as the predictions.

```python
import torch

x = torch.tensor([1, 2, 3])                  # int64 by default
y = torch.tensor([1.0, 2.0, 3.0])            # float32 by default
print(x.dtype, y.dtype)                      # torch.int64 torch.float32
print((x * y).dtype)                         # torch.float32
print(y.to(torch.float16).element_size())    # 2 bytes per element
```

The limitation of lower precision is that small gradients can underflow to zero, and large sums can overflow. Mixed-precision training keeps some values in float32 to avoid this, and you will meet it again when a model runs on a GPU with limited memory.

Do not pass float64 data to a model that expects float32. NumPy creates float64 arrays by default, and `torch.from_numpy` keeps that dtype, so the model either errors or silently runs slower. Convert the data to float32 at the edge of the pipeline, once, not inside the forward pass.

---

## Tensor operations

A library of tensor operations is like a set of kitchen tools. Some combine two ingredients, some reduce a whole bowl to one number, and some rearrange the bowl. Knowing which tool does which job makes the code short and fast.

From first principles, the operations fall into families. Elementwise operations apply a function to each element, such as `torch.relu` or `torch.exp`. Reductions collapse one or more dimensions, such as `x.sum(dim=0)` or `x.mean()`. Linear algebra operations include `@` for matrix multiplication and `torch.linalg` for decompositions. Indexing and slicing use the same syntax as NumPy. Operations ending in an underscore, such as `x.add_(1)`, modify the tensor in place. Autograd, which you meet tomorrow, cannot always track in-place changes to values it needs, so the rule is to avoid them unless you need the memory savings.

```python
import torch

x = torch.arange(12, dtype=torch.float32).reshape(3, 4)

col_means = x.mean(dim=0)           # shape (4,)
row_max = x.max(dim=1).values       # shape (3,)
mask = x > 5                        # bool tensor, shape (3, 4)
picked = x[mask]                    # 1D tensor of the selected values
scaled = torch.relu(x - 6.0)        # elementwise

print(col_means.shape, row_max.shape, picked.shape)
```

The limitation of reductions is that they discard information. Averaging over the batch loses per-example detail, and taking the maximum loses the other values. Check the dimension you reduce, because reducing the wrong axis often gives a plausible but meaningless result.

Do not use the `dim` argument carelessly. `x.sum(dim=0)` sums over rows and leaves columns, and `x.sum(dim=1)` does the opposite. Write a comment with the expected output shape next to every reduction until the habit is automatic. In the real market, reduction axis errors are among the hardest bugs to notice, because the code runs and the metrics move a little.

---

## CPU/GPU movement

Moving a tensor between devices is like shipping a box between warehouses. The transfer takes time, so you do it as few times as possible and keep the work where the data already sits. The `.to(device)` method performs the move and returns a tensor on the new device.

From first principles, a copy from CPU to GPU goes over the PCIe bus, which is far slower than GPU memory. GPU operations are asynchronous: the Python code returns before the GPU has finished, so timing code must call `torch.cuda.synchronize()` before reading the clock. A typical training loop moves each batch to the GPU once, computes everything there, and moves only the scalar loss back to the CPU for logging. Moving the loss at every step with `.item()` forces a synchronization and slows the loop.

```python
import time
import torch

def time_matmul(device, n=2048, repeats=10):
    x = torch.randn(n, n, device=device)
    y = torch.randn(n, n, device=device)
    if device.type == "cuda":
        torch.cuda.synchronize()
    start = time.perf_counter()
    for _ in range(repeats):
        z = x @ y
    if device.type == "cuda":
        torch.cuda.synchronize()
    return (time.perf_counter() - start) / repeats

cpu_time = time_matmul(torch.device("cpu"))
print(f"CPU: {cpu_time * 1000:.1f} ms per matmul")
if torch.cuda.is_available():
    gpu_time = time_matmul(torch.device("cuda"))
    print(f"GPU: {gpu_time * 1000:.1f} ms per matmul")
```

The limitation of benchmarking is that the first call includes warm-up costs, such as loading kernels, so the timing should discard one or more warm-up runs. Also, a result on one machine does not transfer to another.

Do not call `.cpu()` or `.item()` inside the inner loop of training. It forces the GPU to finish its queue and waits for the copy, and the training speed drops. Log every few hundred steps instead. In industry, the same pattern shows up in data loaders, where the CPU prepares batches and the GPU consumes them, and the goal is to keep both busy.

---

## Hands-on exercises

1. **Shape prediction.** Before running code, write the shape of `torch.randn(5, 1) * torch.randn(1, 4)`, `torch.randn(2, 3, 4).sum(dim=1)` and `torch.randn(2, 3, 4).reshape(6, -1)`. Then check your answers in Python.
2. **Broadcasting trap.** Add a bias of shape (32, 1) to a matrix of shape (32, 128). Confirm the code runs, and explain why the result is probably not what you meant. Then fix it with the correct bias shape.
3. **Dtype cost.** Create a tensor of one million float32 values and one of float64 values. Print `element_size()` and `numel()` for each, and compute the memory in megabytes.
4. **View versus reshape.** Transpose a 3 by 4 tensor, then call `.view(12)`, and read the error. Then call `.reshape(12)` and `.contiguous().view(12)`. Explain each result.
5. **Device check.** Write a function that takes a tensor and returns the same tensor on the GPU if available, or the CPU otherwise. Test it with a tensor of your choice and print the device.

---

## Project: tensor basics and the forward pass in PyTorch

**Goal:** install PyTorch 2.5.1, write a device and shape report, port your Day 39 forward pass to tensors, check that the two versions match, and benchmark the matrix multiplication on your hardware.

**Repo name:** `100-days-ai-day-46-pytorch-tensors`

**Repo layout:**

```
100-days-ai-day-46-pytorch-tensors/
├── README.md
├── requirements.txt
├── src/
│   └── tensor_lab/
│       ├── __init__.py
│       ├── report.py
│       ├── forward.py
│       └── devices.py
├── tests/
│   ├── test_forward.py
│   └── test_devices.py
└── experiments/
    └── benchmark_matmul.py
```

**Install PyTorch 2.5.1.** Create a virtual environment with Python 3.11 or 3.12 and activate it. For a CPU-only install on Windows, run `pip install torch==2.5.1 --index-url https://download.pytorch.org/whl/cpu`. For an NVIDIA GPU, run `pip install torch==2.5.1 --index-url https://download.pytorch.org/whl/cu121`, and make sure your driver supports CUDA 12.1 by running `nvidia-smi`. Then install the other packages with `pip install numpy==2.1.3 pytest==8.3.3`. Verify the install with `python -c "import torch; print(torch.__version__, torch.cuda.is_available())"`. The output should start with `2.5.1`.

**Starter code.** Create `requirements.txt` with the pinned versions. Create `src/tensor_lab/report.py`:

```python
import torch


def describe(t: torch.Tensor) -> str:
    return f"shape={tuple(t.shape)} dtype={t.dtype} device={t.device} requires_grad={t.requires_grad}"


def environment_report() -> str:
    lines = [
        f"torch {torch.__version__}",
        f"cuda available: {torch.cuda.is_available()}",
    ]
    if torch.cuda.is_available():
        lines.append(f"gpu: {torch.cuda.get_device_name(0)}")
    return "\n".join(lines)
```

Create `src/tensor_lab/devices.py` with a `pick_device()` function that returns a `torch.device` for the GPU if available and the CPU otherwise. Create `src/tensor_lab/forward.py`:

```python
import torch


def dense(X, W, b, activation=None):
    Z = X @ W + b
    return Z if activation is None else activation(Z)


def two_layer_forward(X, W1, b1, W2, b2):
    H = dense(X, W1, b1, torch.relu)
    return dense(H, W2, b2)
```

**Step-by-step tasks:**

1. Create the repo, initialize Git, make the virtual environment and install PyTorch 2.5.1 as described.
2. Write `src/tensor_lab/report.py` as above, then run `environment_report()` and paste the output into the README.
3. Write `tests/test_forward.py`. Create random NumPy arrays for a batch of 8 inputs with 4 features, a hidden layer of 6 units and an output of 2 units, using a seeded generator. Compute the forward pass in NumPy, convert the arrays to float32 tensors with `torch.tensor`, run `two_layer_forward`, and assert that the outputs match with `torch.allclose` and a tolerance of 1e-5. Include a shape assertion.
4. Write `tests/test_devices.py` to check that `pick_device()` returns a `torch.device` and that a tensor created on that device reports the same device.
5. Write `experiments/benchmark_matmul.py`. Time a 2048 by 2048 float32 matrix multiplication on the CPU and, if available, on the GPU. Discard the first three runs and synchronize before reading the clock on the GPU. Print the mean time for each device.
6. Create a float64 version of the benchmark and compare the time with float32. Record the ratio in the README.
7. Add a short section to the README that lists the device you used and explains any difference between your CPU and GPU result.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with the exact PyTorch install command for your hardware
- [ ] The environment report output
- [ ] How to run the tests and the benchmark
- [ ] The benchmark table with CPU and GPU times and the float32 to float64 ratio
- [ ] A note on the device used and the reason for any surprising result
- [ ] A list of the NumPy and PyTorch functions that map to each other in the forward pass

**Done when:**

- `python -c "import torch; print(torch.__version__)"` prints 2.5.1 in the project environment
- The forward pass test passes with the tolerance stated above
- The device test passes on your machine, whether it has a GPU or not
- The benchmark prints timings with warm-up discarded, and the README explains them
- The repo has at least four commits

**Evidence to commit to GitHub:** the source, the tests, the README with the environment report and the benchmark table, and a `NOTES.md` that lists one shape or dtype error you hit and how the error message pointed to it. The notes show that you can read tensor errors, which you will need on every day from here on.

**Why this project is practical:** every later day in this module starts with a tensor on a device. The report and the forward pass test are the setup check you will repeat in each new project.

---

## Check your understanding

1. A tensor of shape (64, 10) is multiplied by a tensor of shape (10, 5), then a bias of shape (5,) is added. What is the final shape, and what rule makes the bias addition valid?
2. Explain why `x.sum(dim=0)` and `x.sum(dim=1)` give different shapes for a matrix x, and which one you would use to compute the mean of each feature in a batch of examples stored as rows.
3. Your model runs on a GPU, but the loss is computed on a CPU tensor and the code raises a device error. What is the cause, and what is the minimal fix?
4. A teammate converts a dataset of images to float64 to save precision and then complains that training is slow. Explain the cost and suggest the right fix.
5. Why can a timing loop for GPU code report a time that is too short if it does not synchronize?
6. You need to flatten each image in a batch of shape (32, 3, 28, 28) into a vector of length 2352. Give the call you would write and explain why it does not change the data in memory when it works.

**Answers:**

1. The matrix product gives shape (64, 5). The bias of shape (5,) broadcasts across the 64 rows, because the trailing dimension matches and the leading dimension is absent, so it is treated as size 1 and stretched.
2. The first call sums over rows and leaves a shape with one entry per column. The second sums over columns and leaves one entry per row. For a batch with examples as rows, the mean of each feature is the mean over the rows, so `x.mean(dim=0)` is the right call.
3. The loss tensor is on the CPU while the predictions are on the GPU, so an operation between them is impossible. The minimal fix is to move the labels or the loss inputs to the same device as the predictions, typically with `.to(preds.device)`.
4. Float64 uses twice the memory of float32 and is much slower on most GPUs, so the training uses more bandwidth and less of the hardware's throughput. The right fix is to convert the data to float32 once, at the edge of the pipeline, and keep the model in float32 unless there is a specific numerical reason.
5. GPU work is asynchronous, so the Python call returns when the work is queued, not when it finishes. The timer then measures the queueing, not the computation. Synchronizing waits for the queued work to finish, so the elapsed time reflects the real cost.
6. `x.reshape(32, -1)` or `x.reshape(32, 2352)` works. When the tensor is contiguous, `reshape` returns a view that shares the same memory with a different shape, so no data is copied. If the tensor were not contiguous, `reshape` would copy the data.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Tensor (machine learning)", for the definition of a tensor in the context of deep learning
- Wikipedia, "NumPy", for the array library whose interface PyTorch borrows
- Wikipedia, "CUDA", for the GPU programming model
- Wikipedia, "Half-precision floating-point format", for the float16 format and its range
- Wikipedia, "Bfloat16 floating-point format", for the format used in many modern accelerators
- PyTorch documentation, "Tensor Attributes", at pytorch.org/docs/stable, which lists dtype, device and shape properties

**Tutorials**

- PyTorch Tutorials, "Tensors", at pytorch.org/tutorials/beginner/basics/tensorqs_tutorial.html
- PyTorch documentation, "Broadcasting semantics", at pytorch.org/docs/stable, for the exact rules
- NumPy documentation, "Broadcasting", at numpy.org/doc/stable, for the same rules in NumPy, which match PyTorch in most cases
- PyTorch Tutorials, "Accelerating PyTorch with CUDA" or the equivalent current page, for the device and synchronization details

**Watch**

- The PyTorch full course on freeCodeCamp, found by searching YouTube for "freeCodeCamp PyTorch full course", for the tensor basics in a longer context
- The official PyTorch channel on YouTube, for short talks on tensors and devices

**Practice**

- The PyTorch tutorial "Tensors" with its exercises, run in a local notebook. Repeat the operations in the lesson on your machine and compare the shapes you predicted with the output.

---

## Tomorrow

Day 47 is about autograd, the part of PyTorch that records the operations on tensors and computes their gradients automatically. You will see how `requires_grad` marks a tensor for tracking, how `backward` fills in `.grad`, why gradients accumulate, and how to stop tracking with `detach` and `torch.no_grad`. The scalar engine you built on Day 43 is the same idea at a larger scale, and you will check PyTorch's gradients against the ones you computed by hand.
