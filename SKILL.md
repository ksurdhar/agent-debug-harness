---
name: debug-harness
description: Use for debugging complex runtime issues by injecting temporary logging. Activates when debugging behavior that's hard to trace with static analysis.
---

# Agent Debug Harness

A lightweight HTTP server for collecting debug logs during debugging sessions.

## Location

The harness lives at: `/Users/kiransurdhar/projects/agent-debug-harness`

## When to Use

Use this tool when:
- Static code analysis isn't revealing the issue
- You need to trace runtime behavior across multiple functions
- You want to test specific hypotheses about code execution
- Console.log would clutter the codebase or output

## Hypothesis-Driven Debugging Workflow

**IMPORTANT**: Always form hypotheses BEFORE instrumenting code.

### Step 1: Form Hypotheses

Before touching any code, write down 2-3 specific hypotheses:
- H1: "The user object is null when reaching the save function"
- H2: "The API is being called twice due to a race condition"
- H3: "The timeout is firing before the response arrives"

### Step 2: Start the Harness

```bash
~/.bun/bin/bun run /Users/kiransurdhar/projects/agent-debug-harness/src/index.ts &
echo $! > /tmp/debug-harness.pid
```

Verify it's running: `curl http://127.0.0.1:7243/health`

### Step 3: Inject Strategic Logs

Add temporary fetch calls at key points. Always include:
- `location`: file:function:stage (e.g., "auth.ts:login:entry")
- `hypothesisId`: Which hypothesis this tests (e.g., "H1" or "H1,H2")
- `data`: Relevant runtime values

```typescript
// #region agent-debug
fetch('http://127.0.0.1:7243/log', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    location: 'auth.ts:login:beforeSave',
    hypothesisId: 'H1',
    message: 'Checking user object before save',
    data: { user, isNull: user === null },
    timestamp: Date.now()
  })
}).catch(() => {})
// #endregion
```

### Step 4: Trigger the Behavior

Run the code/tests that reproduce the issue.

### Step 5: Analyze Results

```bash
curl -s http://127.0.0.1:7243/logs | jq .
```

Look for:
- Which hypotheses are supported/refuted by the data
- Unexpected values or missing logs (code path not taken)
- Timing patterns

### Step 6: Reset Between Runs

```bash
curl -X POST http://127.0.0.1:7243/reset
```

### Step 7: Clean Up

When done debugging:
1. Remove all `#region agent-debug` blocks from code
2. Stop the harness: `kill $(cat /tmp/debug-harness.pid) && rm /tmp/debug-harness.pid`

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check |
| `/log` | POST | Append log entry |
| `/log/:sessionId` | POST | Append with session grouping |
| `/logs` | GET | Get all logs as JSON array |
| `/reset` | POST | Clear all logs |

## Log Entry Fields

- `timestamp` (auto-added if missing)
- `location` - Where in code: `file:function:stage`
- `message` - Human description
- `data` - Any JSON data
- `hypothesisId` - Which hypothesis this tests (H1, H2, etc.)
- `sessionId` - Group related logs
