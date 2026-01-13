# Textual TUI Framework Guide

## When to use this skill

Invoke this skill when working on:
- Creating or modifying Textual widgets (custom widgets, Canvas, Static, etc.)
- Implementing mouse event handling (clicks, hover, drag, drawing interactions)
- Working with reactive state and watchers (`reactive()`, `watch_*`, `compute_*`)
- Styling with CSS (tcss files, selectors, themes, layout)
- Implementing screens, layouts, or navigation patterns
- Using the `@work` decorator for async operations
- Debugging Textual rendering, event flow, or performance issues
- Testing Textual apps with Pilot
- Questions about Textual architecture, message passing, or best practices

---

# Textual TUI Framework Guidelines

## Quick reference

Textual is a Rapid Application Development framework for Python terminal user interfaces (TUIs). It follows web-inspired patterns with a DOM-like widget tree, reactive attributes, CSS styling, and message-based communication.

**ALWAYS use this import pattern for Textual apps:**

```python
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Button, Static
from textual.containers import Container, Horizontal, Vertical
from textual.reactive import reactive, var
from textual.message import Message
from textual import on, work, events
```

## App guidelines

### App structure

ALWAYS structure your main App class with these key components:

```python
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Static

class MyApp(App):
    CSS_PATH = "styles.tcss"  # External CSS file (recommended)
    TITLE = "My Application"
    BINDINGS = [("q", "quit", "Quit"), ("d", "toggle_dark", "Dark mode")]
    
    def compose(self) -> ComposeResult:
        yield Header()
        yield Static("Hello, World!")
        yield Footer()
    
    def on_mount(self) -> None:
        """Called when app is mounted and ready."""
        pass

if __name__ == "__main__":
    app = MyApp()
    app.run()
```

### App lifecycle events

The app lifecycle follows this order:
1. **Instantiation** - App object created
2. **Load event** - App running but terminal not yet in application mode
3. **Mount event** - App fully mounted, `on_mount()` called
4. **Ready event** - App ready to receive input
5. **Event loop** - Processes messages until exit
6. **Exit** - `app.exit()` called

### Screen management

Use screens for multi-page applications:

```python
from textual.screen import Screen, ModalScreen

class SettingsScreen(Screen):
    BINDINGS = [("escape", "app.pop_screen", "Back")]
    
    def compose(self) -> ComposeResult:
        yield Static("Settings")
        yield Button("Back", id="back")

class MyApp(App):
    SCREENS = {"settings": SettingsScreen}
    
    def action_open_settings(self) -> None:
        self.push_screen("settings")
```

**Screen stack operations:**
- `push_screen(screen)` - Add screen to top of stack
- `pop_screen()` - Remove top screen
- `switch_screen(screen)` - Replace top screen

### Modal dialogs

Use `ModalScreen` for dialogs that block interaction with underlying screens:

```python
from textual.screen import ModalScreen

class ConfirmDialog(ModalScreen[bool]):
    def compose(self) -> ComposeResult:
        yield Container(
            Static("Are you sure?"),
            Horizontal(
                Button("Yes", id="yes", variant="primary"),
                Button("No", id="no"),
            ),
            id="dialog",
        )
    
    def on_button_pressed(self, event: Button.Pressed) -> None:
        self.dismiss(event.button.id == "yes")

# Usage with callback
self.push_screen(ConfirmDialog(), callback=self.handle_confirm)

# Or await result
result = await self.push_screen_wait(ConfirmDialog())
```

## Widget guidelines

### Creating custom widgets

ALWAYS follow this pattern for custom widgets:

```python
from textual.widget import Widget
from textual.reactive import reactive
from textual.app import ComposeResult

class Counter(Widget):
    """A counter widget with best practices."""
    
    DEFAULT_CSS = """
    Counter {
        width: 20;
        height: 3;
        content-align: center middle;
        border: solid $primary;
    }
    """
    
    BINDINGS = [("up", "increment", "Increment"), ("down", "decrement", "Decrement")]
    
    count = reactive(0)
    
    def render(self) -> str:
        return f"Count: {self.count}"
    
    def watch_count(self, count: int) -> None:
        self.log(f"Count changed to {count}")
    
    def action_increment(self) -> None:
        self.count += 1
    
    def action_decrement(self) -> None:
        self.count -= 1
```

### Compose vs Render

- Use `compose()` for compound widgets built from other widgets
- Use `render()` for simple content that returns Rich renderables

```python
# compose() - for compound widgets
def compose(self) -> ComposeResult:
    yield Label(self.title)
    yield Input(placeholder="Enter text")
    yield Button("Submit")

# render() - for simple content
def render(self) -> str:
    return f"Hello, {self.name}!"
```

### Widget lifecycle

1. **compose()** - Declare child widgets (yields widget instances)
2. **on_mount()** - Widget added to DOM, safe to query children
3. **render()** - Return content for display
4. **on_unmount()** - Widget removed, cleanup

### Built-in widgets reference

