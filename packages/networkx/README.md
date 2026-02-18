# networkx

A comprehensive Python library for the creation, manipulation, and study of **complex networks** and **graphs**.

`networkx` provides data structures for graphs (undirected and directed, with and without parallel edges), a large collection of standard graph algorithms, network structure and analysis measures, generators for classic graphs and random graphs, and facilities for reading/writing graphs in many formats. It is the de facto standard for graph and network analysis in the Python ecosystem.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `networkx` |
| **Latest version** | 3.4.x (3.4.2 as of late 2024 / early 2025) |
| **Python support** | Python >= 3.10 |
| **License** | BSD 3-Clause |
| **Repository** | https://github.com/networkx/networkx |
| **Documentation** | https://networkx.org/documentation/stable/ |

NetworkX 3.x dropped support for older Python versions and embraced modern Python features. The API is mature and stable, with occasional deprecations communicated well in advance.

---

## Installation

```bash
# Standard installation
pip install networkx

# With all optional dependencies (matplotlib, scipy, pandas, etc.)
pip install networkx[default]

# With specific extras
pip install networkx[extra]    # lxml, pygraphviz, pydot, sympy
pip install networkx[test]     # pytest and test dependencies
pip install networkx[doc]      # sphinx and documentation dependencies
```

NetworkX itself is **pure Python** with zero required dependencies. Optional dependencies unlock additional features:

| Dependency | Purpose |
|-----------|---------|
| `scipy` | Sparse matrix representations, some algorithms |
| `matplotlib` | Graph drawing and visualization |
| `pandas` | DataFrame-based graph construction |
| `numpy` | Array-based operations, some algorithms |
| `pygraphviz` | Interface to Graphviz layout programs |
| `pydot` | Alternative Graphviz interface |
| `lxml` | GraphML and GML reading/writing |

---

## Core Graph Types

NetworkX provides four graph classes covering all combinations of directed/undirected and single/multi-edge:

| Class | Directed | Parallel Edges | Self-Loops |
|-------|----------|----------------|------------|
| `Graph` | No | No | Yes |
| `DiGraph` | Yes | No | Yes |
| `MultiGraph` | No | Yes | Yes |
| `MultiDiGraph` | Yes | Yes | Yes |

### `Graph` -- Undirected Graph

The simplest graph type. Edges have no direction, and at most one edge can exist between any pair of nodes.

```python
import networkx as nx

G = nx.Graph()
G.add_edge("A", "B")
G.add_edge("B", "C")

# Edges are bidirectional
assert G.has_edge("A", "B")
assert G.has_edge("B", "A")  # same edge

# No parallel edges -- adding the same edge again updates attributes
G.add_edge("A", "B", weight=3.0)
print(G["A"]["B"]["weight"])  # 3.0
```

### `DiGraph` -- Directed Graph

Edges have direction. An edge from A to B is distinct from an edge from B to A.

```python
D = nx.DiGraph()
D.add_edge("A", "B")

assert D.has_edge("A", "B")
assert not D.has_edge("B", "A")  # direction matters

# Predecessors and successors
D.add_edge("C", "A")
print(list(D.predecessors("A")))  # ["C"]
print(list(D.successors("A")))    # ["B"]

# In-degree and out-degree
print(D.in_degree("A"))   # 1
print(D.out_degree("A"))  # 1
```

### `MultiGraph` -- Undirected Graph with Parallel Edges

Allows multiple edges between the same pair of nodes, distinguished by integer keys.

```python
M = nx.MultiGraph()
k1 = M.add_edge("A", "B", weight=1.0)  # returns key 0
k2 = M.add_edge("A", "B", weight=2.0)  # returns key 1

print(M.number_of_edges())  # 2

# Access specific edges by key
print(M["A"]["B"][0]["weight"])  # 1.0
print(M["A"]["B"][1]["weight"])  # 2.0
```

### `MultiDiGraph` -- Directed Graph with Parallel Edges

Combines direction with parallel edges. Useful for modeling transport networks, multigraphs, or any system with multiple typed relationships.

```python
MD = nx.MultiDiGraph()
MD.add_edge("A", "B", relation="friend")
MD.add_edge("A", "B", relation="colleague")
MD.add_edge("B", "A", relation="friend")

print(MD.number_of_edges())  # 3
```

### Converting Between Graph Types

```python
G = nx.Graph()
G.add_edges_from([("A", "B"), ("B", "C"), ("C", "A")])

# Undirected -> Directed (each edge becomes two directed edges)
D = G.to_directed()
print(D.number_of_edges())  # 6 (2 per undirected edge)

# Directed -> Undirected (directed edges merged)
G2 = D.to_undirected()
print(G2.number_of_edges())  # 3

# Any graph -> MultiGraph
M = nx.MultiGraph(G)

# Any graph -> MultiDiGraph
MD = nx.MultiDiGraph(G)
```

---

## Graph Manipulation

### Adding Nodes

Nodes can be any hashable Python object: strings, integers, tuples, frozensets, or custom objects with `__hash__` and `__eq__`.

```python
G = nx.Graph()

# Single node
G.add_node(1)
G.add_node("server-01")
G.add_node((0, 1))  # tuples are hashable

# Multiple nodes
G.add_nodes_from([2, 3, 4, 5])
G.add_nodes_from(range(10, 15))

# Nodes with attributes
G.add_node("server-01", role="web", cpu_cores=8)
G.add_nodes_from([
    ("server-02", {"role": "db", "cpu_cores": 16}),
    ("server-03", {"role": "cache", "cpu_cores": 4}),
])

# Nodes are automatically created when adding edges
G.add_edge("X", "Y")  # creates nodes "X" and "Y" if they don't exist
```

### Adding Edges

