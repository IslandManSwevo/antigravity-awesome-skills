---
name: reefcore-security
description: >
  Use this skill for all security-sensitive code in ReefCore: HMAC webhook verification,
  idempotency key management, audit logging, secrets handling, rate limiting, IP allowlisting,
  and double-spend prevention. Trigger on any mention of: HMAC, signature, idempotency,
  audit log, secret, Doppler, double-spend, timingSafeEqual, gatewayTxnId, webhook,
  claimIdempotencyKey, or any code that touches payment processing atomicity.
---

# ReefCore Security & Idempotency Skill

## Non-Negotiable Security Rules

These rules are never negotiated, never skipped for convenience, never "we'll add it later."

1. **No secrets in `.env` files or repos.** All secrets via Doppler. No exceptions.
2. **No autonomous AI financial decisions.** AI can suggest; humans or deterministic code approve.
3. **Audit logs are append-only.** Never update or delete an audit record. Write a new one.
4. **Double-verify before every sweep.** Check idempotency AND transaction state before any money movement.
5. **PII in a separate database** from transaction data. Never join them in application code.

---

## HMAC Webhook Verification

### Always use timingSafeEqual
```typescript
import { timingSafeEqual, createHmac } from 'crypto';

export function verifyWebhookSignature(
  rawBody: string,
  receivedSig: string,
  secret: string
): void {
  const expected = createHmac('sha256', secret).update(rawBody, 'utf8').digest('hex');
  const a = Buffer.from(expected, 'utf8');
  const b = Buffer.from(receivedSig, 'utf8');

  if (a.length !== b.length || !timingSafeEqual(a, b)) {
    throw new Error('WEBHOOK_SIGNATURE_INVALID');
  }
}
```

**Why**: String equality `===` is vulnerable to timing attacks. An attacker can measure response time to guess the signature one byte at a time.

### Capture raw body before parsing
```typescript
// Route-level middleware — must come before express.json()
router.post('/webhook/paypal',
  express.raw({ type: 'application/json' }), // ← captures rawBody as Buffer
  async (req, res) => {
    const rawBody = req.body.toString('utf8');
    await paypalAdapter.parseWebhook(rawBody, req.headers);
  }
);
```

---

## Idempotency Key Pattern

### The fundamental rule
Before any money movement, **claim** the idempotency key atomically. The claim must:
1. Check `universalTransaction` table for existing `gatewayTxnId`
2. Check `idempotencyKey` table for an in-flight lock
3. Create the lock in a `Serializable` transaction

```typescript
// ✅ Correct: atomic claim with Serializable isolation
const claimed = await claimIdempotencyKey(gatewayTxnId);
if (!claimed) {
  return res.status(200).json({ ok: true, duplicate: true, message: 'Already processed.' });
}
```

### Key naming conventions
Prefix keys by domain to avoid collisions:

| Domain              | Key format                            |
|--------------------|---------------------------------------|
| Webhook events      | `{gatewayTxnId}` (raw gateway ID)     |
| Escrow releases     | `escrow_release_{escrowId}`           |
| VAT sweeps          | `vat_sweep_{txnId}_{date}`            |
| Refunds             | `refund_{gatewayTxnId}`               |

### Release on failure
```typescript
// ✅ Release the key if the handler returns a 4xx/5xx
res.on('finish', async () => {
  if (res.statusCode >= 400) {
    await releaseIdempotencyKey(gatewayTxnId);
  }
});
```

### Escrow-specific idempotency
Write `PENDING_EXTERNAL` to the DB **before** calling the external API. Never call the API and then write to DB.

```typescript
// ✅ Correct order for escrow release
await db.escrow.update({ where: { id }, data: { status: 'PENDING_EXTERNAL' } });
const result = await externalApi.release(escrowId); // if this fails, status is already logged
await db.escrow.update({ where: { id }, data: { status: result.success ? 'RELEASED' : 'FAILED_EXTERNAL' } });
```

---

## Double-Spend Prevention

The double-spend test is the **most critical test** in the system. It must exist before any go-live.

```typescript
// What to test: concurrent requests with the same gatewayTxnId
it('prevents double-spend on concurrent webhook delivery', async () => {
  const [r1, r2] = await Promise.all([
    processWebhook(gatewayTxnId),
    processWebhook(gatewayTxnId), // same ID, concurrent
  ]);

  const processed = [r1, r2].filter(r => r.processed);
  const duplicates = [r1, r2].filter(r => r.duplicate);

  expect(processed).toHaveLength(1);
  expect(duplicates).toHaveLength(1);
});
```

---

## Audit Logging

Every financial event must produce an audit log entry. Append-only, never updated.

```typescript
// Minimum required fields
interface AuditEntry {
  id: string;           // uuid
  tenantId: string;
  eventType: string;    // 'PAYMENT_RECEIVED' | 'ESCROW_RELEASED' | 'REFUND_ISSUED' | etc.
  gatewayTxnId?: string;
  amount?: number;
  currency?: 'USD' | 'BSD';
  actorId?: string;     // userId or 'SYSTEM'
  createdAt: Date;      // set by DB default, never by application
  metadata?: Record<string, unknown>; // JSON blob — no PII
}
```

Do not log: card numbers, full wallet addresses, names, emails, or any PII in the audit table.

---

## Secrets Management

- All secrets managed by Doppler. Config in `doppler.yaml`.
- Local dev: `doppler run -- node server.js` (never copy secrets to `.env`).
- Never `console.log` a secret, API key, or token. Redact in error messages.
- Rotate HMAC secrets by deploying a new Doppler secret and updating all webhook registrations.

---

## Rate Limiting

`rateLimitMiddleware.ts` is applied at the gateway level. Key rules:

- Webhook endpoints get their own rate limit tier (higher volume expected).
- Operator dashboard APIs get per-tenant rate limits.
- Public booking endpoints get IP-based rate limits.
- On `429`, return `Retry-After` header.

---

## IP Allowlisting (`ipAllowlist.ts`)

- Used for Fortress API endpoints (internal escrow control plane).
- Allowlist maintained in Doppler, not hardcoded.
- Log every blocked request with IP and endpoint — these are potential probes.

---

## CI Enforcement

These checks must pass in CI before any PR merges:

- [ ] Coverage thresholds enforced on `adapters/` and `middleware/` directories
- [ ] No `console.log` in production code paths (ESLint rule)
- [ ] No hardcoded secrets (secret-scan in CI pipeline)
- [ ] `timingSafeEqual` used in all HMAC comparisons (grep check)
- [ ] `gatewayTxnId` idempotency check present in all webhook handlers
