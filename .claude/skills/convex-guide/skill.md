# Convex Python Development Guide

## When to use this skill

Invoke this skill when working on:
- Implementing Convex backend functions (queries, mutations, actions in TypeScript)
- Setting up Convex Python client subscriptions and real-time sync
- Defining Convex schemas or database operations
- Troubleshooting Convex authentication, error handling, or WebSocket issues
- Working with files in the `convex/` directory
- Questions about Convex patterns, pagination, workers, or cron jobs
- Debugging Convex function calls or subscription behavior

---

Convex is a reactive serverless backend platform that combines a real-time database, serverless functions, and automatic sync into a unified system. The Python client enables calling Convex functions, subscribing to real-time updates, and building backend-connected Python applications. This guide provides actionable patterns for building robust Convex Python applications.

**What makes Convex different**: Unlike traditional backends that stitch together databases, APIs, and real-time layers, Convex was designed from first principles as an integrated system. Queries are TypeScript functions that run inside the database with automatic dependency tracking—when data changes, subscribed clients receive updates instantly via WebSocket without manual intervention.

## Quick reference

```python
# Installation
pip install convex python-dotenv

# Initialize client
from convex import ConvexClient
client = ConvexClient('https://your-deployment.convex.cloud')

# Basic operations
result = client.query("tasks:list")                    # Read data
client.mutation("tasks:create", {"text": "New task"})  # Write data
result = client.action("ai:generate", {"prompt": "..."})  # External APIs

# Real-time subscriptions
for tasks in client.subscribe("tasks:list"):
    print(tasks)  # Fires on every data change

# Authentication
client.set_auth("jwt-token-from-auth-provider")

# Error handling
from convex import ConvexError
try:
    client.mutation("orders:create", {...})
except ConvexError as err:
    print(err.data)  # Application-level error
```

### Convex function types at a glance

| Function Type | Purpose | Characteristics | Python Call |
|--------------|---------|-----------------|-------------|
| **Query** | Read data | Deterministic, subscribable, cached | `client.query()` |
| **Mutation** | Write data | ACID transactions, auto-retry on conflict | `client.mutation()` |
| **Action** | External APIs | Non-deterministic, can fetch external services | `client.action()` |

### Type mapping between Python and Convex

| Convex/JS Type | Python Type | Notes |
|----------------|-------------|-------|
| `null` | `None` | |
| `number` | `float` or `int` | Becomes Float64 in Convex |
| `bigint` | `ConvexInt64` | Use for 64-bit integers |
| `boolean` | `bool` | |
| `string` | `str` | |
| `Array` | `list` | Also accepts `tuple` |
| `object` | `dict` | |
| `ArrayBuffer` | `bytes` | |
| `Id` | `str` | Document IDs returned as strings |

---

## Client setup and configuration

### Standard initialization pattern

```python
import os
from convex import ConvexClient
from dotenv import load_dotenv

load_dotenv(".env.local")
client = ConvexClient(os.getenv("CONVEX_URL"))

# Enable debug logging to see Convex function logs
client.set_debug(True)
```

### HTTP client for environments without WebSocket support

```python
from convex.http_client import ConvexHttpClient

# Use when WebSockets unavailable (some serverless environments)
client = ConvexHttpClient('https://your-deployment.convex.cloud')
# Note: HTTP client cannot use subscribe()
```

### Finding your deployment URL

Open the Convex dashboard at `dashboard.convex.dev`, select your deployment, and click Settings. The URL follows the format `https://happy-animal-123.convex.cloud`.

---

## Calling Convex functions

### Queries for reading data

Queries are deterministic, read-only functions. They're automatically cached and can be subscribed to for real-time updates.

```python
# Simple query
messages = client.query("messages:list")

# Query with arguments
user = client.query("users:getById", {"userId": "abc123"})

# Query with complex arguments
results = client.query("search:find", {
    "filters": {"category": "books", "minPrice": 10},
    "pagination": {"limit": 20, "cursor": None}
})
```

### Mutations for writing data

Mutations run as ACID transactions with **serializable isolation**—the strictest consistency guarantee. If concurrent writes conflict, Convex automatically retries.