```python
G = nx.Graph()

# Single edge
G.add_edge("A", "B")
G.add_edge("A", "B", weight=4.5, color="red")

# Multiple edges
G.add_edges_from([("A", "C"), ("B", "C"), ("C", "D")])

# Edges with attributes
G.add_edges_from([
    ("A", "B", {"weight": 1.0}),
    ("B", "C", {"weight": 2.5}),
    ("C", "D", {"weight": 0.8}),
])

# Weighted edges shorthand
G.add_weighted_edges_from([
    ("A", "B", 1.0),
    ("B", "C", 2.5),
    ("C", "D", 0.8),
])
```

### Removing Nodes and Edges

```python
G = nx.Graph()
G.add_edges_from([("A", "B"), ("B", "C"), ("C", "D"), ("A", "D")])

# Remove a single node (also removes all its edges)
G.remove_node("D")
print(G.edges())  # [("A", "B"), ("B", "C")]

# Remove multiple nodes
G.add_nodes_from(["X", "Y", "Z"])
G.remove_nodes_from(["X", "Y", "Z"])

# Remove a single edge (nodes remain)
G.remove_edge("A", "B")
print(list(G.nodes()))  # ["A", "B", "C"] -- nodes still present

# Remove multiple edges
G.add_edges_from([("A", "B"), ("A", "C")])
G.remove_edges_from([("A", "B"), ("A", "C")])

# Clear everything
G.clear()        # remove all nodes and edges
G.clear_edges()  # remove only edges, keep nodes
```

### Node and Edge Attributes

```python
G = nx.Graph()
G.add_node("A", color="red", size=10)
G.add_edge("A", "B", weight=3.5, label="primary")

# Read node attributes
print(G.nodes["A"]["color"])  # "red"
print(G.nodes["A"])           # {"color": "red", "size": 10}

# Read edge attributes
print(G["A"]["B"]["weight"])        # 3.5
print(G.edges["A", "B"]["label"])   # "primary"

# Update attributes
G.nodes["A"]["color"] = "blue"
G.edges["A", "B"]["weight"] = 5.0

# Set attributes in bulk
nx.set_node_attributes(G, {"A": "blue", "B": "green"}, name="color")
nx.set_edge_attributes(G, {("A", "B"): 5.0}, name="weight")

# Get attributes in bulk
colors = nx.get_node_attributes(G, "color")    # {"A": "blue", "B": "green"}
weights = nx.get_edge_attributes(G, "weight")   # {("A", "B"): 5.0}
```

### Graph-Level Attributes

```python
G = nx.Graph(name="My Network", created="2025-01-01")
print(G.graph)  # {"name": "My Network", "created": "2025-01-01"}

G.graph["description"] = "A sample graph"
```

### Iterating Over Nodes, Edges, and Neighbors

```python
G = nx.Graph()
G.add_edges_from([
    ("A", "B", {"weight": 1}),
    ("B", "C", {"weight": 2}),
    ("C", "A", {"weight": 3}),
])
G.nodes["A"]["role"] = "server"

# Iterate over nodes
for node in G.nodes():
    print(node)

# Nodes with data
for node, data in G.nodes(data=True):
    print(node, data)
# A {'role': 'server'}
# B {}
# C {}

# Iterate over edges
for u, v in G.edges():
    print(u, v)

# Edges with data
for u, v, data in G.edges(data=True):
    print(u, v, data["weight"])

# Edges with specific attribute (default if missing)
for u, v, w in G.edges(data="weight", default=0):
    print(f"{u}-{v}: {w}")

# Neighbors of a node
for neighbor in G.neighbors("A"):
    print(neighbor)  # "B", "C"

# Adjacency (dict-of-dicts)
for node, neighbors in G.adjacency():
    print(node, dict(neighbors))

# Degree
print(G.degree("A"))            # 2
print(dict(G.degree()))         # {"A": 2, "B": 2, "C": 2}
print(G.degree("A", weight="weight"))  # 4 (weighted degree: 1 + 3)
```

### Subgraphs and Views

NetworkX provides lightweight **view** objects that reference the original graph without copying data:

```python
G = nx.path_graph(10)  # 0-1-2-3-4-5-6-7-8-9

# Subgraph view (no data copy -- changes to G are reflected)
H = G.subgraph([0, 1, 2, 3, 4])
print(list(H.edges()))  # [(0,1), (1,2), (2,3), (3,4)]

# Subgraph view is read-only for structure
# H.add_node(99)  # raises NetworkXError

# To get a mutable subgraph, copy it
H_mutable = G.subgraph([0, 1, 2, 3, 4]).copy()
H_mutable.add_node(99)  # works fine

# Edge subgraph
E = G.edge_subgraph([(0, 1), (1, 2), (2, 3)])
print(list(E.nodes()))  # [0, 1, 2, 3]

# Reverse view for DiGraph (no copy, O(1))
D = nx.DiGraph([(1, 2), (2, 3)])
R = D.reverse(copy=False)  # view -- changes to D are reflected
print(list(R.edges()))  # [(2, 1), (3, 2)]
```

---

## Key Algorithms

### Shortest Path

```python
G = nx.Graph()
G.add_weighted_edges_from([
    ("A", "B", 1), ("B", "C", 2), ("A", "C", 4),
    ("C", "D", 1), ("B", "D", 5),
])

# Unweighted shortest path (BFS)
path = nx.shortest_path(G, source="A", target="D")
print(path)  # ["A", "B", "C", "D"] or ["A", "C", "D"]

# Weighted shortest path (Dijkstra -- default for weighted)
path = nx.shortest_path(G, source="A", target="D", weight="weight")
print(path)  # ["A", "B", "C", "D"] (cost: 1+2+1=4)

# Dijkstra explicitly
path = nx.dijkstra_path(G, source="A", target="D")
length = nx.dijkstra_path_length(G, source="A", target="D")
print(path, length)  # ["A", "B", "C", "D"] 4

# Bellman-Ford (handles negative weights, no negative cycles)
path = nx.bellman_ford_path(G, source="A", target="D")
length = nx.bellman_ford_path_length(G, source="A", target="D")

# All shortest paths between two nodes
all_paths = list(nx.all_shortest_paths(G, source="A", target="D"))

# Shortest path from one source to all targets
lengths = dict(nx.single_source_shortest_path_length(G, "A"))
# {"A": 0, "B": 1, "C": 1, "D": 2}

# All-pairs shortest path length
all_lengths = dict(nx.all_pairs_shortest_path_length(G))

# A* search (requires heuristic)
# nx.astar_path(G, source, target, heuristic=None, weight="weight")
```

