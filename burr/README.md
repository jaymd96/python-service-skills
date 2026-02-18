# Burr -- Application State Machines for AI Agent Workflows

## Overview

**Burr** (from DAGWorks-Inc) is a Python framework for building applications as state machines, particularly suited for AI/LLM agent workflows. It models applications as a series of named actions connected by conditional transitions, with explicit state management. Burr provides persistence, a tracking UI for debugging, and integrations with LLM frameworks. Every action reads from and writes to a typed `State` object, making the entire application flow inspectable, reproducible, and debuggable.

**Key Characteristics:**

- **Version:** 0.30.0 (latest stable as of early 2026)
- **Python:** 3.9+
- **License:** BSD-3-Clause
- **Repository:** DAGWorks-Inc/burr
- **Dependencies:** Minimal core; optional extras for tracking, integrations

**When to use Burr:**

- Building AI agents that need clear, inspectable control flow
- Multi-step LLM workflows (RAG pipelines, chatbots, autonomous agents)
- Applications that need persistence and resumability (long-running workflows)
- Workflows requiring human-in-the-loop steps
- Systems where you need to visualize and debug the decision graph

**Core Design Principles:**

- **Explicit state** -- all application state is in a single `State` object
- **Declarative transitions** -- edges between actions are defined with conditions
- **Observable** -- built-in tracking UI for debugging and monitoring
- **Persistent** -- save and resume application state at any point
- **Testable** -- actions are pure functions of state, easy to unit test

---

## Installation

```bash
# Core library
pip install burr

# With tracking UI
pip install "burr[tracking]"

# With all extras
pip install "burr[all]"

# Launch the tracking UI
burr
```

The tracking UI starts a local web server (default: `http://localhost:7241`) for visualizing application execution.

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

## Graph Visualization

Burr can export the state machine graph for visualization.

```python
app = ApplicationBuilder().with_actions(...).with_transitions(...).build()

# Get the graph as a graphviz object (requires graphviz)
graph = app.graph

# The tracking UI also provides interactive graph visualization
```

---

## Complete Code Examples

### AI Chatbot with Persistence

```python
import openai
from burr.core import ApplicationBuilder, State, action, Result, when, default
from burr.core.persistence import SQLLitePersister
from burr.tracking import LocalTrackingClient

client = openai.OpenAI()

@action(reads=["user_input"], writes=["messages"])
def add_user_message(state: State) -> State:
    return state.append(
        messages={"role": "user", "content": state["user_input"]}
    )

@action(reads=["messages"], writes=["messages", "response"])
def call_llm(state: State) -> State:
    response = client.chat.completions.create(
        model="gpt-4",
        messages=state["messages"],
    )
    reply = response.choices[0].message.content
    return state.update(response=reply).append(
        messages={"role": "assistant", "content": reply}
    )

@action(reads=["response"], writes=["should_continue"])
def check_done(state: State) -> State:
    is_done = "goodbye" in state["response"].lower()
    return state.update(should_continue=not is_done)

def build_chatbot(user_id: str):
    persister = SQLLitePersister(db_path="./chatbot.db", table_name="chat_state")
    persister.initialize()
    tracker = LocalTrackingClient(project="chatbot")

    app = (
        ApplicationBuilder()
        .with_actions(
            add_message=add_user_message,
            call_llm=call_llm,
            check_done=check_done,
            done=Result("response"),
        )
        .with_transitions(
            ("add_message", "call_llm"),
            ("call_llm", "check_done"),
            ("check_done", "done", when(should_continue=False)),
            ("check_done", "add_message", default),
        )
        .with_state(State({
            "messages": [{"role": "system", "content": "You are a helpful assistant."}],
        }))
        .with_entrypoint("add_message")
        .with_persister(persister)
        .with_tracker(tracker)
        .with_identifiers(app_id="chatbot", partition_key=user_id)
        .initialize_from(persister, resume_at_next_action=True)
        .build()
    )
    return app

# Usage
app = build_chatbot("user-123")
while True:
    user_input = input("You: ")
    app.update_state(State({"user_input": user_input}))
    action, result, state = app.run(halt_after=["done", "add_message"], halt_before=["add_message"])
    if action.name == "done":
        print(f"Assistant: {state['response']}")
        break
    print(f"Assistant: {state['response']}")
```

### RAG Pipeline

```python
from burr.core import ApplicationBuilder, State, action, Result, when, default

@action(reads=["query"], writes=["query_embedding"])
def embed_query(state: State) -> State:
    embedding = embedding_model.encode(state["query"])
    return state.update(query_embedding=embedding)

@action(reads=["query_embedding"], writes=["documents"])
def retrieve_documents(state: State) -> State:
    docs = vector_store.search(state["query_embedding"], top_k=5)
    return state.update(documents=docs)

@action(reads=["documents"], writes=["has_relevant_docs"])
def check_relevance(state: State) -> State:
    has_relevant = any(doc.score > 0.7 for doc in state["documents"])
    return state.update(has_relevant_docs=has_relevant)

@action(reads=["query", "documents"], writes=["answer"])
def generate_answer(state: State) -> State:
    context = "\n".join(doc.text for doc in state["documents"])
    prompt = f"Context:\n{context}\n\nQuestion: {state['query']}\nAnswer:"
    answer = llm.generate(prompt)
    return state.update(answer=answer)

@action(reads=["query"], writes=["answer"])
def fallback_answer(state: State) -> State:
    return state.update(answer="I don't have enough information to answer that question.")

app = (
    ApplicationBuilder()
    .with_actions(
        embed=embed_query,
        retrieve=retrieve_documents,
        check=check_relevance,
        generate=generate_answer,
        fallback=fallback_answer,
        result=Result("answer"),
    )
    .with_transitions(
        ("embed", "retrieve"),
        ("retrieve", "check"),
        ("check", "generate", when(has_relevant_docs=True)),
        ("check", "fallback", when(has_relevant_docs=False)),
        ("generate", "result"),
        ("fallback", "result"),
    )
    .with_state(State({"query": "What is Burr?"}))
    .with_entrypoint("embed")
    .build()
)

action, result, state = app.run(halt_after=["result"])
print(state["answer"])
```

