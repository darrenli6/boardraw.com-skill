---
name: generate
description: 
  Convert natural language descriptions into visual flowcharts and automatically upload them to boardraw.com.
---

# Boardraw — Natural Language to Excalidraw Whiteboard

This skill turns a one-line user description ("draw a user registration flow", "make a mind map for this project", "diagram the microservice architecture") into a fully editable `.excalidraw` file — where every element can be moved, restyled, and reconnected inside Excalidraw or Boardraw — and then uploads it directly to boardraw.com.

**Core principle: never hand-write raw `.excalidraw` JSON.** The format has strict requirements around `id`, `seed`, `versionNonce`, and two-way `boundElements` linking. Hand-written JSON is error-prone (text that doesn't move with shapes, arrows that aren't snapped to nodes, etc.). Always use the `ExcalidrawDoc` class in `scripts/excalidraw_builder.py` to build elements.

---

## Workflow

### Step 1: Parse the User's Description into Structure

Mentally (or in a brief note) break the request down into:

- **Node / card list**: the label and type of each node (regular step / decision / start-end / sticky note)
- **Connections**: who connects to whom, direction, and any edge labels (e.g. "Yes / No" on decision branches)
- **Layout direction**: top-down flowchart? left-to-right? radial mind map? tree org chart? grid kanban?
- **Groups / regions**: should any nodes be wrapped in a named frame (e.g. "Frontend" / "Backend")?

If the description is vague (e.g. "draw a flowchart" with no steps given), fill in a sensible example skeleton and briefly note your interpretation in the reply — no need to block on repeated clarification.

Present the design as a **text outline** before generating anything:

```
📋 Diagram Preview
Title: User Registration Flow

  [Start] → [Fill Form] → [Validate Email] → {Email Valid?}
                                                ├── Yes → [Create Account] → [Send Welcome Email] → [End]
                                                └── No  → [Show Error] → [Fill Form]

Color scheme: Start/End = green, Decision = yellow, Action = blue
```

Then ask: **"Does this match what you need? Confirm and I'll generate and upload it to boardraw.com."**

Wait for confirmation before proceeding.

---

### Step 2: Choose a Layout

`scripts/excalidraw_builder.py` ships with coordinate helpers — no need to hand-calculate positions:

- `vertical_flow_positions(count, ...)` — top-down linear flowchart
- `horizontal_flow_positions(count, ...)` — left-to-right linear flowchart
- `grid_positions(count, cols, ...)` — grid layout (kanban, sticky-note wall, parallel cards)

For complex layouts (radial mind map branches, tree org charts, flowcharts with splits/merges), calculate coordinates directly — see the patterns below. The key rule: keep consistent spacing within each level or branch (horizontal flows: 150–250 px gap between nodes; vertical flows: 250–350 px row height avoids text overflow into adjacent nodes).

---

### Step 3: Build Elements with `ExcalidrawDoc`

```python
import sys
sys.path.insert(0, "scripts")   # adjust to the actual skill mount path if needed
from excalidraw_builder import ExcalidrawDoc, vertical_flow_positions

doc = ExcalidrawDoc()

positions = vertical_flow_positions(4, x0=300, y0=40, w=200, h=80, gap=80)
labels = ["Submit Request", "Compliance Check", "Generate Proposal", "Deliver to Client"]

nodes = []
for (x, y), text in zip(positions, labels):
    shape = doc.diamond(x - 20, y, 240, 100, bg="orange") if "Check" in text \
        else doc.rectangle(x, y, 200, 80, bg="blue")
    doc.label(shape, text)   # always bind text via label(), never float a separate text element on top
    nodes.append(shape)

for a, b in zip(nodes, nodes[1:]):
    doc.arrow(a, b)          # arrows auto-snap to shape edges

doc.arrow(nodes[1], nodes[0], label="Non-compliant, return", stroke="red")

doc.save("/mnt/user-data/outputs/request-processing-flow.excalidraw")
```

Run this via the Bash tool (write the script to a `.py` file and execute it). Do not produce Excalidraw JSON by hand.

---

### Step 4: Save, Validate, and Upload to boardraw.com