### Topological Sort

Topological sorting applies to **directed acyclic graphs (DAGs)** only. It produces a linear ordering of nodes such that for every edge (u, v), u comes before v.

```python
D = nx.DiGraph()
D.add_edges_from([
    ("compile", "link"),
    ("link", "deploy"),
    ("test", "deploy"),
    ("compile", "test"),
])

# Single topological sort (returns an iterator)
order = list(nx.topological_sort(D))
print(order)  # ["compile", "test", "link", "deploy"] or similar valid ordering

# All topological sorts (can be exponentially many)
all_orders = list(nx.all_topological_sorts(D))
for o in all_orders:
    print(o)

# Topological generations (nodes grouped by "level")
generations = list(nx.topological_generations(D))
print(generations)
# [["compile"], ["link", "test"], ["deploy"]] -- depends on structure

# Lexicographic topological sort
order = list(nx.lexicographical_topological_sort(D))
```

### Cycle Detection

```python
D = nx.DiGraph()
D.add_edges_from([("A", "B"), ("B", "C"), ("C", "A")])

# Check if a directed graph is acyclic
print(nx.is_directed_acyclic_graph(D))  # False

# Find a single cycle
try:
    cycle = nx.find_cycle(D)
    print(cycle)  # [("A", "B"), ("B", "C"), ("C", "A")]
except nx.NetworkXNoCycle:
    print("No cycle found")

# Find all simple cycles in a directed graph
cycles = list(nx.simple_cycles(D))
print(cycles)  # [["A", "B", "C"]]

# For undirected graphs, use cycle_basis
G = nx.Graph()
G.add_edges_from([(1, 2), (2, 3), (3, 1), (3, 4), (4, 5), (5, 3)])
basis = nx.cycle_basis(G)
print(basis)  # [[1, 2, 3], [3, 4, 5]] -- basis cycles
```

### Connected Components

```python
# Undirected connectivity
G = nx.Graph()
G.add_edges_from([(1, 2), (2, 3), (4, 5)])

print(nx.is_connected(G))  # False (two components)

components = list(nx.connected_components(G))
print(components)  # [{1, 2, 3}, {4, 5}]

print(nx.number_connected_components(G))  # 2

# Get the component containing a specific node
component = nx.node_connected_component(G, 1)
print(component)  # {1, 2, 3}

# Directed connectivity (strong and weak)
D = nx.DiGraph([(1, 2), (2, 3), (3, 1), (3, 4)])

# Strongly connected: every node reachable from every other in the component
print(nx.is_strongly_connected(D))  # False
strong = list(nx.strongly_connected_components(D))
print(strong)  # [{1, 2, 3}, {4}]

# Weakly connected: connected if we ignore edge direction
print(nx.is_weakly_connected(D))  # True
weak = list(nx.weakly_connected_components(D))
print(weak)  # [{1, 2, 3, 4}]
```

### Centrality Measures

Centrality measures quantify the "importance" of nodes in a network.

```python
G = nx.karate_club_graph()  # Zachary's karate club (classic test graph)

# Degree centrality: fraction of nodes each node is connected to
dc = nx.degree_centrality(G)
top_dc = sorted(dc, key=dc.get, reverse=True)[:5]

# Betweenness centrality: fraction of shortest paths passing through a node
bc = nx.betweenness_centrality(G)
top_bc = sorted(bc, key=bc.get, reverse=True)[:5]

# Closeness centrality: inverse average distance to all other nodes
cc = nx.closeness_centrality(G)

# Eigenvector centrality: influence based on connections to high-scoring nodes
ec = nx.eigenvector_centrality(G, max_iter=1000)

# PageRank (directed graphs, but works on undirected too)
pr = nx.pagerank(G, alpha=0.85)

# For directed graphs
D = nx.DiGraph(G)
in_dc = nx.in_degree_centrality(D)
out_dc = nx.out_degree_centrality(D)
```

### Graph Traversal

```python
G = nx.Graph()
G.add_edges_from([
    (1, 2), (1, 3), (2, 4), (2, 5), (3, 6), (3, 7),
])

# BFS (breadth-first search)
bfs = list(nx.bfs_edges(G, source=1))
print(bfs)  # [(1,2), (1,3), (2,4), (2,5), (3,6), (3,7)]

# BFS tree (returns a directed graph rooted at source)
T = nx.bfs_tree(G, source=1)
print(list(T.edges()))

# BFS with depth limit
bfs_limited = list(nx.bfs_edges(G, source=1, depth_limit=1))
print(bfs_limited)  # [(1, 2), (1, 3)]

# BFS layers (nodes grouped by distance from source)
layers = dict(enumerate(nx.bfs_layers(G, sources=[1])))
print(layers)  # {0: [1], 1: [2, 3], 2: [4, 5, 6, 7]}

# DFS (depth-first search)
dfs = list(nx.dfs_edges(G, source=1))
print(dfs)  # [(1,2), (2,4), (2,5), (1,3), (3,6), (3,7)]

# DFS tree
T = nx.dfs_tree(G, source=1)

# DFS with depth limit
dfs_limited = list(nx.dfs_edges(G, source=1, depth_limit=1))

# DFS pre-ordering and post-ordering
pre = list(nx.dfs_preorder_nodes(G, source=1))
post = list(nx.dfs_postorder_nodes(G, source=1))
```

### DAG Operations

