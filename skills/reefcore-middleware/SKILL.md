---
name: reefcore-middleware
description: >
  Use this skill whenever adding, modifying, or debugging any Express middleware in
  ReefCore's API gateway. Covers the mandatory execution order, transactionContext
  object shape, tenant scoping rules, VAT middleware, exemption middleware, and
  auth middleware. Trigger on any mention of: middleware, Express route, req.transactionContext,
  tenantScopingMiddleware, exemptionMiddleware, vatMiddleware, or injectTransactionContext.
---

# ReefCore Middleware Chain Skill

## Mandatory Execution Order

This order is **non-negotiable**. Changing it breaks security or financial correctness.

```
authenticateNextAuthJWT
  → tenantScopingMiddleware
    → injectTransactionContext
      → exemptionMiddleware
        → vatMiddleware
          → route handler
```

### Why this order matters

1. **authenticateNextAuthJWT** — Must be first. All subsequent middleware trusts `req.user`. Never access `req.user` before auth runs.
2. **tenantScopingMiddleware** — Sets `req.tenantId` from the authenticated user. Never use `merchantId` from the request body to scope DB queries. Only use `req.tenantId`.
3. **injectTransactionContext** — Bootstraps `req.transactionContext` from the validated request body. This is the only place raw `req.body` values are trusted and sanitized.
4. **exemptionMiddleware** — Must run **before** vatMiddleware. Sets `gbpaExempt` and `vatSkipped` on context. VAT middleware is intentionally dumb — it trusts these flags blindly.
5. **vatMiddleware** — Reads context, calculates VAT only if `vatSkipped !== true`. Never re-checks exemption logic here.

---

## The transactionContext Object

`req.transactionContext` is the central data bus. Shape after all middleware runs:

```typescript
interface TransactionContext {
  amount: number;              // Validated finite number. Set by injectTransactionContext.
  merchantId?: string;         // Always from req.user.clientId — NEVER from req.body directly.
  merchantCategory?: string;   // Lowercased, trimmed. Used by exemptionMiddleware.
  location?: string;           // Lowercased. 'freeport' or 'grand bahama' triggers HCA check.
  description?: string;
  reference?: string;

  // Set by exemptionMiddleware
  gbpaExempt?: boolean;
  vatSkipped?: boolean;
  exemptionReason?: 'GBPA_HAWKSBILL_CREEK' | null;
  vatApplied?: boolean;

  // Set by vatMiddleware (only if !vatSkipped)
  vatAmount?: number;          // Derived independently from amount. Never chained.
  totalWithVat?: number;       // Derived independently from amount. Never chained.
}
```

### Critical: merchantId authority
```typescript
// ✅ Correct: auth is authoritative
const merchantId = req.body.merchantId ?? req.user?.clientId;
// But in downstream middleware and handlers — ALWAYS use req.tenantId or req.user.clientId.

// ❌ Never: trust body-supplied merchantId for DB scoping
const { merchantId } = req.body; // attacker can impersonate another tenant
```

---

## injectTransactionContext Rules

- Only bootstraps if `req.body.amount` or `req.body.merchantCategory` is present.
- Validates `amount` is a finite number — returns `400` if not.
- Whitelists allowed context keys: `['merchantId', 'merchantCategory', 'location', 'description', 'reference']`.
- Never assign arbitrary `req.body` keys to context — prototype pollution risk.

```typescript
// ✅ Key whitelist pattern
const allowedContextKeys = ['merchantId', 'merchantCategory', 'location', 'description', 'reference'];
const safeContext: Record<string, any> = {};
for (const key of allowedContextKeys) {
  if (Object.prototype.hasOwnProperty.call(req.body.context, key)) {
    safeContext[key] = req.body.context[key];
  }
}
```

---

## exemptionMiddleware Rules

- Checks `merchantCategory` against `GBPA_EXEMPT_CATEGORIES` (see `hca_categories.ts`).
- **Requires both**: category match AND location containing `'freeport'` or `'grand bahama'`.
- Sets `vatAmount: 0` and `totalWithVat: ctx.amount` if exempt. Sets them on context atomically.
- Never mutates `ctx` piecemeal — spread the full update in one assignment.

```typescript
// ✅ Atomic assignment
req.transactionContext = {
  ...ctx,
  gbpaExempt: isExempt,
  vatSkipped: isExempt,
  exemptionReason: isExempt ? 'GBPA_HAWKSBILL_CREEK' : null,
  ...(isExempt ? { vatAmount: 0, totalWithVat: ctx.amount ?? 0, vatApplied: false } : {})
};

// ❌ Never mutate piecemeal
ctx.gbpaExempt = isExempt;
ctx.vatSkipped = isExempt; // race-prone partial state
```

---

## vatMiddleware Rules

- Check `ctx.vatSkipped === true` at the top. If true, call `next()` immediately.
- Derive `vatAmount` and `totalWithVat` **independently** from `ctx.amount`. Never chain.
- Use `parseFloat(X.toFixed(2))` for BSD monetary rounding.

```typescript
// ✅ Independent derivation
const vatAmount = parseFloat((ctx.amount * VAT_RATE).toFixed(2));
const totalWithVat = parseFloat((ctx.amount + vatAmount).toFixed(2));

// ❌ Never chain
const vatAmount = parseFloat((ctx.amount * VAT_RATE).toFixed(2));
const totalWithVat = parseFloat((vatAmount / VAT_RATE + vatAmount).toFixed(2)); // wrong
```

---

## Adding New Middleware

When adding new middleware to the chain:

1. Determine where in the order it belongs based on what it reads vs. writes on `req`.
2. Never read from `req.transactionContext` before `injectTransactionContext` has run.
3. Never scope DB queries by `req.body` fields — always use `req.tenantId`.
4. Write a unit test that verifies the middleware calls `next()` on the happy path AND returns the correct error code on each failure path.
5. Document which `req` properties it reads and which it writes.
