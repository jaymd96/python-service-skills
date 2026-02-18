# NetworkX — API Reference

> Part of the networkx skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core Graph Types](#core-graph-types)
- [Graph Manipulation](#graph-manipulation)
- [Serialization](#serialization)
- [Drawing and Visualization](#drawing-and-visualization)
- [Quick Reference](#quick-reference)

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
