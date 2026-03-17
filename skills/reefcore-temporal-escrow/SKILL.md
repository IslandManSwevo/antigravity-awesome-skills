---
name: reefcore-temporal-escrow
description: >
  Use this skill for all Temporal workflow and activity code in ReefCore, with focus on
  the escrow state machine, sweep workflows, and durable financial operations. Trigger
  on any mention of: Temporal, workflow, activity, escrow, HELD, RELEASED, PENDING_EXTERNAL,
  sleepUntil, sweep, releaseFunds, Documenso webhook, or any durable/stateful financial
  operation. Also use for BullMQ async job patterns where relevant.
---

# ReefCore Temporal & Escrow Skill

## Architecture Principle

Temporal handles all durable, stateful financial workflows. BullMQ handles fire-and-forget async jobs (notifications, emails, non-critical background tasks). **Never** use BullMQ for money movement.

Temporal worker runs as a **separate Railway service** from the API gateway. Cascade failures in the API must not affect in-flight escrow workflows.

---

## Escrow State Machine

```
HELD → PENDING_EXTERNAL → RELEASED
            ↓
       FAILED_EXTERNAL
```

| State             | Meaning                                                              |
|------------------|----------------------------------------------------------------------|
| `HELD`           | Funds captured, escrow active, awaiting release signal              |
| `PENDING_EXTERNAL`| Release signal received, external transfer initiated                |
| `RELEASED`        | External transfer confirmed successful                              |
| `FAILED_EXTERNAL` | External transfer failed — requires manual intervention             |

**There is no automatic retry from `FAILED_EXTERNAL`.** Alert and page. Do not silently retry.

---

## Three Release Paths

### Path 1: Documenso Webhook (Priority)
The document signature triggers escrow release. This is the normal flow.

```typescript
// Workflow receives external signal
await workflow.setHandler(documensoSignalChannel, async (payload) => {
  if (payload.documentId === escrowRecord.documentId && payload.status === 'SIGNED') {
    await releaseFunds(escrowId);
  }
});
```

### Path 2: Manual API Signal via Fortress API
Operator manually releases via internal control plane. Requires IP allowlisting + auth.

```typescript
// Fortress API endpoint sends signal to running workflow
await temporalClient.signal(workflowId, 'manual_release', { actorId, reason });
```

### Path 3: Time-based sleepUntil
Escrow auto-releases at a defined deadline if no other signal received.

```typescript
await workflow.sleep(escrowDeadline); // Temporal's durable sleep
// Workflow resumes here after deadline
await releaseFunds(escrowId);
```

---

## releaseFunds Activity

The `releaseFunds` activity is the critical path. It must follow this exact sequence:

```typescript
async function releaseFunds(escrowId: string): Promise<void> {
  // 1. Claim idempotency key — abort if already claimed
  const key = `escrow_release_${escrowId}`;
  const claimed = await claimIdempotencyKey(key);
  if (!claimed) {
    console.log(`[Escrow] Duplicate release signal for ${escrowId} — skipping.`);
    return;
  }

  // 2. Write PENDING_EXTERNAL BEFORE calling external API
  await db.escrow.update({ where: { id: escrowId }, data: { status: 'PENDING_EXTERNAL' } });

  try {
    // 3. Call the CBOB/Sand Dollar hook (or other gateway)
    await cbobApiService.transferFunds({
      fromWallet: escrowRecord.holdWalletId,
      toWallet: escrowRecord.merchantWalletId,
      amount: escrowRecord.amount,
      reference: `escrow_release_${escrowId}`,
    });

    // 4. Mark RELEASED only after confirmation
    await db.escrow.update({ where: { id: escrowId }, data: { status: 'RELEASED' } });
    await appendAuditLog({ eventType: 'ESCROW_RELEASED', escrowId });

  } catch (err) {
    // 5. On failure: mark FAILED_EXTERNAL, do NOT retry, alert
    await db.escrow.update({ where: { id: escrowId }, data: { status: 'FAILED_EXTERNAL' } });
    await appendAuditLog({ eventType: 'ESCROW_RELEASE_FAILED', escrowId, error: err.message });
    throw new ApplicationFailure(err.message, 'ESCROW_RELEASE_FAILED', true); // non-retryable
  }
}
```

