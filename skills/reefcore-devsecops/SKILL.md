---
name: reefcore-devsecops
description: >
  Use this skill for ALL security-related code, configuration, infrastructure decisions,
  and architectural choices in ReefCore. This is the authoritative security reference for
  passing a professional penetration test and third-party security audit (required for
  CBOB AFI status). Trigger on any mention of: security, pen test, audit, vulnerability,
  CVE, WAF, headers, Helmet, CORS, SQL injection, XSS, CSRF, authentication, authorization,
  JWT, Clerk, mTLS, Vault, KMS, DEK, KEK, secrets, Doppler, rate limit, IP allowlist,
  logging, SIEM, Cloudflare, Railway hardening, dependency scanning, Snyk, Docker,
  trust proxy, PostgreSQL permissions, row-level security, or anything touching
  the security posture of the platform.
---

# ReefCore DevSecOps & Pen Test Readiness

## Audit Scope Summary

ReefCore processes real BSD/USD transactions via CBOB-adjacent rails, stores encrypted
merchant API credentials with envelope encryption (DEK/KEK via HashiCorp Vault), and
exposes multi-tenant financial APIs. The security audit covers:

- API gateway (Express/Node.js on Railway)
- Temporal worker (separate Railway service)
- Next.js frontend (Vercel)
- PostgreSQL (Railway managed)
- HashiCorp Vault (Transit engine / KMS)
- Documenso (Docker, per-tenant Railway)
- Webhook endpoints (PayPal, CIBC, Cash N Go, Documenso)
- Escrow control plane (Fortress API)

---

## OWASP Top 10 — ReefCore Threat Map

| OWASP Category                    | ReefCore Attack Surface                                     | Status Check |
|----------------------------------|-------------------------------------------------------------|--------------|
| A01 Broken Access Control        | Tenant isolation — `clientId` from auth vs body             | See §AUTH    |
| A02 Cryptographic Failures       | DEK/KEK envelope encryption, AES-256-GCM, key rotation      | See §CRYPTO  |
| A03 Injection                    | Prisma parameterized queries, Zod input validation          | See §INPUT   |
| A04 Insecure Design              | Escrow PENDING_EXTERNAL before external call, idempotency   | See §DESIGN  |
| A05 Security Misconfiguration    | Helmet headers, CORS, Railway env exposure, Vault dev mode  | See §CONFIG  |
| A06 Vulnerable Components        | glob CVE in package-lock.json — IMMEDIATE ACTION REQUIRED   | See §DEPS    |
| A07 Auth Failures                | Clerk + MFA enforcement, JWT validation, token expiry       | See §AUTH    |
| A08 Software Integrity Failures  | CI dependency pinning, Dockerfile base image hashing        | See §CICD    |
| A09 Logging Failures             | Audit log append-only, PII excluded, SIEM forwarding        | See §LOGGING |
| A10 SSRF                         | External API calls from Temporal activities — validate URLs | See §SSRF    |

---

## §AUTH — Authentication & Authorization

### Clerk Configuration (Mandatory MFA)

- MFA **must** be enforced at the Clerk organization level — not optional per user.
- Use Clerk's `SessionClaims` to embed `clientId`/`tenantId` into the JWT. Never rely on body-supplied IDs for authorization decisions.
- JWT verification must happen on every request via `authenticateNextAuthJWT` middleware. No "trusted internal" paths that skip auth.

```typescript
// ✅ Always derive tenant identity from verified token
const tenantId = req.user?.clientId; // from JWT claims

// ❌ Never use body-supplied identity for authorization
const tenantId = req.body.merchantId; // attacker-controlled
```

### JWT Hardening

- Verify `aud`, `iss`, and `exp` on every token.
- Use short-lived access tokens (15 min max). Refresh via Clerk's session management.
- On `401`, never reveal whether the token is expired vs. invalid — return identical `401 Unauthorized` for both.

### Tenant Isolation (IDOR Prevention)

Every DB query involving merchant/tenant data must scope by `tenantId` from auth:

