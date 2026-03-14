# Sentinel

Real-time behavioral monitoring engine for AI agents. Processes receipt events, builds per-agent baselines using statistical methods, detects anomalies, and fires configurable alert rules.

Zero runtime dependencies. Pure TypeScript.

## Install

```bash
npm install @authensor/sentinel
```

## What It Detects

| Metric | What it measures | Method |
|--------|-----------------|--------|
| `deny_rate` | Fraction of denied actions per agent | EWMA baseline + CUSUM drift |
| `latency` | Evaluation latency in milliseconds | EWMA baseline + threshold |
| `action_rate` | Actions per time window | Threshold rules |
| `cost_rate` | Monetary cost per time window | Threshold rules |
| `error_rate` | Failed tool executions | EWMA baseline |
| `chain_depth` | Depth of receipt chains (recursive agent calls) | Chain tracker |
| `chain_fanout` | Fan-out from a single parent receipt | Chain tracker |
| `budget_utilization` | Budget consumption rate | Threshold rules |
| `approval_rate` | Rate of approval-required actions | Threshold rules |

## Statistical Methods

### EWMA (Exponentially Weighted Moving Average)

Tracks a smoothed running average of each metric per agent. New values contribute proportionally to a decay factor (alpha). When the current value deviates significantly from the EWMA mean, an anomaly is flagged.

### CUSUM (Cumulative Sum)

Detects sustained drift in a metric away from its baseline. Unlike threshold alerts that fire on single spikes, CUSUM accumulates small deviations over time and triggers when the cumulative sum exceeds a threshold. Useful for detecting gradual behavioral changes (e.g., a slow increase in deny rate over hours).

## Usage

### Basic setup

```typescript
import { Sentinel } from '@authensor/sentinel';

const sentinel = new Sentinel({
  alertRules: [
    {
      id: 'high-deny-rate',
      name: 'High deny rate',
      metric: 'deny_rate',
      operator: 'gt',
      threshold: 0.3,
      windowMs: 5 * 60 * 1000, // 5 minutes
      severity: 'critical',
      enabled: true,
    },
    {
      id: 'cost-spike',
      name: 'Cost spike',
      metric: 'cost_rate',
      operator: 'gt',
      threshold: 10,
      windowMs: 15 * 60 * 1000, // 15 minutes
      severity: 'warning',
      enabled: true,
    },
    {
      id: 'deep-chain',
      name: 'Deep agent chain',
      metric: 'chain_depth',
      operator: 'gt',
      threshold: 5,
      windowMs: 0,
      severity: 'warning',
      enabled: true,
    },
  ],
  onAlert: (alert) => {
    console.log(`ALERT [${alert.severity}]: ${alert.message}`);
  },
  onAnomaly: (anomaly) => {
    console.log(`Anomaly: ${anomaly.metric} for ${anomaly.agentId}`);
  },
});
```

### Processing events

```typescript
// Feed receipt events from the control plane
const { anomalies, alerts } = sentinel.processEvent({
  receiptId: 'rcpt-001',
  envelopeId: 'env-001',
  agentId: 'agent-alpha',
  actionType: 'shell.execute',
  outcome: 'deny',
  timestamp: Date.now(),
  latencyMs: 3,
  cost: 0.01,
  parentReceiptId: undefined,
});
```

### Querying state

```typescript
// Per-agent statistics
const stats = sentinel.getAgentStats('agent-alpha');
// { agentId, totalActions, allowCount, denyCount, denyRate, avgLatencyMs, riskScore, ... }

// All agents
const allStats = sentinel.getAllAgentStats();

// Active alerts
const active = sentinel.getActiveAlerts();

// Chain statistics
const chainStats = sentinel.getChainStats();
// { maxDepth, maxFanout, totalChains }

// Acknowledge an alert
sentinel.acknowledgeAlert('alert-001');

// Overall status
const status = sentinel.getStatus();
// { totalEvents, activeAgents, activeAlerts, uptimeMs, chainStats }
```

### Dynamic alert rules

```typescript
sentinel.addAlertRule({
  id: 'fan-out-limit',
  name: 'Excessive fan-out',
  metric: 'chain_fanout',
  operator: 'gt',
  threshold: 10,
  windowMs: 0,
  severity: 'critical',
  enabled: true,
});

sentinel.removeAlertRule('fan-out-limit');
```

## Receipt Event Schema

```typescript
interface ReceiptEvent {
  receiptId: string;
  envelopeId: string;
  agentId: string;
  actionType: string;
  outcome: 'allow' | 'deny' | 'require_approval' | 'rate_limited' | 'budget_exceeded';
  timestamp: number;       // epoch ms
  latencyMs?: number;
  cost?: number;
  error?: boolean;
  parentReceiptId?: string; // for chain tracking
}
```

## Alert Rule Schema

```typescript
interface AlertRule {
  id: string;
  name: string;
  metric: MetricName;
  operator: 'gt' | 'lt' | 'gte' | 'lte';
  threshold: number;
  windowMs: number;
  severity: 'info' | 'warning' | 'critical';
  enabled: boolean;
}
```

## Integration with the Control Plane

Sentinel is loaded as an optional dependency in the Authensor control plane. Enable it with:

```bash
AUTHENSOR_SENTINEL_ENABLED=true
```

When enabled, the control plane feeds every receipt event to Sentinel automatically. Alerts are available via the `/sentinel/alerts` API endpoint and the dashboard.

## Performance

- Event processing is O(1) amortized per event.
- EWMA and CUSUM updates are constant-time arithmetic operations.
- Chain tracking uses a map-based lookup for depth and fan-out.

## Related

- [Aegis](aegis.md) -- content safety scanning
- [MCP Gateway](mcp-gateway.md) -- transparent MCP proxy with policy enforcement
- [Ecosystem Overview](ecosystem-overview.md) -- how Sentinel fits in the stack
