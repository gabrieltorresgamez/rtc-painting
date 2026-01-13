# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**rtc-painting** is a real-time collaborative ephemeral drawing canvas application. Users can draw trails by holding down the mouse button while moving the cursor, and see all other users' cursors as colored bubbles in real time. The application is anonymous, ephemeral (drawings fade after ~60 seconds), and runs in the terminal (with potential browser support).

**Tech Stack:**
- **Frontend/UI:** Python **Textual** (TUI framework for terminal UIs with mouse events, Canvas/Screen widgets)
- **Backend/Realtime:** **Convex** (reactive serverless backend with real-time database)
- **Package Manager:** `uv` (for Python dependencies)
- **Python Version:** 3.14

## Development Commands

### Environment Setup
```bash
# Install dependencies (uv handles virtualenv automatically)
uv sync

# Activate virtualenv manually if needed
source .venv/bin/activate
```

### Running the Application
```bash
# Run the main application
uv run python main.py

# Or with activated venv
python main.py
```

### Convex Backend
```bash
# Initialize Convex (first time only)
npx convex dev

# Deploy Convex functions
npx convex deploy

# Run a specific Convex function
npx convex run cursors:list
```

### Testing
```bash
# Run all tests with pytest
uv run pytest

# Run specific test file
uv run pytest tests/test_canvas.py

# Run with verbose output
uv run pytest -v
```

### Linting
```bash
# Run ruff linter
uv run ruff check .

# Auto-fix issues
uv run ruff check --fix .

# Format code
uv run ruff format .
```

## Architecture Overview

### Core Concept
The application implements a **"cursor-centric"** real-time collaboration model:

1. **Presence (Always On):** All users' cursor positions are always visible as colored bubbles, regardless of whether they're drawing
2. **Drawing (On-Demand):** Line segments are only created while mouse button is pressed (`MouseDown` → `MouseUp`)
3. **Ephemeral State:** Line segments fade linearly over ~60 seconds and disappear automatically
4. **Anonymous Identity:** Each user gets a persistent anonymous ID (stored locally) with a unique HSL/ANSI color

### Data Flow Pattern

```
User Input (Textual) → Local State Update → Convex Mutation
                  ↓                              ↓
           Local Rendering ←─────────── Convex Real-time Broadcast
                  ↓
         Other Users' Screens
```

**Key principle:** "Attributes down, messages up" (from Textual architecture)
- Parent components update children via attributes
- Children notify parents via message bubbling

### Convex Backend Schema

**Collection:** `cursors`

**Fields per cursor:**
- `id` — unique anonymous user ID (string)
- `x`, `y` — normalized cursor position (0.0 to 1.0)
- `color` — user's assigned color (HSL/hex string)
- `drawing` — boolean flag indicating if user is actively drawing
- `timestamp` — last update time (for staleness detection and fading)

**Update frequency:** Cursor positions throttled to 30-60ms intervals to balance responsiveness and bandwidth.

### Textual Frontend Architecture

**Key Components:**

1. **Canvas Widget (Primary):**
   - Full-screen custom Textual widget
   - Handles all mouse events: `on_mouse_move`, `on_mouse_down`, `on_mouse_up`
   - Renders line segments locally while `drawing=True`
   - Renders all users' cursor bubbles at their latest positions
   - Implements fade-out logic for line segments based on timestamp

2. **Convex Client Integration:**
   - Python Convex client (`convex.ConvexClient`) for real-time subscriptions
   - Subscribe to cursor updates: `client.subscribe("cursors:list")`
   - Push cursor updates: `client.mutation("cursors:update", {...})`

3. **Reactive State Management:**
   - Use Textual's `reactive()` for state that affects UI (cursor positions, drawing state)
   - Use `watch_*()` methods to respond to state changes
   - Use `@work` decorator for non-blocking Convex operations

### Mouse Interaction Pattern

**Critical implementation details:**

