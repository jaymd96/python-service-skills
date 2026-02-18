# NetworkX — Examples & Gotchas

> Part of the networkx skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Node Identity Requires Hashability](#1-node-identity-requires-hashability)
  - [2. Mutating While Iterating](#2-mutating-while-iterating)
  - [3. Memory Usage with Large Graphs](#3-memory-usage-with-large-graphs)
  - [4. Performance for Large-Scale Graphs](#4-performance-for-large-scale-graphs)
  - [5. DiGraph vs Graph Confusion](#5-digraph-vs-graph-confusion)
  - [6. Edge Overwrites in Simple Graphs](#6-edge-overwrites-in-simple-graphs)
  - [7. shortest_path Default Is Unweighted](#7-shortest_path-default-is-unweighted)
  - [8. Generators Are Not Reusable](#8-generators-are-not-reusable)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Building a Dependency DAG](#example-1-building-a-dependency-dag)
  - [Example 2: Topological Sort for Task Scheduling](#example-2-topological-sort-for-task-scheduling)
  - [Example 3: Shortest Path Finding](#example-3-shortest-path-finding)
  - [Example 4: Cycle Detection](#example-4-cycle-detection)
  - [Example 5: Graph Serialization Round-Trip](#example-5-graph-serialization-round-trip)

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