```typescript
// ✅ Correct
await prisma.escrowRecord.findMany({
  where: { tenantId: req.user.clientId } // from JWT — cannot be spoofed
});

// ❌ IDOR vulnerability — pen testers will find this
await prisma.escrowRecord.findMany({
  where: { tenantId: req.body.merchantId } // attacker changes this to another tenant's ID
});
```

### Fortress API (Escrow Control Plane)

- IP allowlist enforced via `ipAllowlist.ts` — allowlist stored in Doppler, not code.
- Ensure `app.set('trust proxy', 1)` is set in `app.ts` for Railway's proxy layer. Without this, `req.ip` returns the proxy IP, making allowlisting useless.
- Fortress API endpoints must additionally require a separate API key (not the same JWT used by the frontend).

---

## §CRYPTO — Cryptography & Key Management

### Envelope Encryption (DEK/KEK) — Current Implementation

The `security.js` DEK/KEK pattern is correct for production. Verify these properties:

- **AES-256-GCM** with 12-byte IV (NIST SP 800-38D compliant) ✓
- **Per-operation DEK** — fresh `randomBytes(32)` every encrypt call ✓
- **DEK wiped after use** — `dek.fill(0)` before any throw or return ✓
- **Auth tag verified** on decrypt — GCM provides authenticated encryption ✓
- **Vault Transit engine** wraps DEK — Vault unavailable = fail closed ✓

### Vault Production Hardening (Pre-Audit Checklist)

The `docker-compose.vault.yml` uses dev mode with `root-token`. This is **prohibited in production**.

```
Pre-production Vault checklist:
[ ] Initialize Vault with Shamir's Secret Sharing (5 key shares, 3 threshold)
[ ] Unseal keys stored in separate physical locations — never in Doppler
[ ] Disable Vault's root token after initial setup
[ ] Create a dedicated AppRole for ReefCore API with limited policy:
      path "transit/encrypt/reefcore-kek" { capabilities = ["update"] }
      path "transit/decrypt/reefcore-kek" { capabilities = ["update"] }
[ ] Enable Vault audit log — forward to SIEM
[ ] Enable Vault TLS — never run Vault over HTTP in production
[ ] Set KEK rotation policy: 90-day automatic rotation
[ ] Vault HA mode if processing > BSD $10k/day
```

### Key Rotation

- When rotating the KEK, decrypt all existing DEKs with the old KEK and re-wrap with the new.
- The `last_rotated_at` column in `security_vault` tracks this — ensure it's updated.
- Key rotation must produce an audit log entry: `KMS_KEY_ROTATION`.

### What NOT to do

```typescript
// ❌ Never store decrypted keys longer than the API call duration
this.apiKey = await decrypt(encryptedKey); // stored on instance — memory exposure

// ❌ Never log key material
console.log('Using API key:', apiKey);

// ❌ Never use ECB mode, MD5 for any cryptographic purpose, or SHA-1 for signatures
```

---

## §INPUT — Input Validation & Injection Prevention

### Zod Schema on Every Inbound Route

Every route handler must validate input with a Zod schema before any business logic:

```typescript
import { z } from 'zod';

const CreateCheckoutSchema = z.object({
  amount: z.number().positive().finite().max(999999.99),
  currency: z.enum(['USD', 'BSD']),
  bookingRef: z.string().uuid(),
  merchantCategory: z.string().max(64).optional(),
});

// ✅ Validate at route entry — before any middleware processes it
const parsed = CreateCheckoutSchema.safeParse(req.body);
if (!parsed.success) {
  return res.status(400).json({ error: 'Invalid request', details: parsed.error.flatten() });
}
```

### Prisma Parameterized Queries

Prisma uses parameterized queries by default. **Never use `$queryRaw` with string interpolation:**

```typescript
// ❌ SQL injection vector
await prisma.$queryRaw`SELECT * FROM merchants WHERE id = '${req.body.id}'`;

// ✅ Safe parameterized
await prisma.$queryRaw`SELECT * FROM merchants WHERE id = ${req.body.id}::uuid`;
// Even better: use Prisma model methods
await prisma.merchant.findUnique({ where: { id: req.body.id } });
```