**Input widgets:** `Button`, `Input`, `Checkbox`, `Switch`, `RadioButton`, `RadioSet`, `Select`, `SelectionList`

**Display widgets:** `Static`, `Label`, `Digits`, `Pretty`, `Markdown`, `MarkdownViewer`, `RichLog`, `Log`

**Data widgets:** `DataTable`, `Tree`, `DirectoryTree`, `ListView`, `OptionList`

**Layout widgets:** `Container`, `Horizontal`, `Vertical`, `VerticalScroll`, `HorizontalScroll`, `Grid`, `TabbedContent`, `Collapsible`

**System widgets:** `Header`, `Footer`, `LoadingIndicator`, `ProgressBar`, `Toast`

## Reactive guidelines

### Reactive attributes

Use `reactive()` for state that affects the UI:

```python
from textual.reactive import reactive, var

class MyWidget(Widget):
    # Triggers refresh on change
    count = reactive(0)
    
    # Dynamic default using callable
    start_time = reactive(time.monotonic)
    
    # No auto-refresh (for internal state)
    internal_state = var("")
```

**Reactive options:**
- `repaint=True` - Refresh widget on change (default)
- `layout=True` - Trigger layout recalculation
- `init=True` - Call watchers on initialization
- `always_update=True` - Call watchers even when value unchanged
- `recompose=True` - Remove children and call `compose()` again

### Watch methods

ALWAYS name watch methods as `watch_<attribute_name>`:

```python
class MyWidget(Widget):
    color = reactive("red")
    
    # Single argument (new value only)
    def watch_color(self, color: str) -> None:
        self.styles.background = color
    
    # Two arguments (old and new values)
    def watch_color(self, old_color: str, new_color: str) -> None:
        self.log(f"Color changed from {old_color} to {new_color}")
```

### Validate methods

Use `validate_<attribute_name>` to constrain values:

```python
class MyWidget(Widget):
    count = reactive(0)
    
    def validate_count(self, count: int) -> int:
        """Clamp count between 0 and 100."""
        return max(0, min(100, count))
```

### Compute methods

Use `compute_<attribute_name>` for derived state:

```python
class ColorMixer(Widget):
    red = reactive(0)
    green = reactive(0)
    blue = reactive(0)
    color = reactive(Color.parse("black"))
    
    def compute_color(self) -> Color:
        """Auto-computed when red, green, or blue changes."""
        return Color(self.red, self.green, self.blue).clamped
    
    def watch_color(self, color: Color) -> None:
        """Required for compute to take effect."""
        self.styles.background = color
```

**Execution order:** `compute_*` → `validate_*` → `watch_*`

### Mutable reactives

Do NOT mutate reactive collections directly without notification:

```python
# WRONG - watch method NOT called
self.items.append("new")

# CORRECT - reassign
self.items = self.items + ["new"]

# CORRECT - use mutate_reactive
self.items.append("new")
self.mutate_reactive(MyWidget.items)
```

### Avoiding watch in constructors

Use `set_reactive()` in `__init__` to avoid triggering watchers before mount:

```python
def __init__(self, greeting: str = "Hello"):
    super().__init__()
    # WRONG - triggers watch before widget is mounted
    # self.greeting = greeting
    
    # CORRECT - bypasses watchers
    self.set_reactive(Greeter.greeting, greeting)
```

### Data binding

Bind parent attributes to child widgets:

```python
class ParentApp(App):
    current_time = reactive(datetime.now)
    
    def compose(self) -> ComposeResult:
        yield Clock().data_bind(ParentApp.current_time)
        # With different attribute name
        yield Clock().data_bind(clock_time=ParentApp.current_time)
```

## Message guidelines

### Attributes down, messages up

ALWAYS follow this unidirectional data flow pattern:
- **Attributes down:** Parents update children by setting attributes
- **Messages up:** Children notify parents via messages that bubble up

```python
# Parent updating child (attributes down)
def action_reset(self) -> None:
    self.query_one(Counter).count = 0

# Child notifying parent (messages up)
class Counter(Widget):
    class Changed(Message):
        def __init__(self, value: int) -> None:
            self.value = value
            super().__init__()
    
    def watch_count(self, count: int) -> None:
        self.post_message(self.Changed(count))
```

### Custom messages

Define messages as nested classes inside widgets:

```python
from textual.message import Message

class ColorPicker(Widget):
    class ColorSelected(Message):
        def __init__(self, color: str) -> None:
            self.color = color
            super().__init__()
    
    def on_button_pressed(self, event: Button.Pressed) -> None:
        self.post_message(self.ColorSelected(event.button.id))

# Parent handles with auto-named handler
class MyApp(App):
    def on_color_picker_color_selected(self, event: ColorPicker.ColorSelected) -> None:
        self.background = event.color
```

### The @on decorator

Use `@on` for targeted message handling:

```python
from textual import on

class MyApp(App):
    @on(Button.Pressed, "#submit")
    def handle_submit(self, event: Button.Pressed) -> None:
        """Only handles button with id='submit'."""
        self.submit_form()
    
    @on(Input.Changed, ".validated")
    def handle_validated_input(self, event: Input.Changed) -> None:
        """Only handles inputs with class 'validated'."""
        self.validate(event.value)
```

