# rtc-painting

Real-time collaborative ephemeral drawing canvas. Draw trails with your mouse, see others' cursors as colored bubbles, everything fades in 60 seconds.

## Tech Stack

- **Frontend:** Python Textual (terminal UI)
- **Backend:** Convex (real-time database)
- **Package Manager:** uv
- **Python:** 3.14

## Quick Start

```bash
# Install dependencies
uv sync

# Start Convex backend
npx convex dev

# Run application
uv run python main.py
```

## Features

- Anonymous collaborative drawing
- Real-time cursor presence
- Ephemeral trails (60s fade)
- Terminal-based UI
- No auth, no storage, no persistence
