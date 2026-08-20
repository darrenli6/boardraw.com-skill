# boardraw-skill

> **Language:** English | [中文](./README-zh.md)

A [Claude Code](https://claude.ai/code) skill that turns natural language into flowcharts and uploads them directly to [boardraw.com](https://boardraw.com).

Describe a process in plain English (or any language) — Claude designs the diagram, asks for your confirmation, then publishes it to your boardraw workspace with a shareable link.

---

## Demo

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

        Does this match what you need? Confirm and I'll upload it to boardraw.com.

You:    Looks good, go ahead.

Claude: ✅ Uploaded!
        🔗 https://www.boardraw.com/board/c2f9b3b6-4b5e-4c22-9a28-fb3e2a4c9d11
```

---

## Requirements

| Requirement | Notes |
|---|---|
| [Claude Code](https://claude.ai/code) | CLI, desktop app, or IDE extension |
| boardraw.com account | **Pro or Team** plan required for API access |
| Python 3.8+ | Only needed if you use the standalone upload script |

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
claude config add skills "$(pwd)/generate"
```

Or register it only for the current project by adding to `.claude/settings.json`:

```json
{
  "skills": [
    "/absolute/path/to/boardraw.com-skill/generate"
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

To get an API key: log in to [boardraw.com](https://boardraw.com) → **Settings → API Keys** → Create new key.

> The skill reads `BOARDRAW_API_KEY` from your shell environment or from a `.env` file anywhere up the directory tree from your working directory.

---

## Usage

Once installed, just describe what you want in Claude Code:

```
Draw a user onboarding flow for a mobile app.
```

```
Create a flowchart showing how our order processing system works:
orders come in, inventory is checked, payment is processed,
then the order ships. If inventory is low, reorder first.
```

```
/generate Draw an architecture diagram for a microservices system
with API gateway, auth service, and three backend services.
```

Claude will:
1. **Preview** the diagram as a text outline
2. **Wait** for your confirmation (or adjustments)
3. **Generate** the Excalidraw JSON
4. **Upload** to boardraw.com and return a shareable link

---

## Standalone Upload Script

If you want to upload a pre-built Excalidraw JSON file without going through the skill:

```bash
# From a file
python generate/scripts/upload.py whiteboard.json

# From stdin
cat whiteboard.json | python generate/scripts/upload.py -
```

The script uses the same `BOARDRAW_API_KEY` lookup (env var → `.env` file). No third-party dependencies required.

---

## Project Structure

```
boardraw.com-skill/
├── generate/
│   ├── SKILL.md          # Skill definition — Claude reads this when invoked
│   └── scripts/
│       └── upload.py     # Standalone Python upload script (stdlib only)
├── .env.example          # API key template
├── README.md             # This file (English)
└── README-zh.md          # Chinese translation
```

---

## How It Works

```
User prompt
    │
    ▼
[Claude parses intent]
    │
    ▼
[Text outline shown to user] ──── user adjusts ────┐
    │ confirmed                                      │
    ▼                                               │
[Excalidraw JSON generated] ◄───────────────────────┘
    │
    ▼
[POST /api/keys/auth → boardraw.com]
    │
    ▼
[Shareable link returned]
```

The generated JSON follows the [Excalidraw](https://excalidraw.com) format — it can also be opened directly in Excalidraw or any compatible tool.

---

## API Reference

**Endpoint:** `POST https://www.boardraw.com/api/keys/auth`

| Item | Value |
|---|---|
| Auth header | `api_key: br_...` |
| Body | `{ "whiteboard": { ...excalidraw object... } }` |
| Success | `201` — returns file UUID and boardraw.com URL |
| Plan required | Pro or Team |

---

## License

MIT — see [LICENSE](./LICENSE).