```python
D = nx.DiGraph()
D.add_edges_from([
    ("A", "B"), ("A", "C"), ("B", "D"), ("C", "D"), ("D", "E"),
])

# Ancestors and descendants
print(nx.ancestors(D, "D"))     # {"A", "B", "C"}
print(nx.descendants(D, "A"))   # {"B", "C", "D", "E"}

# Longest path in a DAG (critical path)
longest = nx.dag_longest_path(D)
print(longest)  # ["A", "B", "D", "E"] or ["A", "C", "D", "E"]

longest_length = nx.dag_longest_path_length(D)
print(longest_length)  # 3

# Transitive closure (add edge for every reachable pair)
TC = nx.transitive_closure(D)
print(TC.has_edge("A", "E"))  # True

# Transitive reduction (minimum edges preserving reachability)
TR = nx.transitive_reduction(D)

# Anti-chains (sets of nodes with no path between them)
antichains = list(nx.antichains(D))

# DAG to branching (arborescence)
# Useful for tree representations of DAGs
```

### Minimum Spanning Tree

```python
G = nx.Graph()
G.add_weighted_edges_from([
    ("A", "B", 4), ("A", "C", 2), ("B", "C", 1),
    ("B", "D", 5), ("C", "D", 8), ("C", "E", 10),
    ("D", "E", 2),
])

# Minimum spanning tree (Kruskal's algorithm by default)
mst = nx.minimum_spanning_tree(G)
print(sorted(mst.edges(data="weight")))
# [("A", "C", 2), ("B", "C", 1), ("C", "D", 8), ("D", "E", 2)]

# Total weight of MST
total = sum(d["weight"] for u, v, d in mst.edges(data=True))

# As edges (iterator, does not build a graph)
mst_edges = list(nx.minimum_spanning_edges(G, data=True))
```

### Cliques and Communities

```python
G = nx.Graph()
G.add_edges_from([
    (1, 2), (1, 3), (2, 3),   # clique of 3
    (3, 4), (4, 5), (4, 6), (5, 6),  # another clique
])

# Find all maximal cliques
cliques = list(nx.find_cliques(G))
print(cliques)  # [[1, 2, 3], [4, 5, 6], [3, 4]]

# Size of the largest clique
print(nx.graph_clique_number(G))  # 3

# Community detection (Louvain method -- requires community extras)
# communities = nx.community.louvain_communities(G)

# Girvan-Newman community detection
from networkx.algorithms.community import girvan_newman
comp = girvan_newman(G)
first_partition = next(comp)
print(first_partition)  # tuple of sets
```

---

## Serialization

### JSON (Node-Link Format)

The most common format for web applications and JavaScript interop.

```python
import json
import networkx as nx
from networkx.readwrite import json_graph

G = nx.Graph()
G.add_node("A", color="red")
G.add_edge("A", "B", weight=1.5)

# Graph -> JSON-serializable dict (node-link format)
data = json_graph.node_link_data(G)
json_string = json.dumps(data, indent=2)
print(json_string)
# {
#   "directed": false,
#   "multigraph": false,
#   "graph": {},
#   "nodes": [
#     {"color": "red", "id": "A"},
#     {"id": "B"}
#   ],
#   "links": [
#     {"weight": 1.5, "source": "A", "target": "B"}
#   ]
# }

# JSON dict -> Graph
G_restored = json_graph.node_link_graph(data)
```

### Adjacency Data Format

```python
# Adjacency data (dict-of-dicts)
data = json_graph.adjacency_data(G)
json_string = json.dumps(data, indent=2)

# Restore from adjacency data
G_restored = json_graph.adjacency_graph(data)
```

### Edge List

Simple text format: one edge per line.

```python
# Write edge list to file
nx.write_edgelist(G, "graph.edgelist")

# Read edge list from file
G = nx.read_edgelist("graph.edgelist")

# Weighted edge list
nx.write_weighted_edgelist(G, "graph.weighted.edgelist")
G = nx.read_weighted_edgelist("graph.weighted.edgelist")
```

### Adjacency List

```python
# Write adjacency list
nx.write_adjlist(G, "graph.adjlist")

# Read adjacency list
G = nx.read_adjlist("graph.adjlist")
```

### GraphML (XML-based, interoperable)

```python
# Write GraphML (requires lxml for full feature support)
nx.write_graphml(G, "graph.graphml")

# Read GraphML
G = nx.read_graphml("graph.graphml")
```

### GML (Graph Modelling Language)

```python
nx.write_gml(G, "graph.gml")
G = nx.read_gml("graph.gml")
```

### GEXF (Gephi format)

```python
nx.write_gexf(G, "graph.gexf")
G = nx.read_gexf("graph.gexf")
```

### Pickle (Python-native, not portable)

```python
import pickle

# Write
with open("graph.pkl", "wb") as f:
    pickle.dump(G, f)

# Read
with open("graph.pkl", "rb") as f:
    G = pickle.load(f)
```

---

## Drawing and Visualization

NetworkX integrates with matplotlib for basic graph visualization. For publication-quality or interactive visualizations, consider exporting to Gephi, Cytoscape, or using libraries like `pyvis`.

```python
import networkx as nx
import matplotlib.pyplot as plt

G = nx.karate_club_graph()

# Basic drawing
nx.draw(G, with_labels=True)
plt.show()

# With layout algorithm
pos = nx.spring_layout(G, seed=42)  # reproducible layout
nx.draw(G, pos, with_labels=True, node_color="lightblue",
        node_size=500, font_size=10, font_weight="bold")
plt.title("Karate Club Graph")
plt.show()

# Draw with edge weights
G = nx.Graph()
G.add_weighted_edges_from([("A", "B", 1), ("B", "C", 3), ("A", "C", 2)])
pos = nx.spring_layout(G)

nx.draw(G, pos, with_labels=True, node_color="lightyellow",
        node_size=700, edgecolors="black", linewidths=1)

# Draw edge labels
edge_labels = nx.get_edge_attributes(G, "weight")
nx.draw_networkx_edge_labels(G, pos, edge_labels=edge_labels)
plt.show()
```

