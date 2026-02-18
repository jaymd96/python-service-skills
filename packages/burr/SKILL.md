---
name: burr
description: State machine framework for AI agent workflows with persistence and tracking UI. Use when building multi-step LLM agents, stateful AI workflows, resumable pipelines, or human-in-the-loop systems. Triggers on burr, state machine, AI agent workflow, LLM pipeline, agent state, action graph, workflow orchestration.
---

# Burr — AI Agent State Machines (v0.30.0)

## Quick Start

```bash
pip install "burr[tracking]"
burr  # launches tracking UI at localhost:7241
```

```python
from burr.core import action, State, ApplicationBuilder

@action(reads=["count"], writes=["count"])
def increment(state: State) -> State:
    return state.update(count=state["count"] + 1)

app = (
    ApplicationBuilder()
    .with_actions(increment=increment)
    .with_transitions(("increment", "increment", default))
    .with_state(count=0)
    .with_entrypoint("increment")
    .build()
)
action, result, state = app.step()
```

## Key Patterns

### Define actions with conditional transitions
```python
from burr.core import action, State, when, default, ApplicationBuilder

@action(reads=["messages"], writes=["messages", "response"])
def call_llm(state: State) -> State:
    messages = state["messages"]
    response = client.chat(messages=messages)
    return state.update(response=response).append(
        messages={"role": "assistant", "content": response}
    )

app = (
    ApplicationBuilder()
    .with_actions(ask=ask_user, respond=call_llm, done=finish)
    .with_transitions(
        ("ask", "done", when(quit=True)),
        ("ask", "respond", default),
        ("respond", "ask", default),
    )
    .with_state(messages=[], quit=False)
    .with_entrypoint("ask")
    .build()
)
action, result, state = app.run(halt_after=["done"])
```

## References

- **[api.md](references/api.md)** — State, @action, ApplicationBuilder, transitions, persistence, tracking UI, hooks, streaming
- **[examples.md](references/examples.md)** — Complete examples (chatbot, RAG, tool-use agent), gotchas
