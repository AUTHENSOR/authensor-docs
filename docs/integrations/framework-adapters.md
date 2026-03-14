# Framework Adapters

Authensor provides adapters for 8 agent frameworks and integration paths. Every adapter follows the same pattern: intercept tool calls, evaluate against the Authensor control plane, and map the decision back to the framework's response format.

```
Agent tool call
  -> Adapter classifies tool -> ActionEnvelope
  -> POST /evaluate to Authensor control plane
  -> Decision: allow / deny / require_approval
  -> Adapter maps decision to framework response
  -> POST /receipts with outcome (async)
```

## Adapter Overview

| Adapter | Package | Language | Integration point |
|---------|---------|----------|-------------------|
| [LangChain / LangGraph](#langchain--langgraph) | `@authensor/langchain` | TypeScript | Tool wrapper / graph validation node |
| [OpenAI Agents SDK](#openai-agents-sdk) | `@authensor/openai` | TypeScript | Pre-tool-call guardrail function |
| [Vercel AI SDK](#vercel-ai-sdk) | `@authensor/vercel-ai-sdk` | TypeScript | Tool wrapper / `needsApproval` hook |
| [Claude Agent SDK](#claude-agent-sdk) | `@authensor/claude-agent-sdk` | TypeScript | Tool handler wrapper with approval flow |
| [CrewAI](#crewai) | `authensor-crewai` | Python | Decorator / guard class |
| [Claude Code hooks](#claude-code-hooks) | Shell script | Bash | `PreToolUse` hook in `.claude/hooks.json` |
| [TypeScript SDK](#typescript-sdk) | `@authensor/sdk-ts` | TypeScript | Direct client library |
| [Python SDK](#python-sdk) | `authensor-sdk-py` | Python | Direct client library |

---

## LangChain / LangGraph

```bash
npm install @authensor/langchain
```

```typescript
import { AuthensorGuard } from '@authensor/langchain';

const guard = new AuthensorGuard({
  controlPlaneUrl: 'http://localhost:3000',
  apiKey: process.env.AUTHENSOR_API_KEY,
  principalId: 'my-langchain-agent',
});

// Wrap a single tool
const safeTool = guard.wrap(myTool);

// Or evaluate manually
const result = await guard.evaluate('search', { query: 'test' });
```

The guard wraps any object with `name` and `invoke` methods. The `evaluate` call constructs an envelope with action type `langchain.<toolName>` and throws on deny.

---

## OpenAI Agents SDK

```bash
npm install @authensor/openai
```

```typescript
import { createAuthensorGuardrail } from '@authensor/openai';

const { evaluate, guard } = createAuthensorGuardrail({
  controlPlaneUrl: 'http://localhost:3000',
  principalId: 'my-openai-agent',
});

// In your agent loop, before executing a tool:
const result = await evaluate('send_email', { to: 'user@example.com' });
if (!result.allowed) {
  // handle denial
}

// Or use guard() which throws on deny:
const receiptId = await guard('send_email', { to: 'user@example.com' });
```

Action type format: `openai.<toolName>`. Returns `{ allowed, receiptId, outcome, reason }`.

---

## Vercel AI SDK

```bash
npm install @authensor/vercel-ai-sdk
```

```typescript
import { AuthensorVercelGuard } from '@authensor/vercel-ai-sdk';
import { tool } from 'ai';
import { z } from 'zod';

const guard = new AuthensorVercelGuard({
  controlPlaneUrl: 'http://localhost:3000',
});

// Option 1: Wrap a tool definition
const safeTool = guard.wrapTool('send_email', {
  description: 'Send an email',
  parameters: z.object({ to: z.string(), body: z.string() }),
  execute: async ({ to, body }) => sendEmail(to, body),
});

// Option 2: Wrap all tools at once
const safeTools = guard.wrapTools({ send_email: emailTool, search: searchTool });

// Option 3: Use needsApproval with Vercel AI SDK's built-in approval flow
const myTool = tool({
  description: 'Send an email',
  parameters: z.object({ to: z.string() }),
  execute: async ({ to }) => sendEmail(to),
  experimental_needsApproval: guard.needsApproval('send_email'),
});
```

Action type format: `vercel-ai.<toolName>`. Throws `AuthensorDeniedError` on deny.

---

## Claude Agent SDK

```bash
npm install @authensor/claude-agent-sdk
```

```typescript
import { AuthensorClaudeGuard } from '@authensor/claude-agent-sdk';

const guard = new AuthensorClaudeGuard({
  controlPlaneUrl: 'http://localhost:3000',
  onApprovalRequired: async (toolName, args, reason) => {
    // Prompt the user for approval
    return await promptUser(`Approve ${toolName}? Reason: ${reason}`);
  },
});

// Wrap a tool handler
const safeHandler = guard.wrapHandler('send_email', originalHandler);

// Wrap a tool definition + handler pair
const { tool, handler } = guard.wrapTool(toolDef, originalHandler);

// Wrap multiple tools
const safePairs = guard.wrapTools([
  { tool: toolDefA, handler: handlerA },
  { tool: toolDefB, handler: handlerB },
]);
```

Action type format: `claude.<toolName>`. Supports an `onApprovalRequired` callback for interactive approval flows.

---

## CrewAI

```bash
pip install authensor-crewai
```

```python
from authensor_crewai import AuthensorGuard

guard = AuthensorGuard(
    control_plane_url="http://localhost:3000",
    api_key=os.environ.get("AUTHENSOR_API_KEY"),
    principal_id="my-crew-agent",
)

# As a decorator
@guard.protect
def search_tool(query: str) -> str:
    return search(query)

# As a pre-check
result = guard.evaluate("search_tool", {"query": "test"})
if result["allowed"]:
    # proceed

# As a guard (raises PermissionError on deny)
receipt_id = guard.guard("search_tool", {"query": "test"})
```

Action type format: `crewai.<toolName>`. Zero external dependencies -- uses `urllib` only.

---

## Claude Code Hooks

No package to install. Create a shell script and wire it into Claude Code's hook system.

### Setup

1. Save the gate script as `~/.claude/hooks/authensor-gate.sh` (see [Claude Code integration guide](claude-code.md) for the full script).

2. Configure `.claude/hooks.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "~/.claude/hooks/authensor-gate.sh"
      }
    ]
  }
}
```

3. Set environment variables:

```bash
export AUTHENSOR_URL="http://localhost:3000"
export AUTHENSOR_TOKEN="authensor_your_key_here"
```

Read-only operations (`Read`, `Glob`, `Grep`, `TodoWrite`, `Skill`, `TaskOutput`) are pre-filtered locally with no network call. All other tools are evaluated via the control plane.

See [Claude Code integration](claude-code.md) for the full guide including SafeClaw setup, Cursor integration, and policy examples.

---

## TypeScript SDK

```bash
npm install @authensor/sdk-ts
```

```typescript
import { AuthensorClient } from '@authensor/sdk-ts';

const client = new AuthensorClient({
  controlPlaneUrl: 'http://localhost:3000',
  apiKey: process.env.AUTHENSOR_API_KEY,
});

// Evaluate an action
const result = await client.evaluate({
  action: { type: 'payment.create', resource: 'stripe://charges' },
  principal: { type: 'agent', id: 'my-agent' },
});

// Guard (throws on deny)
const receiptId = await client.guard('payment.create', { amount: 5000 });
```

The TypeScript SDK provides `AuthensorClient`, envelope construction helpers, and a `guard` function. Used internally by all TypeScript adapters.

---

## Python SDK

```bash
pip install authensor-sdk-py
```

```python
from authensor import AuthensorClient

client = AuthensorClient(
    control_plane_url="http://localhost:3000",
    api_key=os.environ.get("AUTHENSOR_API_KEY"),
)

result = client.evaluate(
    action_type="search.web",
    resource="https://example.com",
    principal_id="my-agent",
)

if result.allowed:
    # proceed
```

---

## Universal Pattern

All adapters share these behaviors:

- **Fail closed**: If the control plane is unreachable, deny by default.
- **Action type namespacing**: Each adapter prefixes tool names with its framework name (`langchain.`, `openai.`, `vercel-ai.`, `claude.`, `crewai.`).
- **Receipt tracking**: Every evaluation produces a receipt ID for the audit trail.
- **Environment-aware**: Reads `AUTHENSOR_API_KEY` from the environment if not provided.
- **No policy logic in the adapter**: The adapter only classifies and forwards. All policy evaluation happens in the control plane.

## Related

- [Claude Code integration](claude-code.md) -- full Claude Code / Cursor setup guide
- [MCP Gateway](../product/mcp-gateway.md) -- transparent MCP proxy (alternative to adapter-based integration)
- [Ecosystem Overview](../product/ecosystem-overview.md) -- how adapters fit in the stack
