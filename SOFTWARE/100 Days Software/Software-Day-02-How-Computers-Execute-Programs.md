# Day 2: How Computers Execute Programs

*100 Days of Software Engineering, Module: Foundations*

**Time:** about 2 to 3 hours
**You will do:** trace one feature from your Day 1 app through the machine, on paper, and run short experiments in a browser and Windows Task Manager. This is a conceptual day, so there is no repo. Your trace goes in a notes file.
**You should know:** Day 1 (the SDLC and your app idea). You need a web browser and Task Manager. Nothing gets installed today.

---

## 1. The big idea

When you click Save, your text becomes numbers. Those numbers sit in memory, the processor runs a long list of tiny instructions on them, and the result is written to a disk that keeps it after the power goes off. Most bugs in this series sit somewhere on that path. A slow page is often data moving between memory and disk. A crash is often a program touching memory it does not own. A wrong total is often a number the machine cannot store exactly.

Engineers who can picture this path debug faster and explain problems in concrete terms. Today you build that picture from the ground up.

---

## 2. From source code to a running program

### 2.1 Source code

**Intuition.** Source code is a recipe written for people. The CPU cannot read it, just as an oven cannot read English.

**Definition.** Source code is plain text in a programming language, saved in files such as `app.py` or `main.c`. People read, change and review it. Before the CPU can run it, a tool has to translate it into machine code, either ahead of time or while the program runs.

**Worked example.** The same function in two languages:

```python
# square.py
def square(x):
    return x * x

print(square(7))
```

```c
/* square.c */
#include <stdio.h>

int square(int x) {
    return x * x;
}

int main(void) {
    printf("%d\n", square(7));
    return 0;
}
```

Both print `49`. The C version declares that `x` is an `int`. The Python version leaves the type open and works it out while the program runs. That difference shapes how each language gets translated.

**Mistakes.** Source code is text, so invisible characters matter. A word processor adds formatting and curly quotes that break code, so use a code editor (Day 3). Windows and Linux also end lines differently, which causes trouble on Day 5.

### 2.2 Compilation

**Intuition.** A translator converts a whole book before anyone reads it. Reading the translation is fast, but every new edition needs a new translation.

**Definition.** A **compiler** translates source code into another form before the program runs. For C, the usual steps are: preprocess (expand `#include`), compile to assembly, assemble into machine code in an object file, then **link** the pieces with libraries into one executable. The compiler also checks the code. It refuses to build when types do not match.

