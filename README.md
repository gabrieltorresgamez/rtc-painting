# rtc-painting

## Concept

Shared full-screen canvas where users can draw ephemeral trails while clicking, and see all other users’ cursors with colored bubbles in real time.
Anonymous, ephemeral, collaborative, playable in terminal or browser.

## Tech Stack

* **Frontend / UI:** Python **Textual** (Canvas / Screen widgets, mouse events, terminal & web serving)
* **Backend / Realtime:** **Convex** (Realtime database for cursor positions and drawing state)

## 1. User Identity

* Anonymous persistent ID stored locally (Textual session / localStorage)
* Unique color per user (HSL / ANSI)
* No usernames or profiles

## 2. Canvas Behavior

* Cursor draws line segments **only while mouse button is pressed** (`MouseDown` → start drawing, `MouseUp` → stop drawing)
* Line segments fade after ~60 seconds (linear fade)
* Drawing happens only while the app is focused

## 3. Presence

* Each user has a **cursor bubble** (small colored circle / marker)
* Bubbles are **always visible**, regardless of whether the user is pressing the mouse
* Cursor positions update in real time via `on_mouse_move` (without drawing)
* Bubbles provide awareness of other users’ presence and movement

## 4. Networking (Convex)

* **Collection:** `cursors`
* **Fields per cursor:**

  * `id` — unique anonymous ID
  * `x`, `y` — normalized cursor position
  * `color` — user color
  * `drawing` — boolean indicating whether to render a trail
  * `timestamp` — time of update
* Clients push cursor positions (throttled 30–60ms)
* Convex broadcasts updates in real time to all connected clients

## 5. Rendering

* Use a full-area Textual widget (`Canvas` or `Screen`)
* Draw line segments locally **while `drawing=True`**
* Render all users’ cursor bubbles at their latest positions
* Fade line segments over time based on `timestamp`
* Interpolate positions for smooth movement

## 6. Constraints

* No erase, undo, or clear
* Line segments disappear automatically
* Minimal controls: move + click

## 7. Goal

A live, shared ephemeral canvas in terminal or browser, combining **click-to-draw trails** and **continuous presence awareness via cursor bubbles**, implemented with **Python Textual** for the frontend/UI and **Convex** for realtime backend.