### Available Layout Algorithms

| Layout | Function | Best For |
|--------|----------|----------|
| Spring (Fruchterman-Reingold) | `nx.spring_layout()` | General purpose, most common |
| Circular | `nx.circular_layout()` | Showing all nodes equally |
| Shell | `nx.shell_layout()` | Layered/hierarchical views |
| Kamada-Kawai | `nx.kamada_kawai_layout()` | Graphs with meaningful distances |
| Spectral | `nx.spectral_layout()` | Clustered structures |
| Planar | `nx.planar_layout()` | Planar graphs (no edge crossings) |
| Random | `nx.random_layout()` | Quick visualization |
| Spiral | `nx.spiral_layout()` | Large graphs |

### Saving Figures

```python
pos = nx.spring_layout(G, seed=42)
nx.draw(G, pos, with_labels=True)
plt.savefig("graph.png", dpi=300, bbox_inches="tight")
plt.savefig("graph.pdf", bbox_inches="tight")
plt.close()
```

---

## Gotchas and Common Mistakes

### 1. Node Identity Requires Hashability

Nodes **must** be hashable. Lists, dicts, and sets cannot be nodes.

```python
G = nx.Graph()

# These work (hashable types):
G.add_node(42)
G.add_node("hello")
G.add_node((1, 2, 3))
G.add_node(frozenset({1, 2}))

# These FAIL (unhashable types):
# G.add_node([1, 2])        # TypeError: unhashable type: 'list'
# G.add_node({"a": 1})      # TypeError: unhashable type: 'dict'
# G.add_node({1, 2})        # TypeError: unhashable type: 'set'
```

**Fix:** Use tuples instead of lists, frozensets instead of sets, or wrap data in a named entity and use its ID as the node.

### 2. Mutating While Iterating

Modifying a graph's structure during iteration leads to `RuntimeError` or undefined behavior.

```python
G = nx.Graph([(1, 2), (2, 3), (3, 4)])

# BUG: modifying graph structure during iteration
# for node in G.nodes():
#     if G.degree(node) == 1:
#         G.remove_node(node)  # RuntimeError!

# FIX: collect nodes first, then modify
to_remove = [n for n in G.nodes() if G.degree(n) == 1]
G.remove_nodes_from(to_remove)
```

The same applies to edge iteration:

```python
# BUG:
# for u, v in G.edges():
#     if some_condition(u, v):
#         G.remove_edge(u, v)  # RuntimeError!

# FIX:
edges_to_remove = [(u, v) for u, v in G.edges() if some_condition(u, v)]
G.remove_edges_from(edges_to_remove)
```

### 3. Memory Usage with Large Graphs

NetworkX stores graphs as nested Python dictionaries. This is flexible but memory-intensive:

- Each node uses ~300-500 bytes of overhead
- Each edge uses ~150-300 bytes of overhead (more with attributes)
- A graph with 1M nodes and 10M edges can easily consume 5-10 GB of RAM

```python
# Check approximate memory usage
import sys

G = nx.gnm_random_graph(100000, 500000)

# Rough estimate (does not count attribute data)
node_mem = sys.getsizeof(G._node) + sum(sys.getsizeof(d) for d in G._node.values())
adj_mem = sys.getsizeof(G._adj) + sum(
    sys.getsizeof(nbrs) + sum(sys.getsizeof(d) for d in nbrs.values())
    for nbrs in G._adj.values()
)
print(f"Approximate memory: {(node_mem + adj_mem) / 1e6:.0f} MB")
```

### 4. Performance for Large-Scale Graphs

NetworkX is written in pure Python and optimized for correctness and flexibility, not raw speed. For graphs with millions of nodes or edges, consider:

| Library | Language | Best For |
|---------|----------|----------|
| **graph-tool** | C++ (Python bindings) | Statistical analysis, fast algorithms |
| **igraph** | C (Python bindings) | Fast graph algorithms, community detection |
| **networkit** | C++ (Python bindings) | Large-scale network analysis |
| **SNAP** | C++ (Python bindings) | Very large graphs (billions of edges) |
| **cuGraph** (RAPIDS) | CUDA/GPU | GPU-accelerated graph analytics |

Typical performance boundary: NetworkX works well up to ~100K nodes and ~1M edges. Beyond that, C/C++-backed libraries are recommended.

### 5. DiGraph vs Graph Confusion

A common mistake is building a `Graph` (undirected) when the problem requires a `DiGraph` (directed), or vice versa.

```python
# Topological sort REQUIRES a DiGraph
G = nx.Graph()
G.add_edges_from([("A", "B"), ("B", "C")])
# nx.topological_sort(G)  # NetworkXError: not a directed graph

# Fix: use DiGraph
D = nx.DiGraph([("A", "B"), ("B", "C")])
print(list(nx.topological_sort(D)))  # ["A", "B", "C"]
```

Algorithms like `topological_sort()`, `is_directed_acyclic_graph()`, `ancestors()`, and `descendants()` require directed graphs. Algorithms like `connected_components()` require undirected graphs (use `weakly_connected_components()` or `strongly_connected_components()` for directed graphs).

### 6. Edge Overwrites in Simple Graphs

In `Graph` and `DiGraph`, adding an edge that already exists **replaces** its attributes:

```python
G = nx.Graph()
G.add_edge("A", "B", weight=1.0, color="red")
G.add_edge("A", "B", weight=2.0)

print(G["A"]["B"])  # {"weight": 2.0} -- "color" is GONE
```

**Fix:** Use `G["A"]["B"].update({"weight": 2.0})` to merge instead of replace, or use `MultiGraph` if you need parallel edges.

### 7. `shortest_path` Default Is Unweighted

`nx.shortest_path()` ignores edge weights by default. It finds the path with the fewest hops, not the lowest total weight.