- Use `on_mouse_move()` to update cursor position continuously (even when not drawing)
- Use `on_mouse_down()` to set `drawing=True` and start capturing
- Use `on_mouse_up()` to set `drawing=False` and release capture
- Use `capture_mouse()` / `release_mouse()` to track dragging outside widget bounds
- Prefer `Click` event over `MouseDown` for non-drawing actions

**Coordinate systems:**
- Store normalized coordinates (0.0 to 1.0) in Convex for resolution independence
- Convert to widget-relative coordinates for local rendering
- Handle terminal resize gracefully by recalculating positions

### Line Segment Fade Logic

Line segments should:
1. Store creation timestamp when drawn
2. Calculate age: `current_time - timestamp`
3. Apply linear alpha fade: `alpha = max(0, 1 - age / 60.0)`
4. Remove segments older than 60 seconds from render queue

## Key Design Constraints

1. **No erase, undo, or clear** — Only automatic fade-out
2. **Anonymous only** — No usernames, profiles, or authentication
3. **Minimal controls** — Move cursor + click to draw, that's it
4. **Focus-aware** — Drawing only happens when app is focused
5. **Ephemeral by design** — Nothing persists beyond 60 seconds

## Convex Integration Notes

**When implementing Convex functions (TypeScript in `convex/` directory):**

- Use `query()` for read-only operations (subscribable)
- Use `mutation()` for writes (ACID transactions, auto-retry on conflict)
- Use `action()` only if calling external APIs (not needed for this project)
- Define schema in `convex/schema.ts` for type safety
- Function naming: `filename:functionName` (e.g., `"cursors:update"`)

**Python client patterns:**

```python
from convex import ConvexClient

client = ConvexClient(os.getenv("CONVEX_URL"))

# Query data
cursors = client.query("cursors:list")

# Mutate data
client.mutation("cursors:update", {
    "id": user_id,
    "x": normalized_x,
    "y": normalized_y,
    "drawing": is_drawing
})

# Subscribe to real-time updates
for cursors in client.subscribe("cursors:list"):
    # Update UI with new cursor data
    update_canvas(cursors)
```

## Textual Development Notes

**Critical patterns from `.claude/rules/TEXTUAL.md`:**

1. Always use `compose()` for compound widgets (built from other widgets)
2. Use `render()` for simple content (returns strings or Rich renderables)
3. Use `reactive()` for state that should trigger UI refresh
4. Use `@work` decorator for async operations to avoid blocking UI
5. Use `async with app.run_test()` for testing with Pilot
6. Run with `textual run --dev` for live CSS hot-reloading
7. Use `textual console` in separate terminal for debugging logs

**Common pitfall to avoid:**
- Do NOT query widgets immediately after `mount()` — use `await self.mount(widget)` and `await pilot.pause()` in tests

## Environment Variables

Store in `.env.local` (not committed):

```bash
CONVEX_URL=https://your-deployment.convex.cloud
```

Get the URL from Convex dashboard after running `npx convex dev`.

## Project Structure (Expected)

```
rtc-painting/
├── main.py                 # Entry point
├── convex/                 # Convex backend functions (TypeScript)
│   ├── schema.ts          # Database schema
│   ├── cursors.ts         # Cursor queries/mutations
│   └── _generated/        # Auto-generated (committed)
├── src/                    # Python source (to be created)
│   ├── app.py             # Textual App class
│   ├── canvas.py          # Canvas widget
│   └── client.py          # Convex client wrapper
├── styles/                 # Textual CSS (to be created)
│   └── canvas.tcss
├── tests/                  # Pytest tests
├── .env.local             # Environment variables (not committed)
├── pyproject.toml         # Python dependencies
└── convex.json            # Convex config (auto-generated)
```

## References

- **Textual docs:** https://textual.textualize.io
- **Convex docs:** https://docs.convex.dev
- **Convex Python client:** https://docs.convex.dev/client/python
- Detailed Textual patterns in `.claude/rules/TEXTUAL.md`
- Detailed Convex patterns in `.claude/rules/CONVEX.md`
