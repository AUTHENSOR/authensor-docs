# MCP Gateway

Transparent authorization proxy between any MCP client and any upstream MCP server. Every tool call is evaluated against Authensor policies before it reaches the upstream server.

## How It Works

The gateway spawns the upstream MCP server as a child process, proxies its `tools/list` response, and intercepts every `tools/call`. Before forwarding a call, it constructs an ActionEnvelope and evaluates it against the Authensor control plane. Allowed calls are forwarded; denied calls return an error to the client. A receipt is emitted for every decision.

```
MCP Client ──► Authensor MCP Gateway ──► Upstream MCP Server
                    │
                    ▼
              Authensor Control Plane
              (policy eval + receipts)
```

## SEP Authorization Protocol

The gateway implements the MCP SEP (Specification Enhancement Proposal) for pre-execution authorization hooks. Three message types:

| Message | Direction | Purpose |
|---------|-----------|---------|
| `authorization/propose` | Client -> Gateway | Client proposes an action envelope before calling a tool |
| `authorization/decide` | Gateway -> Client | Gateway returns the policy decision with a `receiptId` |
| `authorization/receipt` | Gateway -> Client | Gateway emits an immutable receipt after execution |

### Protocol flow

```
Client                          Gateway                     Upstream
  │                                │                           │
  │  authorization/propose ──────► │                           │
  │                                │  POST /evaluate ────────► │ (control plane)
  │  ◄────── authorization/decide  │                           │
  │         { outcome, receiptId } │                           │
  │                                │                           │
  │  tools/call ─────────────────► │                           │
  │  { _meta.authorizationId }     │  tools/call ────────────► │
  │                                │  ◄──── tools/call result  │
  │  ◄──────── tools/call result   │                           │
  │  ◄──── authorization/receipt   │                           │
```

### Decision outcomes

| Outcome | Client action |
|---------|---------------|
| `allow` | Proceed to `tools/call` with the `authorizationId` |
| `deny` | Do not call the tool; report the reason |
| `require_approval` | Wait for human approval before proceeding |
| `rate_limited` | Back off and retry after the rate limit window |

## Backward Compatibility

If the client does not negotiate the `authorization` capability, the gateway falls back to **inline evaluation** -- it evaluates the tool call inside the `tools/call` handler and either forwards or blocks without the propose/decide round trip. No client changes are required.

When `requireAuthorization` is set to `true` in the gateway options and the client has negotiated authorization support, the gateway rejects any `tools/call` that does not carry a valid `authorizationId`.

## Setup

### Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `UPSTREAM_COMMAND` | Yes | Command to spawn the upstream MCP server |
| `CONTROL_PLANE_URL` | No | Authensor control plane URL (default: `http://localhost:3000`) |
| `AUTHENSOR_PRINCIPAL_ID` | No | Principal ID for policy evaluation (default: `mcp-gateway`) |

### Run the gateway

```bash
CONTROL_PLANE_URL=http://localhost:3000 \
UPSTREAM_COMMAND="npx @modelcontextprotocol/server-filesystem /tmp" \
npx authensor-mcp-gateway
```

### Configure in Claude Desktop

Add the gateway to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "filesystem-safe": {
      "command": "npx",
      "args": ["authensor-mcp-gateway"],
      "env": {
        "UPSTREAM_COMMAND": "npx @modelcontextprotocol/server-filesystem /tmp",
        "CONTROL_PLANE_URL": "http://localhost:3000"
      }
    }
  }
}
```

## Programmatic Usage

```typescript
import { createGateway } from '@authensor/mcp-server/gateway';

await createGateway({
  controlPlaneUrl: 'http://localhost:3000',
  upstreamCommand: 'npx @modelcontextprotocol/server-filesystem /tmp',
  principalId: 'my-agent',
  requireAuthorization: true,
});
```

### Gateway options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `controlPlaneUrl` | `string` | required | Authensor control plane URL |
| `upstreamCommand` | `string` | required | Command to spawn the upstream MCP server |
| `upstreamArgs` | `string[]` | `[]` | Arguments for the upstream command |
| `upstreamEnv` | `Record<string, string>` | `{}` | Extra environment variables for the upstream process |
| `principalId` | `string` | `'mcp-gateway'` | Principal ID for policy evaluation |
| `requireAuthorization` | `boolean` | `false` | Require `authorization/propose` before `tools/call` |

## Authorization TTL

Pending authorizations expire after 5 minutes. If a client sends `authorization/propose`, receives an `allow` decision, but does not call the tool within 5 minutes, the authorization is discarded and the client must propose again.

## Fail-Closed Behavior

- If the control plane is unreachable, all tool calls are denied.
- If no policy is configured, the default effect is deny.
- Receipt emission is fire-and-forget -- receipt failures do not affect tool execution.

## Related

- [MCP SEP Specification](../standards/mcp-sep.md) -- full protocol specification
- [Aegis](aegis.md) -- content safety scanning (integrated into the evaluation pipeline)
- [Sentinel](sentinel.md) -- real-time monitoring of receipt streams
