---
name: reefcore-financial-arithmetic
description: >
  Use this skill for any code that calculates money, tax, VAT, fees, or BSD/USD
  amounts in ReefCore. Covers Bahamian VAT (10%), HCA/GBPA exemption logic,
  Decimal vs float rules, rounding standards, and the BahamianFinanceEngine API.
  Trigger on any mention of: VAT, BSD, amount, vatAmount, totalWithVat, HCA,
  GBPA, exemption, Decimal, parseFloat, toFixed, or any monetary calculation.
---

# ReefCore Financial Arithmetic Skill

## The Cardinal Rules

### 1. Never use floating-point arithmetic for money

JavaScript `number` has IEEE 754 precision issues. For any monetary calculation, use the `Decimal` library (or `decimal.js`).

```typescript
import Decimal from 'decimal.js';

// ✅ Correct
const vatAmount = new Decimal(baseAmount).mul(VAT_RATE).toDecimalPlaces(2);
const total = new Decimal(baseAmount).plus(vatAmount);

// ❌ Never
const vatAmount = baseAmount * 0.10; // floating point drift
```

**Exception**: `BahamianFinanceEngine` uses `parseFloat(X.toFixed(2))` as an acceptable approximation for small BSD amounts. New code should prefer `Decimal`.

### 2. Derive vatAmount and totalWithVat independently

Never compute one from the other. Both must be derived from the source `amount`.

```typescript
// ✅ Independent derivation from source
const vatAmount = new Decimal(amount).mul(VAT_RATE).toDecimalPlaces(2);
const totalWithVat = new Decimal(amount).plus(vatAmount).toDecimalPlaces(2);

// ❌ Chained derivation — error amplification
const vatAmount = new Decimal(amount).mul(VAT_RATE).toDecimalPlaces(2);
const totalWithVat = vatAmount.div(VAT_RATE).plus(vatAmount); // wrong
```

### 3. Always validate inputs before arithmetic

```typescript
if (!Number.isFinite(baseAmount) || baseAmount <= 0) {
  throw new Error('[BahamianFinanceEngine] Invalid baseAmount: must be a positive finite number.');
}
```

---

## Bahamian VAT

- **Standard rate: 10% (0.10)**
- Applied to all transactions unless exempted.
- VAT rate constant lives in `utils/tax.ts` as `BAHAMIAN_VAT_RATE`. Import from there — never hardcode `0.10`.
- BSD currency only. PayPal (USD) transactions are not subject to Bahamian VAT in the current gateway config.

```typescript
import { BAHAMIAN_VAT_RATE } from '../utils/tax.js';

const vatAmount = new Decimal(baseAmount).mul(BAHAMIAN_VAT_RATE).toDecimalPlaces(2);
```

---

## HCA / GBPA Exemption Logic

The Hawksbill Creek Agreement (1955) exempts qualifying Freeport/Grand Bahama merchants from VAT.

### Exempt categories (`GBPA_EXEMPT_CATEGORIES`)

```
marina, port_services, industrial_manufacturing, bonded_warehouse,
freight_logistics, ship_chandlery, industrial_supply, export_services
```

### Exemption requires BOTH conditions

1. `merchantCategory` is in the exempt set (lowercased, trimmed match)
2. `location` contains `'freeport'` OR `'grand bahama'` (case-insensitive)

```typescript
const isExempt = GBPA_EXEMPT_CATEGORIES.has(merchantCategory) && isGbpaZone;
```

### In BahamianFinanceEngine.calculateTotals

The engine takes `isGbpaBonded: boolean` and `isB2b: boolean`. Exemption applies only when **both are true** (B2B in GBPA zone). B2C marina transactions still pay VAT.

---

## BahamianFinanceEngine API

```typescript
BahamianFinanceEngine.calculateTotals(
  baseAmount: number,      // pre-VAT amount
  isGbpaBonded: boolean,   // merchant has GBPA bonded license
  isB2b: boolean           // is the transaction B2B?
): {
  baseAmount: number;
  vatAmount: number;       // 0 if exempt
  totalAmount: number;     // baseAmount + vatAmount
  appliedGbpaExemption: boolean;
}
```

Use this for all non-middleware financial calculations. Do not duplicate this logic elsewhere.

### Auto VAT Reserve (Sand Dollar only)

```typescript
await BahamianFinanceEngine.executeAutoVatReserve(vatAmount, clientTaxWalletId);
```

- Only call this on Sand Dollar transactions.
- Guard: `if (vatAmount <= 0) return false` is already inside the method.
- `clientTaxWalletId` must be non-empty string — validate before calling.
- Frame this feature as "automated VAT reserve sweep" — never "DeFi" or "AMM."

---

## Rounding Standards

| Context                    | Method                                        |
|---------------------------|-----------------------------------------------|
| All monetary output (BSD) | `toDecimalPlaces(2, Decimal.ROUND_HALF_EVEN)` |
| Legacy/Engine code        | `parseFloat(X.toFixed(2))`                    |
| Display to user           | `toFixed(2)` (string)                         |
| Database storage          | Store as integer cents OR `Decimal(20,2)` col |

Never store or transmit a monetary value with more than 2 decimal places.

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| `amount * 1.10` for total | Derive `vatAmount` and `total` independently |
| `0.10` literal for VAT rate | Import `BAHAMIAN_VAT_RATE` from `utils/tax.ts` |
| VAT on USD/PayPal transactions | Only apply VAT to BSD amounts |
| Skipping input validation | Always check `isFinite` and `> 0` before math |
| Mutating `ctx` partway through | Assign context atomically in one spread |
| Applying B2C exemption | Exemption requires `isGbpaBonded && isB2b` both true |
