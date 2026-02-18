# NetworkX — Algorithms

> Part of the networkx skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Shortest Path](#shortest-path)
- [Topological Sort](#topological-sort)
- [Cycle Detection](#cycle-detection)
- [Connected Components](#connected-components)
- [Centrality Measures](#centrality-measures)
- [Graph Traversal](#graph-traversal)
- [DAG Operations](#dag-operations)
- [Minimum Spanning Tree](#minimum-spanning-tree)
- [Cliques and Communities](#cliques-and-communities)
- [Algorithm Summary Table](#algorithm-summary-table)

---

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