```python
# Create a document
task_id = client.mutation("tasks:create", {
    "text": "Learn Convex",
    "isCompleted": False
})

# Update a document
client.mutation("tasks:update", {
    "id": task_id,
    "isCompleted": True
})

# Delete a document
client.mutation("tasks:delete", {"id": task_id})
```

### Actions for external operations

Actions can call external APIs and perform non-deterministic operations. They cannot directly access the database—they must call mutations to persist data.

```python
# Call external AI service
response = client.action("ai:generateResponse", {
    "prompt": "Explain quantum computing"
})

# Process payment through external API
result = client.action("payments:processStripe", {
    "amount": 2999,
    "currency": "usd"
})
```

### Function naming convention

Functions are referenced as `"filename:functionName"`:
- `"tasks:list"` → calls `list` export from `convex/tasks.ts`
- `"users:getById"` → calls `getById` export from `convex/users.ts`

---

## Real-time subscriptions

The Python client supports WebSocket-based subscriptions that automatically update when underlying data changes.

### Basic subscription pattern

```python
# Subscribe to query - loops indefinitely
for tasks in client.subscribe("tasks:list"):
    print(f"Tasks updated: {len(tasks)} items")
    # Process updated data
    for task in tasks:
        print(f"  - {task['text']}")
```

### Subscription with arguments

```python
# Subscribe to filtered data
for messages in client.subscribe("messages:byChannel", {"channel": "general"}):
    print(f"Channel has {len(messages)} messages")
    # Handle new messages
```

### Breaking out of subscriptions

```python
import signal
import sys

def signal_handler(sig, frame):
    print("Stopping subscription...")
    sys.exit(0)

signal.signal(signal.SIGINT, signal_handler)

for updates in client.subscribe("presence:online"):
    print(f"{len(updates)} users online")
    if some_condition:
        break  # Exit subscription loop
```

### How subscriptions work under the hood

Convex tracks every data dependency when a query runs. When any dependency changes (a document is inserted, updated, or deleted), Convex re-runs the query and pushes the new result via WebSocket. This happens automatically—no polling, no manual invalidation.

---

## Authentication

### Setting JWT tokens from auth providers

```python
# After user authenticates with Clerk, Auth0, etc.
jwt_token = get_token_from_auth_provider()  # Your auth flow
client.set_auth(jwt_token)

# All subsequent calls include user context
user_data = client.query("users:getCurrentUser")
my_tasks = client.query("tasks:myTasks")
```

### Admin authentication for internal operations

```python
# Use admin key for server-to-server operations
# Get admin key from Convex Dashboard → Settings → Deploy Keys
client.set_admin_auth("your-admin-key")

# Can now call internal functions
result = client.mutation("internal:adminOperation", {...})
```

### Clearing authentication

```python
client.set_auth(None)  # Clear auth token
```

---

## Working with Convex types

### Using ConvexInt64 for large integers

Python integers map to JavaScript Float64 (loses precision beyond 2^53). Use `ConvexInt64` for 64-bit integers:

```python
from convex import ConvexInt64

# For large integers that need precision
big_number = ConvexInt64(2**60)
client.mutation("analytics:recordCount", {"count": big_number})

# Reading back
result = client.query("analytics:getCount")
# result["count"] will be a ConvexInt64 object
```

### Working with document IDs

```python
# IDs are returned as strings
doc = client.query("tasks:getFirst")
task_id = doc["_id"]  # String like "jd7f8g9h..."

# Pass IDs to other functions
client.mutation("tasks:complete", {"id": task_id})
```

### ConvexSet and ConvexMap for JavaScript Set/Map

```python
from convex import ConvexSet, ConvexMap

# For JavaScript Set compatibility
tags = ConvexSet(["python", "convex", "backend"])

# For JavaScript Map compatibility (allows non-string keys)
mapping = ConvexMap([
    ({"id": 1}, "value1"),
    ({"id": 2}, "value2")
])
```

---

## Error handling

### Catching application errors

```python
from convex import ConvexClient, ConvexError

client = ConvexClient(os.environ["CONVEX_URL"])

try:
    client.mutation("orders:create", {
        "productId": "prod_123",
        "quantity": 100
    })
except ConvexError as err:
    # Application-level errors thrown from Convex functions
    if isinstance(err.data, dict):
        code = err.data.get("code")
        if code == "INSUFFICIENT_STOCK":
            available = err.data.get("available", 0)
            print(f"Only {available} items available")
        elif code == "NOT_FOUND":
            print("Product not found")
        else:
            print(f"Error: {err.data.get('message')}")
    elif isinstance(err.data, str):
        print(f"Error: {err.data}")
except Exception as err:
    # Network errors, developer errors, etc.
    print(f"Unexpected error: {err}")
```

