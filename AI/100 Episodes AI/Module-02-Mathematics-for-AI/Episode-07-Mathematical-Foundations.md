# Day 7: Mathematical Foundations

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** beginner. You need high-school algebra and some comfort with Python. If you have forgotten what an exponent is, this day rebuilds it from the ground up.

**By the end of this day, you will have:**

- A one-page notation reference you wrote yourself, with each symbol tied to a small example
- A Python script that evaluates functions, logarithms and sums, and checks the identities that the rest of the module depends on
- A set of plots that show how functions, exponents and logarithms bend, saved as image files in the repo

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 2.0 or newer (`pip install numpy`)
- Matplotlib 3.8 or newer (`pip install matplotlib`), used for the plots
- A code editor such as VS Code (free) or any editor you already use
- Git and a free GitHub account for the project section
- Paper and a pencil for the exercises

**Prerequisites:**

- Days 1 to 6 of this series
- Basic Python: variables, functions, lists, `for` loops and `import`
- Algebra at the level of solving `2x + 3 = 11`

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Variables](#variables)
- [Functions](#functions)
- [Equations](#equations)
- [Exponents](#exponents)
- [Logarithms](#logarithms)
- [Summation notation](#summation-notation)
- [Function composition](#function-composition)
- [Graphs](#graphs)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a notation checker and a plotting lab](#project-a-notation-checker-and-a-plotting-lab)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Almost every model in this series is a function with a large number of adjustable numbers inside it. A neural network takes numbers in, multiplies them by weights, adds biases, applies a function and passes numbers on. The formula that scores how wrong a prediction is, the rule that decides how far to move each weight, and the rule that turns raw scores into probabilities are all written in the algebra of this day.

You do not need to be a mathematician to read these formulas. You do need to read one without freezing, work out what it computes for a specific input, and check that your code computes the same thing. That skill is what this day builds.

On the job, the skill shows up in two places. The first is code review: a teammate writes `exp(x) / sum(exp(x))` in a pull request, and you must see that it is a softmax and that it needs a numerical guard to avoid overflow. The second is debugging: a model that trains without errors but reaches a poor score is often a formula with a missing parenthesis or a logarithm taken of the wrong quantity. Reading notation fluently is how you catch those bugs before they cost a week.

---

## Glossary

- **Variable**: a symbol, usually a letter, that stands for a value that can change or is not yet known.
- **Constant**: a fixed value that does not change inside a particular problem, such as 3 or pi.
- **Function**: a rule that takes an input and returns exactly one output.
- **Domain**: the set of inputs a function accepts.
- **Range**: the set of outputs a function can produce.
- **Equation**: two expressions joined by an equals sign, which is true only for some values of the variables.
- **Exponent**: the small number written above a base that says how many times to multiply the base by itself.
- **Logarithm**: the exponent you must raise a base to in order to get a given number.
- **Summation**: adding a sequence of terms, written with the Greek capital letter sigma.
- **Index**: the variable under a summation sign that counts through the terms.
- **Composition**: applying one function to the output of another.
- **Graph**: the picture of all input-output pairs of a function, drawn on axes.
- **Axis**: one of the reference lines of a graph. The horizontal one is usually the x-axis and the vertical one the y-axis.

---

### Variables

Think of a variable as a labeled box. You write `x` on the box and put a value inside. Later, you can take the value out, replace it with another value, or write a formula that refers to the box rather than to a particular number. A recipe that says "add x grams of flour" works for any amount of flour, and `x` is the box that holds the amount you actually use.

In math, a variable is a name bound to a value within a context. In `y = 2x + 1`, `x` is the input you choose and `y` is the output the formula produces. When you substitute `x = 4`, you get `y = 9`. The equals sign here assigns meaning, so it sets up a definition rather than stating something already true.

In Python, a variable is a name in memory that refers to an object:

```python
x = 4
y = 2 * x + 1
print(y)  # 9
```

A common mistake is treating the same letter as two different things in one problem. In machine learning papers, `x` may mean a single feature in one line and a whole dataset in the next. When you read a formula, first write down what each symbol stands for in this specific problem, including its shape if it is a list of numbers. Variables appear in every training loop, where the weights are variables that the optimizer updates on each step. Frameworks such as PyTorch and TensorFlow call these variables parameters or trainable tensors.

### Functions

Picture a vending machine. You put in a coin of a given value, you press a button, and you receive one specific item. If you press the same button with the same coin, you always get the same item. That reliability is what makes it a function. A function takes an input, applies a fixed rule, and gives back exactly one output.

Formally, a function `f` maps each element of its domain to exactly one element of its range. The notation `f(x) = 3x^2 + 2x + 1` says that for every input `x`, the output is three times `x` squared, plus two times `x`, plus one. Two inputs may share an output, as `f(-1)` and `f(1)` do in `x^2`, but a single input cannot have two outputs.

In code, a function is a named block that returns a value:

```python
def f(x):
    return 3 * x**2 + 2 * x + 1

print(f(2))  # 3*4 + 2*2 + 1 = 17
print(f(0))  # 1
```

A common mistake is writing a function that is secretly not a function, for example one that depends on a global variable that changes between calls. The output then depends on something other than the input, and your tests will pass or fail at random. Keep the rule inside the function and pass everything it needs as an argument. On the job, the loss function in a training pipeline is written this way so that the same batch of predictions always produces the same loss, which makes experiments reproducible.

### Equations

An equation is a balance scale. Both sides must weigh the same, and when you add or remove weight on one side, you must do the same to the other. Solving an equation means finding the values of the unknown that keep the scale level.

An equation states that two expressions have equal values. The equation `2x + 3 = 11` is true only when `x = 4`. You find that value by undoing operations in reverse order: subtract 3 from both sides to get `2x = 8`, then divide both sides by 2 to get `x = 4`. Equations with two unknowns, such as `x + y = 10` and `x - y = 2`, have a single solution pair when the two equations are independent. Day 9 returns to systems of equations and writes them in matrix form.

NumPy can check a solution without any manual algebra:

```python
import numpy as np

x = 4
print(2 * x + 3 == 11)  # True
```

Solving the equation without checking it is a common mistake. Substituting the answer back into the original equation takes seconds and catches sign errors and division by zero. The other trap is confusing an equation with an assignment. In Python, `x = x + 1` is an update, while in mathematics `x = x + 1` is a contradiction. Equations describe the world, and assignments change it. In industry, the normal equation for linear regression, which gives the best weights in closed form, is an equation you solve with linear algebra in libraries such as scikit-learn, and the same idea appears in the closed-form solution for least squares.

### Exponents

Compound interest is the everyday example. Put 100 dollars in an account that pays 10 percent a year. After one year you have 110. After two years you have 121, because the second year's interest is paid on the first year's interest too. After ten years you have 100 times 1.1 multiplied by itself ten times. The exponent is the number of times you multiply.

Formally, `b^n` means multiply the base `b` by itself `n` times. Several rules follow directly. `b^m * b^n = b^(m+n)`, because you are multiplying one stack of factors by another. `(b^m)^n = b^(m*n)`. `b^0 = 1` for any nonzero `b`, and `b^(-n) = 1 / b^n`. A fractional exponent is a root: `b^(1/2)` is the square root of `b`, and `8^(1/3)` is the cube root of 8.

```python
print(2 ** 10)        # 1024
print(10 ** -2)       # 0.01
print(8 ** (1 / 3))   # 2.0
```

A common mistake is writing `(a + b)^2` as `a^2 + b^2`. The square of a sum includes a cross term, `a^2 + 2ab + b^2`. Exponents also appear in the activation functions used by neural networks, in the softmax function and in the normal distribution, so getting them wrong quietly changes the model. Large exponents create a practical problem too: `np.exp(1000)` overflows to infinity in double-precision floating point, which is why libraries subtract the maximum value before applying `exp` when they compute softmax.

### Logarithms

A logarithm answers the question "how many times must I multiply by the base to get this number?" The base-10 logarithm of 1000 is 3, because 10 multiplied by itself three times is 1000. The logarithm is the inverse of the exponential. If `b^y = x`, then `log_b(x) = y`.

Logarithms turn multiplication into addition. `log(a * b) = log(a) + log(b)`, because adding exponents corresponds to multiplying the bases. They also turn huge ranges into small ones. The difference between 1 and 1,000,000 is six orders of magnitude, and the base-10 logarithm turns that range into the interval from 0 to 6, which is easy to plot and to compare.

```python
import math

print(round(math.log(1000, 10), 6))   # 3.0
print(round(math.log(8, 2), 6))       # 3.0
print(round(math.log(6 * 7), 6))      # the product rule
print(round(math.log(6) + math.log(7), 6))
```

The natural logarithm, written `ln` in mathematics and `math.log` in Python, uses the base e, which is about 2.718. Most machine learning code uses the natural logarithm because it has the simplest derivative. A common mistake is that `math.log(1000, 10)` returns `2.9999999999999996` rather than exactly 3, because floating-point arithmetic cannot represent every real number. Always round when you compare results, or use `math.isclose`. Another mistake is taking the logarithm of zero or of a negative number, which is undefined for real numbers. In the wild, the cross-entropy loss used to train classifiers and language models takes a logarithm of a predicted probability, and a probability of exactly zero makes the loss infinite, so libraries clip probabilities to a small minimum.

### Summation notation

Suppose you want the total of a list of numbers. You could write the list out in full, or you could use a compact rule. The symbol sigma, written as a large capital Greek S, means "add up the terms," and the expression under it tells you which terms to add. A sum of squares looks like this in mathematical notation:

`sum from i = 1 to 4 of i^2`

This means `1^2 + 2^2 + 3^2 + 4^2`, which equals 30. The variable `i` is the index. The lower and upper limits say where counting starts and stops.

A sum is a loop with an accumulator. Each term is computed from the index, and the results are added to a running total. Two properties matter for AI work. Constants can move outside the sum, so `sum of c * x_i` equals `c` times `sum of x_i`. And the sum of a sum splits into two sums, which is why a loss over a dataset can be written as a sum over examples, one term per example.

```python
import numpy as np

print(sum(i**2 for i in range(1, 5)))  # 30
print(np.sum(np.arange(1, 5) ** 2))    # 30
```

Off-by-one errors in the limits are a common mistake. The summation `sum from i = 1 to 4` has four terms, but `range(1, 4)` in Python gives only three, because Python excludes the stop value. Python ranges are half-open, and mathematical sums are closed. The second mistake is treating the index as a value you can use after the sum has finished. The index is local to the summation. In the real market, mean squared error over a batch is a sum divided by the number of examples, and the loss that a training loop logs is usually this sum computed in code.

### Function composition

A factory line is a good picture of composition. The first machine cuts a piece of metal, and the second machine bends it. The bending machine does not care how the piece was cut, only that it receives a piece. Composition chains two functions so the output of one becomes the input of the next.

If `g(x) = x + 1` and `h(x) = x^2`, then the composition `h(g(x))` means first apply `g`, then apply `h`. The result is `(x + 1)^2`. The composition in the other order, `g(h(x))`, is `x^2 + 1`. Composition is generally not commutative, so the order of the functions changes the result. Neural networks are deep because they compose many functions: each layer is a function, and the network is the layer applied to the output of the layer before it.

```python
def g(x):
    return x + 1

def h(x):
    return x ** 2

print(h(g(3)))  # (3 + 1)^2 = 16
print(g(h(3)))  # 3^2 + 1 = 10
```

A common mistake is reading `f(g(x))` from right to left in one place and left to right in another. Always evaluate the innermost function first, then the next one out. Another mistake is assuming the output type of one function matches the input type of the next. Composing a function that returns a list with one that expects a number fails at runtime, and in PyTorch it fails with a shape error. On the job, a data pipeline is composition: load, clean, transform, featurize, predict. Each stage is a function, and the bugs usually come from a stage returning something the next stage did not expect.

### Graphs

A graph is a map of a function. Each input value has a location on a horizontal line and each output value has a location on a vertical line, and the function's pairs become points on a plane. Looking at the points tells you things that a formula hides, such as where a function rises, where it flattens and where it crosses zero.

A graph of `f` is the set of all pairs `(x, f(x))` for `x` in the domain. You draw it on two perpendicular axes. The x-axis is horizontal and holds the input. The y-axis is vertical and holds the output. A straight line has the form `y = m x + b`, where `m` is the slope and `b` is the intercept. A parabola like `y = x^2` is symmetric around the vertical axis. An exponential such as `y = 2^x` grows faster and faster, and a logarithm such as `y = log2(x)` grows slowly.

```python
import numpy as np
import matplotlib.pyplot as plt

x = np.linspace(0.1, 8, 200)
plt.plot(x, 2 ** x, label="2^x")
plt.plot(x, np.log2(x), label="log2(x)")
plt.plot(x, x ** 2, label="x^2")
plt.legend()
plt.savefig("growth_curves.png")
```

Reading a graph without checking the scales is a common mistake. A curve that looks steep on a linear axis can be a straight line on a logarithmic axis. Plot the same data both ways when you compare models, and always label your axes. Another mistake is sampling a function too coarsely: plotting `x^2` with only three points gives a misleading shape. In the real market, dashboards that track training loss over time are graphs of a function of the training step, and reading them correctly is how engineers decide whether to stop a run, lower the learning rate or keep going.

---

## Hands-on exercises

Work these on paper first, then check them in Python.

1. Evaluate `f(x) = 3x^2 + 2x + 1` by hand for `x = -2`, `x = 0` and `x = 5`. Write each step. Then write a Python function with the same formula and confirm that it returns the same three values.
2. Solve `5x - 7 = 3x + 9` by hand. Substitute your answer into both sides and confirm they match. Repeat with `(x + 2) / 4 = 3`.
3. Simplify `2^3 * 2^4`, `(2^3)^2` and `2^(-2)` without a calculator. Then check each with `2 ** 3 * 2 ** 4`, `(2 ** 3) ** 2` and `2 ** -2` in Python.
4. Compute `log2(64)`, `log10(0.001)` and `ln(e^3)` by hand. Verify with `math.log2(64)`, `math.log10(0.001)` and `math.log(math.e ** 3)`. Round each result before you compare.
5. Compute `sum from i = 1 to 5 of (2i + 1)` by hand, then verify with `sum(2 * i + 1 for i in range(1, 6))`. Explain in one sentence why the Python range stops at 5, not 6.
6. With `g(x) = 2x` and `h(x) = x + 3`, compute `g(h(4))` and `h(g(4))` by hand. Verify with Python and explain why the answers differ.
7. Using the earlier graph code, create `growth_curves.png` and open it. Find the x value where `2^x` first exceeds `x^2`. Hint: test integer values of x in Python before you look at the picture.

Check that each answer is correct before you move on. Write any mistakes in your notes with the reason you made them. Keep the reasons next to the answers.

---

## Project: a notation checker and a plotting lab

This project is light on purpose. Today's topic is vocabulary and a set of rules you must recognize, not a full system to build. A focused script and a short write-up are enough to prove that you can read the notation and test it. The heavier implementation work starts on Day 8 and reaches its peak on Day 16.

**Repo name:** `100-days-ai-engineering-day-07-mathematical-foundations`

**Goal:** write a short Python script that verifies the algebraic identities from today, plot the functions they describe and commit the evidence with a README that explains each result.

**Repo layout:**

```
100-days-ai-engineering-day-07-mathematical-foundations/
├── README.md
├── requirements.txt
├── identities.py
├── plot_functions.py
└── results/
    └── growth_curves.png
```

**Starter code for `identities.py`:**

```python
import math
import numpy as np


def check(name, left, right, tol=1e-9):
    """Print whether two values match within a tolerance."""
    ok = math.isclose(left, right, abs_tol=tol)
    print(f"{'PASS' if ok else 'FAIL'}  {name}: {left} vs {right}")
    return ok


def main():
    results = []
    # Exponent rule: b^m * b^n = b^(m + n)
    results.append(check("exponent product", 2**3 * 2**4, 2**7))
    # Logarithm product rule: log(a*b) = log(a) + log(b)
    results.append(check("log product", math.log(6 * 7), math.log(6) + math.log(7)))
    # TODO: add a check for the summation of i^2 from 1 to 4 (expect 30)
    # TODO: add a check for composition order: h(g(3)) != g(h(3))
    print(f"{sum(results)} of {len(results)} checks passed")


if __name__ == "__main__":
    main()
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add a `requirements.txt` that lists `numpy` and `matplotlib`.
2. Run `identities.py`. Confirm that the two starter checks pass.
3. Add the two TODO checks. The summation check should compare the Python sum with `np.sum` for the same range, and the composition check should assert that the two orders give different results.
4. Add three more checks of your own from the exercises, for example a power rule, a logarithm base change or a quadratic evaluated at a point.
5. Write `plot_functions.py` so that it saves `results/growth_curves.png` containing `2^x`, `x^2` and `log2(x)` on the same axes, with a legend and labeled axes.
6. In the README, write one paragraph for each check that says what it tests and why the result is expected.
7. Commit the files and push to GitHub.

**README checklist:**

- [ ] The project goal is written in one sentence
- [ ] The commands to install and run are listed (`pip install -r requirements.txt`, `python identities.py`, `python plot_functions.py`)
- [ ] Every check has a sentence that says what it tests
- [ ] The plot is embedded with a caption that says what it shows
- [ ] One common mistake from this day is named, with the check that would catch it

**Done when:**

- `python identities.py` prints a pass line for every check and the summary says all checks passed
- The composition check demonstrates that order matters
- `results/growth_curves.png` exists, has labeled axes and a legend
- The README explains each check in plain language
- The repo is public and the last commit message describes the day's work

**What to commit as evidence:** the script, the plot image, the README and the terminal output of the final run pasted into the README under a "Results" heading.

---

## Check your understanding

1. A training script computes `math.log(p)` for predicted probabilities `p`. One value in the batch is exactly 0. What happens to the loss, and which part of today's logarithm section explains why?
2. The formula `(a + b)^2` appears in a teammate's code as `a**2 + b**2`. Give a numerical example that shows the two expressions differ, and explain what the missing term represents.
3. Two functions `g` and `h` are composed as `h(g(x))`. You swap the order and the model's output changes. Explain why composition is not commutative, and describe a real pipeline where the order matters.
4. A summation in a paper runs from `i = 1` to `n`, and the code uses `range(1, n)`. Predict the size of the error for `n = 10`, and explain how you would catch this kind of bug with a test.
5. A plot of loss against training step looks like a straight line on a log scale and a steep curve on a linear scale. What does the log scale reveal, and why might a training team choose to plot it that way?
6. You are asked whether the equation `x = x + 1` has a solution. Explain how the answer differs between mathematics and Python, and why the difference causes bugs when people transfer formulas into code.
7. A function `f(x) = 2^x` is evaluated for `x = 2000` in code and returns infinity. Name the underlying cause and describe one fix a library would use for a related computation such as softmax.

### Answers

1. The loss becomes infinite (or `math.log(0)` raises an error, depending on the library), because the logarithm of zero is undefined and tends to negative infinity. The section on logarithms explains that the domain is positive numbers, and that libraries clip probabilities to a small minimum before taking the log.
2. For `a = 1` and `b = 2`, `(a + b)^2` is 9, while `a^2 + b^2` is 5. The missing term is `2ab`, the cross term that comes from multiplying out the square of a sum.
3. The output of `g` feeds into `h` in one case and `h` feeds into `g` in the other, so the inner and outer functions change roles. A pipeline that normalizes features and then applies a model gives different results from one that applies the model first, so the order is part of the design.
4. The sum for `n = 10` has 10 terms, but `range(1, 10)` gives 9, so the code omits the last term, `10^2 = 100` if the term is `i^2`. A test that compares the code to a hand-computed sum for a small `n`, such as `n = 4` with the expected answer 30, catches the error.
5. A straight line on a log scale means the quantity falls by a constant factor per step, which is an exponential decay. The log scale makes the rate of decrease visible over many orders of magnitude, which a linear scale hides at the end of training.
6. In mathematics, no number equals its own successor plus one, so there is no solution. In Python, `x = x + 1` is a statement that rebinds the name `x` to a new value. Engineers who copy formulas literally into code can misread assignments as equations, and the reverse mistake also happens.
7. `2^2000` is far larger than the largest number a 64-bit floating-point value can hold, so it overflows to infinity. Softmax implementations subtract the maximum input before exponentiating, which keeps every exponent at or below zero and avoids overflow.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Function (mathematics)](https://en.wikipedia.org/wiki/Function_(mathematics))
- Wikipedia: [Variable (mathematics)](https://en.wikipedia.org/wiki/Variable_(mathematics))
- Wikipedia: [Equation](https://en.wikipedia.org/wiki/Equation)
- Wikipedia: [Exponentiation](https://en.wikipedia.org/wiki/Exponentiation)
- Wikipedia: [Logarithm](https://en.wikipedia.org/wiki/Logarithm)
- Wikipedia: [Summation](https://en.wikipedia.org/wiki/Summation)
- Wikipedia: [Function composition](https://en.wikipedia.org/wiki/Function_composition)
- Wikipedia: [Graph of a function](https://en.wikipedia.org/wiki/Graph_of_a_function)

**Tutorials**

- Khan Academy, "Algebra" and "Precalculus" courses, sections on functions, exponents and logarithms. Search the title on khanacademy.org.
- GeeksforGeeks, search "Python math module" for a walkthrough of `math.log` and related functions that complements the official reference.

**Watch**

- 3Blue1Brown (YouTube), "But what is a neural network?" from the neural networks series. It shows a function with many inputs and parameters, which is today's idea at full scale.
- MIT OpenCourseWare, "18.01 Single Variable Calculus" (Prof. David Jerison). Search the course on ocw.mit.edu and watch the early lectures on functions and graphs.

**Practice**

- Python documentation, [math module](https://docs.python.org/3/library/math.html). Read the entries for `log`, `exp`, `isclose` and `fsum`, and run each one in a REPL.

---

## Tomorrow

Tomorrow moves from single numbers to vectors, which are ordered lists of numbers that describe a point, a direction or a feature set. Everything you wrote today, the functions, exponents and sums, reappears as operations on vectors. You will implement addition, scaling, magnitude and the dot product with NumPy, and you will see why the dot product measures how closely two feature vectors point in the same direction.

**Next:** [Day 8: Vectors](Day-08-Vectors.md)