```python
G = nx.Graph()
G.add_edge("A", "B", weight=100)
G.add_edge("A", "C", weight=1)
G.add_edge("C", "B", weight=1)

# Unweighted (default) -- fewest hops
print(nx.shortest_path(G, "A", "B"))  # ["A", "B"] (1 hop, weight=100)

# Weighted -- lowest total weight
print(nx.shortest_path(G, "A", "B", weight="weight"))  # ["A", "C", "B"] (weight=2)
```

### 8. Generators Are Not Reusable

Many NetworkX algorithm functions return generators/iterators, not lists. They can only be consumed once.

```python
components = nx.connected_components(G)

# First consumption works
first = next(components)

# But you cannot "reset" -- the generator is partially consumed
# list(components) will miss the first component

# Fix: convert to list immediately if you need to use it multiple times
components = list(nx.connected_components(G))
```

---

## Complete Code Examples

### Example 1: Building a Dependency DAG

Model package dependencies and analyze the dependency tree.

```python
import networkx as nx

# Build a dependency graph (edge means "depends on")
deps = nx.DiGraph()

# Add dependencies: (package, depends_on)
deps.add_edges_from([
    ("myapp", "flask"),
    ("myapp", "sqlalchemy"),
    ("myapp", "celery"),
    ("flask", "werkzeug"),
    ("flask", "jinja2"),
    ("flask", "click"),
    ("sqlalchemy", "greenlet"),
    ("celery", "kombu"),
    ("celery", "billiard"),
    ("celery", "vine"),
    ("kombu", "amqp"),
    ("kombu", "vine"),
    ("jinja2", "markupsafe"),
    ("werkzeug", "markupsafe"),
])

# Verify it is a DAG (no circular dependencies)
assert nx.is_directed_acyclic_graph(deps), "Circular dependency detected!"

# Find all dependencies of myapp (transitive)
all_deps = nx.descendants(deps, "myapp")
print(f"myapp depends on {len(all_deps)} packages: {sorted(all_deps)}")
# myapp depends on 11 packages: ['amqp', 'billiard', 'celery', 'click',
#   'flask', 'greenlet', 'jinja2', 'kombu', 'markupsafe', 'sqlalchemy', ...]

# Find packages with no dependencies (leaf nodes)
leaves = [n for n in deps.nodes() if deps.out_degree(n) == 0]
print(f"Leaf packages (no deps): {sorted(leaves)}")
# ['amqp', 'billiard', 'click', 'greenlet', 'markupsafe', 'vine']

# Find packages that nothing depends on (root nodes, besides myapp)
roots = [n for n in deps.nodes() if deps.in_degree(n) == 0]
print(f"Root packages: {roots}")
# ['myapp']

# What packages would be affected if markupsafe had a breaking change?
affected = nx.ancestors(deps, "markupsafe")
print(f"Packages affected by markupsafe change: {sorted(affected)}")
# ['flask', 'jinja2', 'myapp', 'werkzeug']

# Install order (topological sort -- dependencies first)
install_order = list(reversed(list(nx.topological_sort(deps))))
print(f"Install order: {install_order}")
# markupsafe, click, greenlet, vine, amqp, billiard, werkzeug, jinja2,
# sqlalchemy, kombu, flask, celery, myapp
```

### Example 2: Topological Sort for Task Scheduling

Schedule build tasks respecting dependencies, and identify the critical path.

```python
import networkx as nx

# Build task graph with durations
# Edge (A, B) means "A must complete before B can start"
tasks = nx.DiGraph()

task_durations = {
    "fetch_data":       5,
    "parse_config":     2,
    "validate_schema":  3,
    "transform_data":   8,
    "run_tests":        10,
    "generate_report":  4,
    "deploy":           6,
    "notify":           1,
}

for task, duration in task_durations.items():
    tasks.add_node(task, duration=duration)

tasks.add_edges_from([
    ("fetch_data", "transform_data"),
    ("parse_config", "validate_schema"),
    ("validate_schema", "transform_data"),
    ("transform_data", "run_tests"),
    ("transform_data", "generate_report"),
    ("run_tests", "deploy"),
    ("generate_report", "deploy"),
    ("deploy", "notify"),
])

assert nx.is_directed_acyclic_graph(tasks), "Cycle detected in task graph!"

# Execution order
order = list(nx.topological_sort(tasks))
print("Execution order:")
for i, task in enumerate(order, 1):
    duration = tasks.nodes[task]["duration"]
    deps = list(tasks.predecessors(task))
    dep_str = f" (after: {', '.join(deps)})" if deps else " (no deps)"
    print(f"  {i}. {task} [{duration}s]{dep_str}")

# Topological generations: tasks that can run in parallel
print("\nParallel execution schedule:")
for gen_num, generation in enumerate(nx.topological_generations(tasks)):
    gen_tasks = sorted(generation)
    durations = [tasks.nodes[t]["duration"] for t in gen_tasks]
    wall_time = max(durations)
    print(f"  Stage {gen_num}: {gen_tasks} (wall time: {wall_time}s)")

# Critical path (longest path through the DAG)
critical = nx.dag_longest_path(tasks, weight="duration")
critical_length = nx.dag_longest_path_length(tasks, weight="duration")
print(f"\nCritical path: {' -> '.join(critical)}")
print(f"Minimum total time: {critical_length}s")
```

### Example 3: Shortest Path Finding

Find optimal routes in a transportation network.

