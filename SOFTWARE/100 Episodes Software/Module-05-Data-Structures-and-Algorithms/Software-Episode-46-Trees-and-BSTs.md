# Day 46: Trees and BSTs

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** intermediate. You should have finished Day 45 and be comfortable with recursion at a basic level, meaning a function that calls itself on a smaller input. This day is for anyone who has seen file system folders or HTML documents and wants to know the structure behind them.

**By the end of this day, you will have:**

- The vocabulary of trees: root, parent, child, leaf, height and depth
- A binary search tree (BST) with search, insert and delete, and a clear rule for each of the three delete cases
- The four traversal orders, preorder, inorder, postorder and level order, each with a use
- A demonstration that a BST's height decides its speed, shown by inserting random and sorted keys
- A range query on the BST, tested against a sorted list, committed with CI

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 45
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)
- Paper for drawing trees, which you will use in every exercise

**Prerequisites:**

- Day 43: Linked Lists (nodes with references to other nodes)
- Day 44: Stacks and Queues (the traversal order of a stack and a queue)
- Day 45: Hash Tables (why a hash table cannot give you the next key)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Tree terminology](#tree-terminology)
  - [Traversals](#traversals)
  - [Search insert and delete](#search-insert-and-delete)
  - [Recursion](#recursion)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an ordered timestamp index](#project-an-ordered-timestamp-index)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A hash table can tell you whether a key is present in about one step, but it cannot tell you what comes after a key, or which keys fall between two values. A log viewer that shows events from 9:00 to 9:15 needs that ability. So does a file browser that lists folders in order, and an autocomplete box that offers every name starting with "ca". Trees solve those problems. A binary search tree keeps its keys in sorted order by shape, so a search, an insert or a range query can skip most of the data.

Trees are also everywhere in the software you already use. The folders on your disk form a tree. An HTML page is a tree of elements, and the browser builds the DOM as a tree before it renders anything. A parser turns source code into an abstract syntax tree before the compiler optimizes it. The database indexes behind most web applications are B-trees, which are a close relative of the BST in this day.

The outline asks for terminology, traversals, search and insert and delete, and recursion. The project turns these into an index that answers range queries over timestamps, and it measures the one weakness that a plain BST has. That weakness, a tree that grows lopsided when the input is sorted, is why balanced trees exist, and it is the reason the project matters.

---

## Glossary

- **Tree:** a set of nodes connected by edges, with no cycles, and one node designated as the root.
- **Root:** the top node of a tree, with no parent.
- **Parent and child:** a node is the parent of the nodes directly below it, and those are its children.
- **Leaf:** a node with no children.
- **Depth:** the number of edges from the root to a node. The root has depth 0.
- **Height:** the number of nodes on the longest path from the root to a leaf. An empty tree has height 0 in this day's code, and a single node has height 1.
- **Binary tree:** a tree where each node has at most two children, called left and right.
- **Binary search tree (BST):** a binary tree where every key in a node's left subtree is smaller than the node's key, and every key in its right subtree is larger.
- **Subtree:** a node and all of the nodes below it.
- **Traversal:** visiting every node of a tree in a defined order.
- **Balanced tree:** a tree whose height stays close to the minimum for its number of nodes, usually O(log n).
- **Range query:** finding every key between a low and a high value.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Tree terminology

Think of a company org chart. The CEO is at the top, managers are below, and teams sit under the managers. Nobody reports to two people, and nobody reports to themselves. That is a tree, and the words you need come straight from the chart: the top is the root, a person with no reports is a leaf, and the number of levels is the height.

From first principles, a tree has one root and every other node has exactly one parent. Adding one edge between two nodes in a tree creates a cycle, which is why a tree with n nodes always has n minus 1 edges. A binary tree limits each node to two children. In code, a node is an object with a key and references to its children, and the whole tree is reachable from the root. If you lose the root reference, the entire tree is lost.

```python
from dataclasses import dataclass

@dataclass(eq=False, repr=False)
class TreeNode:
    key: int
    left: "TreeNode | None" = None
    right: "TreeNode | None" = None

#        5
#       / \
#      3   8
#     / \
#    1   4
root = TreeNode(5, TreeNode(3, TreeNode(1), TreeNode(4)), TreeNode(8))
```

The limitation of the tree model is that it does not capture shared structure. A file that is linked from two folders forms a graph, and Day 48 covers that case. A tree can also hide its shape when you print it as a list, so draw it when you debug.

The common mistake is to mix up height and depth. The depth of a node is measured from the root down, and the height of the tree is the maximum depth plus one, counting nodes. To spot it, check your definitions on a three-node chain. The depth of the bottom node is 2, and the height is 3 nodes.

Trees appear in file systems, in the DOM of web pages, in the abstract syntax trees of compilers and in the indexes of databases.

### Traversals

Imagine a family tree printed on a page. You can read it top to bottom, left to right, or in the order that you visit each person. Each choice gives a different list from the same tree. A traversal is a rule for the order of visits, and the four standard rules are preorder, inorder, postorder and level order.

From first principles, the three depth-first orders differ only in where you put the visit to the current node relative to the two recursive calls. Preorder visits the node first, then the left and right subtrees. Inorder visits the left subtree, the node and then the right subtree. Postorder visits both subtrees before the node. Level order visits each depth in turn, which needs a queue instead of recursion. All four take O(n) time, because each node is visited once.

```python
def inorder(node, out):
    if node is not None:
        inorder(node.left, out)
        out.append(node.key)
        inorder(node.right, out)
    return out

def preorder(node, out):
    if node is not None:
        out.append(node.key)
        preorder(node.left, out)
        preorder(node.right, out)
    return out

tree = root  # the tree from the previous example
print(inorder(tree, []))    # [1, 3, 4, 5, 8]
print(preorder(tree, []))   # [5, 3, 1, 4, 8]
```

Inorder traversal of a BST returns the keys in sorted order, which is the most useful fact about the structure. Preorder is useful for copying a tree, because you create each node before its children. Postorder is useful for deleting a tree, because you free the children before the parent. Level order is useful when you need the nodes by depth, such as drawing a tree one row at a time.

The limitation of recursive traversal is the call stack. A tree with a depth of 10000 needs 10000 nested calls, which exceeds Python's default recursion limit of 1000. Day 50 explains the call stack and how to avoid the limit with an explicit stack.

The common mistake is to write the recursive call for the right subtree before the left one, which reverses the order of inorder output. To spot it, trace a three-node tree on paper. The output for the tree above must start with 1.

Traversals show up in expression evaluators, which use postorder, in file system tools that compute folder sizes, and in web page rendering, where layout visits the DOM in document order.

### Search insert and delete

Consider a dictionary arranged so that every word in the left half of a page comes before the word at the top of the page. To find a word, you start at the middle and move left or right. Each comparison rules out half of the remaining pages, if the book is balanced. A BST works the same way: each node splits the keys into smaller and larger.

From first principles, search compares the key with the node. If equal, you are done. If smaller, search the left subtree. If larger, search the right subtree. Insert follows the same path until it finds an empty child slot, and places the new node there. Both operations cost O(h), where h is the height. In a balanced tree h is about log2 n, so a million keys need about 20 comparisons. In a degenerate tree, h equals n, and the cost is O(n).

Delete has three cases. If the node is a leaf, remove it. If it has one child, replace the node with that child. If it has two children, find the smallest key in the right subtree, which is the inorder successor. Copy its key into the node, then delete the successor from the right subtree. The successor has no left child, so its deletion is one of the two easy cases.

```python
def _delete(self, node, key, removed):
    if node is None:
        return None
    if key < node.key:
        node.left = self._delete(node.left, key, removed)
    elif key > node.key:
        node.right = self._delete(node.right, key, removed)
    else:
        removed[0] = True
        if node.left is None:
            return node.right
        if node.right is None:
            return node.left
        successor = node.right
        while successor.left is not None:
            successor = successor.left
        node.key = successor.key
        node.right = self._delete(node.right, successor.key, removed)
    return node
```

The limitation of a plain BST is that nothing keeps it balanced. Inserting keys in sorted order makes each new key the right child of the last one, and the tree becomes a chain. Searches then take O(n), and the structure gives no advantage over a list. Balanced variants such as AVL trees and red-black trees fix that by rotating nodes after inserts. Those are beyond this day, but the project measures the problem.

The common mistake in delete is to forget that the successor's key is still in the right subtree after you copy it, so the key appears twice. The fix is the recursive call on `node.right` that removes the successor by its key. The second mistake is to copy the wrong successor, such as the largest key in the left subtree, which breaks the BST rule for some inputs. To spot both, check the inorder output after each delete.

BSTs and their balanced variants power the ordered maps in many standard libraries. Java's `TreeMap` and C++'s `std::map` are red-black trees, and database indexes use B-trees built on the same ideas.

### Recursion

Think of a set of Russian nesting dolls. To open the smallest doll, you must open the one that contains it, and so on up to the largest. Each opening is the same action on a smaller doll, and the process ends when a doll does not open. A recursive function works that way: it solves a problem by solving a smaller version of the same problem, until it reaches a base case.

From first principles, a recursive function needs two things. A base case returns an answer without calling itself, such as `node is None`. A recursive case calls the function on a smaller input that moves toward the base case. Each call adds a frame to the call stack, which holds its arguments and local variables. The depth of the recursion is the largest number of frames at once, and it is the height of the tree for tree traversals.

```python
def height(node) -> int:
    if node is None:                      # base case: an empty tree has height 0
        return 0
    return 1 + max(height(node.left), height(node.right))

print(height(root))   # 3, the path 5 to 3 to 1 has three nodes
```

The limitation of recursion is the stack depth. Python limits recursion to 1000 frames by default, which you can read with `sys.getrecursionlimit()`. A degenerate tree with 2000 nodes will fail in a recursive `height` call. The fix is an explicit stack or queue, or a balanced tree that keeps the depth small.

The common mistake is a missing or wrong base case. If `height` tests `node.left is None` instead of `node is None`, it crashes when a node has only one child. To spot it, test a single node, a chain and an empty tree. Each one exercises a different base case.

Recursion appears in parsers, in file system walkers, in divide-and-conquer algorithms such as merge sort on Day 49 and in every tree and graph algorithm in this module.

---

## Hands-on exercises

Work in the repository from Day 45. Keep the BST in `dsa/bst.py` and the demo in `height_demo.py`.

Exercise 1 draws a tree. Insert the keys 5, 3, 8, 1, 4, 7 and 9 into a tree by hand, drawing the tree after each insert. Write the inorder, preorder and postorder lists before you check them in code. Then build the same tree with `BinarySearchTree` and compare.

Done when your three lists match the code output and your drawing matches the tree's shape.

Exercise 2 traces delete. Starting from the tree in exercise 1, draw the result of deleting 3, which has two children, and then the result of deleting 8. Check your drawings against the inorder list after each delete.

Done when the inorder list after deleting 3 is 1, 4, 5, 7, 8, 9, and after deleting 8 it is 1, 4, 5, 7, 9.

Exercise 3 shows the degenerate case. Insert the keys 1 to 30 in order into a BST and print its height. Then insert the same 30 keys in a shuffled order using `random.Random(1)` and print the height again. Record both in `docs/bst-notes.md` and explain the difference.

Done when your notes show a height of 30 for the sorted insert, and a smaller height for the shuffled insert.

Exercise 4 tests delete on a random workload. Add a randomized test in `tests/test_bst.py` that runs 300 random inserts and deletes, using `random.Random(9)` and keys from 0 to 50, and checks after each operation that `tree.inorder()` equals `sorted(mirror)`.

Done when the randomized test passes and you can describe one case it would catch that a leaf-only delete would miss.

Exercise 5 finds the limit. Build a tree from the sorted keys 0 to 1100 and call `height()`. Read the traceback and identify the recursion limit that you hit. Then write a version of `height` that uses an explicit stack, and check that it returns 1101.

Done when the recursive version fails with `RecursionError` and the iterative version returns 1101.

---

## Project: an ordered timestamp index

The project is an ordered index over event timestamps. The BST stores timestamps as keys, so the tree can answer "which events happened between these two times" by visiting only the subtrees that can contain them. The project adds a `range_keys` method, a demo that measures the height of random and sorted inputs, and tests that check the answers against a sorted list.

The project uses integers as timestamps, which keeps the example simple. In a real log system, timestamps would be stored as sortable values, such as Unix seconds or ISO 8601 strings. Strings sort correctly in ISO format because the fields are in decreasing significance.

Suggested repo name: `100-days-software-day-46-trees-and-bsts`. Copy the Day 45 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-46-trees-and-bsts/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── count_words.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   ├── bst.py
│   ├── browser_history.py
│   ├── complexity.py
│   ├── hash_table.py
│   ├── linked_list.py
│   ├── stacks_queues.py
│   └── word_count.py
├── docs/
│   ├── bst-notes.md
│   └── hash-notes.md
├── height_demo.py
├── print_queue.py
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   ├── test_bst.py
│   ├── test_browser_history.py
│   ├── test_complexity.py
│   ├── test_hash_table.py
│   ├── test_linked_list.py
│   ├── test_print_queue.py
│   ├── test_stacks_queues.py
│   └── test_word_count.py
└── README.md
```

Starter code for `dsa/bst.py`:

```python
from __future__ import annotations

from collections import deque
from dataclasses import dataclass


@dataclass(eq=False, repr=False)
class TreeNode:
    key: int
    left: TreeNode | None = None
    right: TreeNode | None = None


def _inorder(node: TreeNode | None, out: list[int]) -> list[int]:
    if node is not None:
        _inorder(node.left, out)
        out.append(node.key)
        _inorder(node.right, out)
    return out


def _preorder(node: TreeNode | None, out: list[int]) -> list[int]:
    if node is not None:
        out.append(node.key)
        _preorder(node.left, out)
        _preorder(node.right, out)
    return out


def _postorder(node: TreeNode | None, out: list[int]) -> list[int]:
    if node is not None:
        _postorder(node.left, out)
        _postorder(node.right, out)
        out.append(node.key)
    return out


def _height(node: TreeNode | None) -> int:
    if node is None:
        return 0
    return 1 + max(_height(node.left), _height(node.right))


class BinarySearchTree:
    def __init__(self, keys=()) -> None:
        self.root: TreeNode | None = None
        self._size = 0
        for key in keys:
            self.insert(key)

    def __len__(self) -> int:
        return self._size

    def __contains__(self, key) -> bool:
        node = self.root
        while node is not None:
            if key == node.key:
                return True
            node = node.left if key < node.key else node.right
        return False

    def insert(self, key) -> bool:
        """Insert key if it is not present. Return True if it was added."""
        if self.root is None:
            self.root = TreeNode(key)
            self._size += 1
            return True
        node = self.root
        while True:
            if key == node.key:
                return False
            if key < node.key:
                if node.left is None:
                    node.left = TreeNode(key)
                    break
                node = node.left
            else:
                if node.right is None:
                    node.right = TreeNode(key)
                    break
                node = node.right
        self._size += 1
        return True

    def delete(self, key) -> bool:
        """Remove key if present. Return True if it was removed."""
        removed = [False]
        self.root = self._delete(self.root, key, removed)
        if removed[0]:
            self._size -= 1
        return removed[0]

    def _delete(self, node: TreeNode | None, key, removed: list[bool]) -> TreeNode | None:
        if node is None:
            return None
        if key < node.key:
            node.left = self._delete(node.left, key, removed)
        elif key > node.key:
            node.right = self._delete(node.right, key, removed)
        else:
            removed[0] = True
            if node.left is None:
                return node.right
            if node.right is None:
                return node.left
            successor = node.right
            while successor.left is not None:
                successor = successor.left
            node.key = successor.key
            node.right = self._delete(node.right, successor.key, removed)
        return node

    def inorder(self) -> list[int]:
        return _inorder(self.root, [])

    def preorder(self) -> list[int]:
        return _preorder(self.root, [])

    def postorder(self) -> list[int]:
        return _postorder(self.root, [])

    def level_order(self) -> list[list[int]]:
        levels: list[list[int]] = []
        if self.root is None:
            return levels
        queue = deque([self.root])
        while queue:
            level = []
            for _ in range(len(queue)):
                node = queue.popleft()
                level.append(node.key)
                if node.left is not None:
                    queue.append(node.left)
                if node.right is not None:
                    queue.append(node.right)
            levels.append(level)
        return levels

    def height(self) -> int:
        return _height(self.root)

    def minimum(self):
        if self.root is None:
            raise ValueError("empty tree")
        node = self.root
        while node.left is not None:
            node = node.left
        return node.key

    def range_keys(self, low, high) -> list[int]:
        """Keys k with low <= k <= high, in sorted order, skipping subtrees that cannot match."""
        out: list[int] = []

        def walk(node: TreeNode | None) -> None:
            if node is None:
                return
            if low < node.key:
                walk(node.left)
            if low <= node.key <= high:
                out.append(node.key)
            if node.key < high:
                walk(node.right)

        walk(self.root)
        return out
```

Starter code for `height_demo.py`:

```python
import random

from dsa.bst import BinarySearchTree


def heights(n: int, seed: int = 1) -> tuple[int, int]:
    """Height of a BST built from n random keys, and from n sorted keys."""
    rng = random.Random(seed)
    random_keys = rng.sample(range(10 * n), n)
    random_height = BinarySearchTree(random_keys).height()
    sorted_height = BinarySearchTree(range(n)).height()
    return random_height, sorted_height


if __name__ == "__main__":
    for n in [50, 200, 500]:
        random_height, sorted_height = heights(n)
        print(f"n={n:>4}  random height={random_height:>3}  sorted height={sorted_height:>4}")
```

Starter code for `tests/test_bst.py`:

```python
import random
import unittest

from dsa.bst import BinarySearchTree
from height_demo import heights


class BstTests(unittest.TestCase):
    def setUp(self):
        self.tree = BinarySearchTree([5, 3, 8, 1, 4, 7, 9])

    def test_traversals(self):
        self.assertEqual(self.tree.inorder(), [1, 3, 4, 5, 7, 8, 9])
        self.assertEqual(self.tree.preorder(), [5, 3, 1, 4, 8, 7, 9])
        self.assertEqual(self.tree.postorder(), [1, 4, 3, 7, 9, 8, 5])
        self.assertEqual(
            self.tree.level_order(), [[5], [3, 8], [1, 4, 7, 9]]
        )

    def test_search_and_minimum(self):
        self.assertIn(4, self.tree)
        self.assertNotIn(6, self.tree)
        self.assertEqual(self.tree.minimum(), 1)

    def test_insert_duplicate_is_rejected(self):
        self.assertFalse(self.tree.insert(4))
        self.assertEqual(len(self.tree), 7)

    def test_delete_leaf(self):
        self.assertTrue(self.tree.delete(1))
        self.assertEqual(self.tree.inorder(), [3, 4, 5, 7, 8, 9])

    def test_delete_node_with_two_children(self):
        self.assertTrue(self.tree.delete(3))
        self.assertEqual(self.tree.inorder(), [1, 4, 5, 7, 8, 9])
        self.assertEqual(len(self.tree), 6)

    def test_delete_missing_key(self):
        self.assertFalse(self.tree.delete(100))
        self.assertEqual(len(self.tree), 7)

    def test_height_of_chain_equals_size(self):
        self.assertEqual(BinarySearchTree(range(50)).height(), 50)

    def test_range_keys(self):
        self.assertEqual(self.tree.range_keys(3, 7), [3, 4, 5, 7])
        self.assertEqual(self.tree.range_keys(10, 20), [])

    def test_random_workload_matches_sorted_set(self):
        rng = random.Random(9)
        tree = BinarySearchTree()
        mirror: set[int] = set()
        for _ in range(300):
            key = rng.randint(0, 50)
            if rng.random() < 0.6:
                self.assertEqual(tree.insert(key), key not in mirror)
                mirror.add(key)
            else:
                self.assertEqual(tree.delete(key), key in mirror)
                mirror.discard(key)
            with self.subTest(size=len(mirror)):
                self.assertEqual(tree.inorder(), sorted(mirror))
                self.assertEqual(len(tree), len(mirror))

    def test_random_keys_are_shallower_than_sorted(self):
        random_height, sorted_height = heights(200)
        self.assertLess(random_height, sorted_height)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 45 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python height_demo.py`. Copy the three lines into `docs/bst-notes.md` and explain why the sorted height grows with n while the random height grows slowly.
3. Add a `range_keys` test that checks an interval whose low end is smaller than every key, and one whose high end is larger than every key.
4. Add a `preorder`-based `copy` method that builds a new `BinarySearchTree` with the same keys, and test that `inorder()` on the copy equals the original.
5. Write `docs/bst-notes.md` with one paragraph on why the degenerate case is O(n) and one paragraph naming a balanced tree (AVL or red-black) that fixes it, with a sentence saying what rotation does. Do not implement the balanced tree in this day.
6. Commit with the message "Add binary search tree, traversals and range queries", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python height_demo.py` prints three lines, and the sorted height equals n for each size
- The randomized test matches a Python set on 300 operations
- `docs/bst-notes.md` explains the degenerate case with numbers
- The latest commit on `main` has a green check

Evidence to commit: `dsa/bst.py`, `height_demo.py`, `tests/test_bst.py`, `docs/bst-notes.md` and the Actions run. The sorted-height numbers are the evidence that the structure has a limit, and the randomized test is the evidence that delete is correct.

This project is light on new code and heavy on careful reasoning about shape. The index you built is the same idea that database engines use, with more engineering around it.

---

## Check your understanding

1. A binary search tree has 7 nodes and a height of 7. What does its shape look like, and what is the cost of searching for a missing key?
2. Why does inorder traversal of a BST return sorted keys, and why would the same claim be false for preorder?
3. Explain why deleting a node with two children copies the successor's key instead of deleting the node and rebuilding its subtrees.
4. In `delete`, after copying the successor key into the node, why must you still call `_delete` on the right subtree?
5. A teammate says a recursive `height` function is O(n) and therefore a bad idea. Give the correct time complexity, and state the real limit of the recursive version.
6. Why does `range_keys` skip the left subtree when `low` is not less than the node's key, and what would go wrong if it skipped the right subtree when the node's key is already at least `high`?

### Answers

1. A height of 7 with 7 nodes means each node has exactly one child, so the tree is a chain. A search for a missing key follows the chain to the end, which costs O(n), or 7 comparisons here.
2. Each node's left subtree holds smaller keys and its right subtree holds larger keys, so visiting left, node, right visits keys in increasing order. Preorder visits the node before its subtrees, so the root comes first, which is not sorted order in general.
3. Replacing the key keeps the node in place, so its parent and its subtrees do not need to change. The successor is the smallest key on the right side, so it can replace the node's key without breaking the BST rule, and it has no left child, so its own removal is simple.
4. The successor's key still exists in the right subtree, so it must be removed there. Without the call, the same key would appear twice and the size would be wrong.
5. The time complexity of `height` is O(n), because it visits every node once, and it is correct to call that linear. The real limit is the call stack: a degenerate tree deeper than the recursion limit raises `RecursionError`, so the recursive version is not safe for arbitrary input.
6. If `low` is not less than the node's key, all keys in the left subtree are smaller than the node, so they are also below `low`, and skipping them is safe. If the node's key is already at least `high`, all keys in the right subtree are larger, so skipping them is safe. Skipping the right subtree when the node is already at or above `high` is correct, but skipping it when the node is below `high` would drop matching keys.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Tree (data structure)](https://en.wikipedia.org/wiki/Tree_(data_structure))
- [Wikipedia: Binary search tree](https://en.wikipedia.org/wiki/Binary_search_tree)
- [Wikipedia: Tree traversal](https://en.wikipedia.org/wiki/Tree_traversal)
- [Wikipedia: AVL tree](https://en.wikipedia.org/wiki/AVL_tree) and [Wikipedia: Red-black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree), the two balanced variants to read after this day
- The Python documentation for `sys.getrecursionlimit`, at docs.python.org/3/library/sys.html

*Tutorials*

- GeeksforGeeks, "Binary Search Tree Data Structure" and "Tree Traversals" tutorials. Search the site for each title.
- W3Schools, the Python section on recursion, which covers the base case and the call stack. Search W3Schools for "Python recursion".

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on binary search trees and balanced trees. Search the course site for "binary search trees".
- William Fiset on YouTube, the playlist on trees and binary search trees. Search YouTube for "William Fiset binary search tree".

*Practice*

- VisuAlgo at visualgo.net. Open the BST visualization, insert the keys from exercise 3 and watch the shape change.
- Python Tutor at pythontutor.com. Paste the `_delete` function from this day and step through a delete with two children.

---

## Tomorrow

Day 47 covers heaps and priority queues. A heap is a tree with a different rule: a parent is always smaller than its children, which lets you find the minimum in O(1) and remove it in O(log n). It is stored in an array rather than with node references, and it is the structure behind task schedulers and top-k queries. The height argument from today applies again, because a heap is a complete binary tree, so its height is always about log2 n.