### NoSQL / JSON Injection in JSONB Columns

`conditions JSONB` in `escrow_records` is user-controlled. Validate with Zod before storing:

```typescript
const EscrowConditionsSchema = z.object({
  releaseOn: z.enum(['SIGNATURE', 'MANUAL', 'TIMEOUT']),
  documentId: z.string().uuid().optional(),
  timeoutAt: z.string().datetime().optional(),
}).strict(); // .strict() rejects unexpected keys
```

### Path Traversal

If any endpoint accepts file paths or tenant-specific asset paths (marina templates), validate against an allowlist of known paths. Never construct filesystem paths from user input.

---

## §CONFIG — Security Configuration

### Express Hardening (`app.ts`)

```typescript
import helmet from 'helmet';
import cors from 'cors';

// Trust Railway's proxy — required for req.ip to work correctly
app.set('trust proxy', 1);

// Helmet: sets 14 security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],  // no inline scripts
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
  hsts: {
    maxAge: 31536000,       // 1 year
    includeSubDomains: true,
    preload: true,
  },
}));

// CORS: explicit allowlist — never use wildcard '*' on financial APIs
const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') ?? [];
app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('CORS_BLOCKED'));
    }
  },
  credentials: true,
}));

// Never expose stack traces in production
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error('[ERROR]', err.message); // log internally
  res.status(500).json({ error: 'Internal server error' }); // never leak err.message
});
```

### Required HTTP Headers (Pen Test Will Check)

A pen tester will run the request through a header scanner. All of these must be present:

| Header                           | Value                                      |
|---------------------------------|--------------------------------------------|
| `Strict-Transport-Security`      | `max-age=31536000; includeSubDomains; preload` |
| `X-Content-Type-Options`         | `nosniff`                                  |
| `X-Frame-Options`                | `DENY`                                     |
| `Content-Security-Policy`        | Explicit `defaultSrc 'self'`               |
| `Referrer-Policy`                | `strict-origin-when-cross-origin`          |
| `Permissions-Policy`             | `geolocation=(), microphone=()`            |
| `X-Powered-By`                   | Must be **absent** (Helmet removes it)     |
| `Server`                         | Must be **absent** or generic              |

### CORS Policy for Webhook Endpoints

Webhook endpoints receive server-to-server POST requests. They must:

- Return `405` for `OPTIONS` preflight (no CORS headers needed — not browser-originated).
- Accept only `Content-Type: application/json` or `application/octet-stream`.

---

## §DEPS — Dependency Vulnerability Management

### IMMEDIATE ACTION: glob CVE

A pre-existing CVE on `glob` has been flagged in `package-lock.json`. This will be the first thing an automated pen test tool catches and will be a finding in the audit report.

```bash
# Immediate remediation steps:
npm audit                              # identify severity and path
npm audit fix                          # auto-fix where safe
npm audit fix --force                  # force — review breaking changes manually
npm ls glob                            # find which package pulls glob transitively
```

If `glob` is pulled in transitively and can't be upgraded, use `overrides` in `package.json`:

```json
{
  "overrides": {
    "glob": "^10.4.5"
  }
}
```

### CI Gate: Block Deployments on Critical CVEs

Add to CI pipeline (GitHub Actions example):

```yaml
- name: Security audit
  run: npm audit --audit-level=high
  # Fails build on HIGH or CRITICAL CVEs — MODERATE allowed until addressed
```

### Dependency Pinning

In production `Dockerfile`, pin Node.js base image by digest:

```dockerfile
# ❌ Mutable tag — supply chain risk
FROM node:20-alpine

# ✅ Pinned digest — immutable
FROM node:20-alpine@sha256:<digest>
```

### Snyk Integration

Add Snyk to CI for deeper analysis (covers license compliance for CBOB audit too):