```python
import networkx as nx

# Build a city transportation network
city = nx.Graph()

# Edges: (city_a, city_b, distance_in_km)
routes = [
    ("New York", "Boston", 346),
    ("New York", "Philadelphia", 151),
    ("New York", "Washington DC", 363),
    ("Boston", "Portland", 175),
    ("Philadelphia", "Washington DC", 224),
    ("Philadelphia", "Pittsburgh", 491),
    ("Washington DC", "Richmond", 171),
    ("Washington DC", "Pittsburgh", 393),
    ("Pittsburgh", "Cleveland", 216),
    ("Pittsburgh", "Columbus", 264),
    ("Cleveland", "Columbus", 228),
    ("Cleveland", "Detroit", 265),
    ("Columbus", "Indianapolis", 284),
    ("Detroit", "Chicago", 452),
    ("Indianapolis", "Chicago", 290),
    ("Richmond", "Charlotte", 443),
    ("Charlotte", "Atlanta", 364),
]

city.add_weighted_edges_from(routes)

# Shortest path by distance
source, target = "New York", "Chicago"
path = nx.dijkstra_path(city, source, target, weight="weight")
distance = nx.dijkstra_path_length(city, source, target, weight="weight")
print(f"Shortest route {source} -> {target}:")
print(f"  Path: {' -> '.join(path)}")
print(f"  Distance: {distance} km")

# Shortest path by number of stops (unweighted)
path_hops = nx.shortest_path(city, source, target)
print(f"\nFewest stops {source} -> {target}:")
print(f"  Path: {' -> '.join(path_hops)}")
print(f"  Stops: {len(path_hops) - 1}")

# All shortest paths (same minimum distance)
all_paths = list(nx.all_shortest_paths(city, source, target, weight="weight"))
print(f"\nAll optimal routes: {len(all_paths)}")
for p in all_paths:
    print(f"  {' -> '.join(p)}")

# Distance from New York to all cities
distances = dict(nx.single_source_dijkstra_path_length(city, "New York"))
print(f"\nDistances from New York:")
for dest, dist in sorted(distances.items(), key=lambda x: x[1]):
    print(f"  {dest}: {dist} km")

# Find all cities within 400 km of New York
nearby = {c: d for c, d in distances.items() if d <= 400}
print(f"\nCities within 400 km of New York: {sorted(nearby.keys())}")
```

### Example 4: Cycle Detection

Detect circular dependencies in a configuration system.

```python
import networkx as nx


def check_dependencies(dependency_map: dict[str, list[str]]) -> dict:
    """
    Analyze a dependency map for circular dependencies and report findings.

    Args:
        dependency_map: dict mapping each item to its list of dependencies.

    Returns:
        dict with analysis results.
    """
    G = nx.DiGraph()

    for item, deps in dependency_map.items():
        G.add_node(item)
        for dep in deps:
            G.add_edge(item, dep)

    result = {
        "is_valid": nx.is_directed_acyclic_graph(G),
        "total_items": G.number_of_nodes(),
        "total_dependencies": G.number_of_edges(),
        "cycles": [],
        "install_order": None,
    }

    if result["is_valid"]:
        # No cycles -- provide installation order
        result["install_order"] = list(reversed(list(nx.topological_sort(G))))
    else:
        # Find all cycles
        result["cycles"] = list(nx.simple_cycles(G))

        # Find strongly connected components (groups of mutual dependencies)
        sccs = [
            scc for scc in nx.strongly_connected_components(G)
            if len(scc) > 1
        ]
        result["circular_groups"] = [sorted(scc) for scc in sccs]

    return result


# Test with a valid dependency graph
valid_deps = {
    "app": ["auth", "database", "logging"],
    "auth": ["database", "crypto"],
    "database": ["logging"],
    "crypto": [],
    "logging": [],
}

result = check_dependencies(valid_deps)
print("Valid dependencies:")
print(f"  Is valid DAG: {result['is_valid']}")
print(f"  Install order: {result['install_order']}")
# Install order: ['logging', 'crypto', 'database', 'auth', 'app']

# Test with circular dependencies
circular_deps = {
    "A": ["B"],
    "B": ["C"],
    "C": ["A"],    # circular!
    "D": ["E"],
    "E": ["D"],    # another cycle!
    "F": ["A"],
}

result = check_dependencies(circular_deps)
print("\nCircular dependencies:")
print(f"  Is valid DAG: {result['is_valid']}")
print(f"  Cycles found: {result['cycles']}")
print(f"  Circular groups: {result['circular_groups']}")
# Cycles found: [['A', 'B', 'C'], ['D', 'E']]
# Circular groups: [['A', 'B', 'C'], ['D', 'E']]
```

### Example 5: Graph Serialization Round-Trip

Build a graph, serialize to multiple formats, and restore.