### Stopping message propagation

```python
def on_button_pressed(self, event: Button.Pressed) -> None:
    event.stop()  # Prevent bubbling to parent
```

### Preventing messages temporarily

```python
with self.prevent(Input.Changed):
    self.query_one(Input).value = "new value"  # Won't trigger Changed
```

## Mouse handling guidelines

### Mouse events overview

Textual sends events for mouse movements, clicks, and scroll actions. Mouse coordinates are given as `(x, y)` pairs where x is character offset and y is line offset.

**Key mouse events:**
- `MouseMove` - Cursor movement over widgets
- `MouseDown` / `MouseUp` - Button press/release
- `Click` - Complete click (after MouseUp)
- `Enter` / `Leave` - Cursor entering/leaving widget
- `MouseScrollDown` / `MouseScrollUp` - Scroll wheel
- `MouseScrollLeft` / `MouseScrollRight` - Horizontal scroll (if supported)

### Mouse coordinate systems

```python
from textual import events

def on_mouse_move(self, event: events.MouseMove) -> None:
    # Screen coordinates (relative to terminal)
    screen_x, screen_y = event.screen_x, event.screen_y
    screen_offset = event.screen_offset  # Offset object
    
    # Widget coordinates (relative to widget)
    widget_x, widget_y = event.x, event.y
    widget_offset = event.offset  # Offset object
    
    # Delta since last mouse event
    delta_x, delta_y = event.delta_x, event.delta_y
```

### Handling mouse movements

Use `on_mouse_move` to track cursor position:

```python
from textual import events

class MyWidget(Widget):
    cursor_position = reactive((0, 0))
    
    def on_mouse_move(self, event: events.MouseMove) -> None:
        # Update position relative to widget
        self.cursor_position = (event.x, event.y)
        
        # Check modifier keys
        if event.ctrl:
            self.log("Ctrl held during mouse move")
        if event.shift:
            self.log("Shift held during mouse move")
```

### Click handling

ALWAYS prefer `Click` event over `MouseDown`/`MouseUp`:

```python
from textual import events, on

class InteractiveWidget(Widget):
    # Method 1: Using on_click handler
    def on_click(self, event: events.Click) -> None:
        self.log(f"Clicked at {event.x}, {event.y}")
        
        # Check for double-click, triple-click, etc.
        if event.chain == 2:
            self.on_double_click(event)
    
    # Method 2: Using @on decorator with selector
    @on(Click, "#my-button")
    def handle_button_click(self, event: events.Click) -> None:
        self.notify("Button clicked!")
    
    def on_double_click(self, event: events.Click) -> None:
        self.log("Double-clicked!")
```

### Double-click detection

Use the `chain` attribute to detect multiple clicks:

```python
from textual.events import Click

class MyWidget(Widget):
    CLICK_CHAIN_TIME_THRESHOLD = 0.5  # Seconds between clicks (default 0.5)
    
    def on_click(self, event: Click) -> None:
        match event.chain:
            case 1:
                self.single_click(event)
            case 2:
                self.double_click(event)
            case 3:
                self.triple_click(event)
```

### Enter and Leave events

Detect when cursor enters/leaves a widget:

```python
from textual import events

class HoverWidget(Widget):
    is_hovered = reactive(False)
    
    def on_enter(self, event: events.Enter) -> None:
        """Cursor entered widget."""
        self.is_hovered = True
        
        # Check if this widget was the original target
        if event.node is self:
            self.add_class("hover")
    
    def on_leave(self, event: events.Leave) -> None:
        """Cursor left widget."""
        self.is_hovered = False
        
        # Check if this widget was the original target
        if event.node is self:
            self.remove_class("hover")
    
    def watch_is_hovered(self, is_hovered: bool) -> None:
        """React to hover state changes."""
        self.log(f"Hover state: {is_hovered}")
```

### Mouse capture

Capture all mouse events regardless of cursor position:

```python
from textual import events

class DraggableWidget(Widget):
    dragging = False
    drag_start = (0, 0)
    
    def on_mouse_down(self, event: events.MouseDown) -> None:
        """Start drag on mouse down."""
        self.dragging = True
        self.drag_start = event.screen_offset
        self.capture_mouse()  # Capture mouse events
    
    def on_mouse_move(self, event: events.MouseMove) -> None:
        """Handle drag movement."""
        if self.dragging:
            delta = event.screen_offset - self.drag_start
            self.offset = delta
    
    def on_mouse_up(self, event: events.MouseUp) -> None:
        """End drag on mouse up."""
        if self.dragging:
            self.dragging = False
            self.release_mouse()  # Release capture
    
    def on_mouse_capture(self, event: events.MouseCapture) -> None:
        """Called when mouse is captured."""
        self.log("Mouse captured")
    
    def on_mouse_release(self, event: events.MouseRelease) -> None:
        """Called when mouse is released."""
        self.log("Mouse released")
```

