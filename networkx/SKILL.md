---
name: networkx
description: Graph and network analysis with NetworkX. Use when working with directed/undirected graphs, DAGs, topological sorting, dependency resolution, shortest paths, cycle detection, or graph algorithms. Triggers on graphs, DAG, topological sort, networkx, dependency graph, DiGraph, shortest path, graph algorithms.
---

# NetworkX — Graph Analysis (v3.4)

## Quick Start

```bash
pip install networkx
```

```python
import networkx as nx

G = nx.DiGraph()
G.add_edges_from([("A", "B"), ("A", "C"), ("B", "D"), ("C", "D")])

# Topological sort (dependency order)
list(nx.topological_sort(G))  # ['A', 'C', 'B', 'D'] or similar valid order

# Check for cycles
nx.is_directed_acyclic_graph(G)  # True
```

## Key Patterns

### DAG operations (most common use case)
```python
nx.topological_sort(G)           # generator of nodes in dependency order
nx.topological_generations(G)    # groups of independent nodes (parallel exec)
nx.ancestors(G, "D")             # all transitive predecessors: {'A', 'B', 'C'}
nx.descendants(G, "A")           # all transitive successors: {'B', 'C', 'D'}
nx.dag_longest_path(G)           # critical path
```

### Shortest path
```python
nx.shortest_path(G, "A", "D")                    # unweighted
nx.dijkstra_path(G, "A", "D", weight="cost")     # weighted
```

### Graph types
```python
nx.Graph()          # undirected
nx.DiGraph()        # directed
nx.MultiGraph()     # undirected, parallel edges
nx.MultiDiGraph()   # directed, parallel edges
```

## References

- **[api.md](references/api.md)** — Core graph API, node/edge operations, graph types, attributes
- **[algorithms.md](references/algorithms.md)** — Shortest paths, centrality, community detection, traversal algorithms
- **[examples.md](references/examples.md)** — Complete examples, gotchas, visualization notes
