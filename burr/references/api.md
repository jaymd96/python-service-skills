# Burr — API Reference

> Part of the burr skill. See [SKILL.md](../SKILL.md) for overview.

- [Core API](#core-api)
  - [State](#state)
  - [@action Decorator](#action-decorator)
  - [Action Classes](#action-classes)
  - [Result](#result)
- [ApplicationBuilder](#applicationbuilder)
  - [Builder Methods](#builder-methods)
  - [Running the Application](#running-the-application)
- [Transitions and Conditions](#transitions-and-conditions)
  - [Unconditional Transitions](#unconditional-transitions)
  - [Conditional Transitions with when](#conditional-transitions-with-when)
  - [default Transition](#default-transition)
  - [Custom Conditions](#custom-conditions)
- [Persistence](#persistence)
  - [Built-in Persisters](#built-in-persisters)
  - [Saving and Loading](#saving-and-loading)
  - [Other Persisters](#other-persisters)
- [Tracking UI](#tracking-ui)
  - [Enabling Tracking in Code](#enabling-tracking-in-code)
- [Hooks](#hooks)
  - [Available Hook Types](#available-hook-types)
- [Streaming](#streaming)
- [Graph Visualization](#graph-visualization)

## Core API

### `State`

The `State` object is an immutable container for all application data. Actions read from state and return updated state.

```python
from burr.core import State

# Create initial state
state = State({"messages": [], "count": 0, "model": "gpt-4"})

# Access values
state["messages"]    # []
state["count"]       # 0
state.get("missing") # None

# Update creates a new State (immutable)
new_state = state.update(count=1)
new_state = state.update(messages=[{"role": "user", "content": "hello"}])

# Append to lists
new_state = state.append(messages={"role": "user", "content": "hello"})

# Delete keys
new_state = state.wipe(keep=["messages"])  # Keep only "messages"
```

**Important:** `State` is immutable. `update()`, `append()`, and other mutation methods return a new `State` object.

### `@action` Decorator

Actions are the building blocks of a Burr application. Each action is a function that reads from state and returns a result and updated state.

```python
from burr.core import action, State

@action(reads=["messages"], writes=["messages", "response"])
def call_llm(state: State) -> State:
    messages = state["messages"]
    response = openai_client.chat.completions.create(
        model="gpt-4",
        messages=messages,
    )
    reply = response.choices[0].message.content
    return state.update(response=reply).append(
        messages={"role": "assistant", "content": reply}
    )
```

#### `reads` and `writes` Parameters

- `reads` declares which state keys the action accesses. Burr uses this for validation and optimization.
- `writes` declares which state keys the action modifies.

```python
@action(reads=["query"], writes=["search_results"])
def search(state: State) -> State:
    results = search_engine.search(state["query"])
    return state.update(search_results=results)
```

### Action Classes

For more complex actions, use class-based actions:

```python
from burr.core import Action, State

class CallLLM(Action):
    def __init__(self, model: str = "gpt-4"):
        super().__init__()
        self.model = model

    @property
    def reads(self) -> list[str]:
        return ["messages"]

    @property
    def writes(self) -> list[str]:
        return ["messages", "response"]

    def run(self, state: State, **run_kwargs) -> dict:
        messages = state["messages"]
        response = openai_client.chat.completions.create(
            model=self.model,
            messages=messages,
        )
        return {"response": response.choices[0].message.content}

    def update(self, result: dict, state: State) -> State:
        return state.update(response=result["response"]).append(
            messages={"role": "assistant", "content": result["response"]}
        )
```

Class-based actions separate `run()` (computation) from `update()` (state mutation), which is useful for streaming, async, and testing.

### `Result`

The `Result` action marks terminal states in the state machine. When a transition leads to a `Result`, the application halts.

```python
from burr.core import ApplicationBuilder, State, action, Result

@action(reads=["response"], writes=[])
def format_output(state: State) -> State:
    return state

app = (
    ApplicationBuilder()
    .with_actions(
        process=process_input,
        call_llm=call_llm,
        format_output=format_output,
        final=Result("response"),  # Terminal action, returns "response" from state
    )
    .with_transitions(
        ("process", "call_llm"),
        ("call_llm", "format_output"),
        ("format_output", "final"),
    )
    .with_entrypoint("process")
    .build()
)
```

## ApplicationBuilder

The `ApplicationBuilder` is the main way to construct Burr applications. It uses a fluent builder pattern.

```python
from burr.core import ApplicationBuilder, State

app = (
    ApplicationBuilder()
    .with_actions(
        action_a=action_a,
        action_b=action_b,
        action_c=action_c,
        result=Result("output"),
    )
    .with_transitions(
        ("action_a", "action_b", when(condition=True)),
        ("action_a", "action_c", when(condition=False)),
        ("action_b", "result"),
        ("action_c", "result"),
    )
    .with_state(State({"input": None}))
    .with_entrypoint("action_a")
    .build()
)
```

### Builder Methods

| Method | Description |
|--------|-------------|
| `.with_actions(**actions)` | Register named actions |
| `.with_transitions(*transitions)` | Define edges between actions |
| `.with_state(state)` | Set initial state |
| `.with_entrypoint(action_name)` | Set the starting action |
| `.with_tracker(tracker)` | Attach a tracker for the UI |
| `.with_persister(persister)` | Attach a state persister |
| `.with_hooks(*hooks)` | Add lifecycle hooks |
| `.with_identifiers(app_id, ...)` | Set application identifiers for tracking |
| `.build()` | Construct the application |

### Running the Application

```python
# Step through one action at a time
action, result, state = app.step()

# Run until a halt condition
action, result, state = app.run(halt_after=["result"])

# Run until a specific action completes
action, result, state = app.run(halt_before=["human_review"])

# Iterate through steps
for action, result, state in app.iterate(halt_after=["result"]):
    print(f"Executed: {action.name}")
```

## Transitions and Conditions

Transitions define the edges in the state machine graph. Each transition is a tuple of `(from_action, to_action, condition)`.

### Unconditional Transitions

```python
.with_transitions(
    ("step_a", "step_b"),  # Always go from step_a to step_b
)
```

### Conditional Transitions with `when`

```python
from burr.core import when

.with_transitions(
    ("classify", "handle_positive", when(sentiment="positive")),
    ("classify", "handle_negative", when(sentiment="negative")),
    ("classify", "handle_neutral", default),
)
```

`when()` checks state values. Transitions are evaluated in order; the first matching transition is taken.

### `default` Transition

The `default` transition matches when no other transition from that action matches:

```python
from burr.core import default

.with_transitions(
    ("router", "specialized_handler", when(category="special")),
    ("router", "general_handler", default),
)
```

### Custom Conditions

```python
from burr.core import Condition

def is_long_text(state: State) -> bool:
    return len(state.get("text", "")) > 1000

.with_transitions(
    ("analyze", "summarize", Condition.expr(is_long_text)),
    ("analyze", "respond", default),
)
```

## Persistence

Burr supports persisting application state so workflows can be paused and resumed.

### Built-in Persisters

```python
from burr.core.persistence import SQLLitePersister

# SQLite persistence
persister = SQLLitePersister(db_path="./burr_state.db", table_name="app_state")
persister.initialize()  # Create tables

app = (
    ApplicationBuilder()
    .with_actions(...)
    .with_transitions(...)
    .with_state(State({}))
    .with_entrypoint("start")
    .with_persister(persister)
    .with_identifiers(app_id="my-chatbot", partition_key="user-123")
    .build()
)
```

### Saving and Loading

State is automatically saved after each action when a persister is configured. To resume:

```python
app = (
    ApplicationBuilder()
    .with_actions(...)
    .with_transitions(...)
    .with_state(State({}))
    .with_entrypoint("start")
    .with_persister(persister)
    .with_identifiers(app_id="my-chatbot", partition_key="user-123")
    .initialize_from(persister, resume_at_next_action=True)
    .build()
)
# If previous state exists, the app resumes from where it left off
```

### Other Persisters

Burr provides persistence backends for PostgreSQL, Redis, and custom stores. Implement the `BaseStatePersister` interface for custom backends.

## Tracking UI

Burr includes a web-based tracking UI for visualizing and debugging application execution.

```bash
# Start the tracking server
burr
# Opens at http://localhost:7241
```

### Enabling Tracking in Code

```python
from burr.tracking import LocalTrackingClient

tracker = LocalTrackingClient(project="my-chatbot")

app = (
    ApplicationBuilder()
    .with_actions(...)
    .with_transitions(...)
    .with_state(State({}))
    .with_entrypoint("start")
    .with_tracker(tracker)
    .build()
)
```

The tracking UI shows:

- State machine graph visualization
- Step-by-step execution history
- State at each step
- Action inputs and outputs
- Timing information

## Hooks

Hooks let you inject custom logic at various points in the execution lifecycle.

```python
from burr.core import ApplicationBuilder
from burr.lifecycle import PostRunStepHook, PreRunStepHook

class LoggingHook(PostRunStepHook, PreRunStepHook):
    def pre_run_step(self, *, action, state, **kwargs):
        print(f"About to run: {action.name}")

    def post_run_step(self, *, action, state, result, **kwargs):
        print(f"Completed: {action.name}, result keys: {list(result.keys())}")

app = (
    ApplicationBuilder()
    .with_actions(...)
    .with_transitions(...)
    .with_state(State({}))
    .with_entrypoint("start")
    .with_hooks(LoggingHook())
    .build()
)
```

### Available Hook Types

| Hook | When It Runs |
|------|-------------|
| `PreRunStepHook` | Before each action executes |
| `PostRunStepHook` | After each action completes |
| `PreStartSpanHook` | Before a tracking span starts |
| `PostEndSpanHook` | After a tracking span ends |

## Streaming

Burr supports streaming results from actions, which is essential for LLM applications that stream tokens.

```python
from burr.core import action, State
from burr.core.action import StreamingResultContainer

@action(reads=["messages"], writes=["messages", "response"])
def stream_llm_response(state: State) -> StreamingResultContainer:
    messages = state["messages"]

    def generator():
        response = openai_client.chat.completions.create(
            model="gpt-4",
            messages=messages,
            stream=True,
        )
        full_response = ""
        for chunk in response:
            delta = chunk.choices[0].delta.content or ""
            full_response += delta
            yield {"delta": delta}, None  # Intermediate result, no state update
        # Final yield includes the state update
        new_state = state.update(response=full_response).append(
            messages={"role": "assistant", "content": full_response}
        )
        yield {"delta": "", "response": full_response}, new_state

    return StreamingResultContainer.from_generator(generator())

# Consuming the stream
action, streaming_container = app.stream_result(halt_after=["stream_llm_response"])
for result_chunk in streaming_container:
    print(result_chunk["delta"], end="", flush=True)
final_state = streaming_container.get()
```

## Graph Visualization

Burr can export the state machine graph for visualization.

```python
app = ApplicationBuilder().with_actions(...).with_transitions(...).build()

# Get the graph as a graphviz object (requires graphviz)
graph = app.graph

# The tracking UI also provides interactive graph visualization
```
