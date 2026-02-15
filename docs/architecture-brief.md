# Sachiv-AI: Architecture Brief

## Vision

Sachiv-AI is a hierarchical multi-agent AI assistant framework. It operates like an organization — with a CEO, department managers, and doers — where tasks flow down the hierarchy and results flow back up.

The framework is **standalone** — it is the AI brain. Physical embodiments (like OLAF robot hardware) connect as MCP servers, making them optional extensions rather than core dependencies.

## Core Concept

Every agent in the system — whether it's a simple doer or a manager with 50 agents beneath it — has the **same interface**:

```
Input (text/image) → [Agent] → Output (text)
```

This makes the hierarchy **recursive**. A manager doesn't know or care if a report is a simple doer or another manager with its own team. It sends work down and gets results back.

## Architecture

### Hierarchy

```
CEO (top-level agent)
  └── Team A (manager + reports)
        ├── Doer 1 (leaf agent with MCP tools)
        ├── Doer 2
        └── Team A1 (another manager + reports)
              ├── Doer 3
              └── Doer 4
```

- **CEO**: Understands user intent, splits into department-level tasks, synthesizes final response
- **Managers**: Plan subtasks for their reports, delegate, optionally review results, retry if not satisfied
- **Doers**: Execute work using MCP tools and functions, return results

### Agent Roles

**Managers:**
- Read task from above
- Plan — break into subtasks for their reports
- Delegate — assign to the right report(s)
- Review — check output quality (if task requires it)
- Retry — send back with feedback if not satisfied
- Summarize — report results up the chain

**Doers:**
- Receive a task
- Execute using MCP tools / functions
- Return result

### Static Configuration

The agent hierarchy is **defined at startup**, not dynamically spawned at runtime. Configuration files define:
- The org chart (who reports to whom)
- Which model each agent uses
- Which MCP servers each doer connects to

Each agent has a **.md file** describing:
- What it can do (capabilities)
- What its reports can do (for managers)
- Quality criteria for reviewing output
- Input/output expectations

## Task Flow

```
User: "I'm cold and it's dark"
  → CEO reads department .md files, creates two tasks
    → Home Manager receives tasks, reads its reports' .md files
      → Climate Doer: set_temperature(72) → success
      → Lights Doer: set_scene("warm") → success
    → Home Manager summarizes results → CEO
  → CEO: "I've warmed up the house and set warm lighting"
```

## Review System

Not every task needs review. The task definition carries a review flag:

- **none** — fire and forget, trust the doer
- **status** — check success/failure (code logic, no LLM call needed)
- **quality** — LLM reviews the output (for important tasks like research)

```
Task:
  action: "turn on living room lights"
  review: none

Task:
  action: "research best pizza places nearby"
  review: quality
  max_retries: 3
```

## Model Strategy

Each agent can use a **different LLM provider and model**, chosen for cost/capability tradeoff:

| Role | Model | Why |
|---|---|---|
| CEO | Claude Sonnet | Best reasoning for complex intent understanding |
| Managers | Groq Llama 70B or Qwen 32B | Fast, cheap, good enough for planning |
| Doers | Groq Llama 8B | Dirt cheap, blazing fast for simple tool execution |
| Future | Fine-tuned local models | Zero cost, runs on Pi |

### Provider Options

- **Groq** — cheapest and fastest (Llama 8B at $0.05/$0.08 per 1M tokens, ~800 tok/s)
- **Google Gemini** — good free tier, cheap Flash-Lite models
- **Anthropic Claude** — best reasoning quality, used selectively for CEO/complex tasks
- **OpenAI** — GPT-5 Nano as cheap alternative
- **Local (Ollama)** — future option for zero-cost routing on Pi

## Token and Cost Logging

Every agent call logs time and token usage — pure bookkeeping, no LLM needed:

- Time elapsed per task
- Tokens used (input/output) per agent
- Cost per task, per team, per day
- Retry counts
- Can set budget limits per department

## Framework: Google ADK

**Google Agent Development Kit (ADK)** was chosen because it supports:

- **Recursive agent hierarchy** via `BaseAgent` subclass
- **Mixed LLM providers** via LiteLLM integration (Groq, Claude, Gemini, local — all in one tree)
- **Custom agent loops** — full code control over plan/delegate/review/retry
- **Built-in memory and RAG** via `MemoryService`
- **Structured output** with Pydantic schema validation
- **MCP support** via `McpToolset`
- **Multimodal input** (text + images)

### Why Not Other Frameworks

| Framework | Reason for rejection |
|---|---|
| Claude Agent SDK | Cannot nest agents (single level only), Claude models only |
| OpenAI Agents SDK | Flat handoff pattern, no deep hierarchy support |
| CrewAI | Opinionated, would fight our custom design |
| Custom from scratch | Too much plumbing (output parsing, retries, memory, etc.) |

## Integration Points

### MCP Servers (plugins)

The AI connects to capabilities via MCP servers:

```
Sachiv-AI
  ├── MCP Server: OLAF hardware (ears, eyes, neck, LEDs)
  ├── MCP Server: Smart Home (lights, thermostat, locks)
  ├── MCP Server: Calendar (events, reminders)
  ├── MCP Server: Web Search
  └── MCP Server: ... (anything)
```

OLAF robot hardware is just one MCP server — the AI works without it.

### Deployment

Both Sachiv-AI and OLAF run on a Raspberry Pi:

```
[Sachiv-AI] --MCP--> [OLAF MCP Server] --ROS2--> [Hardware Drivers] --serial--> [Servos/LEDs]
```

## Future Considerations (Out of Scope for v1)

- **RAG / Graph RAG** — slots in as a Knowledge department (manager + RAG doer + search doer)
- **Scheduling** — recurring tasks (cron-like, no LLM needed to trigger)
- **Priority / Interrupts** — urgent tasks preempt ongoing work
- **User profile** — shared preferences doc that agents learn from
- **Fine-tuning** — train small models per department using collected conversation data
- **Shared skills** — internet search, file I/O as shared MCP servers any team can access
- **Health dashboard** — web UI showing agent status, costs, failures

## v1 Scope

- Framework: hierarchy, task flow, .md configs, token logging
- CEO + one manager + couple of doers
- Google ADK with LiteLLM for multi-provider
- OLAF MCP server as first hardware integration
- Simple memory at agent level
- Review system (none / status / quality)
