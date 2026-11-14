# Day 43: Linked Lists

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** beginner to intermediate. You should have finished Day 42, be comfortable with classes and `dataclass`, and be willing to draw boxes and arrows on paper. This day is for anyone who has written `node.next = node.next.next` and was not sure what it did.

**By the end of this day, you will have:**

- A working singly linked list with push, append, pop, search, delete and reverse
- A doubly linked list where you can remove a node in O(1) when you already hold a reference to it
- The pointer-reasoning habit: drawing the before and after state of every pointer change
- A browser history built on a doubly linked list, with back, forward and visit
- The repository from Day 42 extended with new modules and tests that pass in GitHub Actions

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 42
- Paper and a pencil for drawing pointers, which is more useful here than any tool
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 41: Complexity Analysis, especially the operation counting habit
- Day 42: Arrays and Strings, especially the difference between indexing and traversal
- Module 3: Python classes, `dataclass` and generators with `yield`

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Nodes](#nodes)
  - [Singly linked lists](#singly-linked-lists)
  - [Doubly linked lists](#doubly-linked-lists)
  - [Insertion and deletion](#insertion-and-deletion)
  - [Pointer reasoning](#pointer-reasoning)
- [Hands-on exercises](#hands-on-exercises)
- [Project: browser history with a doubly linked list](#project-browser-history-with-a-doubly-linked-list)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A Python list is a block of memory with values side by side. Adding to the end is cheap, but inserting at the front means moving every element one place to the right. A linked list takes the opposite approach. Each value lives in its own small object, called a node, and each node holds a reference to the next one. Nothing has to move when you insert, because you only change two references.

That trade is the whole subject of today. Linked lists give up fast indexing, so `items[500]` now means walking 500 nodes, and in exchange they give you cheap changes at a position you already hold. Operating systems use them for free lists of memory blocks. Browsers use doubly linked structures for history and undo. Caches use them to track the least recently used item in O(1) time, which is the design behind LRU caches. The skill you need is the ability to draw the pointers before and after a change and check that no node is lost.

This is also where pointer bugs show up. A missing line can leave a list with a dangling tail, a cycle or a silently truncated chain. Those bugs are hard to see in a debugger, and easy to see on paper. The habit built here, drawing and then testing, will matter in every tree and graph you write in the rest of this module.

---

## Glossary

- **Node:** a small object that stores one value and one or more references to other nodes.
- **Head:** the first node in a linked list. Some implementations also keep a reference to the last node, called the tail.
- **Tail:** the last node in a list. Keeping a tail reference makes appending O(1).
- **Singly linked list:** a list where each node points only to the next node.
- **Doubly linked list:** a list where each node points to both the next and the previous node.
- **Sentinel:** a placeholder node that holds no real value, used to avoid special cases at the head or tail.
- **Dangling reference:** a pointer to a node that the list no longer holds, which can keep memory alive or cause a lost update.
- **Cycle:** a chain of references that leads back to a node already visited, so a traversal never ends.
- **Pointer reasoning:** the practice of predicting the state of every reference before and after an operation.
- **LRU cache:** a cache that discards the least recently used item when full. It is often built from a doubly linked list plus a hash table.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Nodes

Think of a scavenger hunt. Each clue is a card with a message and the location of the next card. You do not know where card 40 is until you have read card 39. A node is one of those cards, and a linked list is the chain of cards.

From first principles, a node has two parts: its data and a reference to another node. In Python, a reference is a name bound to an object, so `node.next` is a field that either holds another `Node` or holds `None` to mark the end. A node does not know its position in the chain, and it does not know how many nodes come before it. Those facts have to be computed by walking the chain.

```python
from dataclasses import dataclass

@dataclass(eq=False, repr=False)
class Node:
    value: object
    next: "Node | None" = None

a = Node("first")
b = Node("second", next=None)
a.next = b                  # the chain is now first -> second
print(a.next.value)         # second
```

The `eq=False` setting matters. Without it, `dataclass` compares nodes field by field, and that includes the whole chain after each node. Two nodes with the same value but different positions would then look equal, which is wrong for a structure where identity matters.

The common mistake is to treat a node as a value. If you write `a = b` and then change `a.value`, you also change `b.value`, because both names refer to the same object. To spot it, ask whether two names point to one node or two. Python does not copy objects on assignment.

Nodes appear in almost every pointer-based structure, including trees, graphs and the free lists inside memory allocators. Any time you read about a "node" in a data structure paper, the same first principles apply.

### Singly linked lists

Imagine a line of people where each person only knows the person directly behind them. To find the tenth person, you start at the front and ask each person in turn. To add someone at the front, the new person tells the old front person where they are standing. Nobody else has to move.

A singly linked list is that line. The list keeps a `head` reference to the first node, and each node points to the next. The class below keeps a `tail` reference too, so that appending does not walk the whole chain:

```python
from collections.abc import Iterator

class SinglyLinkedList:
    def __init__(self, values=()):
        self.head: Node | None = None
        self.tail: Node | None = None
        self._size = 0
        for value in values:
            self.append(value)

    def __len__(self) -> int:
        return self._size

    def __iter__(self) -> Iterator:
        node = self.head
        while node is not None:
            yield node.value
            node = node.next

    def push_front(self, value) -> None:
        self.head = Node(value, self.head)
        if self.tail is None:
            self.tail = self.head
        self._size += 1

    def append(self, value) -> None:
        new_node = Node(value)
        if self.tail is None:
            self.head = self.tail = new_node
        else:
            self.tail.next = new_node
            self.tail = new_node
        self._size += 1

    def pop_front(self):
        if self.head is None:
            raise IndexError("pop from empty list")
        value = self.head.value
        self.head = self.head.next
        if self.head is None:
            self.tail = None
        self._size -= 1
        return value
```

`push_front` and `pop_front` are O(1), because they touch only the head. `append` is O(1) because of the tail. Without the tail, `append` would be O(n), since it would have to walk to the end. Searching is always O(n), because the nodes are not in an array you can index.

The limitation is memory overhead and cache behavior. Each node is a separate object, with its own header and a pointer, so a linked list of a million integers uses far more memory than a Python list of a million integers. Walking the nodes also jumps around memory, which is slower in practice than a sequential scan of an array, even when both are O(n). Use a linked list when you need cheap changes in the middle of a sequence you are already traversing. Choose an array when you need fast random access.

The common mistake is to forget to update `tail` in the same operation that changes the last node. If you delete the last node and leave `tail` pointing at it, the next `append` attaches a new node to a node that is no longer in the list, and the value silently disappears. The method `delete` below handles that case explicitly.

Singly linked lists show up as the chains of buckets inside hash tables, where each slot holds the entries that collided. Day 45 uses them for that purpose.

### Doubly linked lists

Think of a train with a coupling at each end of each car. You can walk forward or backward through the train, and you can unhook a car without hunting for the car in front of it. A doubly linked list gives each node a `prev` reference as well as a `next` reference, which buys you two things. You can walk in both directions, and you can remove a node when you hold the node itself, without searching from the head.

The first principle is that every link must be kept in both directions. When you insert or remove, you update up to four references: the new node's `prev` and `next`, and the neighbors' `next` and `prev`. Missing one of them is the most common source of bugs in this structure.

```python
@dataclass(eq=False, repr=False)
class DoubleNode:
    value: object
    prev: "DoubleNode | None" = None
    next: "DoubleNode | None" = None


class DoublyLinkedList:
    def __init__(self):
        self.head: DoubleNode | None = None
        self.tail: DoubleNode | None = None
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def __iter__(self) -> Iterator:
        node = self.head
        while node is not None:
            yield node.value
            node = node.next

    def __reversed__(self) -> Iterator:
        node = self.tail
        while node is not None:
            yield node.value
            node = node.prev

    def push_back(self, value) -> DoubleNode:
        node = DoubleNode(value, prev=self.tail)
        if self.tail is None:
            self.head = node
        else:
            self.tail.next = node
        self.tail = node
        self._size += 1
        return node

    def push_front(self, value) -> DoubleNode:
        node = DoubleNode(value, next=self.head)
        if self.head is None:
            self.tail = node
        else:
            self.head.prev = node
        self.head = node
        self._size += 1
        return node
```

Both `push` methods return the new node, which the caller can keep. That returned node is what makes O(1) removal possible later.

The limitation is that each node carries two references instead of one, so the memory overhead is higher. Doubly linked lists also need more care at the boundaries. The head's `prev` and the tail's `next` must stay `None`, or a traversal runs off the end.

The common mistake is to write the two directions in different orders, such as updating `next` on a neighbor before reading the node's own `next`, and then losing the reference. A useful habit is to read every neighbor into a local variable first, then change the references. The `remove` method below follows that habit.

Doubly linked lists power undo stacks in editors, the back and forward buttons in browsers and the recency order of LRU caches in web servers.

### Insertion and deletion

Think of removing a card from a chain of paper clips. If you can reach the clip you want to remove, you only need to unhook it from its two neighbors. If you have to search for it first, the search costs far more than the unhooking. This is the central point of linked list performance: the change is cheap, and finding the place to change is the expensive part.

From first principles, deleting a node in a singly linked list needs its predecessor. You set `previous.next` to `node.next`. Finding the predecessor from the head is O(n) in the worst case. Deleting in a doubly linked list needs only the node, because `node.prev` gives you the predecessor. Once you have the node, the change is O(1).

The `delete` method in the project's `SinglyLinkedList` searches from the head, keeping a `previous` reference one step behind the current node. When it finds the value, it bypasses the node. It is O(n), because it searches. The three cases are the head, a middle node and the tail, and each one changes a different reference. The tail case is the one that is easy to forget.

Here is the O(1) removal in the doubly linked list, which the project uses for browser history:

```python
def remove(self, node: DoubleNode) -> None:
    if node.prev is None:
        self.head = node.next
    else:
        node.prev.next = node.next
    if node.next is None:
        self.tail = node.prev
    else:
        node.next.prev = node.prev
    node.prev = node.next = None
    self._size -= 1
```

Insertion has the same shape. To insert after a known node, you create the new node, point it at the old next node, and then point the known node at the new one. Inserting before a known node in a singly linked list needs the predecessor, so it is not O(1) without extra work.

The common mistake in deletion is to change `node.next` before reading it, which loses the rest of the list. The safe order is read, then update. The second mistake is forgetting to update the tail or the head when the change happens at an end. Test those edges on every list change, using a list with one element, a list with two elements and a list with three.

Deletion and insertion in the middle of a sequence are the operations that linked lists make cheap. Text editors used linked structures for the document model in older designs, and today's editors use related structures, such as gap buffers and ropes, that solve the same problem with better memory behavior.

### Pointer reasoning

Pointer reasoning is the habit of drawing a list before a change, drawing it after the change, and checking that every node is still reachable. It is the most useful skill in this day, because it works for any structure that uses references, including trees and graphs.

Start with three nodes labeled A, B and C, with arrows from A to B and from B to C. Suppose you want to remove B. The before picture is A points at B, and B points at C. The after picture should be A points at C, and B points at nothing. Now check the code: `previous.next = node.next` makes A point at C, and the reference to B is dropped, which is correct. If you wrote `node.next = None` first, you would lose C.

The method to practice has three steps. Draw the state. Name the references that change. For each change, write the new value and check whether the old value is still needed. A simple way to test reasoning is to reverse a list in your head with three nodes, then check the code against your drawing.

```python
def reverse(self) -> None:
    previous = None
    node = self.head
    self.tail = self.head
    while node is not None:
        following = node.next     # save the rest of the list first
        node.next = previous      # flip this arrow
        previous = node           # move the previous pointer forward
        node = following          # move along the original chain
    self.head = previous
```

The line that saves `following` is the pointer reasoning in one line. Without it, flipping `node.next` would make the rest of the list unreachable, and the loop would stop after one node.

The limitation of pointer reasoning on paper is that it misses bugs that depend on sequence, such as an operation that is correct alone but wrong after another. Test the operations in combination. A randomized test that compares the linked list with a Python list catches those cases, and the project uses one.

The common mistake is to draw the picture and then forget to update the length or the tail. The picture shows the references, but the counters are separate state. Keep the counter update next to every structural change, so the two cannot drift apart.

Pointer reasoning is used when engineers debug cache eviction code, implement concurrency locks on linked queues and review patches to kernels and drivers, where a missed pointer update can corrupt memory.

---

## Hands-on exercises

Work in the repository from Day 42. Keep a sheet of paper next to you for the drawings.

Exercise 1 draws a list. Create a singly linked list with the values 10, 20 and 30 by hand. Draw the nodes and arrows. Then draw the state after `push_front(5)`, after `pop_front()` and after `delete_by_value(20)`. Compare each drawing with the code output.

Done when each drawing matches the list printed by `list(lst)` after the same operation.

Exercise 2 checks the tail. Write a test that builds a list with three values, deletes the last value, and then appends a new value. Confirm that the new value appears in the iteration, and that `len` returns the correct number.

Done when the test passes, and you can explain what would happen to the appended value if `delete` did not update `tail`.

Exercise 3 writes a middle finder. Using two pointers, a slow pointer that moves one step and a fast pointer that moves two steps, write a function `middle_value(lst)` that returns the middle value of a singly linked list. For an even number of values, return the second of the two middle values. Test it on lists of length 1, 2, 3 and 4.

Done when all four lengths return the expected values in tests. Hint: the fast pointer must check two nodes ahead before moving.

Exercise 4 tests against a reference. Write a randomized test that performs 500 random operations on a `SinglyLinkedList` and a Python list in parallel, using `random.Random(3)`. The operations are `push_front`, `append`, `pop_front` (only when the list is not empty) and `delete` with a random value. After each operation, assert that the two containers hold the same values in the same order.

Done when the randomized test passes and you can name the operation you found hardest to mirror in Python.

Exercise 5 ties it together. Commit the work and open the Actions tab on GitHub. Make sure the run for your latest commit is green, and write one sentence in `docs/linked-list-notes.md` on why `remove` takes a node and not a value.

Done when the workflow is green and the notes file has the sentence.

---

## Project: browser history with a doubly linked list

The project is a `BrowserHistory` class that stores visited pages in a doubly linked list. It supports three operations: `visit` a new page, which discards any forward history, `back` to move the current page backward, and `forward` to move it forward. The current page is a node reference, so moving back and forward is a pointer walk, and removing the forward history uses the O(1) `remove` method from this day.

This is a real use case. The back and forward buttons in a browser behave the same way, and the forward history is cut off when you open a new page from an older one.

Suggested repo name: `100-days-software-day-43-linked-lists`. Copy the Day 42 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-43-linked-lists/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   ├── browser_history.py
│   ├── complexity.py
│   └── linked_list.py
├── docs/
│   ├── linked-list-notes.md
│   └── window-notes.md
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   ├── test_browser_history.py
│   ├── test_complexity.py
│   └── test_linked_list.py
└── README.md
```

Starter code for `dsa/linked_list.py`:

```python
from __future__ import annotations

from collections.abc import Iterator
from dataclasses import dataclass


@dataclass(eq=False, repr=False)
class Node:
    value: object
    next: Node | None = None


class SinglyLinkedList:
    def __init__(self, values=()):
        self.head: Node | None = None
        self.tail: Node | None = None
        self._size = 0
        for value in values:
            self.append(value)

    def __len__(self) -> int:
        return self._size

    def __iter__(self) -> Iterator:
        node = self.head
        while node is not None:
            yield node.value
            node = node.next

    def push_front(self, value) -> None:
        self.head = Node(value, self.head)
        if self.tail is None:
            self.tail = self.head
        self._size += 1

    def append(self, value) -> None:
        new_node = Node(value)
        if self.tail is None:
            self.head = self.tail = new_node
        else:
            self.tail.next = new_node
            self.tail = new_node
        self._size += 1

    def pop_front(self):
        if self.head is None:
            raise IndexError("pop from empty list")
        value = self.head.value
        self.head = self.head.next
        if self.head is None:
            self.tail = None
        self._size -= 1
        return value

    def find(self, value) -> Node | None:
        node = self.head
        while node is not None:
            if node.value == value:
                return node
            node = node.next
        return None

    def insert_after(self, node: Node, value) -> None:
        node.next = Node(value, node.next)
        if node is self.tail:
            self.tail = node.next
        self._size += 1

    def delete(self, value) -> bool:
        """Remove the first node holding value. Return True if one was removed."""
        previous = None
        node = self.head
        while node is not None:
            if node.value == value:
                if previous is None:
                    self.head = node.next
                else:
                    previous.next = node.next
                if node is self.tail:
                    self.tail = previous
                self._size -= 1
                return True
            previous, node = node, node.next
        return False

    def reverse(self) -> None:
        previous = None
        node = self.head
        self.tail = self.head
        while node is not None:
            following = node.next
            node.next = previous
            previous = node
            node = following
        self.head = previous


@dataclass(eq=False, repr=False)
class DoubleNode:
    value: object
    prev: DoubleNode | None = None
    next: DoubleNode | None = None


class DoublyLinkedList:
    def __init__(self):
        self.head: DoubleNode | None = None
        self.tail: DoubleNode | None = None
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def __iter__(self) -> Iterator:
        node = self.head
        while node is not None:
            yield node.value
            node = node.next

    def __reversed__(self) -> Iterator:
        node = self.tail
        while node is not None:
            yield node.value
            node = node.prev

    def push_back(self, value) -> DoubleNode:
        node = DoubleNode(value, prev=self.tail)
        if self.tail is None:
            self.head = node
        else:
            self.tail.next = node
        self.tail = node
        self._size += 1
        return node

    def push_front(self, value) -> DoubleNode:
        node = DoubleNode(value, next=self.head)
        if self.head is None:
            self.tail = node
        else:
            self.head.prev = node
        self.head = node
        self._size += 1
        return node

    def remove(self, node: DoubleNode) -> None:
        """Unlink a node in O(1), given the node itself."""
        if node.prev is None:
            self.head = node.next
        else:
            node.prev.next = node.next
        if node.next is None:
            self.tail = node.prev
        else:
            node.next.prev = node.prev
        node.prev = node.next = None
        self._size -= 1
```

The `from __future__ import annotations` line lets the type hints refer to classes that are defined later in the file, such as `Node | None` inside `Node`. It is a standard pattern for self-referencing types.

Starter code for `dsa/browser_history.py`:

```python
from dsa.linked_list import DoubleNode, DoublyLinkedList


class BrowserHistory:
    def __init__(self, homepage: str):
        self._pages = DoublyLinkedList()
        self._current: DoubleNode = self._pages.push_back(homepage)

    def current(self) -> str:
        return self._current.value

    def visit(self, url: str) -> None:
        """Open a new page, discarding any forward history."""
        node = self._current.next
        while node is not None:
            following = node.next
            self._pages.remove(node)
            node = following
        self._current = self._pages.push_back(url)

    def back(self, steps: int = 1) -> str:
        for _ in range(steps):
            if self._current.prev is None:
                break
            self._current = self._current.prev
        return self.current()

    def forward(self, steps: int = 1) -> str:
        for _ in range(steps):
            if self._current.next is None:
                break
            self._current = self._current.next
        return self.current()
```

Starter code for `tests/test_linked_list.py`:

```python
import random
import unittest

from dsa.linked_list import DoublyLinkedList, SinglyLinkedList


class SinglyLinkedListTests(unittest.TestCase):
    def test_push_front_and_append_order(self):
        lst = SinglyLinkedList([2, 3])
        lst.push_front(1)
        lst.append(4)
        self.assertEqual(list(lst), [1, 2, 3, 4])
        self.assertEqual(len(lst), 4)

    def test_pop_front_on_empty_raises(self):
        with self.assertRaises(IndexError):
            SinglyLinkedList().pop_front()

    def test_delete_tail_then_append(self):
        lst = SinglyLinkedList([1, 2, 3])
        self.assertTrue(lst.delete(3))
        lst.append(9)
        self.assertEqual(list(lst), [1, 2, 9])

    def test_delete_missing_returns_false(self):
        lst = SinglyLinkedList([1])
        self.assertFalse(lst.delete(5))
        self.assertEqual(len(lst), 1)

    def test_reverse(self):
        lst = SinglyLinkedList([1, 2, 3])
        lst.reverse()
        self.assertEqual(list(lst), [3, 2, 1])
        lst.append(0)
        self.assertEqual(list(lst), [3, 2, 1, 0])

    def test_insert_after_tail_updates_tail(self):
        lst = SinglyLinkedList([1, 2])
        lst.insert_after(lst.find(2), 3)
        lst.append(4)
        self.assertEqual(list(lst), [1, 2, 3, 4])

    def test_random_operations_match_python_list(self):
        rng = random.Random(3)
        lst = SinglyLinkedList()
        mirror: list[int] = []
        for _ in range(500):
            op = rng.choice(["push", "append", "pop", "delete"])
            if op == "push":
                value = rng.randint(0, 9)
                lst.push_front(value)
                mirror.insert(0, value)
            elif op == "append":
                value = rng.randint(0, 9)
                lst.append(value)
                mirror.append(value)
            elif op == "pop" and mirror:
                self.assertEqual(lst.pop_front(), mirror.pop(0))
            elif op == "delete":
                value = rng.randint(0, 9)
                removed = lst.delete(value)
                if value in mirror:
                    mirror.remove(value)
                    self.assertTrue(removed)
                else:
                    self.assertFalse(removed)
            self.assertEqual(list(lst), mirror)


class DoublyLinkedListTests(unittest.TestCase):
    def test_iterate_both_ways(self):
        lst = DoublyLinkedList()
        for value in [1, 2, 3]:
            lst.push_back(value)
        self.assertEqual(list(lst), [1, 2, 3])
        self.assertEqual(list(reversed(lst)), [3, 2, 1])

    def test_remove_head_middle_and_tail(self):
        lst = DoublyLinkedList()
        nodes = [lst.push_back(v) for v in [1, 2, 3, 4]]
        lst.remove(nodes[0])
        lst.remove(nodes[2])
        lst.remove(nodes[3])
        self.assertEqual(list(lst), [2])
        self.assertEqual(len(lst), 1)
        self.assertIs(lst.tail, nodes[1])

    def test_remove_only_node_leaves_empty_list(self):
        lst = DoublyLinkedList()
        node = lst.push_front("solo")
        lst.remove(node)
        self.assertEqual(len(lst), 0)
        self.assertIsNone(lst.head)
        self.assertIsNone(lst.tail)
        lst.push_back("again")
        self.assertEqual(list(lst), ["again"])


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_browser_history.py`:

```python
import unittest

from dsa.browser_history import BrowserHistory


class BrowserHistoryTests(unittest.TestCase):
    def test_back_and_forward_within_bounds(self):
        history = BrowserHistory("home")
        history.visit("a")
        history.visit("b")
        self.assertEqual(history.back(1), "a")
        self.assertEqual(history.back(5), "home")
        self.assertEqual(history.forward(1), "a")
        self.assertEqual(history.forward(10), "b")

    def test_visit_discards_forward_history(self):
        history = BrowserHistory("home")
        history.visit("a")
        history.visit("b")
        history.back(2)
        history.visit("c")
        self.assertEqual(history.forward(3), "c")
        self.assertEqual(history.back(2), "home")

    def test_back_on_homepage_stays_put(self):
        history = BrowserHistory("home")
        self.assertEqual(history.back(3), "home")
        self.assertEqual(history.forward(3), "home")


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 42 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Draw the list state for `BrowserHistory` through this sequence: start at "home", visit "a", visit "b", back 2, visit "c". Check the drawing against `history.forward()`, which should stay on "c".
3. Add a method `history_list(self) -> list[str]` to `BrowserHistory` that returns every page in order, from the first visited to the last. Add a test that checks it after a visit that truncates forward history.
4. Write `docs/linked-list-notes.md` with three sections: why `remove` takes a node, why `visit` is O(k) for k discarded pages with an amortized argument, and what a singly linked list would cost for `back`.
5. Commit with the message "Add linked lists and browser history", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- The randomized test in `tests/test_linked_list.py` passes with 500 operations
- `docs/linked-list-notes.md` explains the cost of `visit` with a reason
- The latest commit on `main` has a green check

Evidence to commit: `dsa/linked_list.py`, `dsa/browser_history.py`, both test files, `docs/linked-list-notes.md` and the Actions run. The randomized test is the strongest evidence that the pointers stay correct across many sequences of operations.

This project is light on new theory and heavy on pointer care. The browser history is a small, real case, and the habits it builds will show up in the tree and graph code later in this module.

---

## Check your understanding

1. You hold a reference to a node in a singly linked list and you want to delete it. Why can you not do it in O(1) without the predecessor, and what would change if the list were doubly linked?
2. A teammate writes `node.next = None` before `previous.next = node.next` in a deletion function. Describe what goes wrong, using a three-node list as an example.
3. Explain why `append` is O(1) only because the list keeps a tail reference. What would the cost be without it, and why does the tail need to change in `delete`?
4. Compare a Python list and a singly linked list for the operation "insert a value at the front, a million times". Which is faster in practice, and why might the answer differ for "read the element at index 500000"?
5. In `BrowserHistory.visit`, the loop removes forward nodes one at a time. Why is removing a node O(1), and why is the whole loop still correct when the removed nodes include the tail?
6. A linked list has a cycle, so iterating it never ends. Explain how a cycle can form from a bad pointer update, and describe a test that would catch it.

### Answers

1. In a singly linked list, you need the predecessor to change its `next` reference, and the only way to find it is to walk from the head, which is O(n). In a doubly linked list, `node.prev` gives you the predecessor directly, so the removal is O(1) once you hold the node.
2. The line `node.next = None` erases the reference to the rest of the list before it is used. In a list A to B to C, deleting B by setting B's `next` to `None` first and then setting A's `next` to `B.next` makes A point at `None`, so C is lost. The correct order reads `node.next` before any change, or updates the predecessor first.
3. Without a tail, `append` walks from the head to the last node, which is O(n). A tail makes it O(1), but the tail must be updated whenever the last node changes. In `delete`, removing the last node means the tail must move to the previous node, or `append` would attach to a node that is no longer in the list.
4. Inserting at the front of a Python list shifts every element, so a million insertions cost about a million times the average shift, which is far slower than the linked list's O(1) per insertion. Reading index 500000 from a Python list is O(1), while the linked list walks 500000 nodes, so the answer flips for random access.
5. Removing a node only changes its neighbors' references and the list's head or tail, so the cost is constant, which is O(1). The loop captures `node.next` before removing, so it never loses the remaining chain. If the removed node is the tail, `remove` moves the tail back to the current page, so the next `push_back` attaches to the right place.
6. A cycle forms when a reference points to a node that comes earlier in the chain, for example when `tail.next` is set to the head by mistake. A test can catch it by iterating with a counter capped at the expected length, or by using a seen set of node identities and failing when a node repeats.
---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Linked list](https://en.wikipedia.org/wiki/Linked_list)
- [Wikipedia: Doubly linked list](https://en.wikipedia.org/wiki/Doubly_linked_list)
- [Wikipedia: Sentinel node](https://en.wikipedia.org/wiki/Sentinel_node)
- [Wikipedia: Cache replacement policies](https://en.wikipedia.org/wiki/Cache_replacement_policies), the section on least recently used
- The Python documentation on `dataclasses`, at docs.python.org/3/library/dataclasses.html

*Tutorials*

- GeeksforGeeks, "Linked List Data Structure" and "Doubly Linked List" tutorials. Search the site for each title.
- W3Schools, "Python Linked Lists" tutorial. Search the W3Schools Python section for "linked list" and compare its implementation with the one in this day.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on data structures that cover linked lists and their operations. Search the course site for "linked list".
- Abdul Bari on YouTube, the playlist on linked lists. Search YouTube for "Abdul Bari linked list".

*Practice*

- Python Tutor at pythontutor.com. Paste the `SinglyLinkedList` class and a short script that calls `reverse`, then step through each pointer change.
- LeetCode, the "Linked List" study plan at leetcode.com. Start with "Reverse Linked List" and "Delete Node in a Linked List", then compare your code with the drawings you made today.

---

## Tomorrow

Day 44 covers stacks and queues. A stack is a last-in, first-out structure, and a queue is first-in, first-out, and both are often built on top of the linked lists from today. You will build a stack and a queue, see how a deque handles both ends, and use a monotonic stack to answer "next greater element" questions in O(n). The pointer habit from today will help you check that a queue's head and tail stay consistent.