```yaml
- uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

---

## §LOGGING — Audit Logging & Observability

### Append-Only Audit Tables

The `kms_audit_log` and `escrow_audit_logs` tables must be append-only at the database permission level — not just by convention.

```sql
-- Run as superuser during initial setup
-- Create a restricted app user that cannot UPDATE/DELETE audit tables
REVOKE UPDATE, DELETE ON kms_audit_log FROM reefcore_app_user;
REVOKE UPDATE, DELETE ON escrow_audit_logs FROM reefcore_app_user;
REVOKE UPDATE, DELETE ON financial_audit_log FROM reefcore_app_user;

-- Confirm with:
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'kms_audit_log';
```

### What Must Be Logged (Every Financial Event)

```
PAYMENT_RECEIVED     — gatewayTxnId, amount, currency, tenantId
PAYMENT_FAILED       — gatewayTxnId, errorCode, tenantId
ESCROW_CREATED       — escrowId, amount, tenantId, actorId
ESCROW_RELEASED      — escrowId, releaseStrategy, actorId
ESCROW_RELEASE_FAILED— escrowId, errorMessage
REFUND_ISSUED        — gatewayTxnId, amount, actorId
KMS_ENCRYPT          — merchantId, requestId (no plaintext)
KMS_DECRYPT          — merchantId, requestId (no plaintext)
KMS_KEY_ROTATION     — merchantId, timestamp
SWEEP_INITIATED      — sweepId, amount, merchantId
SWEEP_COMPLETED      — sweepId, cbobTxnHash
SWEEP_FAILED         — sweepId, errorMessage
ADMIN_ACTION         — actorId, action, targetId
```

### What Must NEVER Appear in Logs

```
Card numbers, CVVs, full account numbers
API keys or tokens (even partial — hash them if needed: sha256(key).slice(0,8))
Passwords or credential material
Full wallet addresses (truncate: first 6 + last 4 chars)
PII: names, emails, phone numbers, national ID numbers
Vault DEK or KEK material
JWT contents
```

### Structured Logging Format

Use JSON structured logging (not console.log strings) for SIEM ingestion:

```typescript
import { createLogger, format, transports } from 'winston';

const logger = createLogger({
  format: format.combine(format.timestamp(), format.json()),
  transports: [new transports.Console()],
});

// Every financial log entry
logger.info({
  event: 'PAYMENT_RECEIVED',
  tenantId,
  gatewayTxnId,
  amount,     // NOT the decrypted API key
  currency,
  requestId: req.headers['x-request-id'],
  timestamp: new Date().toISOString(),
});
```

---

## §SSRF — Server-Side Request Forgery

ReefCore makes outbound HTTP calls from Temporal activities to CBOB, CIBC, PayPal, Cash N Go. An SSRF vulnerability here could allow an attacker to redirect these calls to internal Railway metadata endpoints.

```typescript
// Validate all externally-influenced URLs before calling
function isSafeOutboundUrl(url: string): boolean {
  try {
    const parsed = new URL(url);
    // Block internal/cloud metadata endpoints
    const blocked = ['169.254.169.254', '::1', 'localhost', '127.0.0.1', 'metadata.google.internal'];
    if (blocked.some(h => parsed.hostname === h)) return false;
    // Only allow HTTPS
    if (parsed.protocol !== 'https:') return false;
    // Allowlist known gateway domains
    const allowedHosts = ['api.paypal.com', 'sandbox.paypal.com', /* CIBC, CBOB endpoints */];
    return allowedHosts.includes(parsed.hostname);
  } catch {
    return false;
  }
}
```

---

## §DESIGN — Secure Design Patterns

### Fail Closed by Default

```typescript
// ✅ Fail closed: if Vault is down, encryption fails hard
try {
  wrappedDek = await wrapDek(dek.toString('base64'));
} catch (err) {
  dek.fill(0);
  throw new Error('[SECURITY] Vault unreachable — refusing to encrypt.');
}

// ❌ Fail open: fall back to plaintext if encrypted path fails (never do this)
try {
  wrappedDek = await wrapDek(dek.toString('base64'));
} catch {
  wrappedDek = dek.toString('base64'); // storing plaintext DEK — catastrophic
}
```

### Error Response Standardization

Never expose implementation details in error responses. Pen testers enumerate error messages.

```typescript
// ❌ Leaks stack trace, file paths, internal state
res.status(500).json({ error: err.stack });
res.status(400).json({ error: `Prisma error: ${err.message}` });