### Error types in Convex

| Error Type | Source | Handling |
|------------|--------|----------|
| `ConvexError` | Explicitly thrown in functions | Catch and handle by error code |
| Developer errors | Bugs in Convex functions | Log and fix code |
| Network errors | Connection issues | Retry with backoff |
| Internal errors | Convex infrastructure | Auto-retried by Convex |

---

## Pagination

### Implementing cursor-based pagination

```python
def fetch_all_messages():
    """Fetch all messages using cursor pagination."""
    all_messages = []
    cursor = None

    while True:
        result = client.query("messages:list", {
            "paginationOpts": {
                "numItems": 50,
                "cursor": cursor
            }
        })

        all_messages.extend(result["page"])
        cursor = result["continueCursor"]

        print(f"Fetched {len(result['page'])} messages")

        if result["isDone"]:
            break

    return all_messages

# Fetch page by page
def fetch_page(cursor=None, page_size=20):
    """Fetch a single page of results."""
    return client.query("messages:list", {
        "paginationOpts": {
            "numItems": page_size,
            "cursor": cursor
        }
    })
```

---

## Backend function patterns

The Python client calls functions defined on the Convex backend in TypeScript/JavaScript. Understanding backend patterns helps you design effective Python integrations.

### Defining backend functions

```typescript
// convex/tasks.ts
import { query, mutation } from "./_generated/server";
import { v } from "convex/values";

// Query - read-only, subscribable
export const list = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.query("tasks").order("desc").collect();
  },
});

// Mutation - transactional writes
export const create = mutation({
  args: { text: v.string() },
  handler: async (ctx, args) => {
    return await ctx.db.insert("tasks", {
      text: args.text,
      isCompleted: false,
    });
  },
});

// Query with authentication
export const myTasks = query({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    return await ctx.db
      .query("tasks")
      .withIndex("by_user", (q) => q.eq("userId", identity.subject))
      .collect();
  },
});
```

### Schema definition

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  tasks: defineTable({
    text: v.string(),
    isCompleted: v.boolean(),
    userId: v.optional(v.string()),
  })
    .index("by_user", ["userId"])
    .index("by_completed", ["isCompleted"]),

  users: defineTable({
    name: v.string(),
    email: v.string(),
    tokenIdentifier: v.string(),
  }).index("by_token", ["tokenIdentifier"]),
});
```

### Database operations reference

| Operation | Method | Example |
|-----------|--------|---------|
| Create | `ctx.db.insert()` | `await ctx.db.insert("tasks", {...})` |
| Read by ID | `ctx.db.get()` | `await ctx.db.get(taskId)` |
| Query | `ctx.db.query()` | `await ctx.db.query("tasks").collect()` |
| Update (partial) | `ctx.db.patch()` | `await ctx.db.patch(id, {field: value})` |
| Update (replace) | `ctx.db.replace()` | `await ctx.db.replace(id, newDoc)` |
| Delete | `ctx.db.delete()` | `await ctx.db.delete(id)` |

---

## Advanced features

### Scheduled functions

Schedule functions to run in the future from the backend:

```typescript
// convex/messages.ts
export const sendExpiringMessage = mutation({
  args: { body: v.string() },
  handler: async (ctx, args) => {
    const id = await ctx.db.insert("messages", { body: args.body });
    // Self-destruct after 5 seconds
    await ctx.scheduler.runAfter(5000, internal.messages.delete, { id });
    return id;
  },
});
```

Call from Python:

```python
# Message will auto-delete after 5 seconds
message_id = client.mutation("messages:sendExpiringMessage", {
    "body": "This message will self-destruct"
})
```

### Cron jobs

Define recurring tasks in `convex/crons.ts`:

```typescript
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";

const crons = cronJobs();

// Every 30 seconds
crons.interval("cleanup", { seconds: 30 }, internal.maintenance.cleanup);