**The CBOB hook gap**: As of the current implementation, `releaseFunds` calls a stub. When CBOB AFI status is active, wire `cbobApiService.transferFunds()` here. The activity structure is already in place.

---

## Temporal Activity Design Rules

### Activities must be idempotent
Temporal may replay activities. Every activity must be safe to run twice with the same inputs.

```typescript
// ✅ Idempotent: check before create
const existing = await db.escrow.findUnique({ where: { id: escrowId } });
if (existing?.status === 'RELEASED') return; // already done
```

### Use ApplicationFailure for non-retryable errors
```typescript
import { ApplicationFailure } from '@temporalio/workflow';

// Non-retryable: business logic failure (REJECTED, duplicate, invalid state)
throw new ApplicationFailure('Escrow already released', 'ALREADY_RELEASED', true);

// Retryable: transient infrastructure failure (leave as regular Error)
throw new Error('Database connection timeout');
```

**Explicit rule**: A `REJECTED` status from a gateway must throw as non-retryable. Do not retry a rejected payment.

### Never put DB calls directly in workflow code
DB calls go in activities. Workflows must be deterministic. Non-deterministic operations (DB, HTTP, filesystem, Date.now()) must be activities.

```typescript
// ❌ Never in workflow code
const escrow = await db.escrow.findUnique({ where: { id } }); // side effect in workflow

// ✅ Wrap in activity
const escrow = await workflow.executeActivity(getEscrowActivity, { escrowId });
```

---

## Sweep Workflow Pattern

The sweep engine moves settled funds to merchant accounts on a schedule.

```typescript
// Temporal cron workflow — double-verify before every sweep
export async function sweepSettledFunds(): Promise<void> {
  // Step 1: fetch settled transactions not yet swept
  const settled = await workflow.executeActivity(fetchSettledForSweep, {});

  for (const txn of settled) {
    // Step 2: per-merchant pools — NEVER comingle funds across tenants
    const key = `vat_sweep_${txn.id}_${today}`;
    const claimed = await workflow.executeActivity(claimSweepKey, { key });
    if (!claimed) continue; // already processing

    // Step 3: execute sweep
    await workflow.executeActivity(executeMerchantSweep, { txnId: txn.id });
  }
}
```

**Per-merchant pools are mandatory.** Commingling funds across tenants is a regulatory violation. Never aggregate across `tenantId` in sweep logic.

---

## GovTech: Playwright Automation Workflows

Government portal automation (NIB, VAT, Business Licence) uses Playwright wrapped in Temporal activities. Key rules:

- Each portal session is one Temporal activity with a long `scheduleToCloseTimeout`.
- On portal error or session timeout, throw a retryable Error (Temporal will retry with backoff).
- On logic failure (invalid data, already filed), throw `ApplicationFailure` non-retryable.
- Screenshot on failure, store in audit bucket (no PII in filename).
- Never store portal credentials in the activity. Fetch from Doppler at activity start.

```typescript
async function submitNibFilingActivity(payload: NibPayload): Promise<void> {
  const credentials = await doppler.getSecret('NIB_PORTAL_CREDENTIALS');
  const browser = await chromium.launch({ headless: true });
  // ... Playwright automation
}
```

---

## Checklist: New Temporal Workflow

- [ ] All non-deterministic operations are in activities (no direct DB/HTTP in workflow)
- [ ] Every activity is idempotent (safe to replay)
- [ ] `FAILED_EXTERNAL` / non-retryable states use `ApplicationFailure(msg, type, true)`
- [ ] Idempotency key claimed before any money movement
- [ ] `PENDING_EXTERNAL` written to DB before external API call
- [ ] Audit log entry written on every state transition
- [ ] Per-merchant funds isolation enforced (no cross-tenant aggregation)
- [ ] Temporal worker deployed as separate Railway service from API gateway