### Multi-Step Agent with Tool Use

```python
from burr.core import ApplicationBuilder, State, action, Result, when, default
import json

@action(reads=["messages"], writes=["messages", "llm_output", "needs_tool"])
def plan(state: State) -> State:
    response = client.chat.completions.create(
        model="gpt-4",
        messages=state["messages"],
        tools=tool_definitions,
    )
    msg = response.choices[0].message
    needs_tool = msg.tool_calls is not None and len(msg.tool_calls) > 0
    return state.update(
        llm_output=msg,
        needs_tool=needs_tool,
    ).append(messages=msg.to_dict())

@action(reads=["llm_output"], writes=["messages", "tool_results"])
def execute_tools(state: State) -> State:
    results = []
    for call in state["llm_output"].tool_calls:
        fn = tool_registry[call.function.name]
        args = json.loads(call.function.arguments)
        result = fn(**args)
        results.append({"tool_call_id": call.id, "output": result})
    return state.update(tool_results=results).append(
        messages={"role": "tool", "content": json.dumps(results)}
    )

@action(reads=["llm_output"], writes=["final_answer"])
def extract_answer(state: State) -> State:
    return state.update(final_answer=state["llm_output"].content)

app = (
    ApplicationBuilder()
    .with_actions(
        plan=plan,
        execute_tools=execute_tools,
        extract=extract_answer,
        done=Result("final_answer"),
    )
    .with_transitions(
        ("plan", "execute_tools", when(needs_tool=True)),
        ("plan", "extract", when(needs_tool=False)),
        ("execute_tools", "plan"),  # Loop back to plan after tool execution
        ("extract", "done"),
    )
    .with_state(State({
        "messages": [{"role": "user", "content": "What's the weather in NYC?"}],
    }))
    .with_entrypoint("plan")
    .build()
)

action, result, state = app.run(halt_after=["done"])
print(state["final_answer"])
```

---

## Gotchas and Common Mistakes

### 1. State Is Immutable

`state.update()` and `state.append()` return new `State` objects. The original is unchanged. Always use the return value.

```python
# WRONG: state is not modified in place
@action(reads=["count"], writes=["count"])
def increment(state: State) -> State:
    state.update(count=state["count"] + 1)  # Return value discarded!
    return state  # Returns original, unmodified state

# CORRECT: return the new state
@action(reads=["count"], writes=["count"])
def increment(state: State) -> State:
    return state.update(count=state["count"] + 1)
```

### 2. `reads` and `writes` Must Be Accurate

Burr uses `reads` and `writes` declarations for validation, graph construction, and persistence optimization. If your action reads a key not listed in `reads`, or writes a key not listed in `writes`, you will get runtime errors or unexpected behavior.

### 3. Transition Order Matters

Transitions are evaluated in the order they are defined. The first matching transition is taken. Always put more specific conditions before `default`.

```python
# WRONG: default is checked first, specific condition never reached
.with_transitions(
    ("classify", "general", default),
    ("classify", "special", when(category="special")),  # Never reached
)

# CORRECT: specific conditions first
.with_transitions(
    ("classify", "special", when(category="special")),
    ("classify", "general", default),
)
```

### 4. Forgetting `with_entrypoint()`

Every application must have an entrypoint. Omitting `.with_entrypoint()` will raise an error at build time.

### 5. Persister Must Be Initialized

When using `SQLLitePersister` or similar, you must call `.initialize()` to create the database tables before building the application.

```python
persister = SQLLitePersister(db_path="./state.db", table_name="app")
persister.initialize()  # Do not forget this
```

### 6. `halt_after` vs `halt_before`

- `halt_after=["action_name"]` runs the action and then stops.
- `halt_before=["action_name"]` stops before running the action.

This distinction matters for human-in-the-loop workflows where you want to pause before a specific step.

### 7. Streaming Requires Special Action Return Types

Streaming actions must return a `StreamingResultContainer`, not a regular `State`. Use `app.stream_result()` instead of `app.step()` or `app.run()` to consume streaming actions.

### 8. Application IDs for Persistence

When using persistence, `app_id` and `partition_key` together uniquely identify a stored application state. Using the same IDs will resume from the persisted state, which may not be what you want during development.

```python
# Use unique IDs during development
.with_identifiers(app_id="chatbot", partition_key=f"dev-{uuid.uuid4()}")
```

### 9. Actions Must Return State

Every function-based action (decorated with `@action`) must return a `State` object. Returning `None` or forgetting to return will cause errors.

### 10. Tracker vs Persister

- **Tracker** records execution history for visualization (read-only, append-only).
- **Persister** saves and loads application state for resumability.

These serve different purposes. You typically want both in production.