// Daily at 9:30 AM UTC
crons.daily("report", { hourUTC: 9, minuteUTC: 30 }, internal.reports.generate);

// Standard cron syntax
crons.cron("weekly", "0 0 * * 0", internal.backup.run);

export default crons;
```

### HTTP actions for webhooks

Expose HTTP endpoints for external integrations:

```typescript
// convex/http.ts
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";

const http = httpRouter();

http.route({
  path: "/webhook/stripe",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const payload = await request.json();
    await ctx.runMutation(internal.payments.process, { payload });
    return new Response(JSON.stringify({ received: true }), {
      status: 200,
      headers: { "Content-Type": "application/json" },
    });
  }),
});

export default http;
```

Call HTTP endpoints from Python:

```python
import requests

# HTTP actions are at .convex.site (not .convex.cloud)
CONVEX_SITE_URL = "https://your-deployment.convex.site"

response = requests.post(
    f"{CONVEX_SITE_URL}/webhook/stripe",
    json={"event": "payment.completed", "amount": 9999}
)
```

### File storage

Upload and manage files:

```python
import requests

# Step 1: Get upload URL
upload_url = client.mutation("files:generateUploadUrl", {})

# Step 2: Upload file
with open("document.pdf", "rb") as f:
    response = requests.post(
        upload_url,
        data=f.read(),
        headers={"Content-Type": "application/pdf"}
    )
    storage_id = response.json()["storageId"]

# Step 3: Save reference in database
client.mutation("files:save", {
    "storageId": storage_id,
    "filename": "document.pdf"
})

# Get file URL for download
url = client.query("files:getUrl", {"storageId": storage_id})
```

---

## Project structure recommendations

```
my-convex-app/
├── convex/                      # All backend code
│   ├── _generated/              # Auto-generated (commit this!)
│   ├── schema.ts                # Database schema
│   ├── auth.config.ts           # Auth provider config
│   ├── crons.ts                 # Cron job definitions
│   ├── http.ts                  # HTTP endpoints
│   ├── tasks.ts                 # Domain functions
│   ├── users.ts                 # User functions
│   └── model/                   # Business logic helpers
│       └── validation.ts
├── python/                      # Python application
│   ├── __init__.py
│   ├── client.py                # Convex client wrapper
│   ├── handlers.py              # Business logic
│   └── main.py                  # Entry point
├── .env.local                   # Environment variables (don't commit!)
├── convex.json                  # Convex configuration
└── package.json                 # Node dependencies for Convex
```

---

## Common patterns

### CRUD wrapper class

```python
class ConvexRepository:
    """Generic repository pattern for Convex operations."""

    def __init__(self, client: ConvexClient, table: str):
        self.client = client
        self.table = table

    def list(self, **filters):
        return self.client.query(f"{self.table}:list", filters or {})

    def get(self, id: str):
        return self.client.query(f"{self.table}:get", {"id": id})

    def create(self, **data):
        return self.client.mutation(f"{self.table}:create", data)

    def update(self, id: str, **data):
        return self.client.mutation(f"{self.table}:update", {"id": id, **data})

    def delete(self, id: str):
        return self.client.mutation(f"{self.table}:delete", {"id": id})

# Usage
tasks = ConvexRepository(client, "tasks")
all_tasks = tasks.list()
new_id = tasks.create(text="New task", isCompleted=False)
tasks.update(new_id, isCompleted=True)
```

### Background job pattern

```python
class BackgroundJobManager:
    """Manage async background jobs through Convex."""

    def __init__(self, client: ConvexClient):
        self.client = client

    def submit_job(self, job_type: str, payload: dict) -> str:
        """Submit a job and return job ID."""
        return self.client.mutation("jobs:submit", {
            "type": job_type,
            "payload": payload
        })

    def get_status(self, job_id: str) -> dict:
        """Check job status."""
        return self.client.query("jobs:getStatus", {"id": job_id})

    def wait_for_completion(self, job_id: str, timeout: int = 60):
        """Poll until job completes."""
        import time
        start = time.time()
        while time.time() - start < timeout:
            status = self.get_status(job_id)
            if status["state"] in ("completed", "failed"):
                return status
            time.sleep(1)
        raise TimeoutError(f"Job {job_id} did not complete in {timeout}s")

