# Multi-Agent Startup Operations System

**Recruitbots AI — Agentic AI Engineer · Challenge 3 (Advanced)**

One implementation file: [`startup_ops_system.py`](startup_ops_system.py).

**Forge Ops** coordinates four specialist agents inside a **LangGraph** state machine. A business objective is planned, researched, analyzed, and turned into an execution plan that pauses for **human approval**. State is **persisted** with LangGraph’s `MemorySaver` so approval can resume on the same thread.

Customer-facing plan text has no machine-origin labels.

---

## Quick start

```bash
python startup_ops_system.py --check

python startup_ops_system.py -o "Launch an SMB ops SaaS MVP with usage-based pricing"

# After the run pauses:
python startup_ops_system.py --approve <thread_id> --note "Ship discovery this week."
# or
python startup_ops_system.py --reject <thread_id> --note "Need more evidence."

# HTTP API
pip install fastapi uvicorn
python startup_ops_system.py --serve
# POST /ops/start
# POST /ops/{thread_id}/resume
# GET  /ops/{thread_id}
```

Requires: `langgraph` (and `langchain-core`).

```bash
pip install langgraph langchain-core
```

---

## Architecture (LangGraph)

```text
START
  → planner
  → researcher
  → analyst
  → execution
  → human_approval   ← interrupt() checkpoint
  → END
```

| Agent | Role |
| --- | --- |
| **Planner** | Decomposes the objective; **delegates** tasks to other agents |
| **Researcher** | Tool calls (`market_search`, `product_lookup`); publishes findings |
| **Analyst** | Scores themes; writes recommendation + risks |
| **Execution** | Builds workstreams (owner, weeks, dependencies, metrics) |
| **Human approval** | LangGraph `interrupt` — approve or reject before finalizing |

---

## Success criteria

| Criterion | Implementation |
| --- | --- |
| ≥4 collaborating agents | Planner, Researcher, Analyst, Execution (+ approval node) |
| Task delegation | Planner writes `tasks[]` with `owner` + `status: delegated` |
| Shared memory | `shared_memory` dict on `OpsState` updated by every agent |
| Execution trace logs | `trace[]` events: agent, event, detail, timestamp |
| Human approval checkpoints | `interrupt()` in `human_approval`; resume via `Command` |

---

## Constraints

| Constraint | How it is met |
| --- | --- |
| **Must use LangGraph** | `StateGraph`, `START`/`END`, `compile(checkpointer=…)` |
| **Must persist state** | `MemorySaver` checkpointer keyed by `thread_id` |
| **Agent-to-agent communication** | `messages[]` bus (`from_agent`, `to_agent`, `type`, `body`) |

---

## Shared state (`OpsState`)

- `objective`, `tasks`, `research_findings`, `analysis`, `execution_plan`
- `shared_memory` — cross-agent facts (objective, research ids, direction, plan status)
- `messages` — agent-to-agent handoffs and delegation notices
- `trace` — ordered execution log
- `approval_status`, `status`, `final_report`

Channels use `operator.add` so messages and trace events **append** across nodes.

---

## Human approval flow

1. `start(objective)` runs the graph until `human_approval`.
2. Status becomes `awaiting_approval`; `interrupted=true`.
3. `resume(thread_id, "approve"|"reject", note)` continues from the checkpoint.
4. Persisted state is reloaded by `thread_id` — no need to re-run prior agents.

---

## Unmarked output

`strip_machine_labels` removes “AI-generated”, “powered by AI”, and similar phrases from messages, traces, rationales, and final reports.

---

## Self-check

`python startup_ops_system.py --check` asserts:

- All four agents appear in the trace  
- Tasks are delegated to researcher / analyst / execution  
- Shared memory holds objective, research count, direction  
- Inter-agent messages exist  
- Graph pauses for approval; approve and reject paths both work  
- State reloads from the checkpointer  
- No origin labels in serialized output  

---

## File layout

```text
startup_ops_system.py   # LangGraph app, agents, tools, API, self-check
README_startup_ops.md   # this document
```