**Worked example.** Open [Compiler Explorer](https://godbolt.org/), choose C and an x86-64 compiler such as gcc or clang, add `-O2` to the options box, and paste the `square` function. The output pane shows something like this (exact output varies by compiler and version):

```
square:
        mov     eax, edi
        imul    eax, edi
        ret
```

The argument arrives in register `edi`, gets multiplied, and the result goes back in `eax`. Three instructions for one line of C. Change `x * x` to `x * x + 1` and one more instruction appears.

**Mistakes.** A compiled program targets one CPU type and one OS. A Windows `.exe` will not run on a Linux ARM server without a rebuild. Also, "it compiles" does not mean "it is correct." The compiler catches type errors, not wrong logic.

### 2.3 Interpretation

**Intuition.** A live interpreter at a meeting translates each sentence as it is spoken. Nothing is prepared in advance, so you can change what you say at any moment, but every sentence costs translation time.

**Definition.** An **interpreter** reads code and carries it out while the program runs. Most "interpreted" languages mix the two approaches. CPython, the standard Python, compiles your source to **bytecode** (a compact instruction format), then a virtual machine inside the interpreter runs that bytecode one instruction at a time. JavaScript engines such as V8 start by interpreting, then **JIT-compile** frequently used functions to machine code while the program runs. Java compiles to bytecode ahead of time, and the JVM JIT-compiles it at run time.

**Worked example.** Python includes `dis`, a tool that prints bytecode. You can paste this into an online Python runner:

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
```

On Python 3.12 the output includes these lines:

```
LOAD_FAST                0 (a)
LOAD_FAST                1 (b)
BINARY_OP                0 (+)
RETURN_VALUE
```

Python 3.13 combines some of these steps (it shows `LOAD_FAST_LOAD_FAST`). Bytecode is an internal detail, and it changes between releases. Each bytecode instruction runs as many machine instructions inside the interpreter, which is a big reason pure Python loops run slower than the same loop in C.

**Mistakes.** "Python has no compile step" is false. Python compiles to bytecode, and you will see `__pycache__` folders holding `.pyc` files. Interpretation also defers errors to run time. An error on a line that rarely runs can sit unnoticed for months.

---

## 3. Inside the machine

Everything in this section fills in one picture:

```
  Your source code  (app.py, main.c)            lives on STORAGE (SSD)
          |
          |  compile and/or interpret
          v
  Instructions the CPU understands              loaded into MEMORY (RAM)
          |
          |  the OS starts a PROCESS
          v
  +-------------------- Process (PID 4120) ----------------------+
  |  code  |  data  |  heap (grows up)   ...   stack (grows down) |
  +------------------------------------------------------------+
          |
          |  the CPU fetches, decodes and executes, billions of times a second
          v
  Results: pixels on screen, bytes written to storage, packets sent
```

### 3.1 The CPU

**Intuition.** Picture a fast, literal cook with a recipe card. The cook reads one line, does exactly that, then reads the next. The cook never improvises and never skips ahead unless a line says to.

**Definition.** The CPU repeats a **fetch, decode, execute** cycle. It fetches the next instruction from memory (the address sits in a register called the program counter), decodes what the instruction means, executes it with its arithmetic logic unit or by moving data, then moves on. Each instruction is tiny: add two numbers, compare two values, copy a value from memory into a register, or jump to another address.

**Worked example.** A 3 GHz core runs 3 billion cycles per second. At 2 instructions per cycle, that is about 6 billion instructions per second. The statement `total = price * quantity` might become three instructions: load `price` into a register, multiply by `quantity`, store the result in `total`. The core could run that two billion times in a second, but only if the data is already close by. The next section explains why that "if" matters.

**Mistakes.** Beginners assume a faster clock makes a faster program. Often it does not, because the CPU spends most of its time waiting for data from memory. Also, extra cores help only when the program splits its work across them. Many scripts use exactly one core.

### 3.2 Memory

**Intuition.** RAM is your desk. Storage is the filing cabinet across the room. You can only work on papers that are on the desk, and the desk is cleared every night.

**Definition.** RAM is a long row of bytes, each with a numeric address. The CPU reads and writes RAM by address. RAM is volatile: when power goes off, the contents go with it. RAM is also much slower than the CPU, so chips add layers of **cache** (L1, L2, L3) that hold recently used data. That stack of faster, smaller layers is the **memory hierarchy**.

**Worked example.** These are rough, commonly quoted orders of magnitude from the "Latency Numbers Every Programmer Should Know" list. Real hardware varies.

| Where the data is | Approximate time to reach it | If 1 ns were 1 second |
|---|---|---|
| CPU register | under 1 ns | instant |
| L1 cache | about 1 ns | 1 second |
| Main memory (RAM) | about 100 ns | almost 2 minutes |
| SSD random read | about 100,000 ns (100 µs) | about 1 day |
| Hard disk seek | about 10,000,000 ns (10 ms) | about 4 months |

The table explains why reading a file into memory once is usually faster than reading it from disk line by line many times.

**Mistakes.** Running out of RAM does not always crash a program right away. The OS starts moving memory to disk (paging), and the program slows to a crawl. If Task Manager shows memory near 100% and your laptop crawls, paging is the likely cause. Also, do not treat memory as unlimited. Loading a 10 GB log into a script on an 8 GB laptop will fail.

### 3.3 Storage

**Intuition.** The filing cabinet. It is slow to walk to, but it keeps everything overnight.

**Definition.** Storage is non-volatile: it keeps data without power. Solid-state drives (SSDs) use flash memory and have no moving parts. Hard disk drives (HDDs) use spinning platters and a moving head, which is why their access time is so much worse. The **file system** (NTFS on Windows, ext4 on most Linux systems) turns raw storage blocks into named files and folders. A program such as `notepad.exe` lives on storage. To run it, the OS copies its instructions into RAM.

**Worked example.** In File Explorer, right-click any file and choose **Properties**. Compare "Size" with "Size on disk." A 1-byte text file often shows about 4 KB on disk, because the file system allocates space in fixed blocks (clusters), commonly 4 KB on NTFS.

**Mistakes.** Data in memory is lost unless it reaches storage. Apps that crash before saving lose work, which is why databases use a write-ahead log and flush to disk in a careful order. SSDs also wear out with many writes, so a loop that rewrites a tiny file thousands of times costs more than speed.

### 3.4 Processes

**Intuition.** A recipe in a book is a program. Cooking it right now, with your own pans and ingredients, is a process. Two cooks can follow the same recipe at the same time in separate kitchens.

**Definition.** A **process** is a running instance of a program. The OS gives each one a PID, a private **virtual address space** (so one process cannot read another's memory by accident), and a share of CPU time. The OS **scheduler** switches between processes so fast that they seem to run at once, even on one core. A process can contain several **threads**, which share its memory but run separately.

**Worked example.** Open Notepad twice. Open Task Manager (Ctrl+Shift+Esc), go to the **Details** tab, and find two `Notepad.exe` rows with different PIDs. One program file, two processes. Close one, and the other keeps running. PowerShell shows the same thing:

```powershell
Get-Process notepad
```

**Mistakes.** People say "the app" when they mean several processes. Chrome runs many processes, so "Chrome uses 3 GB" is a sum across them. Also, two processes writing the same file at once can corrupt it. Private memory does not protect shared storage.

---

## 4. Stack and heap

**Intuition.** The stack is a pile of sticky notes on your desk. Each time a task starts a subtask, you add a note on top. When the subtask finishes, you remove that note. The heap is a storage room. You put a box there when you do not know how long you need it, and someone has to clear it out later.

**Definition.** Each thread has a **call stack**. Calling a function pushes a **stack frame** that holds its arguments, its local variables and the address to return to. Returning pops the frame. This is fast and automatic, but the stack is small: by default about 8 MB per thread on Linux and 1 MB on Windows. The **heap** is a larger region for data that must outlive one function call, or whose size is known only at run time. In C you request heap memory with `malloc` and release it with `free`. In Python and JavaScript, objects live on the heap, and a **garbage collector** frees them once nothing refers to them.

**Worked example.** Paste this into [Python Tutor](https://pythontutor.com/) and click **Next** through each step:

```python
def factorial(n):
    if n == 0:
        return 1
    return n * factorial(n - 1)

result = factorial(3)
print(result)
```

At the deepest point, the stack holds four frames: `factorial(3)`, `factorial(2)`, `factorial(1)` and `factorial(0)`. Then they pop off one by one, and `result` becomes 6.

```
  call factorial(3)            returns
  +--------------+
  | factorial n=0|  top        returns 1
  | factorial n=1|             returns 1*1 = 1
  | factorial n=2|             returns 2*1 = 2
  | factorial n=3|             returns 3*2 = 6
  | <module>     |  bottom
  +--------------+
```

Now change the call to `factorial(5000)`. Python stops with `RecursionError: maximum recursion depth exceeded`. The default limit is 1000 frames (check it with `sys.getrecursionlimit()`), which keeps Python from overflowing the real stack.

**Mistakes.** Unbounded recursion causes a **stack overflow**. In C, that crashes the process. On the heap, the classic failure is a **memory leak**: memory that is never freed. Garbage-collected languages leak too, if you keep references you no longer need. A cache dictionary that only grows is a common example. The symptom is memory use that climbs for hours.

---

## 5. Binary and data representation

**Intuition.** A light switch is on or off. Eight switches in a row make 256 patterns. Everything a computer stores, from your name to a photo, is a pattern of switches plus an agreement about what the pattern means.

**Definition.** A **bit** is 0 or 1. A **byte** is 8 bits, which can hold 256 values (0 to 255). Numbers are written in **binary** (base 2), where each position is worth double the one to its right. **Hexadecimal** (base 16) is shorthand: one hex digit is exactly 4 bits, so one byte is two hex digits. The same bytes mean different things depending on the agreed type or encoding.

**Worked examples.**

- *Binary to decimal.* `1011` is 1×8 + 0×4 + 1×2 + 1×1 = 11.
- *Decimal to binary.* 13 = 8 + 4 + 1, so `1101`.
- *Hex.* `0xFF` = 15×16 + 15 = 255, which is `11111111`, the largest value in one byte.
- *Negative numbers (two's complement).* To store -5 in 8 bits, write 5 (`00000101`), flip every bit (`11111010`), then add 1 to get `11111011`. A signed 8-bit integer runs from -128 to 127. Adding 1 to 127 wraps to -128. That is **integer overflow**.
- *Fractions (floating point).* Most languages store decimals as 64-bit IEEE 754 floating-point numbers. Many decimals, including 0.1, have no exact binary form, just as 1/3 has no exact decimal form. Press F12 in your browser, open the **Console** tab, and type `0.1 + 0.2`. You get `0.30000000000000004`.
- *Text.* Characters are stored as numbers through an encoding. In ASCII and UTF-8, `A` is 65 (`0x41`). UTF-8 uses 1 to 4 bytes per character: English letters take 1, `é` takes 2 (`0xC3 0xA9`), and most emoji take 4. In the same console:

```javascript
new TextEncoder().encode("é")   // Uint8Array [195, 169]
"é".length                      // 1 character
```

In Python (from Day 3), the same conversions are `bin(11)` gives `'0b1011'`, `int("1011", 2)` gives `11`, `hex(255)` gives `'0xff'`, `ord("A")` gives `65`, and `"é".encode("utf-8")` gives `b'\xc3\xa9'`.

**Mistakes.** Never store money as floating point. Store cents as integers, or use a decimal type (Python's `decimal` module, PostgreSQL's `numeric`). Payment APIs such as Stripe send amounts as integers in the smallest currency unit for this reason. Never assume one character is one byte. A name field limited to "50 bytes" can cut a non-English name in half. Text such as `JosÃ©` where `José` should be means UTF-8 bytes were read with the wrong encoding.

---

## 6. Exercises (about 45 minutes, no installs)

Write your answers in a notes file. You will use them in the trace below.

1. **Processes (10 minutes).** Open Notepad twice and open Task Manager. On the **Details** tab, write down both PIDs. On the **Performance** tab, write down your CPU model, core count, clock speed, total RAM and disk type. Close one Notepad and confirm the other PID is still there. *Check:* you have two different PIDs written down.
2. **Bytes on disk (5 minutes).** Create a text file containing `hi` on your desktop. In **Properties**, note "Size" and "Size on disk." *Check:* you can explain why they differ.
3. **Stack frames (15 minutes).** Step through the `factorial(3)` code in Python Tutor. Draw the stack at its deepest point. Then run `factorial(5000)` and copy the error message. *Check:* your drawing has four `factorial` frames.
4. **Compilation (10 minutes).** In Compiler Explorer, compile the C `square` function with `-O0`, then with `-O2`. Count the instructions in each. *Check:* `-O2` has fewer, and you can explain in one sentence why (the optimizer removed steps that do not change the result).
5. **Binary by hand (15 minutes).** Without a calculator, convert 6, 19, 100 and 255 to binary; `10110` and `11111111` to decimal; `0x2A` to decimal; and -1 to 8-bit two's complement. Then check each answer in the browser console with `(19).toString(2)`, `parseInt("10110", 2)` and `0x2A`. *Check:* all eight match.
6. **Floating point and text (5 minutes).** In the console, run `0.1 + 0.2`, `0.1 + 0.2 === 0.3` and `new TextEncoder().encode("Café 😀")`. *Check:* you can explain why the byte count is larger than the character count.
7. **Trace one feature (about 30 minutes).** Take one feature from your Day 1 app, such as "a user adds a habit called Read 20 pages." On one page, write the following:
   1. The source code: a 5 to 15 line sketch in pseudocode or Python.
   2. Translation: would you build this in a compiled or an interpreted language? What runs first?
   3. The process: which process runs this code (a phone app, a browser tab, a server)? What does it need from the OS?
   4. Memory: which values live in stack frames, and which live on the heap? Draw the stack at its deepest point.
   5. Data: write the habit name as UTF-8 bytes. Name one number that must not be a float, and say why.
   6. Storage: when does the data reach disk, and what does a crash one second earlier lose?
   7. Failure: three failure modes from this lesson that could hit this feature.

   *Check:* someone who skipped this lesson could read your page and explain the difference between stack and heap.

---

## 7. Check your understanding

1. Your laptop has a 4.5 GHz CPU and your friend's has 3.0 GHz, yet your friend's runs a data script faster. Give two reasons this can happen.
2. Why does the OS give each process its own virtual address space instead of letting processes share RAM directly?
3. A teammate says, "Python doesn't compile anything." Correct them using what you saw today.
4. A Python function calls itself with no base case and crashes with `RecursionError`. Which memory region is involved, and why does Python stop at a limit?
5. An invoice app stores prices as floats, and a customer's total is off by one cent. What is the cause, and what is the fix?
6. A user's name displays as `JosÃ©`. What went wrong between storage and screen?
7. You compile a C program on Windows and copy the `.exe` to a Linux server. It will not run. Why, and what do you do?

**Answers (check after you try):**

1. The script may be limited by memory or disk speed rather than clock speed. Your friend may have more cache, faster RAM or an SSD. The script may also use several cores, and your friend's CPU may have more of them.
2. Isolation. A bug or an attack in one process cannot read or overwrite another process's memory, and each program can assume it has a clean, private address space.
3. CPython compiles source code to bytecode first. You can see the bytecode with `dis` and in the `.pyc` files in `__pycache__`. Then the virtual machine interprets that bytecode.
4. The call stack. Each call adds a frame. Without a base case, the frames pile up until the stack would overflow. Python's default limit of 1000 raises a clear error before the real stack runs out.
5. Values like 0.1 have no exact binary floating-point form, so tiny errors add up across many items. Store money as integer cents or use a decimal type.
6. The name was stored as UTF-8 (`é` is two bytes, `C3 A9`) but read with a different encoding, which showed each byte as its own character. Use UTF-8 consistently at every step.
7. Machine code and executable formats are specific to a CPU architecture and OS. Rebuild the program for Linux on the server's CPU type.

---

## 8. Sources to read and watch

**Read:**
- [Wikipedia: Central processing unit](https://en.wikipedia.org/wiki/Central_processing_unit)
- [Wikipedia: Instruction cycle](https://en.wikipedia.org/wiki/Instruction_cycle)
- [Wikipedia: Memory hierarchy](https://en.wikipedia.org/wiki/Memory_hierarchy)
- [Wikipedia: Process (computing)](https://en.wikipedia.org/wiki/Process_(computing))
- [Wikipedia: Compiler](https://en.wikipedia.org/wiki/Compiler)
- [Wikipedia: Interpreter (computing)](https://en.wikipedia.org/wiki/Interpreter_(computing))
- [Wikipedia: Call stack](https://en.wikipedia.org/wiki/Call_stack)
- [Wikipedia: Two's complement](https://en.wikipedia.org/wiki/Two%27s_complement)
- [Wikipedia: UTF-8](https://en.wikipedia.org/wiki/UTF-8)
- [Python docs: dis, Disassembler for Python bytecode](https://docs.python.org/3/library/dis.html)
- [Python docs: Floating-Point Arithmetic: Issues and Limitations](https://docs.python.org/3/tutorial/floatingpoint.html)
- "Latency Numbers Every Programmer Should Know": search the title. Several versions on GitHub credit Jeff Dean and Peter Norvig.

**Tutorials:**
- GeeksforGeeks: search "Stack vs Heap Memory Allocation" and "Difference between Compiler and Interpreter".
- [CS50x course site](https://cs50.harvard.edu/x/): the Week 0 notes (binary, representing text) and the Week 4 notes (memory, pointers, stack and heap).

**Watch:**
- CS50x Lecture 0 (binary and data) and Lecture 4 (memory), on the course site and the CS50 YouTube channel.
- CrashCourse on YouTube, *Crash Course Computer Science*: search "The Central Processing Unit (CPU): Crash Course Computer Science #7" and "Instructions & Programs: Crash Course Computer Science #8".
- [Computerphile](https://www.youtube.com/@Computerphile): search "Floating Point Numbers" for a clear explanation of why 0.1 + 0.2 misbehaves.

**Practice:**
- [Python Tutor](https://pythontutor.com/) to step through stack frames and heap objects.
- [Compiler Explorer](https://godbolt.org/) to see what compilers produce.
- Long-term side project: [Nand2Tetris](https://www.nand2tetris.org/) builds a computer from logic gates up to a compiler. It is a full course.

**Note on links:** open each one once. If a page has moved, search its title.

---

## 9. Tomorrow

Day 3, **Developer Environment**, turns your laptop into a working engineer's machine. You will install VS Code, Git and Python on Windows 11, set up WSL 2 so you have a real Linux system for Days 4 and 5, and learn how the terminal finds programs through PATH and environment variables. Today's ideas come back in each part. The terminal starts processes, PATH decides which program file gets loaded into memory, and the debugger pauses the CPU on a line so you can inspect the stack frames you drew today.