# Usage
jobs = BackgroundJobManager(client)
job_id = jobs.submit_job("image_generation", {"prompt": "A sunset"})
result = jobs.wait_for_completion(job_id)
```

### Real-time sync with callbacks

```python
import threading
from typing import Callable, Any

class RealtimeSync:
    """Manage real-time subscriptions with callbacks."""

    def __init__(self, client: ConvexClient):
        self.client = client
        self._subscriptions = {}
        self._threads = {}

    def subscribe(self, query: str, callback: Callable[[Any], None], args: dict = None):
        """Start subscription in background thread."""
        def run():
            for data in self.client.subscribe(query, args or {}):
                callback(data)

        thread = threading.Thread(target=run, daemon=True)
        thread.start()
        self._threads[query] = thread

    def unsubscribe(self, query: str):
        """Stop subscription (thread will terminate on next update)."""
        if query in self._threads:
            del self._threads[query]

# Usage
sync = RealtimeSync(client)
sync.subscribe("tasks:list", lambda tasks: print(f"Tasks: {len(tasks)}"))
sync.subscribe("presence:online", lambda users: print(f"Online: {len(users)}"))
```

---

## Common pitfalls and how to avoid them

### Pitfall 1: Not awaiting promises in backend code

**Problem**: Forgetting `await` causes silent failures.

```typescript
// ❌ BAD - scheduler.runAfter not awaited
export const sendMessage = mutation({
  handler: async (ctx, args) => {
    ctx.scheduler.runAfter(5000, internal.messages.expire, {...});  // Missing await!
  },
});

// ✅ GOOD - properly awaited
export const sendMessage = mutation({
  handler: async (ctx, args) => {
    await ctx.scheduler.runAfter(5000, internal.messages.expire, {...});
  },
});
```

**Fix**: Enable the `no-floating-promises` ESLint rule.

### Pitfall 2: Using filter instead of indexes

**Problem**: `.filter()` scans all documents—same as loading everything into memory.

```typescript
// ❌ BAD - full table scan
const movies = await ctx.db.query("movies")
  .filter(q => q.eq(q.field("director"), "Spielberg"))
  .collect();

// ✅ GOOD - index-powered query
const movies = await ctx.db.query("movies")
  .withIndex("by_director", q => q.eq("director", "Spielberg"))
  .collect();
```

### Pitfall 3: Unbounded queries

**Problem**: `.collect()` without limits can return thousands of documents.

```python
# ❌ BAD - could return millions of records
all_logs = client.query("logs:getAll")

# ✅ GOOD - paginate or limit
page = client.query("logs:getRecent", {"limit": 100})
```

### Pitfall 4: Frontend-only authentication

**Problem**: Checking auth only in frontend leaves backend vulnerable.

```typescript
// ❌ BAD - no backend auth check
export const deleteAccount = mutation({
  args: { userId: v.id("users") },
  handler: async (ctx, args) => {
    await ctx.db.delete(args.userId);  // Anyone can delete any user!
  },
});

// ✅ GOOD - verify identity
export const deleteAccount = mutation({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    const user = await ctx.db
      .query("users")
      .withIndex("by_token", q => q.eq("tokenIdentifier", identity.tokenIdentifier))
      .unique();

    if (!user) throw new Error("User not found");
    await ctx.db.delete(user._id);
  },
});
```

### Pitfall 5: Exposing internal functions

**Problem**: Using `api.*` for scheduled functions exposes them publicly.

```typescript
// ❌ BAD - public function can be called by anyone
crons.daily("cleanup", {...}, api.maintenance.cleanup);

// ✅ GOOD - internal function not externally accessible
crons.daily("cleanup", {...}, internal.maintenance.cleanup);
```

### Pitfall 6: Calling actions from mutations

**Problem**: Actions can fail and aren't transactional—don't call them from mutations.

```typescript
// ❌ BAD - action in mutation breaks atomicity
export const processOrder = mutation({
  handler: async (ctx, args) => {
    await ctx.db.insert("orders", {...});
    await ctx.runAction(api.payments.charge, {...});  // If this fails, order still exists!
  },
});

