# Boardraw Flowchart Generator

Convert natural language descriptions into visual flowcharts and automatically upload them to [boardraw.com](https://boardraw.com).

## How to Trigger

User describes any process, architecture, or system design — or invokes this skill directly.

---

## Execution Flow

### Step 1: Understand and Design the Diagram

Analyze the user's input and extract key nodes and relationships. Present the design as a **text outline** before generating any JSON:

```
📋 Diagram Preview
Title: User Registration Flow

Nodes:
  [Start] → [Fill Form] → [Validate Email] → {Email Valid?}
                                                ├── Yes → [Create Account] → [Send Welcome Email] → [End]
                                                └── No  → [Show Error] → [Fill Form]

Color scheme: Start/End = green, Decision = yellow, Action = blue
```

Then explicitly ask: **"Does this design match what you need? Please confirm and I'll upload it to boardraw.com."**

Wait for the user's reply before proceeding.

---

### Step 2: Generate Excalidraw JSON

After confirmation, build the complete Excalidraw JSON structure.

**Element Specs:**

#### Rectangle Node (regular step)
```json
{
  "id": "rect_1",
  "type": "rectangle",
  "x": 100, "y": 100,
  "width": 160, "height": 70,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "#e9ecef",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": {"type": 3},
  "seed": 123456,
  "version": 1,
  "versionNonce": 654321,
  "isDeleted": false,
  "boundElements": [{"type": "text", "id": "text_1"}],
  "updated": 1786809030962,
  "link": null,
  "locked": false
}
```

#### Text (bound inside a container)
```json
{
  "id": "text_1",
  "type": "text",
  "x": 112, "y": 120,
  "width": 136, "height": 30,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": null,
  "seed": 789012,
  "version": 1,
  "versionNonce": 210987,
  "isDeleted": false,
  "boundElements": null,
  "updated": 1786809030962,
  "link": null,
  "locked": false,
  "text": "Step Name",
  "fontSize": 16,
  "fontFamily": 1,
  "textAlign": "center",
  "verticalAlign": "middle",
  "containerId": "rect_1",
  "originalText": "Step Name",
  "lineHeight": 1.25,
  "baseline": 14
}
```

#### Diamond Node (decision/branch)
Use `"type": "diamond"` — all other fields same as rectangle.

#### Arrow (connection)
```json
{
  "id": "arrow_1",
  "type": "arrow",
  "x": 260, "y": 135,
  "width": 60, "height": 0,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": {"type": 2},
  "seed": 345678,
  "version": 1,
  "versionNonce": 876543,
  "isDeleted": false,
  "boundElements": null,
  "updated": 1786809030962,
  "link": null,
  "locked": false,
  "points": [[0, 0], [60, 0]],
  "lastCommittedPoint": null,
  "startBinding": {"elementId": "rect_1", "focus": 0, "gap": 4},
  "endBinding": {"elementId": "rect_2", "focus": 0, "gap": 4},
  "startArrowhead": null,
  "endArrowhead": "arrow"
}
```

**Layout Rules:**
- Horizontal flow: node spacing 250px (node width 160 + gap 90)
- Vertical flow: node spacing 150px (node height 70 + gap 80)
- Start at x=80, y=80
- Title text: y=30, fontSize=24, strokeColor="#1e1e1e"
- Node color scheme:
  - Start / End: `strokeColor: "#2f9e44"`, `backgroundColor: "#b2f2bb"`
  - Regular step: `strokeColor: "#1e1e1e"`, `backgroundColor: "#e9ecef"`
  - Decision: `strokeColor: "#f08c00"`, `backgroundColor: "#ffec99"`
  - Highlight: `strokeColor: "#1971c2"`, `backgroundColor: "#a5d8ff"`
  - Warning / Error: `strokeColor: "#c92a2a"`, `backgroundColor: "#ffc9c9"`

**Complete whiteboard structure:**
```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://boardraw.com",
  "elements": ["...all nodes and arrows..."],
  "appState": {
    "gridSize": 20,
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
}
```

---

### Step 3: Upload to boardraw.com

Read the API key from the `BOARDRAW_API_KEY` environment variable. If not set, try loading it from a `.env` file in the current working directory. If still missing, prompt the user to configure it (see below).

Use the Bash tool to run:
```bash
curl -s -X POST "https://www.boardraw.com/api/keys/auth" \
  -H "Content-Type: application/json" \
  -H "api_key: $BOARDRAW_API_KEY" \
  -d "{\"whiteboard\": $WHITEBOARD_JSON}"
```

**Handle responses:**

| Status | Action |
|--------|--------|
| 201 | Extract `file.uuid`, build link `https://www.boardraw.com/board/{uuid}`, show to user |
| 401 (missing key) | Tell user to set `BOARDRAW_API_KEY` |
| 401 (invalid/revoked) | Tell user to verify the API key in boardraw.com settings |
| 403 | Tell user to upgrade to Pro or Team plan |
| 400 | Check the generated whiteboard JSON for syntax errors |

---

### Step 4: Return the Result

On success, output:

```
✅ Diagram uploaded to boardraw.com!

🔗 View online: https://www.boardraw.com/board/{uuid}
📁 File name:   {fileName}
👤 Account:     {user.name} ({user.email})
```

---

## Technical Rules

1. **`seed` and `versionNonce`**: Use distinct random integers per element.
2. **`id` uniqueness**: Every element id must be unique within the `elements` array.
3. **Text binding**: Text inside a container must set `containerId`; the container must list that text in `boundElements`.
4. **Arrow coordinates**: `x`/`y` on an arrow is the absolute start point; `points` are offsets relative to that start point.
5. **`updated` timestamp**: Use the current millisecond timestamp.
6. **Valid JSON**: Ensure the whiteboard object has no comments or trailing commas before uploading.

---

## API Key Setup (show to user if missing)

```
To use this skill, configure your boardraw.com API Key:

1. Log in to boardraw.com → Settings → API Keys
2. Create a new API Key (requires Pro or Team plan)
3. Add it to a .env file in your project root:
   BOARDRAW_API_KEY=br_your_api_key_here
   
   Or export it as an environment variable:
   export BOARDRAW_API_KEY=br_your_api_key_here
```
