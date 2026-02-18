# Burr — Examples & Gotchas

> Part of the burr skill. See [SKILL.md](../SKILL.md) for overview.

- [Complete Code Examples](#complete-code-examples)
  - [AI Chatbot with Persistence](#ai-chatbot-with-persistence)
  - [RAG Pipeline](#rag-pipeline)
  - [Multi-Step Agent with Tool Use](#multi-step-agent-with-tool-use)
- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. State Is Immutable](#1-state-is-immutable)
  - [2. reads and writes Must Be Accurate](#2-reads-and-writes-must-be-accurate)
  - [3. Transition Order Matters](#3-transition-order-matters)
  - [4. Forgetting with_entrypoint()](#4-forgetting-with_entrypoint)
  - [5. Persister Must Be Initialized](#5-persister-must-be-initialized)
  - [6. halt_after vs halt_before](#6-halt_after-vs-halt_before)
  - [7. Streaming Requires Special Action Return Types](#7-streaming-requires-special-action-return-types)
  - [8. Application IDs for Persistence](#8-application-ids-for-persistence)
  - [9. Actions Must Return State](#9-actions-must-return-state)
  - [10. Tracker vs Persister](#10-tracker-vs-persister)

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