// ✅ GOOD - mutation schedules action
export const processOrder = mutation({
  handler: async (ctx, args) => {
    const orderId = await ctx.db.insert("orders", {...});
    await ctx.scheduler.runAfter(0, internal.payments.charge, { orderId });
  },
});
```

---

## Decision framework

### When to use queries vs mutations vs actions

```
┌─────────────────────────────────────────────────────────────────┐
│                     What does your function do?                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │   Does it modify the database? │
              └───────────────────────────────┘
                     │              │
                    No             Yes
                     │              │
                     ▼              ▼
            ┌──────────────┐  ┌─────────────────────────┐
            │    QUERY     │  │ Does it call external   │
            │              │  │ APIs or need side       │
            │ • Read-only  │  │ effects?                │
            │ • Subscribable│  └─────────────────────────┘
            │ • Cached     │         │           │
            └──────────────┘        No          Yes
                                    │           │
                                    ▼           ▼
                            ┌──────────┐  ┌──────────────┐
                            │ MUTATION │  │ MUTATION +   │
                            │          │  │ scheduled    │
                            │ • ACID   │  │ ACTION       │
                            │ • Retry  │  │              │
                            └──────────┘  │ Mutation     │
                                          │ writes DB,   │
                                          │ Action calls │
                                          │ external API │
                                          └──────────────┘
```

### When to use Python client vs HTTP actions

| Use Python Client | Use HTTP Actions |
|-------------------|------------------|
| Real-time subscriptions needed | Webhook receivers |
| Interactive applications | REST API endpoints |
| Background services | Third-party integrations |
| Direct function calls | CORS-enabled endpoints |

### Index design decisions

```
┌─────────────────────────────────────────────────────────────────┐
│              Do you need to query by this field?                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                Yes                        No
                 │                          │
                 ▼                          ▼
    ┌───────────────────────┐    ┌────────────────────┐
    │ Will you filter by    │    │  No index needed   │
    │ multiple fields       │    └────────────────────┘
    │ together?             │
    └───────────────────────┘
           │           │
          Yes         No
           │           │
           ▼           ▼
    ┌────────────┐  ┌─────────────┐
    │ Compound   │  │ Single-     │
    │ index      │  │ field       │
    │            │  │ index       │
    │ Put        │  └─────────────┘
    │ equality   │
    │ fields     │
    │ first,     │
    │ range      │
    │ fields     │
    │ last       │
    └────────────┘
```

---

## Testing strategies

### Unit testing Convex functions

```typescript
// convex/tasks.test.ts
import { convexTest } from "convex-test";
import { describe, it, expect } from "vitest";
import { api } from "./_generated/api";
import schema from "./schema";

describe("tasks", () => {
  it("creates and retrieves tasks", async () => {
    const t = convexTest(schema);

    await t.mutation(api.tasks.create, { text: "Test task" });

    const tasks = await t.query(api.tasks.list);
    expect(tasks).toHaveLength(1);
    expect(tasks[0].text).toBe("Test task");
  });

  it("requires authentication for myTasks", async () => {
    const t = convexTest(schema);

    // Without auth
    await expect(t.query(api.tasks.myTasks)).rejects.toThrow("Unauthorized");

    // With auth
    const authed = t.withIdentity({ subject: "user123" });
    const tasks = await authed.query(api.tasks.myTasks);
    expect(tasks).toEqual([]);
  });
});
```

### Integration testing Python client

```python
import pytest
from convex import ConvexClient

@pytest.fixture
def client():
    return ConvexClient(os.environ["CONVEX_URL"])

def test_create_and_list_tasks(client):
    # Create task
    task_id = client.mutation("tasks:create", {"text": "Test task"})
    assert task_id is not None

    # Verify it appears in list
    tasks = client.query("tasks:list")
    assert any(t["_id"] == task_id for t in tasks)

    # Cleanup
    client.mutation("tasks:delete", {"id": task_id})

def test_error_handling(client):
    from convex import ConvexError

    with pytest.raises(ConvexError) as exc_info:
        client.mutation("orders:create", {"productId": "nonexistent"})

    assert exc_info.value.data["code"] == "NOT_FOUND"
```

---

## Complete application example

```python
"""
Complete Convex Python application demonstrating common patterns.
"""
import os
import time
import threading
from dataclasses import dataclass
from typing import Optional, Callable, Any
from convex import ConvexClient, ConvexError
from dotenv import load_dotenv

load_dotenv(".env.local")