### Scroll events

Handle scroll wheel input:

```python
from textual import events

class ScrollableWidget(Widget):
    scroll_position = reactive(0)
    
    def on_mouse_scroll_down(self, event: events.MouseScrollDown) -> None:
        """Scroll down (wheel down)."""
        self.scroll_position += 1
        event.stop()  # Prevent parent from scrolling
    
    def on_mouse_scroll_up(self, event: events.MouseScrollUp) -> None:
        """Scroll up (wheel up)."""
        self.scroll_position = max(0, self.scroll_position - 1)
        event.stop()
    
    # Horizontal scrolling (if terminal supports it)
    def on_mouse_scroll_left(self, event: events.MouseScrollLeft) -> None:
        self.horizontal_scroll(-1)
    
    def on_mouse_scroll_right(self, event: events.MouseScrollRight) -> None:
        self.horizontal_scroll(1)
```

### Mouse event properties

All mouse events provide:

```python
def on_click(self, event: Click) -> None:
    # Position relative to screen
    event.screen_x, event.screen_y
    event.screen_offset  # Offset(x, y)
    
    # Position relative to widget
    event.x, event.y
    event.offset  # Offset(x, y)
    
    # Modifier keys
    event.ctrl    # Ctrl key held
    event.shift   # Shift key held
    event.meta    # Meta/Command key held
    event.alt     # Alt key held
    
    # Button information (for MouseDown/MouseUp)
    event.button  # 0=left, 1=middle, 2=right
    
    # Mouse delta (for MouseMove)
    event.delta_x, event.delta_y
    
    # Style under cursor
    event.style  # Rich Style object
    
    # Widget at click position
    event.widget  # Widget instance
```

### Hover effects with CSS

Use `:hover` pseudo-class for visual feedback:

```css
Button {
    background: $surface;
}

Button:hover {
    background: $primary-lighten-1;
}

.item:hover {
    outline: solid $accent;
}
```

### Combining mouse and keyboard

```python
from textual import events

class InteractiveWidget(Widget):
    def on_click(self, event: events.Click) -> None:
        if event.ctrl:
            self.ctrl_click(event)
        elif event.shift:
            self.shift_click(event)
        elif event.alt:
            self.alt_click(event)
        else:
            self.normal_click(event)
    
    def on_mouse_down(self, event: events.MouseDown) -> None:
        # Check which button
        if event.button == 0:  # Left
            self.left_button_down(event)
        elif event.button == 1:  # Middle
            self.middle_button_down(event)
        elif event.button == 2:  # Right
            self.right_button_down(event)
```

### Preventing mouse event bubbling

```python
def on_click(self, event: Click) -> None:
    # Handle click
    self.toggle_selected()
    
    # Stop event from bubbling to parent
    event.stop()
```

### Common mouse patterns

**Pattern 1: Clickable items in list**
```python
class ListItem(Static):
    class Selected(Message):
        def __init__(self, item: "ListItem") -> None:
            self.item = item
            super().__init__()
    
    def on_click(self) -> None:
        self.post_message(self.Selected(self))

class ItemList(Widget):
    def on_list_item_selected(self, event: ListItem.Selected) -> None:
        self.notify(f"Selected: {event.item.id}")
```

**Pattern 2: Drag and drop**
```python
class DragDrop(Widget):
    dragging = None
    
    def on_mouse_down(self, event: MouseDown) -> None:
        widget = self.get_widget_at(*event.screen_offset)
        if isinstance(widget, DraggableItem):
            self.dragging = widget
            self.capture_mouse()
    
    def on_mouse_move(self, event: MouseMove) -> None:
        if self.dragging:
            self.dragging.offset = event.screen_offset
    
    def on_mouse_up(self, event: MouseUp) -> None:
        if self.dragging:
            drop_target = self.get_widget_at(*event.screen_offset)
            self.handle_drop(self.dragging, drop_target)
            self.dragging = None
            self.release_mouse()
```

**Pattern 3: Drawing/painting**
```python
class Canvas(Widget):
    drawing = False
    
    def on_mouse_down(self, event: MouseDown) -> None:
        self.drawing = True
        self.draw_pixel(event.x, event.y)
        self.capture_mouse()
    
    def on_mouse_move(self, event: MouseMove) -> None:
        if self.drawing:
            self.draw_pixel(event.x, event.y)
    
    def on_mouse_up(self, event: MouseUp) -> None:
        self.drawing = False
        self.release_mouse()
```

### Mouse handling pitfalls

**Pitfall 1: Forgetting to release mouse capture**
```python
# WRONG - mouse stays captured forever
def on_mouse_down(self, event: MouseDown) -> None:
    self.capture_mouse()

# CORRECT - always release
def on_mouse_up(self, event: MouseUp) -> None:
    self.release_mouse()
```

**Pitfall 2: Using MouseDown instead of Click**
```python
# WRONG - doesn't work well with all pointing devices
def on_mouse_down(self, event: MouseDown) -> None:
    self.activate()

# CORRECT - use Click for actions
def on_click(self, event: Click) -> None:
    self.activate()
```

