# Aegis

Content safety scanner for AI agents. Detects prompt injection, PII leakage, credential exposure, data exfiltration, memory poisoning, dangerous code patterns, and multimodal threats.

Zero runtime dependencies. Pure TypeScript. Sub-millisecond latency.

## Install

```bash
npm install @authensor/aegis
```

## Detection Capabilities

### Prompt injection (19 rules)

Detects direct instruction overrides, delimiter-based injection, role manipulation, system prompt extraction, and encoding-based evasion. Covers the attack patterns catalogued in OWASP LLM01.

### Memory poisoning (21 MINJA rules)

Detects MINJA-style attacks where adversaries inject malicious instructions into agent memory, RAG documents, conversation history, or vector DB entries. Catches instructions disguised as stored data, hidden directives in retrieved content, and poisoned tool results (OWASP ASI06).

### PII detection (9 rules)

Identifies Social Security numbers, email addresses, phone numbers, credit card numbers (with Luhn validation), IP addresses, and other personally identifiable information.

### Credential scanning (12 rules)

Detects API keys from major providers (OpenAI, AWS, GitHub, Stripe, Anthropic, Google, Azure, Slack, etc.), private keys, JWTs, and other secrets.

### Data exfiltration (13 rules)

Catches DNS exfiltration, webhook data smuggling, base64-encoded data transfer, URL-encoded exfiltration, and steganographic data hiding patterns.

### Code safety (18 rules)

Flags dangerous shell commands (`rm -rf`, reverse shells, privilege escalation), unsafe file operations, code injection patterns, and environment variable access.

### Output injection (14 rules)

Scans tool results for hidden instructions, markdown-based exfiltration, tool manipulation attempts, and indirect prompt injection via fetched content (OWASP ASI01).

### Multimodal safety

Heuristic checks on image and file attachments: SVG script injection, data URI script embedding, tracking pixels, QR code exfiltration, unexpected MIME types, and phishing domain detection. Supports pluggable vision backends for deeper image analysis.

## Usage

### Basic scan

```typescript
import { AegisScanner } from '@authensor/aegis';

const aegis = new AegisScanner();

const result = aegis.scan('My SSN is 123-45-6789');
// {
//   safe: false,
//   threatLevel: 'medium',
//   detections: [{ type: 'pii', subType: 'SSN', content: '123-45-6789', ... }],
//   scanTimeMs: 0.4,
// }
```

### Scan with specific detectors

```typescript
const result = aegis.scan(content, {
  detectors: ['injection', 'credentials'],
  mode: 'block',
});
```

### Scan tool output

```typescript
const result = aegis.scanOutput(toolResult, {
  toolName: 'http.request',
  includeStandardDetectors: true,
});
// Runs output-specific rules (hidden instructions, markdown exfiltration)
// plus standard detectors (PII, credentials, exfiltration)
```

### Scan multimodal content

```typescript
const result = await aegis.scanMultimodal({
  text: 'Check this image',
  attachments: [
    { mimeType: 'image/svg+xml', data: svgContent },
    { url: 'https://example.com/image.png' },
  ],
});
```

### Redaction mode

```typescript
const result = aegis.scan('My SSN is 123-45-6789 and key is sk-proj-abc', {
  mode: 'redact',
  redactWith: 'REDACTED',
});
console.log(result.redacted);
// "My SSN is [SSN_REDACTED] and key is [OPENAI_KEY_REDACTED]"
```

## Scan Modes

| Mode | Behavior |
|------|----------|
| `block` | Critical threats cause the action to be denied |
| `redact` | Detected content is replaced with redaction tags |
| `warn` | Detections are recorded but do not affect the decision |

## Advanced Features

### Canary tokens

Inject canary tokens into prompts to detect system prompt leakage:

```typescript
import { generateCanaryToken, checkCanaryLeak } from '@authensor/aegis';

const canary = generateCanaryToken();
const prompt = `${canary.token}\nYour system prompt here`;

// Later, check if the canary leaked in agent output
const leak = checkCanaryLeak(agentOutput, canary);
```

### Entropy analysis

Detect adversarial suffixes and encoded payloads by measuring Shannon entropy:

```typescript
import { analyzeEntropy } from '@authensor/aegis';

const result = analyzeEntropy(suspiciousText);
// { entropy, isAnomalous, ... }
```

### Heuristic scoring

Combine multiple detection signals into a single risk score:

```typescript
import { computeHeuristicScore } from '@authensor/aegis';

const result = computeHeuristicScore(signals);
// { score, verdict, ... }
```

### Custom rules

```typescript
const scanner = new AegisScanner([
  ...INJECTION_RULES,
  ...PII_RULES,
  {
    id: 'custom-block-internal-urls',
    type: 'exfiltration',
    subType: 'INTERNAL_URL',
    pattern: /https?:\/\/internal\..*/gi,
    confidence: 0.9,
    description: 'Block references to internal URLs',
  },
]);
```

## Integration with the Control Plane

Aegis is loaded as an optional dependency in the Authensor control plane. Enable it with:

```bash
AUTHENSOR_AEGIS_ENABLED=true
```

When enabled, the control plane scans every action envelope through Aegis before policy evaluation. Critical threats can override the policy decision to deny, regardless of what the policy rules say.

## Rule Counts Summary

| Detector | Rules |
|----------|-------|
| Prompt injection | 19 |
| Memory poisoning (MINJA) | 21 |
| Code safety | 18 |
| Output injection | 14 |
| Exfiltration | 13 |
| Credentials | 12 |
| PII | 9 |
| **Total** | **106** |

## Related

- [Sentinel](sentinel.md) -- real-time behavioral monitoring
- [MCP Gateway](mcp-gateway.md) -- transparent MCP proxy with policy enforcement
- [Ecosystem Overview](ecosystem-overview.md) -- how Aegis fits in the stack