// ✅ Generic message + internal logging
logger.error({ event: 'UNHANDLED_ERROR', message: err.message, stack: err.stack });
res.status(500).json({ error: 'Internal server error', requestId });
```

### Rate Limiting Improvements

The current `rateLimitMiddleware.ts` uses an in-memory Map. This resets on restart and doesn't work across multiple Railway replicas.

For production (and for the audit), migrate to Redis-backed rate limiting:

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

export const apiRateLimit = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redisClient.sendCommand(args) }),
  keyGenerator: (req) => req.user?.clientId ?? req.ip,
});

// Stricter limits for auth and financial endpoints
export const paymentRateLimit = rateLimit({ windowMs: 60000, max: 20, store: redisStore });
export const webhookRateLimit = rateLimit({ windowMs: 60000, max: 500, store: redisStore });
```

---

## §CICD — CI/CD Security Gates

Every PR must pass ALL of these before merge. Add as required GitHub Actions status checks:

```yaml
security-gates:
  steps:
    - name: Dependency audit
      run: npm audit --audit-level=high

    - name: Secret scan
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: ${{ github.event.repository.default_branch }}

    - name: SAST — static analysis
      uses: github/codeql-action/analyze@v3
      with:
        languages: typescript, javascript

    - name: HMAC safety check
      run: |
        # Fail if timingSafeEqual is missing from any HMAC comparison
        grep -rn "=== " --include="*.ts" src/adapters/ src/middleware/ | grep -i "hmac\|sig\|hash" && echo "TIMING ORACLE RISK" && exit 1 || true

    - name: No secrets in code
      run: |
        grep -rn "VAULT_TOKEN\|API_KEY\|SECRET" --include="*.ts" src/ | grep -v "process.env\|Doppler\|// " && exit 1 || true

    - name: Coverage threshold enforcement
      run: npm test -- --coverage --coverageThreshold='{"./src/adapters/":{"lines":80},"./src/middleware/":{"lines":85}}'

    - name: License compliance
      run: npx license-checker --onlyAllow "MIT;ISC;Apache-2.0;BSD-2-Clause;BSD-3-Clause"
```

---

## §DB — PostgreSQL Security Hardening

### Least Privilege DB Users

```sql
-- App user: cannot touch audit tables destructively
CREATE USER reefcore_app WITH PASSWORD '<from-doppler>';
GRANT SELECT, INSERT, UPDATE ON merchants, sweep_configs, sweep_logs, escrow_records TO reefcore_app;
GRANT SELECT, INSERT ON kms_audit_log, escrow_audit_logs, financial_audit_log TO reefcore_app;
-- Note: no UPDATE or DELETE on audit tables

-- Read-only reporting user (for future analytics/GovTech)
CREATE USER reefcore_reader WITH PASSWORD '<from-doppler>';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reefcore_reader;

-- Migrations user: separate, used only in CI/CD
CREATE USER reefcore_migrate WITH PASSWORD '<from-doppler>';
GRANT ALL ON SCHEMA public TO reefcore_migrate;
```

### Row-Level Security (Multi-Tenant Isolation)

Enable RLS on all tenant-scoped tables as a defense-in-depth layer:

```sql
ALTER TABLE escrow_records ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON escrow_records
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Set in Prisma middleware before each query
await prisma.$executeRaw`SELECT set_config('app.current_tenant_id', ${tenantId}, true)`;
```

### Connection Security

- All Railway PostgreSQL connections use TLS. Verify with `?sslmode=require` in `DATABASE_URL`.
- Connection pool max: set to Railway's limit — do not leave at Prisma default (10) without checking.
- Enable `pgaudit` extension on Railway for SQL-level audit logging (send to SIEM).

---

## §NETWORK — Infrastructure & Network Security

### Cloudflare WAF (Required Pre-Audit)

Place all public endpoints behind Cloudflare with these rules:

