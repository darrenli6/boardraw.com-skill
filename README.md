# boardraw-skill

> **Language:** English | [中文](./README-zh.md)

A [Claude Code](https://claude.ai/code) skill that turns natural language into fully editable flowcharts and uploads them directly to [boardraw.com](https://boardraw.com).

Describe a process in plain English (or any language) — Claude designs the diagram, asks for your confirmation, builds it with the `ExcalidrawDoc` builder (so every element stays movable, restyable, and reconnectable), then publishes it to your boardraw workspace with a shareable link.

---

## Demo

./assets/boardraw-skill.mp4

```
You:    Draw a CI/CD pipeline: code push triggers tests,
        if tests pass deploy to staging, manual approval
        then deploy to production, notify Slack on success or failure.

Claude: 📋 Diagram Preview
        Title: CI/CD Pipeline

          [Code Push] → [Run Tests] → {Tests Pass?}
                                        ├── Yes → [Deploy Staging] → {Approved?}
                                        │                               ├── Yes → [Deploy Production] → [Notify Slack ✅]
                                        │                               └── No  → [Notify Slack ❌]
                                        └── No  → [Notify Slack ❌]

        Does this match what you need? Confirm and I'll generate and upload it.

You:    Looks good, go ahead.

Claude: ✅ Diagram uploaded to boardraw.com!

        🔗 View / edit online: https://www.boardraw.com/workspace/c2f9b3b6-4b5e-4c22-9a28-fb3e2a4c9d11
        📁 File name:          skill--a83f29cd
        👤 Account:            Olivia Carter (olivia@example.com)

        The file can also be opened directly in Excalidraw or any compatible tool.
```

---

## What You Can Draw

| Type | Example prompt |
|---|---|
| Flowchart | "Draw a user registration flow with email validation" |
| Mind map | "Make a mind map for our Q3 product roadmap" |
| Architecture diagram | "Diagram a microservices system: API gateway, auth service, three backends" |
| Org chart | "Draw the engineering org: CTO → 3 EMs → 5 ICs each" |
| Kanban / sticky-note wall | "Create a kanban board with Todo, In Progress, Done columns" |

---

## Requirements

| Requirement | Notes |
|---|---|
| [Claude Code](https://claude.ai/code) | CLI, desktop app, or IDE extension |
| boardraw.com account | **Pro or Team** plan required for API access |
| Python 3.8+ | Required — the builder script generates Excalidraw JSON |

---

## Installation

### 1. Clone this repository

```bash
git clone https://github.com/your-org/boardraw.com-skill.git
cd boardraw.com-skill
```

### 2. Register the skill with Claude Code

Add the skill to your Claude Code user settings so it's available in every project:

```bash
claude config add skills "$(pwd)/boardraw"
```

Or register it only for the current project by adding to `.claude/settings.json`:

```json
{
  "skills": [
    "/absolute/path/to/boardraw.com-skill/boardraw"
  ]
}
```

### 3. Configure your API key

Copy the example env file and fill in your key:

```bash
cp .env.example .env
```

Edit `.env`:

```env
BOARDRAW_API_KEY=br_your_api_key_here
```

To get an API key: log in to [boardraw.com](https://boardraw.com) → **Settings → API Keys** → Create new key (requires Pro or Team plan).

> The skill reads `BOARDRAW_API_KEY` from your shell environment or from a `.env` file anywhere up the directory tree from your working directory.

---

## Usage

Once installed, just describe what you want in Claude Code:

```
Draw a user onboarding flow for a mobile app.
```

```
Create a flowchart for our order processing system:
orders come in, inventory is checked, payment is processed,
then the order ships. If inventory is low, trigger a reorder first.
```

```
/generate Draw an architecture diagram: React frontend → API gateway →
auth service + order service + notification service → PostgreSQL + Redis
```

Claude will:
1. **Preview** the diagram as a text outline
2. **Wait** for your confirmation (or adjustments)
3. **Build** the `.excalidraw` file using `scripts/excalidraw_builder.py`
4. **Validate** the JSON, then **upload** to boardraw.com
5. **Return** a shareable link — fully editable in boardraw or Excalidraw

---

## Standalone Scripts

### Upload an existing `.excalidraw` file

```bash
# From a file
python boardraw/scripts/upload.py whiteboard.excalidraw

# From stdin
cat whiteboard.excalidraw | python boardraw/scripts/upload.py -
```

### Use the builder directly

```python
from boardraw.scripts.excalidraw_builder import ExcalidrawDoc, vertical_flow_positions

doc = ExcalidrawDoc()
positions = vertical_flow_positions(3, x0=300, y0=40, w=200, h=80, gap=80)

for (x, y), label in zip(positions, ["Start", "Process", "End"]):
    shape = doc.rectangle(x, y, 200, 80, bg="blue")
    doc.label(shape, label)

doc.save("output.excalidraw")
```

Both scripts use the same `BOARDRAW_API_KEY` lookup (env var → `.env` file walk-up). No third-party dependencies required.

---

## Project Structure

```
boardraw.com-skill/
├── boardraw/
│   ├── SKILL.md                    # Skill definition — Claude reads this when invoked
│   ├── scripts/
│   │   ├── excalidraw_builder.py   # ExcalidrawDoc builder — generates valid Excalidraw JSON
│   │   └── upload.py               # Standalone upload script (stdlib only)
│   └── references/
│       └── excalidraw-schema.md    # Raw JSON field reference for advanced customisation
├── .env.example                    # API key template
├── README.md                       # This file (English)
└── README-zh.md                    # Chinese translation
```

---

## How It Works

```
User prompt
    │
    ▼
[Claude parses intent → text outline]
    │
    │ user confirms (or adjusts)
    ▼
[excalidraw_builder.py builds elements]   ← never hand-written JSON
    │
    ▼
[JSON validated]
    │
    ▼
[POST /api/keys/auth → boardraw.com]
    │
    ▼
[Shareable link returned]
```

**Why a builder script?** The Excalidraw format requires strict two-way binding between shapes and their labels (`boundElements` + `containerId`), precise arrow snapping (`startBinding` / `endBinding`), and unique random `seed` / `versionNonce` per element. Hand-writing this JSON reliably is error-prone. The builder handles all of it automatically.

---

## API Reference

**Endpoint:** `POST https://www.boardraw.com/api/keys/auth`

| Item | Value |
|---|---|
| Auth header | `x-api-key: br_...` |
| Body | `{ "whiteboard": { ...excalidraw object... } }` |
| Success | `201` — returns `file.uuid` and user info |
| Plan required | Pro or Team |

**Error codes:**

| Status | Meaning |
|---|---|
| 401 | Missing or invalid / revoked API key |
| 403 | Free plan — upgrade required |
| 400 | Malformed whiteboard JSON |

---

## License

MIT — see [LICENSE](./LICENSE).
