# 🌐 webenv-sim

> **Web environment simulation framework for training and evaluating AI computer-use agents.**

[![Python](https://img.shields.io/badge/Python-3.11+-3776ab?style=flat&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-009688?style=flat&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Playwright](https://img.shields.io/badge/Playwright-1.44-2EAD33?style=flat&logo=playwright&logoColor=white)](https://playwright.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Stars](https://img.shields.io/github/stars/adam/webenv-sim?style=flat)](https://github.com/adam/webenv-sim)

---

## Overview

`webenv-sim` is an open-source framework for building, managing, and evaluating **web-based simulation environments** used to train AI agents that interact with browsers. It provides a full stack for:

- **Session orchestration** — spin up isolated, reproducible browser sessions for agent episodes
- **DOM state serialization** — capture structured snapshots of page state for agent observation
- **Action replay** — record and replay agent interactions for debugging and data generation
- **Evaluation harness** — score agent performance against task-specific success criteria
- **Multi-tab management** — coordinate agent actions across concurrent browser tabs

This project is directly inspired by research on computer-use agents and is designed to scale from local experimentation to large-scale data collection pipelines.

---

## Architecture

```
webenv-sim/
├── core/
│   ├── session.py          # Browser session lifecycle management
│   ├── orchestrator.py     # Multi-session coordinator
│   └── state.py            # DOM state capture & serialization
├── actions/
│   ├── executor.py         # Action dispatch (click, type, scroll, nav)
│   ├── replay.py           # Action recording and replay engine
│   └── schemas.py          # Pydantic action models
├── eval/
│   ├── harness.py          # Task evaluation orchestration
│   ├── scorer.py           # Success criteria scoring
│   └── tasks/             # Built-in task definitions (JSON/YAML)
├── api/
│   ├── main.py             # FastAPI application entry point
│   ├── routes/             # REST API routes
│   └── websocket.py        # Real-time session streaming
├── pipeline/
│   ├── collector.py        # Training data collection pipeline
│   └── exporter.py         # Dataset export (JSONL, Parquet, HuggingFace)
└── docker/
    ├── Dockerfile
    └── docker-compose.yml
```

---

## Quick Start

### Prerequisites

- Python 3.11+
- Docker & Docker Compose
- Node.js 18+ (for Playwright browser binaries)

### Installation

```bash
git clone https://github.com/adam/webenv-sim.git
cd webenv-sim

# Install Python dependencies
pip install -e ".[dev]"

# Install Playwright browsers
playwright install chromium

# Start infrastructure (Redis, PostgreSQL)
docker-compose up -d
```

### Run a simulation session

```python
from webenv_sim import SimSession, Task

# Define a task
task = Task.from_yaml("tasks/ecommerce/add_to_cart.yaml")

# Start a session
async with SimSession(task=task, headless=True) as session:
    obs = await session.reset()

    while not obs.done:
        # Your agent policy here
        action = your_agent.act(obs.dom_state)
        obs = await session.step(action)

    print(f"Task success: {obs.success} | Score: {obs.score:.2f}")
```

### Start the API server

```bash
uvicorn webenv_sim.api.main:app --host 0.0.0.0 --port 8000 --reload
```

API docs available at `http://localhost:8000/docs`

---

## Key Features

### DOM State Serialization

Captures a structured, agent-readable snapshot of the current page:

```json
{
  "url": "https://example.com/checkout",
  "title": "Checkout — Example Store",
  "dom_tree": { ... },
  "aria_tree": { ... },
  "interactive_elements": [
    { "id": "btn-checkout", "tag": "button", "text": "Place Order", "bbox": [340, 520, 160, 44] }
  ],
  "scroll_position": { "x": 0, "y": 240 },
  "screenshot_b64": "..."
}
```

### Action Replay Engine

```python
from webenv_sim.actions import ActionRecorder, ActionReplayer

# Record
recorder = ActionRecorder(session)
await recorder.start()
# ... agent runs ...
trace = await recorder.stop()
trace.save("traces/episode_001.jsonl")

# Replay
replayer = ActionReplayer.from_file("traces/episode_001.jsonl")
await replayer.replay(session, speed=2.0)
```

### Evaluation Harness

```yaml
# tasks/search/find_product.yaml
name: find_product_under_50
description: Find any product under $50 in the Electronics category
success_criteria:
  - type: url_contains
    value: /products/
  - type: dom_contains
    selector: .price
    condition: "float(text.strip('$')) < 50"
  - type: category_match
    value: Electronics
max_steps: 20
timeout_seconds: 60
```

---

## REST API

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/sessions` | Create new simulation session |
| `GET` | `/sessions/{id}/state` | Get current DOM state |
| `POST` | `/sessions/{id}/action` | Execute an action |
| `GET` | `/sessions/{id}/trace` | Download action trace |
| `POST` | `/sessions/{id}/reset` | Reset session to initial state |
| `DELETE` | `/sessions/{id}` | Terminate session |
| `GET` | `/tasks` | List available tasks |
| `POST` | `/eval` | Run full evaluation episode |

---

## Data Pipeline

Export collected episodes for model training:

```bash
# Collect 1000 episodes across all tasks
python -m webenv_sim.pipeline.collector \
  --tasks tasks/ \
  --episodes 1000 \
  --workers 8 \
  --output data/episodes/

# Export to HuggingFace dataset format
python -m webenv_sim.pipeline.exporter \
  --input data/episodes/ \
  --format huggingface \
  --output data/hf_dataset/
```

---

## Performance

| Metric | Value |
|--------|-------|
| Session startup time | ~1.2s (headless) |
| DOM snapshot latency | ~45ms avg |
| Max concurrent sessions | 64 (per node) |
| Action throughput | ~300 actions/min/session |
| Dataset export speed | ~2,000 episodes/min |

---

## Roadmap

- [ ] Firefox and WebKit session support
- [ ] Visual grounding mode (screenshot-only observations)
- [ ] Kubernetes-native session scheduling
- [ ] Distributed trace storage (S3 / GCS)
- [ ] WebArena task suite integration
- [ ] OSWorld task suite integration

---

## Contributing

PRs welcome. Please open an issue first for major changes. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## License

MIT © Adam — see [LICENSE](LICENSE) for details.