```
WAF Rules:
- Block all non-HTTPS traffic (enforce at Cloudflare level)
- Geo-restrict: allow only BS (Bahamas), US, CA, GB — adjust for marina clients
- Block known bot signatures and scanner user-agents
- Rate limit: 100 req/min per IP on /api/* before hitting Railway
- Challenge (CAPTCHA) on /api/auth/* after 5 failures from same IP
- Block path traversal patterns: /../, /etc/, /proc/
- Enable OWASP Core Rule Set (CRS) in blocking mode
```

### Railway Service Isolation

- API gateway and Temporal worker must be in **separate Railway services** — not the same service with different start commands.
- Temporal worker should have **no public HTTP port**. It connects outbound to Temporal Cloud only.
- Vault should not be exposed publicly — access only from Railway private network (`railway.internal`).
- Set `RAILWAY_ENVIRONMENT=production` and use it to gate dev-only behaviors.

### TLS / mTLS

- All service-to-service communication within Railway private network still uses TLS.
- For CBOB/CIBC connections: confirm whether mTLS client certificates are required (likely yes for CBOB FPS). The `mtls_cert_reference` column in `security_vault` is the hook for this.
- Never disable TLS verification in production: `rejectUnauthorized: false` is a critical finding.

---

## Pre-Pen-Test Checklist

Run through this list the week before the audit engagement:

### Application Layer

- [ ] All OWASP headers present (verify with securityheaders.com)
- [ ] `X-Powered-By` header absent
- [ ] CORS policy explicit, no wildcard `*` on any authenticated endpoint
- [ ] No stack traces in API error responses
- [ ] Zod validation on every inbound route
- [ ] `timingSafeEqual` in all HMAC comparisons (grep check)
- [ ] No `console.log` with sensitive data (grep check)
- [ ] Rate limiting Redis-backed (not in-memory)
- [ ] `trust proxy` set correctly for Railway

### Authentication & Authorization

- [ ] Clerk MFA enforced at org level
- [ ] All DB queries scoped by `req.user.clientId` (not `req.body`)
- [ ] JWT expiry ≤ 15 minutes
- [ ] Fortress API uses separate credential from user JWT
- [ ] IP allowlist reads from Doppler, not hardcoded

### Cryptography & Secrets

- [ ] Vault in production mode (not dev mode with root-token)
- [ ] No `.env` files in repo (`.gitignore` + `trufflehog` scan clean)
- [ ] `npm audit` returns zero HIGH/CRITICAL
- [ ] glob CVE patched
- [ ] Docker base image pinned to digest

### Database

- [ ] `reefcore_app` user lacks UPDATE/DELETE on audit tables
- [ ] RLS enabled on `escrow_records` and `sweep_configs`
- [ ] `DATABASE_URL` has `sslmode=require`
- [ ] No raw `$queryRaw` with string interpolation

### Infrastructure

- [ ] Cloudflare WAF in blocking mode
- [ ] Temporal worker has no public HTTP port
- [ ] Vault not publicly accessible
- [ ] All internal service URLs use `railway.internal` private networking
- [ ] Prometheus metrics endpoint is NOT public

### Logging & Monitoring

- [ ] Audit tables are INSERT-only for app user (verified with PSQL `\dp`)
- [ ] No PII in any log line
- [ ] Sentry configured, not leaking stack traces to client
- [ ] Structured JSON logging (not ad-hoc console.log strings)

---

## Pen Test Engagement Notes

The pen test scope per the Risk Model covers:

1. **Sweep engine** — double-spend, race conditions, replay attacks on sweep triggers
2. **Payment rails** — HMAC bypass attempts, webhook replay, currency confusion
3. **Escrow release paths** — signal spoofing, state machine bypass, Documenso webhook forgery
4. **Admin/Fortress API** — IP allowlist bypass (test with X-Forwarded-For spoofing), IDOR

Provide testers with:

- A staging environment with real (non-production) Vault instance
- Separate test tenant credentials — never use production merchant data
- API documentation (use Scalar/Stoplight to generate from OpenAPI spec)
- Out-of-scope declaration: Clerk authentication infrastructure, Railway platform itself, PayPal/CIBC production systems