@dataclass
class Task:
    id: str
    text: str
    is_completed: bool
    created_at: float

class TaskManager:
    """Full-featured task manager using Convex."""

    def __init__(self):
        self.client = ConvexClient(os.environ["CONVEX_URL"])
        self._subscription_thread: Optional[threading.Thread] = None
        self._on_update: Optional[Callable[[list[Task]], None]] = None

    def authenticate(self, token: str):
        """Set authentication token."""
        self.client.set_auth(token)

    def list_tasks(self) -> list[Task]:
        """Get all tasks."""
        raw_tasks = self.client.query("tasks:list")
        return [self._to_task(t) for t in raw_tasks]

    def get_task(self, task_id: str) -> Optional[Task]:
        """Get single task by ID."""
        raw = self.client.query("tasks:get", {"id": task_id})
        return self._to_task(raw) if raw else None

    def create_task(self, text: str) -> str:
        """Create new task, returns task ID."""
        try:
            return self.client.mutation("tasks:create", {"text": text})
        except ConvexError as e:
            if isinstance(e.data, dict) and e.data.get("code") == "VALIDATION_ERROR":
                raise ValueError(e.data.get("message", "Invalid input"))
            raise

    def complete_task(self, task_id: str):
        """Mark task as completed."""
        self.client.mutation("tasks:update", {
            "id": task_id,
            "isCompleted": True
        })

    def delete_task(self, task_id: str):
        """Delete task."""
        self.client.mutation("tasks:delete", {"id": task_id})

    def subscribe_to_updates(self, callback: Callable[[list[Task]], None]):
        """Subscribe to real-time task updates."""
        self._on_update = callback

        def run_subscription():
            for raw_tasks in self.client.subscribe("tasks:list"):
                if self._on_update:
                    tasks = [self._to_task(t) for t in raw_tasks]
                    self._on_update(tasks)

        self._subscription_thread = threading.Thread(target=run_subscription, daemon=True)
        self._subscription_thread.start()

    def _to_task(self, raw: dict) -> Task:
        """Convert raw dict to Task object."""
        return Task(
            id=raw["_id"],
            text=raw["text"],
            is_completed=raw["isCompleted"],
            created_at=raw["_creationTime"]
        )

# Example usage
if __name__ == "__main__":
    manager = TaskManager()

    # Create some tasks
    task1_id = manager.create_task("Learn Convex")
    task2_id = manager.create_task("Build an app")
    print(f"Created tasks: {task1_id}, {task2_id}")

    # List all tasks
    tasks = manager.list_tasks()
    print(f"Found {len(tasks)} tasks")

    # Complete a task
    manager.complete_task(task1_id)
    print(f"Completed task {task1_id}")

    # Subscribe to real-time updates
    def on_tasks_updated(tasks: list[Task]):
        print(f"Tasks updated! Now have {len(tasks)} tasks")
        for task in tasks:
            status = "✓" if task.is_completed else "○"
            print(f"  {status} {task.text}")

    manager.subscribe_to_updates(on_tasks_updated)

    # Keep running for real-time updates
    print("Listening for updates (Ctrl+C to stop)...")
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("Shutting down...")
```

---

## Environment variables and deployment

### Required environment variables

```bash
# .env.local (development)
CONVEX_URL=https://your-deployment.convex.cloud

# For HTTP actions (webhooks)
CONVEX_SITE_URL=https://your-deployment.convex.site

# Authentication (if using)
AUTH_TOKEN=your-jwt-token
```

### Setting backend environment variables

```bash
# Via CLI
npx convex env set OPENAI_API_KEY sk-xxx
npx convex env set STRIPE_SECRET_KEY sk_live_xxx

# List variables
npx convex env list
```

### Deployment commands

```bash
# Development (watches for changes)
npx convex dev

# Production deployment
npx convex deploy

# Run a function directly
npx convex run tasks:list
npx convex run tasks:create '{"text": "New task"}'
```

---

## Resources

- **Documentation**: https://docs.convex.dev
- **Python Client**: https://docs.convex.dev/client/python
- **GitHub**: https://github.com/get-convex/convex-py
- **PyPI**: https://pypi.org/project/convex/
- **Dashboard**: https://dashboard.convex.dev
- **Discord**: https://convex.dev/community