**1. Save** to `/mnt/user-data/outputs/<meaningful-name>.excalidraw` (UTF-8, no BOM; Chinese filenames are fine).

**2. Validate** the JSON is well-formed:
```bash
python3 -c "import json; json.load(open('output.excalidraw')); print('OK')"
```

**3. Upload** — read the API key from `BOARDRAW_API_KEY` (env var) or from a `.env` file walking up the directory tree. Then:

```bash
curl -s -X POST "https://www.boardraw.com/api/keys/auth" \
  -H "Content-Type: application/json" \
  -H "x-api-key: $BOARDRAW_API_KEY" \
  -d "{\"whiteboard\": $(cat output.excalidraw)}"
```

Or use `scripts/upload.py`:
```bash
python scripts/upload.py output.excalidraw
```

**Handle API responses:**

| Status | Action |
|--------|--------|
| 201 | Extract `file.uuid` → show `https://www.boardraw.com/workspace/{uuid}` |
| 401 (missing key) | Ask user to set `BOARDRAW_API_KEY` |
| 401 (invalid/revoked) | Ask user to verify the key in boardraw.com Settings |
| 403 | Ask user to upgrade to Pro or Team plan |
| 400 | The whiteboard JSON is malformed — re-run the builder script |

**4. Reply** — no need to list every node. A single line is enough:

```
✅ Diagram uploaded to boardraw.com!

🔗 View / edit online: https://www.boardraw.com/workspace/{uuid}
📁 File name:          {fileName}
👤 Account:            {user.name} ({user.email})

The file can also be opened directly in Excalidraw or any compatible tool.
```

---

## Common Diagram Patterns

### Flowchart
- Start / End → rounded rectangle or ellipse
- Process step → rectangle
- Decision → diamond (`doc.diamond`); label each outgoing arrow with "Yes / No" or the condition
- Merge branches by pointing multiple arrows at the same downstream node — no special handling needed

### Mind Map
- Central topic: a larger ellipse at canvas center, e.g. `doc.ellipse(x, y, 220, 100, bg="violet")`
- First-level branches: radiate outward (compute angle + radius with trigonometry; 8 branches → `angle = i * 360/8`)
- Connect with `doc.line()` or `doc.arrow(start_arrowhead=None, end_arrowhead=None)`
- Colour-code branch groups with `stroke=` (red / green / blue / orange / purple)

### Architecture / System Diagram
- Use `doc.frame(x, y, w, h, name="Frontend")` to group services by tier, then `doc.put_in_frame(el, frame_el)` to place elements inside
- Services → rectangle; databases / storage → ellipse or rounded rectangle with a distinct background
- Calls → arrows with `label=` for protocol / method (e.g. "HTTPS", "gRPC", "Redis pub/sub")

### Org Chart / Tree
- Place each level at an increasing `y` offset; `grid_positions` or manual coordinates both work
- Connect parent → child with arrows; `end_arrowhead="triangle"` looks more org-chart-like; `None` for plain lines

### Kanban / Sticky-Note Wall
- `grid_positions(count, cols=3)` for a uniform grid of coloured rectangles
- Use `bg=` to colour-code by category, priority, or owner
- No arrows needed

---

## Technical Rules

1. **`seed` and `versionNonce`**: distinct random integers per element (the builder handles this automatically).
2. **`id` uniqueness**: every element id must be unique in the `elements` array.
3. **Text binding**: text inside a container must set `containerId`; the container must include that text id in `boundElements`. Use `doc.label()` — it handles both sides.
4. **Arrow coordinates**: `x`/`y` is the absolute start point; `points` are offsets relative to that start. Use `doc.arrow()` — it calculates these automatically.
5. **`updated` timestamp**: use the current millisecond timestamp.
6. **Valid JSON**: no comments, no trailing commas.

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

---

## References

- `references/excalidraw-schema.md` — full raw JSON field reference (common element fields, text/arrow/line-specific fields, colour palette, frame rules). Only consult this when `excalidraw_builder.py`'s methods don't cover a custom requirement (freedraw, image elements, non-default font sizes, special corner radii). The builder's methods are sufficient for the vast majority of diagrams.