**Pitfall 3: Negative coordinates with mouse capture**
```python
# WRONG - doesn't handle negative coords
def on_mouse_move(self, event: MouseMove) -> None:
    self.offset = (event.x, event.y)  # Can be negative!

# CORRECT - clamp or check coordinates
def on_mouse_move(self, event: MouseMove) -> None:
    x = max(0, min(event.x, self.size.width - 1))
    y = max(0, min(event.y, self.size.height - 1))
    self.offset = (x, y)
```

**Pitfall 4: Not checking event.node for Enter/Leave**
```python
# WRONG - triggers for child widgets too
def on_enter(self, event: Enter) -> None:
    self.add_class("hover")  # Triggers even for children!

# CORRECT - check if this widget is the target
def on_enter(self, event: Enter) -> None:
    if event.node is self:
        self.add_class("hover")
```

## CSS guidelines

### CSS loading

**External CSS files (recommended):**
```python
class MyApp(App):
    CSS_PATH = "styles.tcss"
    # Or multiple files
    CSS_PATH = ["base.tcss", "widgets.tcss"]
```

**Inline CSS:**
```python
class MyApp(App):
    CSS = """
    Screen {
        background: $primary;
    }
    """
```

**Widget bundled CSS:**
```python
class MyWidget(Widget):
    DEFAULT_CSS = """
    MyWidget {
        height: auto;
        border: solid $primary;
    }
    """
```

**Specificity:** App CSS > External CSS > DEFAULT_CSS

### Selectors

```css
/* Type selector - matches widget class name */
Button { width: 16; }

/* ID selector - matches widget id attribute */
#submit { outline: red; }

/* Class selector - matches CSS classes */
.error { background: red; }
.error.disabled { background: darkred; }

/* Universal selector */
* { outline: solid red; }

/* Descendant combinator */
#sidebar Button { text-style: bold; }

/* Child combinator */
#form > Input { margin: 1; }
```

### Pseudo-classes

```css
Button:hover { background: $primary-lighten-1; }
Button:focus { outline: double $accent; }
Button:disabled { opacity: 0.5; }
MyWidget:dark Label { color: white; }
MyWidget:light Label { color: black; }
ListItem:even { background: $surface; }
```

### Key CSS properties

**Dimensions:**
```css
Widget {
    width: 50;           /* Cells */
    width: 50%;          /* Percentage */
    width: 1fr;          /* Fraction units */
    width: auto;         /* Fit content */
    height: 10vh;        /* Viewport height */
    min-width: 20;
    max-width: 100;
}
```

**Layout:**
```css
Container {
    layout: horizontal;  /* or vertical, grid */
    align: center middle;
    content-align: center middle;
}

#header {
    dock: top;
    height: 3;
}
```

**Grid layout:**
```css
Screen {
    layout: grid;
    grid-size: 3 2;          /* columns rows */
    grid-columns: 2fr 1fr 1fr;
    grid-rows: 1fr 2fr;
    grid-gutter: 1 2;
}
#wide-cell {
    column-span: 2;
}
```

**Colors and appearance:**
```css
Widget {
    background: $surface;
    color: $text;
    border: solid $primary;
    outline: double $accent;
    opacity: 0.8;
}
```

**Text:**
```css
Label {
    text-style: bold italic;
    text-align: center;
    text-overflow: ellipsis;
}
```

### Theme variables

Use theme variables for consistent styling:

| Variable | Purpose |
|----------|---------|
| `$primary` | Branding, titles, emphasis |
| `$secondary` | Alternative branding |
| `$background` | Screen background |
| `$surface` | Widget backgrounds |
| `$panel` | UI panels |
| `$success`, `$warning`, `$error` | Semantic colors |
| `$accent` | Attention-drawing elements |
| `$text`, `$text-muted` | Text colors |

**Auto-generated shades:**
```css
$primary-lighten-1, $primary-lighten-2, $primary-lighten-3
$primary-darken-1, $primary-darken-2, $primary-darken-3
$primary-muted  /* Blended with background */
```

### CSS dynamic classes

Toggle CSS classes for state changes:

```python
def on_click(self) -> None:
    self.add_class("active")
    self.remove_class("inactive")
    self.toggle_class("selected")
```

```css
.item { background: $surface; }
.item.selected { background: $primary; }
.item.active { border: solid $accent; }
```

## Worker guidelines

### Background tasks

Use `@work` decorator for non-blocking operations:

```python
from textual import work

class MyApp(App):
    @work
    async def fetch_data(self, url: str) -> None:
        """Async worker - doesn't block UI."""
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            self.display_data(response.json())
    
    @work(thread=True)
    def heavy_computation(self) -> None:
        """Thread worker - for CPU-bound tasks."""
        result = process_large_dataset()
        self.call_from_thread(self.update_ui, result)
```

### Exclusive workers

Cancel previous worker when starting new one:

