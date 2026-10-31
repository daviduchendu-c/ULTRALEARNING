# Day 48: Graphs

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 to 4 hours
**Level:** intermediate. You should have finished Day 47, be comfortable with trees and queues, and be ready to reason about visited sets. This day is for anyone who has worked on a social network, a route planner, a dependency graph or a build system and wanted to know what the code underneath does.

**By the end of this day, you will have:**

- A `Graph` class that stores directed or undirected edges as adjacency lists, with a matrix view for comparison
- Breadth-first search (BFS) that returns shortest edge counts, and depth-first search (DFS) that returns a visit order
- A count of connected components and a cycle check for undirected graphs
- A cycle check and topological order for directed graphs, using Kahn's algorithm
- A build-order tool that uses the directed graph to order tasks and to report circular dependencies, committed with CI

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 47
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 44: Stacks and Queues (the stack and queue behavior that DFS and BFS rely on)
- Day 46: Trees and BSTs (tree traversals, which are a special case of graph search)
- Day 47: Heaps and Priority Queues (the priority queue used in shortest weighted paths, which this day only mentions)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Representations](#representations)
  - [Breadth-first search](#breadth-first-search)
  - [Depth-first search](#depth-first-search)
  - [Connected components](#connected-components)
  - [Cycles](#cycles)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a build order tool with cycle detection](#project-a-build-order-tool-with-cycle-detection)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A tree is a special kind of graph. In a tree, every node has one parent and there are no cycles. Most real relationships do not look like that. Friends on a social network point to each other in loops. Roads connect cities in a web with many routes between the same pair. Packages depend on other packages, and sometimes two of them depend on each other. A graph is the general model for all of these: a set of vertices, which are the things, and a set of edges, which are the relationships between them.

Graph algorithms answer the questions people actually ask. Can I reach this user from that one, and in how many hops? Which users form separate groups? Does this build plan have a loop that makes it impossible? Which order should I install packages in? Each question maps to one of the operations in this day: a search, a component count, a cycle check or a topological order. The same few ideas answer all of them, which is why graphs are one of the most reused structures in software.

The outline asks for representations, BFS, DFS, connected components and cycles. This day builds all five on one `Graph` class, and the project uses the directed half of that class to order build tasks safely.

---

## Glossary

- **Vertex (node):** one of the things in a graph.
- **Edge:** a connection between two vertices. It can be directed (from one to another) or undirected (in both directions).
- **Directed graph:** a graph where each edge has a direction.
- **Undirected graph:** a graph where each edge goes both ways.
- **Adjacency list:** a representation that stores, for each vertex, the list of vertices it connects to.
- **Adjacency matrix:** a representation that stores a grid of size V by V, where entry (u, v) says whether an edge exists.
- **Path:** a sequence of vertices where each consecutive pair is joined by an edge.
- **Breadth-first search (BFS):** a search that visits all vertices at distance 1, then 2, and so on. It finds shortest paths in edge count.
- **Depth-first search (DFS):** a search that follows one path as far as it goes before backtracking.
- **Connected component:** a maximal set of vertices where each vertex can reach every other one. In a directed graph, this sense is called strongly connected, which this day does not cover.
- **Cycle:** a path that starts and ends at the same vertex, with at least one edge.
- **Topological order:** an ordering of the vertices of a directed acyclic graph so that every edge goes from an earlier vertex to a later one.
- **Visited set:** the set of vertices a search has already seen, which prevents infinite loops.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Representations

Imagine a city map. One way to record it is a list for each intersection, naming the streets that leave it. Another way is a large table with a row and column for every intersection, and a mark wherever a street connects two of them. The map is the same in both cases, but the cost of asking questions differs. That is the choice between an adjacency list and an adjacency matrix.

From first principles, a graph with V vertices and E edges can be stored in two main ways. An adjacency list uses O(V plus E) space, and it lists neighbors in time proportional to the number of neighbors. An adjacency matrix uses O(V squared) space, and it checks whether an edge exists in O(1). For sparse graphs, where E is much smaller than V squared, the list wins on space. For dense graphs, the matrix is simple and fast. Most real graphs are sparse, so the list is the default.

```python
from dsa.graph import Graph

g = Graph(directed=False)
g.add_edge("ann", "ben")
g.add_edge("ben", "cy")
print(g.neighbors("ben"))           # ['ann', 'cy']
print(g.adjacency_matrix())         # rows and columns follow the order of insertion
```

The limitation of an adjacency list is that checking whether a specific edge exists takes time proportional to the neighbor count of its source. The matrix avoids that, but it wastes space on sparse data. A hybrid is common in production, where each vertex keeps a set of neighbors instead of a list, which gives O(1) edge checks at the cost of more memory.

The common mistake is to forget that an undirected edge is stored twice, once in each direction. If you count the entries in the lists and divide by two, you get the edge count. If you forget the second entry, BFS can miss vertices that are reachable only one way. To spot it, add one edge and print both neighbor lists.

Graph representations appear in road routing, in dependency managers that store package graphs and in recommendation systems that store user-item connections.

### Breadth-first search

Think of a rumor in a small town. The person who starts it tells their friends, who tell theirs, and so on. Everyone at distance one hears it before anyone at distance two. That spread is breadth-first search, and it explains why BFS finds the shortest path in edge count: the first time it reaches a vertex is the shortest route to it.

From first principles, BFS keeps a queue of vertices to visit and a record of the distance to each one it has seen. It starts at the source with distance zero, removes the front of the queue, and adds every unseen neighbor with distance one more than the current vertex. Each vertex enters the queue once and each edge is examined once, so the total time is O(V plus E).

```python
from collections import deque

def bfs(self, start) -> dict:
    dist = {start: 0}
    queue = deque([start])
    while queue:
        u = queue.popleft()
        for v in self._adj[u]:
            if v not in dist:
                dist[v] = dist[u] + 1
                queue.append(v)
    return dist
```

The limitation is that BFS counts edges, not weights. If roads have different lengths, the shortest path in hops can be the longest one in distance. Weighted shortest paths need Dijkstra's algorithm with a priority queue, which Day 47 prepared you for, and which this module does not implement.

The common mistake is to mark a vertex as visited when it is removed from the queue instead of when it is added. A vertex can then be queued several times, and the distances can be wrong on graphs with many paths. To spot the problem, test a graph where two paths of different lengths lead to the same vertex.

BFS powers friend-of-friend suggestions, network crawlers that need to visit pages in layers and the level-by-level garbage collection in some runtimes.

### Depth-first search

Picture a maze. You choose a corridor and follow it until you hit a dead end, then go back to the last junction where you had another choice. That is depth-first search. It goes deep first and backtracks when it has no new place to go, which makes it natural for exploring every reachable part of a structure, and for the order in which vertices are discovered.

From first principles, DFS uses a stack instead of a queue. Recursion and an explicit stack do the same job, because a recursive call is a frame on the call stack. The iterative version below pushes neighbors in reverse order, so that the first neighbor in the list is visited first, which matches the recursive order. Like BFS, DFS takes O(V plus E) time.

```python
def dfs(self, start) -> list:
    order = []
    seen = set()
    stack = [start]
    while stack:
        u = stack.pop()
        if u in seen:
            continue
        seen.add(u)
        order.append(u)
        for v in reversed(self._adj[u]):
            if v not in seen:
                stack.append(v)
    return order
```

The limitation of DFS is that it does not give shortest paths, because it may follow a long route before it finds a short one. It is also sensitive to the order of neighbors, so two runs on the same graph can produce different orders if the adjacency lists are built differently.

The common mistake is to check `seen` only when a vertex is popped and forget to check it before pushing, which pushes the same vertex many times. The code above checks both places. The second mistake is to use a recursive DFS on a deep graph and hit Python's recursion limit, which Day 50 explains. To spot it, test a chain of 2000 vertices.

DFS is used in compilers to detect unreachable code, in test runners to discover tests, and in game engines to find all rooms connected to a starting room.

### Connected components

Think of islands on a map. Two land pieces belong to the same island if you can walk from one to the other without a boat. A component is one island, and counting components means counting how many separate islands there are. A search from any land piece finds its whole island, and the next unvisited piece starts a new island.

From first principles, loop over the vertices. For each vertex not yet seen, run a search that marks everything it can reach, and record that set as one component. Every vertex is visited once overall, so the total work is O(V plus E), not O(V times the number of components).

```python
def connected_components(self) -> list[list]:
    seen: set = set()
    components: list[list] = []
    for start in self._adj:
        if start in seen:
            continue
        component = []
        stack = [start]
        seen.add(start)
        while stack:
            u = stack.pop()
            component.append(u)
            for v in self._adj[u]:
                if v not in seen:
                    seen.add(v)
                    stack.append(v)
        components.append(component)
    return components
```

The limitation is that this is the undirected version. In a directed graph, reachability is not symmetric, so the same loop does not find the groups you want. Finding strongly connected components in a directed graph needs a different algorithm, such as Tarjan's.

The common mistake is to count components by counting search calls from the top level, without the check that skips seen vertices. That reports one component per vertex. To spot it, run the function on a graph with two isolated vertices and check that the count is two.

Component analysis is used to find clusters of related users in analytics, to split a large mesh into independent parts for simulation, and to find isolated subsystems in dependency audits.

### Cycles

Think of a set of one-way streets. If you can drive in a loop and come back to the same corner, you have a cycle. Cycles matter in two different ways. In an undirected graph, a cycle means there is more than one route between some pair, which matters for network redundancy. In a directed graph, a cycle means a dependency cannot be satisfied, because each step waits for the next.

From first principles, the undirected test is simple once you know the components. A forest, which is a graph with no cycles, has exactly V minus C edges, where C is the number of components. If the graph has more edges than that, some edge must close a loop. The directed test uses Kahn's algorithm: repeatedly remove a vertex with no incoming edges. If every vertex can be removed, there is no cycle. If some remain, each one waits on another, which is a cycle.

```python
def has_cycle_undirected(self) -> bool:
    edges = sum(len(ns) for ns in self._adj.values()) // 2
    return edges > len(self._adj) - len(self.connected_components())

def _kahn_order(self) -> list:
    indegree = {v: 0 for v in self._adj}
    for u in self._adj:
        for v in self._adj[u]:
            indegree[v] += 1
    queue = deque(v for v, d in indegree.items() if d == 0)
    order = []
    while queue:
        u = queue.popleft()
        order.append(u)
        for v in self._adj[u]:
            indegree[v] -= 1
            if indegree[v] == 0:
                queue.append(v)
    return order
```

The limitation of the edge-count test is that it works only for undirected graphs, and that it treats a self-loop or a duplicate edge as a cycle, which is usually what you want but must be stated. The limitation of Kahn's algorithm is that it finds whether a cycle exists, but it does not name the cycle. For that you need a DFS with a color marker on each vertex, which is a useful extension.

The common mistake is to run Kahn's algorithm on an undirected graph. Each undirected edge becomes two directed edges, so every vertex with a neighbor has indegree at least one, and the algorithm reports a cycle on any graph with an edge. The code in this day raises an error in that case. To spot it, test a single edge and see what the function returns.

Cycle detection appears in package managers, which refuse to install circular dependencies, in spreadsheet engines, which report circular references, and in compilers, which reject circular type definitions.

---

## Hands-on exercises

Work in the repository from Day 47. Keep the graph in `dsa/graph.py` and the build order tool in `dsa/build_order.py`.

Exercise 1 traces BFS by hand. Draw a graph with six vertices and seven edges on paper. Run BFS from one vertex by writing the queue and the distance table after each step. Compare your distances with `g.bfs(start)`.

Done when your distance table matches the code, and the queue contents match the order vertices were added.

Exercise 2 traces DFS. Using the same graph, write the order DFS visits the vertices, following the rule that the first neighbor in the list goes first. Compare with `g.dfs(start)`.

Done when the orders match, and you can explain one case where BFS and DFS visit the same vertex in a different position.

Exercise 3 counts components. Build a graph with 10 vertices and 6 edges that forms three components, and check `len(g.connected_components())` returns 3. Then add one edge between two components and check the count drops to 2.

Done when both counts match and your notes state the rule for each change.

Exercise 4 tests cycles. Write tests for `has_cycle_undirected` on a tree, a triangle, two disjoint edges, and a single self-loop. Then write tests for `has_cycle_directed` on a three-vertex chain, a directed three-cycle, and a graph where two paths meet at a vertex without a cycle.

Done when all the cases pass, and the notes name the result of each case in one line.

Exercise 5 compares matrix and list. Build the same 50-vertex random graph with 100 edges using `random.Random(6)`. Print the number of entries in the adjacency matrix and the number of entries in the adjacency lists. Record both in `docs/graph-notes.md` and explain which is smaller.

Done when your notes show both counts and a sentence explaining why the list is smaller for a sparse graph.

---

## Project: a build order tool with cycle detection

The project is a small build order tool. You give it a list of pairs, where each pair names a task and one of its prerequisites. It returns an order in which every task comes after everything it depends on. If the dependencies form a loop, it reports the problem instead of returning a wrong order. Build systems, package managers and course planners all solve this same problem.

The tool uses the directed graph from this day. Each prerequisite points to the tasks that depend on it, and the topological order from Kahn's algorithm gives the build sequence.

Suggested repo name: `100-days-software-day-48-graphs`. Copy the Day 47 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-48-graphs/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── build_demo.py
├── count_words.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   ├── bst.py
│   ├── browser_history.py
│   ├── build_order.py
│   ├── complexity.py
│   ├── graph.py
│   ├── hash_table.py
│   ├── heap.py
│   ├── linked_list.py
│   ├── scheduler.py
│   ├── stacks_queues.py
│   └── word_count.py
├── docs/
│   ├── bst-notes.md
│   ├── graph-notes.md
│   ├── hash-notes.md
│   └── heap-notes.md
├── height_demo.py
├── print_queue.py
├── schedule_demo.py
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   ├── test_bst.py
│   ├── test_browser_history.py
│   ├── test_build_order.py
│   ├── test_complexity.py
│   ├── test_graph.py
│   ├── test_hash_table.py
│   ├── test_heap.py
│   ├── test_linked_list.py
│   ├── test_print_queue.py
│   ├── test_scheduler.py
│   ├── test_stacks_queues.py
│   └── test_word_count.py
└── README.md
```

Starter code for `dsa/graph.py`:

```python
from __future__ import annotations

from collections import deque
from collections.abc import Hashable


class Graph:
    """A graph stored as adjacency lists. Set directed=True for one-way edges."""

    def __init__(self, directed: bool = False) -> None:
        self.directed = directed
        self._adj: dict[Hashable, list[Hashable]] = {}

    def add_vertex(self, v: Hashable) -> None:
        self._adj.setdefault(v, [])

    def add_edge(self, u: Hashable, v: Hashable) -> None:
        self.add_vertex(u)
        self.add_vertex(v)
        self._adj[u].append(v)
        if not self.directed:
            self._adj[v].append(u)

    def neighbors(self, v: Hashable) -> list[Hashable]:
        return self._adj[v]

    def vertices(self) -> list[Hashable]:
        return list(self._adj)

    def adjacency_matrix(self) -> list[list[int]]:
        order = list(self._adj)
        index = {v: i for i, v in enumerate(order)}
        matrix = [[0] * len(order) for _ in order]
        for u in order:
            for v in self._adj[u]:
                matrix[index[u]][index[v]] = 1
        return matrix

    def bfs(self, start: Hashable) -> dict[Hashable, int]:
        """Edge count from start to every reachable vertex."""
        dist = {start: 0}
        queue = deque([start])
        while queue:
            u = queue.popleft()
            for v in self._adj[u]:
                if v not in dist:
                    dist[v] = dist[u] + 1
                    queue.append(v)
        return dist

    def dfs(self, start: Hashable) -> list[Hashable]:
        """Vertices in the order a depth-first search first reaches them."""
        order: list[Hashable] = []
        seen: set[Hashable] = set()
        stack = [start]
        while stack:
            u = stack.pop()
            if u in seen:
                continue
            seen.add(u)
            order.append(u)
            for v in reversed(self._adj[u]):
                if v not in seen:
                    stack.append(v)
        return order

    def connected_components(self) -> list[list[Hashable]]:
        seen: set[Hashable] = set()
        components: list[list[Hashable]] = []
        for start in self._adj:
            if start in seen:
                continue
            component: list[Hashable] = []
            stack = [start]
            seen.add(start)
            while stack:
                u = stack.pop()
                component.append(u)
                for v in self._adj[u]:
                    if v not in seen:
                        seen.add(v)
                        stack.append(v)
            components.append(component)
        return components

    def has_cycle_undirected(self) -> bool:
        if self.directed:
            raise ValueError("use has_cycle_directed for directed graphs")
        edges = sum(len(ns) for ns in self._adj.values()) // 2
        return edges > len(self._adj) - len(self.connected_components())

    def _kahn_order(self) -> list[Hashable]:
        if not self.directed:
            raise ValueError("topological order needs a directed graph")
        indegree: dict[Hashable, int] = {v: 0 for v in self._adj}
        for u in self._adj:
            for v in self._adj[u]:
                indegree[v] += 1
        queue = deque(v for v, d in indegree.items() if d == 0)
        order: list[Hashable] = []
        while queue:
            u = queue.popleft()
            order.append(u)
            for v in self._adj[u]:
                indegree[v] -= 1
                if indegree[v] == 0:
                    queue.append(v)
        return order

    def has_cycle_directed(self) -> bool:
        return len(self._kahn_order()) < len(self._adj)

    def topological_order(self) -> list[Hashable]:
        order = self._kahn_order()
        if len(order) < len(self._adj):
            raise ValueError("graph has a cycle")
        return order
```

Starter code for `dsa/build_order.py`:

```python
from dsa.graph import Graph


def build_order(dependencies: list[tuple[str, str]]) -> list[str]:
    """Each pair (task, prerequisite) means the prerequisite must finish first.
    Return an order where every task comes after its prerequisites."""
    graph = Graph(directed=True)
    for task, prerequisite in dependencies:
        graph.add_edge(prerequisite, task)
    return graph.topological_order()
```

Starter code for `build_demo.py`:

```python
from dsa.build_order import build_order

if __name__ == "__main__":
    deps = [
        ("serve", "bake"),
        ("bake", "mix"),
        ("mix", "buy"),
        ("frost", "bake"),
    ]
    print(" -> ".join(build_order(deps)))
```

Running `python build_demo.py` prints `buy -> mix -> bake -> serve -> frost`. Check that answer by hand: buy has no prerequisites, so it is first, and each later task appears after the task it needs.

Starter code for `tests/test_graph.py`:

```python
import random
import unittest

from dsa.graph import Graph


def sample_graph() -> Graph:
    g = Graph()
    for u, v in [("a", "b"), ("a", "c"), ("b", "d"), ("c", "d"), ("d", "e")]:
        g.add_edge(u, v)
    return g


class TraversalTests(unittest.TestCase):
    def test_bfs_distances(self):
        self.assertEqual(sample_graph().bfs("a"), {"a": 0, "b": 1, "c": 1, "d": 2, "e": 3})

    def test_dfs_order_follows_first_neighbor(self):
        self.assertEqual(sample_graph().dfs("a"), ["a", "b", "d", "c", "e"])

    def test_adjacency_matrix_is_symmetric_for_undirected(self):
        m = sample_graph().adjacency_matrix()
        for i in range(len(m)):
            for j in range(len(m)):
                self.assertEqual(m[i][j], m[j][i])

    def test_bfs_random_graph_edges_differ_by_at_most_one(self):
        rng = random.Random(6)
        g = Graph()
        for _ in range(120):
            g.add_edge(rng.randint(0, 39), rng.randint(0, 39))
        dist = g.bfs(0)
        for u in dist:
            for v in g.neighbors(u):
                self.assertLessEqual(abs(dist[u] - dist[v]), 1)


class ComponentTests(unittest.TestCase):
    def test_three_components(self):
        g = Graph()
        g.add_edge(1, 2)
        g.add_edge(3, 4)
        g.add_vertex(5)
        components = [set(c) for c in g.connected_components()]
        self.assertEqual(len(components), 3)
        self.assertIn({5}, components)

    def test_joining_components_reduces_count(self):
        g = Graph()
        g.add_edge(1, 2)
        g.add_edge(3, 4)
        g.add_edge(2, 3)
        self.assertEqual(len(g.connected_components()), 1)


class CycleTests(unittest.TestCase):
    def test_undirected_tree_has_no_cycle(self):
        g = Graph()
        g.add_edge(1, 2)
        g.add_edge(1, 3)
        self.assertFalse(g.has_cycle_undirected())

    def test_undirected_triangle_has_cycle(self):
        g = Graph()
        for u, v in [(1, 2), (2, 3), (3, 1)]:
            g.add_edge(u, v)
        self.assertTrue(g.has_cycle_undirected())

    def test_undirected_disjoint_edges_have_no_cycle(self):
        g = Graph()
        g.add_edge(1, 2)
        g.add_edge(3, 4)
        self.assertFalse(g.has_cycle_undirected())

    def test_directed_chain_has_no_cycle(self):
        g = Graph(directed=True)
        g.add_edge("a", "b")
        g.add_edge("b", "c")
        self.assertFalse(g.has_cycle_directed())
        self.assertEqual(g.topological_order(), ["a", "b", "c"])

    def test_directed_cycle_is_detected(self):
        g = Graph(directed=True)
        for u, v in [("a", "b"), ("b", "c"), ("c", "a")]:
            g.add_edge(u, v)
        self.assertTrue(g.has_cycle_directed())
        with self.assertRaises(ValueError):
            g.topological_order()

    def test_wrong_method_for_graph_kind_raises(self):
        with self.assertRaises(ValueError):
            Graph(directed=True).has_cycle_undirected()
        with self.assertRaises(ValueError):
            Graph().has_cycle_directed()


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_build_order.py`:

```python
import unittest

from dsa.build_order import build_order


class BuildOrderTests(unittest.TestCase):
    def assertRespectsDependencies(self, order, dependencies):
        position = {task: i for i, task in enumerate(order)}
        for task, prerequisite in dependencies:
            self.assertLess(position[prerequisite], position[task])

    def test_order_respects_every_dependency(self):
        deps = [("serve", "bake"), ("bake", "mix"), ("mix", "buy"), ("frost", "bake")]
        order = build_order(deps)
        self.assertEqual(set(order), {"serve", "bake", "mix", "buy", "frost"})
        self.assertRespectsDependencies(order, deps)

    def test_circular_dependency_is_reported(self):
        with self.assertRaises(ValueError):
            build_order([("a", "b"), ("b", "a")])

    def test_no_dependencies(self):
        self.assertEqual(sorted(build_order([])), [])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 47 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python build_demo.py` and confirm the output matches the order you worked out by hand.
3. Add a `shortest_path(self, start, goal)` method to `Graph` that uses the `bfs` distances and walks backward from the goal. Keep a parent dictionary during BFS, and return the list of vertices on one shortest path, or `None` if the goal is unreachable. Add tests for a reachable goal, an unreachable goal and the start equal to the goal.
4. Extend `build_order` so that a task with a missing prerequisite is still handled. Decide whether the missing prerequisite should become a vertex, and write a test that states your choice.
5. Write `docs/graph-notes.md` with the matrix and list counts from exercise 5, one sentence on the cycle test for undirected graphs, and one sentence on why Kahn's algorithm names no cycle.
6. Commit with the message "Add graph, traversals and build order tool", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python build_demo.py` prints `buy -> mix -> bake -> serve -> frost`
- The build order test checks every dependency, and the cycle test raises `ValueError`
- `docs/graph-notes.md` contains the counts from exercise 5 and explains the sparse-graph choice
- The latest commit on `main` has a green check

Evidence to commit: `dsa/graph.py`, `dsa/build_order.py`, `build_demo.py`, both test files, `docs/graph-notes.md` and the Actions run. The dependency check in the build order test proves that the output is a valid order.

This project is light on new theory and heavy on the difference between a graph that is a DAG and one that is not. The skill that transfers is recognizing dependency and cycle problems and choosing the directed or undirected model that fits them.

---

## Check your understanding

1. A graph has 100 vertices and 150 undirected edges. Compare the space for an adjacency list and an adjacency matrix, and say which you would choose and why.
2. Why does BFS find the shortest path in edge count, but DFS does not?
3. A teammate counts connected components by calling `dfs` from every vertex and counting the calls. Explain why the count is wrong, and what check fixes it.
4. For an undirected graph with 8 vertices, 3 components and 6 edges, does it contain a cycle? Show the calculation with the formula used in this day.
5. A build tool runs Kahn's algorithm and finds that 3 of 10 tasks are never added to the output. What does that tell you about the graph, and what would you show the user?
6. Why does `has_cycle_undirected` raise an error on a directed graph, and what would the edge-count formula report if it did not?

### Answers

1. The adjacency list uses about V plus 2E entries, which is about 100 plus 300 equals 400 entries. The matrix uses V squared, which is 10000 cells. The graph is sparse, since 150 edges is far fewer than 100 squared, so the list is much smaller, and it is the right choice.
2. BFS explores all vertices at distance k before any at distance k plus one, so the first time it reaches a vertex, it has used the fewest edges. DFS follows one path to its end before trying others, so the first path it finds can be long, even when a shorter one exists.
3. Each call from a vertex already in the same component finds that component again, so the count is the number of vertices that were not reached by an earlier call, not the number of components. The fix is to check the visited set before starting a new search, as the `connected_components` method does.
4. Use the formula: edges should be V minus C, which is 8 minus 3, or 5, for a forest. The graph has 6 edges, which is more than 5, so it contains at least one cycle.
5. The three tasks that were never added are in or downstream of a cycle, because each one waits for a predecessor that never gets indegree zero. The user should see the names of the tasks left over, and ideally the cycle among them, so they can fix the dependency that closes the loop.
6. An undirected edge is stored in both directions, so Kahn's algorithm would see a cycle at every edge, and the directed test would be wrong. The edge-count formula would also be meaningless, because it assumes each edge is counted once and that the graph has no direction to follow.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Graph (abstract data type)](https://en.wikipedia.org/wiki/Graph_(abstract_data_type))
- [Wikipedia: Breadth-first search](https://en.wikipedia.org/wiki/Breadth-first_search)
- [Wikipedia: Depth-first search](https://en.wikipedia.org/wiki/Depth-first_search)
- [Wikipedia: Topological sorting](https://en.wikipedia.org/wiki/Topological_sorting), which covers Kahn's algorithm
- [Wikipedia: Connected component (graph theory)](https://en.wikipedia.org/wiki/Connected_component_(graph_theory))
- The Python documentation for `collections.deque`, at docs.python.org/3/library/collections.html

*Tutorials*

- GeeksforGeeks, "Graph and its representations" and "Breadth First Search or BFS for a Graph". Search the site for each title.
- GeeksforGeeks, "Detect Cycle in a Directed Graph" and "Topological Sorting". Search the site for each title.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on graph search and on topological sort. Search the course site for "breadth-first search".
- William Fiset on YouTube, the playlist on graph algorithms, including BFS, DFS and topological sort. Search YouTube for "William Fiset graph algorithms".

*Practice*

- VisuAlgo at visualgo.net, the graph visualization for DFS, BFS and topological sort. Run each one on the sample graph from this day and compare the order with your answer.
- LeetCode, the "Graph" study plan at leetcode.com. Start with "Number of Islands" for components and "Course Schedule" for directed cycle detection.

---

## Tomorrow

Day 49 covers sorting. You have now used sorting several times without building one, including the sorted lists in the top-k checks and the tests that compare results to `sorted`. Day 49 builds merge sort, quicksort and heap sort, compares their complexity, and explains stability, which is the property that equal items keep their original order. The recursion pattern from the tree and graph days returns in merge sort, which splits the input in half at each step.
