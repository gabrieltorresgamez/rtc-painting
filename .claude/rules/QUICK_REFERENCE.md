# Quick Reference

This file contains essential patterns for the rtc-painting project. For comprehensive framework documentation, use the skills:
- `/convex-guide` - Convex backend patterns and Python client usage
- `/textual-guide` - Textual TUI framework patterns and mouse handling

## Critical Patterns

### Textual (Terminal UI)

**Imports:**
```python
from textual.app import App, ComposeResult
from textual.widgets import Static
from textual.reactive import reactive
from textual import events, on, work
```

**Widget patterns:**
- Use `compose()` for compound widgets (built from other widgets)
- Use `render()` for simple content (returns strings or Rich renderables)
- Use `reactive()` for state that should trigger UI refresh
- Always `await self.mount()` before querying newly mounted widgets

**Mouse handling:**
- `on_click()` for user actions (preferred over MouseDown)
- `on_mouse_move()` to track cursor continuously
- `on_mouse_down()` + `capture_mouse()` for dragging
- `on_mouse_up()` + `release_mouse()` to end dragging
- Check `event.x, event.y` for widget-relative coordinates
- Check `event.screen_x, event.screen_y` for terminal-relative coordinates

**Async operations:**
```python
@work(exclusive=True)
async def fetch_data(self):
    # Non-blocking async operation
    ...
```

### Convex (Backend)

**Python client:**
```python
from convex import ConvexClient

client = ConvexClient(os.getenv("CONVEX_URL"))

# Query (read)
data = client.query("file:function", args)

# Mutate (write)
client.mutation("file:function", args)

# Subscribe (real-time)
for updates in client.subscribe("file:function", args):
    handle_updates(updates)
```

**Function naming:** `"filename:functionName"` maps to `convex/filename.ts` export

**Type mapping:**
- Python `dict` → Convex `object`
- Python `list` → Convex `Array`
- Python `str` (for IDs) → Convex `Id<"table">`
- Normalize coordinates to 0.0-1.0 range before storing

## Project-Specific Patterns

### Cursor Position Updates

**Throttle updates:**
- Update Convex every 30-60ms max
- Use `time.monotonic()` to track last update time
- Only send if enough time has elapsed

**Normalize coordinates:**
```python
# Store in Convex (0.0 to 1.0)
normalized_x = event.x / widget.size.width
normalized_y = event.y / widget.size.height

# Restore for rendering
pixel_x = int(normalized_x * widget.size.width)
pixel_y = int(normalized_y * widget.size.height)
```

### Drawing vs Presence

**Key distinction:**
- **Presence (always):** Cursor bubbles visible for all users at all times
- **Drawing (conditional):** Line segments only created while `drawing=True` (mouse button held)

**State tracking:**
```python
drawing = False  # Local state

def on_mouse_down(self, event: MouseDown) -> None:
    self.drawing = True
    self.capture_mouse()
    # Update Convex with drawing=True

def on_mouse_move(self, event: MouseMove) -> None:
    # Always update cursor position
    if self.drawing:
        # Also create line segment locally

def on_mouse_up(self, event: MouseUp) -> None:
    self.drawing = False
    self.release_mouse()
    # Update Convex with drawing=False
```

### Line Segment Fading

**Implement linear fade over 60 seconds:**
```python
age = current_time - segment.timestamp
alpha = max(0, 1 - age / 60.0)

if age > 60:
    # Remove from render queue
    segments.remove(segment)
```

## Development Workflow

**Run Convex backend:**
```bash
npx convex dev  # Watch mode with hot reload
```

**Run Textual app:**
```bash
uv run python main.py
```

**Debug Textual:**
```bash
# Terminal 1:
textual console

# Terminal 2:
textual run --dev main.py
```

## Common Gotchas

1. **Never mutate reactive collections in-place without notification:**
   ```python
   # WRONG
   self.items.append("new")

   # CORRECT
   self.items = self.items + ["new"]
   # OR
   self.items.append("new")
   self.mutate_reactive(type(self).items)
   ```

2. **Always release mouse capture:**
   ```python
   def on_mouse_down(self, event):
       self.capture_mouse()  # Capture

   def on_mouse_up(self, event):
       self.release_mouse()  # Always release!
   ```

3. **Await mount before querying:**
   ```python
   # WRONG
   self.mount(Widget())
   self.query_one(Widget)  # NoMatches error!

   # CORRECT
   await self.mount(Widget())
   self.query_one(Widget)  # Works!
   ```

4. **Convex queries return fresh data, mutations trigger updates:**
   - Queries are subscribable, mutations are transactional
   - Never mix: don't call actions from mutations
   - Use scheduled actions if you need both

## Anonymous User Identity

**Generate and store locally:**
```python
import uuid
import random

user_id = str(uuid.uuid4())
user_color = f"hsl({random.randint(0, 360)}, 70%, 60%)"

# Store in Textual app state, not in Convex
# Each user has unique color for cursor bubble
```