```python
@work(exclusive=True)
async def search(self, query: str) -> None:
    """Only one search runs at a time."""
    results = await api.search(query)
    self.update_results(results)
```

### Thread safety

NEVER update UI directly from thread workers:

```python
# WRONG - not thread-safe
@work(thread=True)
def bad_worker(self) -> None:
    self.query_one(Label).update("Done")  # CRASHES!

# CORRECT - use call_from_thread
@work(thread=True)
def good_worker(self) -> None:
    result = heavy_computation()
    self.call_from_thread(self.update_label, result)

def update_label(self, text: str) -> None:
    self.query_one(Label).update(text)
```

### Worker exception handling

Handle exceptions inside workers:

```python
@work(exclusive=True)
async def fetch_url(self, url: str) -> None:
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            self.update(response.text)
    except httpx.HTTPError as e:
        self.notify(f"Error: {e}", severity="error")
```

Or handle outside with `exit_on_error=False`:

```python
@work(exit_on_error=False)
async def risky_operation(self) -> None:
    ...

def on_worker_state_changed(self, event: Worker.StateChanged) -> None:
    if event.worker.state == WorkerState.ERROR:
        self.notify("Operation failed", severity="error")
```

## Testing guidelines

### Pilot testing

Use `run_test()` with Pilot for headless testing:

```python
import pytest
from textual.color import Color

async def test_button_changes_color():
    app = MyApp()
    async with app.run_test() as pilot:
        await pilot.click("#red-button")
        assert app.screen.styles.background == Color.parse("red")

async def test_keyboard_input():
    app = MyApp()
    async with app.run_test() as pilot:
        await pilot.press("tab", "enter")
        assert app.query_one(Input).value == "submitted"
```

### Pilot methods

- `pilot.press("key")` - Simulate key press
- `pilot.click("#selector")` - Click widget by selector
- `pilot.click(offset=(x, y))` - Click at coordinates
- `pilot.hover("#widget")` - Hover over widget
- `pilot.pause()` - Wait for message processing

### Snapshot testing

```bash
pip install pytest-textual-snapshot
```

```python
def test_app_appearance(snap_compare):
    assert snap_compare("path/to/app.py")

def test_after_input(snap_compare):
    assert snap_compare("path/to/app.py", press=["1", "2", "3"])

def test_custom_size(snap_compare):
    assert snap_compare("path/to/app.py", terminal_size=(80, 24))
```

First run always fails (creates baseline). Use `pytest --snapshot-update` after verification.

### Testing async properly

ALWAYS await mount before querying:

```python
async def test_dynamic_widget():
    app = MyApp()
    async with app.run_test() as pilot:
        await pilot.press("a")  # Triggers mount
        await pilot.pause()     # Wait for processing
        # Now safe to query
        assert app.query_one("#dynamic-widget").value == "expected"
```

## Debugging guidelines

### Textual console

Run in two terminals:

```bash
# Terminal 1: Start console
textual console

# Terminal 2: Run app with dev mode
textual run --dev my_app.py
```

### Logging

```python
from textual import log

class MyApp(App):
    def on_mount(self) -> None:
        log("Simple message")
        log(locals())  # Log variables
        log(key="value", count=42)  # Named values
        self.log("Using widget log method")
```

### Live CSS editing

Run with `--dev` to hot-reload CSS:

```bash
textual run --dev my_app.py
```

Changes to `.tcss` files apply immediately.

## Common pitfalls

### Mounting timing

Do NOT query widgets immediately after `mount()`:

```python
# WRONG - widget not yet in DOM
def on_key(self) -> None:
    self.mount(MyWidget())
    self.query_one(MyWidget).value = "test"  # NoMatches error!

# CORRECT - await the mount
async def on_key(self) -> None:
    await self.mount(MyWidget())
    self.query_one(MyWidget).value = "test"  # Works!
```

### Reactive mutations

Do NOT expect watch to trigger on in-place mutations:

```python
# WRONG - watch NOT called
self.items.append("new")

# CORRECT - reassign or mutate_reactive
self.items = [*self.items, "new"]
# OR
self.items.append("new")
self.mutate_reactive(type(self).items)
```

### Watch before mount

Do NOT rely on query in watch methods during `__init__`:

```python
# WRONG - query fails during construction
def __init__(self):
    super().__init__()
    self.value = "test"  # Triggers watch, but not mounted!

def watch_value(self, value):
    self.query_one(Label).update(value)  # FAILS!

# CORRECT - use set_reactive in __init__
def __init__(self):
    super().__init__()
    self.set_reactive(MyWidget.value, "test")
```

### Blocking handlers

Do NOT run slow operations in message handlers:

```python
# WRONG - blocks entire UI
async def on_button_pressed(self) -> None:
    data = await slow_api_call()  # UI frozen!

# CORRECT - use workers
@work(exclusive=True)
async def fetch_data(self) -> None:
    data = await slow_api_call()
    self.update(data)

def on_button_pressed(self) -> None:
    self.fetch_data()  # Non-blocking
```

### Compute without watch