```python
import json
import tempfile
import os
import networkx as nx
from networkx.readwrite import json_graph


def build_sample_graph() -> nx.DiGraph:
    """Build a sample project dependency graph with metadata."""
    G = nx.DiGraph(
        name="project-dependencies",
        version="1.0",
        description="Python project dependency graph",
    )

    packages = [
        ("requests", {"version": "2.31.0", "category": "http"}),
        ("flask", {"version": "3.0.0", "category": "web"}),
        ("sqlalchemy", {"version": "2.0.25", "category": "database"}),
        ("click", {"version": "8.1.7", "category": "cli"}),
        ("jinja2", {"version": "3.1.3", "category": "templating"}),
        ("werkzeug", {"version": "3.0.1", "category": "http"}),
        ("markupsafe", {"version": "2.1.4", "category": "security"}),
        ("urllib3", {"version": "2.1.0", "category": "http"}),
        ("certifi", {"version": "2024.2.2", "category": "security"}),
    ]

    for name, attrs in packages:
        G.add_node(name, **attrs)

    dependencies = [
        ("flask", "werkzeug", {"type": "runtime"}),
        ("flask", "jinja2", {"type": "runtime"}),
        ("flask", "click", {"type": "runtime"}),
        ("jinja2", "markupsafe", {"type": "runtime"}),
        ("werkzeug", "markupsafe", {"type": "runtime"}),
        ("requests", "urllib3", {"type": "runtime"}),
        ("requests", "certifi", {"type": "runtime"}),
        ("sqlalchemy", "markupsafe", {"type": "optional"}),
    ]

    for src, dst, attrs in dependencies:
        G.add_edge(src, dst, **attrs)

    return G


G = build_sample_graph()

# --- JSON (node-link format) ---
data = json_graph.node_link_data(G)
json_str = json.dumps(data, indent=2)
print(f"JSON size: {len(json_str)} bytes")

# Restore from JSON
G_from_json = json_graph.node_link_graph(data)
assert set(G_from_json.nodes()) == set(G.nodes())
assert set(G_from_json.edges()) == set(G.edges())
print("JSON round-trip: OK")

# --- Edge list ---
with tempfile.NamedTemporaryFile(mode="w", suffix=".edgelist",
                                  delete=False) as f:
    edgelist_path = f.name
    nx.write_edgelist(G, f.name)

G_from_edgelist = nx.read_edgelist(edgelist_path, create_using=nx.DiGraph)
print(f"Edge list nodes: {G_from_edgelist.number_of_nodes()}")
print(f"Edge list edges: {G_from_edgelist.number_of_edges()}")
os.unlink(edgelist_path)
print("Edge list round-trip: OK")

# --- GML ---
with tempfile.NamedTemporaryFile(mode="w", suffix=".gml",
                                  delete=False) as f:
    gml_path = f.name

nx.write_gml(G, gml_path)
G_from_gml = nx.read_gml(gml_path)
assert set(G_from_gml.nodes()) == set(G.nodes())
os.unlink(gml_path)
print("GML round-trip: OK")

# --- Custom JSON serialization for specific use cases ---
def graph_to_dict(G: nx.DiGraph) -> dict:
    """Custom serialization preserving all metadata."""
    return {
        "metadata": dict(G.graph),
        "nodes": {
            node: dict(data)
            for node, data in G.nodes(data=True)
        },
        "edges": [
            {"source": u, "target": v, **dict(data)}
            for u, v, data in G.edges(data=True)
        ],
    }


def dict_to_graph(d: dict) -> nx.DiGraph:
    """Custom deserialization."""
    G = nx.DiGraph(**d["metadata"])
    for node, attrs in d["nodes"].items():
        G.add_node(node, **attrs)
    for edge in d["edges"]:
        src = edge.pop("source")
        tgt = edge.pop("target")
        G.add_edge(src, tgt, **edge)
    return G


custom_data = graph_to_dict(G)
print(f"\nCustom format:\n{json.dumps(custom_data, indent=2)[:500]}...")

G_restored = dict_to_graph(custom_data)
assert set(G_restored.nodes()) == set(G.nodes())
assert set(G_restored.edges()) == set(G.edges())
assert G_restored.graph["name"] == "project-dependencies"
print("Custom serialization round-trip: OK")
```

---

## Quick Reference

### Graph Construction Shortcuts

```python
import networkx as nx

# From edge list
G = nx.Graph([(1, 2), (2, 3), (3, 1)])

# Classic generators
K5 = nx.complete_graph(5)          # complete graph
P10 = nx.path_graph(10)            # path graph
C7 = nx.cycle_graph(7)             # cycle graph
S5 = nx.star_graph(5)              # star graph
grid = nx.grid_2d_graph(4, 4)     # 4x4 grid
petersen = nx.petersen_graph()     # Petersen graph

# Random graphs
er = nx.erdos_renyi_graph(100, 0.05)       # Erdos-Renyi
ba = nx.barabasi_albert_graph(100, 3)       # Barabasi-Albert
ws = nx.watts_strogatz_graph(100, 4, 0.3)   # Watts-Strogatz

# From adjacency matrix (requires numpy)
import numpy as np
A = np.array([[0, 1, 1], [1, 0, 0], [1, 0, 0]])
G = nx.from_numpy_array(A)

# From pandas DataFrame
import pandas as pd
df = pd.DataFrame({"source": ["A", "B", "C"], "target": ["B", "C", "A"]})
G = nx.from_pandas_edgelist(df, source="source", target="target")

# From a dict-of-lists
G = nx.from_dict_of_lists({1: [2, 3], 2: [1, 3], 3: [1, 2]})
```

### Common Graph Properties

```python
G = nx.karate_club_graph()

print(f"Nodes: {G.number_of_nodes()}")
print(f"Edges: {G.number_of_edges()}")
print(f"Density: {nx.density(G):.4f}")
print(f"Is connected: {nx.is_connected(G)}")
print(f"Diameter: {nx.diameter(G)}")
print(f"Average shortest path: {nx.average_shortest_path_length(G):.2f}")
print(f"Clustering coefficient: {nx.average_clustering(G):.4f}")
print(f"Transitivity: {nx.transitivity(G):.4f}")
```

### Algorithm Summary Table

| Category | Key Functions |
|----------|---------------|
| **Shortest Path** | `shortest_path()`, `dijkstra_path()`, `bellman_ford_path()`, `astar_path()`, `all_shortest_paths()` |
| **Topological Sort** | `topological_sort()`, `all_topological_sorts()`, `lexicographical_topological_sort()`, `topological_generations()` |
| **Cycles** | `find_cycle()`, `simple_cycles()`, `cycle_basis()`, `is_directed_acyclic_graph()` |
| **Components** | `connected_components()`, `strongly_connected_components()`, `weakly_connected_components()`, `is_connected()` |
| **Centrality** | `degree_centrality()`, `betweenness_centrality()`, `closeness_centrality()`, `eigenvector_centrality()`, `pagerank()` |
| **Traversal** | `bfs_edges()`, `dfs_edges()`, `bfs_tree()`, `dfs_tree()`, `bfs_layers()` |
| **DAG** | `ancestors()`, `descendants()`, `dag_longest_path()`, `dag_longest_path_length()`, `transitive_closure()`, `transitive_reduction()` |
| **Matching** | `max_weight_matching()`, `min_weight_matching()`, `is_perfect_matching()` |
| **Flow** | `maximum_flow()`, `minimum_cut()`, `max_flow_min_cost()` |
| **MST** | `minimum_spanning_tree()`, `minimum_spanning_edges()` |
| **Cliques** | `find_cliques()`, `graph_clique_number()`, `graph_number_of_cliques()` |
| **Communities** | `louvain_communities()`, `girvan_newman()`, `greedy_modularity_communities()` |