`compute_*` methods require a corresponding `watch_*` to execute:

```python
# WRONG - compute_color never runs
color = reactive(Color.parse("black"))

def compute_color(self) -> Color:
    return Color(self.r, self.g, self.b)

# CORRECT - add watch method
def watch_color(self, color: Color) -> None:
    self.styles.background = color
```

### Large DataTable performance

Standard DataTable is slow with large datasets (>10k rows):

```python
# For large datasets, use textual-fastdatatable
from textual_fastdatatable import ArrowBackend, DataTable

backend = ArrowBackend.from_parquet("large_data.parquet")
yield DataTable(backend=backend)
```

## Project structure

### Recommended layout

```
my_app/
├── __init__.py
├── app.py              # Main App class
├── screens/
│   ├── __init__.py
│   ├── main.py
│   └── settings.py
├── widgets/
│   ├── __init__.py
│   └── custom.py
├── styles/
│   ├── base.tcss
│   └── widgets.tcss
└── tests/
    ├── test_app.py
    └── snapshots/
```

### Widget distribution

Bundle CSS with widgets using `DEFAULT_CSS`:

```python
class DistributableWidget(Widget):
    DEFAULT_CSS = """
    DistributableWidget {
        height: auto;
        padding: 1;
        border: solid $primary;
    }
    """
    SCOPED_CSS = True  # CSS only affects this widget
```

# Examples

## Example: Todo app

### Task
Create a simple todo application with add, complete, and delete functionality.

### Implementation

**app.py:**
```python
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Input, Button, Static
from textual.containers import Horizontal, VerticalScroll
from textual.reactive import reactive
from textual.message import Message
from textual import on

class TodoItem(Static):
    class Deleted(Message):
        def __init__(self, item: "TodoItem") -> None:
            self.item = item
            super().__init__()
    
    completed = reactive(False)
    
    def __init__(self, text: str) -> None:
        super().__init__()
        self.text = text
    
    def compose(self) -> ComposeResult:
        yield Static(self.text, id="text")
        yield Button("✓", id="complete", variant="success")
        yield Button("✕", id="delete", variant="error")
    
    def watch_completed(self, completed: bool) -> None:
        self.set_class(completed, "completed")
    
    @on(Button.Pressed, "#complete")
    def toggle_complete(self) -> None:
        self.completed = not self.completed
    
    @on(Button.Pressed, "#delete")
    def delete_item(self) -> None:
        self.post_message(self.Deleted(self))

class TodoApp(App):
    CSS_PATH = "todo.tcss"
    BINDINGS = [("q", "quit", "Quit")]
    
    def compose(self) -> ComposeResult:
        yield Header()
        yield Horizontal(
            Input(placeholder="What needs to be done?", id="new-todo"),
            Button("Add", id="add", variant="primary"),
            id="input-row",
        )
        yield VerticalScroll(id="todo-list")
        yield Footer()
    
    @on(Button.Pressed, "#add")
    @on(Input.Submitted, "#new-todo")
    async def add_todo(self) -> None:
        input_widget = self.query_one("#new-todo", Input)
        if input_widget.value.strip():
            await self.query_one("#todo-list").mount(TodoItem(input_widget.value))
            input_widget.value = ""
            input_widget.focus()
    
    def on_todo_item_deleted(self, event: TodoItem.Deleted) -> None:
        event.item.remove()

if __name__ == "__main__":
    TodoApp().run()
```

**todo.tcss:**
```css
#input-row {
    height: auto;
    padding: 1;
    dock: top;
}

#new-todo {
    width: 1fr;
}

#add {
    width: auto;
    margin-left: 1;
}

#todo-list {
    padding: 1;
}

TodoItem {
    layout: horizontal;
    height: auto;
    padding: 1;
    margin: 0 0 1 0;
    background: $surface;
}

TodoItem #text {
    width: 1fr;
    content-align: left middle;
}

TodoItem Button {
    width: auto;
    min-width: 5;
    margin-left: 1;
}

TodoItem.completed #text {
    text-style: strike;
    color: $text-muted;
}
```

## Example: API data viewer

### Task
Create an app that fetches and displays data from an API with loading states and error handling.

### Implementation

```python
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Static, Input, DataTable
from textual.containers import Container
from textual import on, work
import httpx

class DataViewer(App):
    CSS = """
    #search { dock: top; padding: 1; }
    #results { padding: 1; }
    #error { color: $error; padding: 1; display: none; }
    #error.visible { display: block; }
    """
    
    BINDINGS = [("q", "quit", "Quit")]
    
    def compose(self) -> ComposeResult:
        yield Header()
        yield Input(placeholder="Enter search query...", id="search")
        yield Static(id="error")
        yield Container(DataTable(id="results"), id="table-container")
        yield Footer()
    
    def on_mount(self) -> None:
        table = self.query_one(DataTable)
        table.add_columns("ID", "Title", "Status")
    
    @on(Input.Submitted, "#search")
    def handle_search(self, event: Input.Submitted) -> None:
        if event.value.strip():
            self.search_api(event.value)
    
    @work(exclusive=True)
    async def search_api(self, query: str) -> None:
        table = self.query_one(DataTable)
        error = self.query_one("#error")
        
        table.loading = True
        error.remove_class("visible")
        
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"https://api.example.com/search?q={query}",
                    timeout=10.0
                )
                response.raise_for_status()
                data = response.json()
                
            table.clear()
            for item in data["results"]:
                table.add_row(item["id"], item["title"], item["status"])
                
        except httpx.HTTPError as e:
            error.update(f"Error: {e}")
            error.add_class("visible")
        finally:
            table.loading = False

if __name__ == "__main__":
    DataViewer().run()
```

## Example: Interactive drawing canvas

### Task
Create a simple drawing canvas that responds to mouse clicks and drags.

### Implementation

```python
from textual.app import App, ComposeResult
from textual.widgets import Static
from textual import events
from textual.reactive import reactive

class Canvas(Static):
    """A simple drawing canvas."""
    
    DEFAULT_CSS = """
    Canvas {
        width: 100%;
        height: 100%;
        background: $surface;
        border: solid $primary;
    }
    """
    
    pixels = reactive(set())
    drawing = False
    
    def __init__(self) -> None:
        super().__init__()
        self.pixels = set()
    
    def render(self) -> str:
        """Render the canvas with drawn pixels."""
        if not self.size:
            return ""
        
        lines = []
        for y in range(self.size.height):
            line = ""
            for x in range(self.size.width):
                if (x, y) in self.pixels:
                    line += "█"
                else:
                    line += " "
            lines.append(line)
        return "\n".join(lines)
    
    def on_mouse_down(self, event: events.MouseDown) -> None:
        """Start drawing on mouse down."""
        self.drawing = True
        self.draw_pixel(event.x, event.y)
        self.capture_mouse()
    
    def on_mouse_move(self, event: events.MouseMove) -> None:
        """Continue drawing on mouse move if button held."""
        if self.drawing:
            self.draw_pixel(event.x, event.y)
    
    def on_mouse_up(self, event: events.MouseUp) -> None:
        """Stop drawing on mouse up."""
        self.drawing = False
        self.release_mouse()
    
    def draw_pixel(self, x: int, y: int) -> None:
        """Draw a pixel at the given coordinates."""
        if 0 <= x < self.size.width and 0 <= y < self.size.height:
            self.pixels.add((x, y))
            self.refresh()

class DrawingApp(App):
    """Simple drawing application."""
    
    CSS = """
    Screen {
        align: center middle;
    }
    
    Canvas {
        width: 80;
        height: 24;
    }
    """
    
    BINDINGS = [
        ("q", "quit", "Quit"),
        ("c", "clear", "Clear"),
    ]
    
    def compose(self) -> ComposeResult:
        yield Canvas()
    
    def action_clear(self) -> None:
        """Clear the canvas."""
        canvas = self.query_one(Canvas)
        canvas.pixels = set()
        canvas.refresh()

if __name__ == "__main__":
    DrawingApp().run()
```

## Key patterns summary

| Pattern | When to Use |
|---------|-------------|
| `compose()` | Building widget tree declaratively |
| `reactive()` | State that affects UI display |
| `watch_*()` | Respond to reactive changes |
| `@on(Event, selector)` | Handle specific widget events |
| `@work` | Non-blocking async/background tasks |
| `post_message()` | Child-to-parent communication |
| `query_one()/query()` | Find widgets in DOM |
| `push_screen()` | Multi-screen navigation |
| `ModalScreen` | Dialogs and overlays |
| `CSS classes` | Dynamic styling based on state |
| `on_click()` | Handle mouse clicks |
| `on_mouse_move()` | Track cursor movement |
| `capture_mouse()` | Capture all mouse events |
| `on_enter()/on_leave()` | Detect hover state |

## Decision framework

**Choosing reactive vs var:**
- Use `reactive()` when changes should refresh the widget
- Use `var()` for internal state that doesn't affect display

**Choosing compose vs render:**
- Use `compose()` for widgets containing other widgets
- Use `render()` for simple text/Rich content

**Choosing async worker vs thread worker:**
- Use `@work` (async) for I/O-bound operations (API calls, file reads)
- Use `@work(thread=True)` for CPU-bound operations

**Choosing Screen vs ModalScreen:**
- Use `Screen` for full page navigation
- Use `ModalScreen` for dialogs that overlay current content

**Choosing inline CSS vs external CSS:**
- Use external `.tcss` files for app styling (enables live reload)
- Use `DEFAULT_CSS` for distributable widgets
- Use inline `CSS` class variable for simple apps

**Choosing Click vs MouseDown/MouseUp:**
- Use `Click` for actions (preferred for all pointing devices)
- Use `MouseDown`/`MouseUp` only when you need to distinguish press/release

**Choosing mouse capture:**
- Use `capture_mouse()` for drag operations or when mouse might leave widget
- Release with `release_mouse()` when operation completes