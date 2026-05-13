# SaaS Image Generator — System Architecture

_Date: 2026-05-09_
_Author: Backend Architect_
_Status: Blueprint — ready for implementation_

---

## 1. System Architecture Overview

```
                                    +-----------------+
                                    |   Midtrans      |
                                    +--------+--------+
                                             |
                                             | webhooks (/api/v1/webhooks/midtrans)
                                             |
+----------+     +----------------+     +-----+-------+     +-----------------+
|          |     |                |     |             |     |                 |
|  React   +---->  FastAPI        +----->  Credit     +----->  kie.ai API     |
|  SPA     | HTTPS|  Backend      | HTTP |  Manager   | REST | (nano-banana)  |
|  (Vercel)| <----+  (VPS/Docker) | <----+             | <----+                 |
|          |     |                |     |             |     |                 |
+----+-----+     +-------+--------+     +------+------+     +-----------------+
     |                   |                     |
     |                   |                     |
     |            +------+-------+     +-------+-------+
     |            |              |     |               |
     |            |   SQLite /   |     |  Image        |
     |            |   PostgreSQL |     |  Storage      |
     |            |              |     |  (local/S3)   |
     |            +--------------+     +---------------+
     |
     |  +-----------+
     +-->  CDN       |
        |  (Vercel)  |
        +-----------+

Data Flow Legend:
  ---->  HTTPS request/response (synchronous)
  ---->  HTTP polling (async task pattern)
  ---->  Webhook callback (Midtrans -> Backend)
```

### Component Responsibilities

| Component | Role |
|---|---|
| **React SPA** | User-facing UI: signup, login, image editor, credit purchase, generation history |
| **FastAPI Backend** | API server: auth, credit management, generation orchestration, Midtrans integration |
| **SQLite/PostgreSQL** | Persistent storage: users, credits, generations, transactions |
| **kie.ai API** | External image generation service (async: POST task -> poll status -> download result) |
| **Midtrans** | Payment processing: Snap checkout, QRIS/GoPay/VA, webhook notifications |
| **Image Storage** | Generated images stored locally (MVP) or S3-compatible (production) |

---

## 2. API Design

Base URL: `/api/v1`

### 2.1 Authentication Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/auth/register` | Create new account | No |
| `POST` | `/auth/login` | Login, get tokens | No |
| `POST` | `/auth/refresh` | Refresh access token | Refresh token |
| `POST` | `/auth/logout` | Revoke refresh token | Yes |
| `GET` | `/auth/me` | Get current user profile | Yes |

#### POST /auth/register
```json
// Request
{
  "email": "user@example.com",
  "password": "SecureP@ss1",
  "name": "Jane Doe"           // optional
}

// Response 201
{
  "id": "usr_2xKj9mNp",
  "email": "user@example.com",
  "name": "Jane Doe",
  "credits": 20,               // 5 free gens x 4 credits
  "free_gens_remaining": 5,
  "created_at": "2026-05-09T10:00:00Z"
}
```

#### POST /auth/login
```json
// Request
{
  "email": "user@example.com",
  "password": "SecureP@ss1"
}

// Response 200
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 900,           // 15 minutes
  "user": {
    "id": "usr_2xKj9mNp",
    "email": "user@example.com",
    "name": "Jane Doe",
    "credits": 20,
    "free_gens_remaining": 5
  }
}
```

#### POST /auth/refresh
```json
// Request
{
  "refresh_token": "eyJ..."
}

// Response 200
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",    // rotated
  "token_type": "bearer",
  "expires_in": 900
}
```

### 2.2 Credits Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `GET` | `/credits/balance` | Get current credit balance | Yes |
| `GET` | `/credits/history` | Transaction history (paginated) | Yes |

#### GET /credits/balance
```json
// Response 200
{
  "credits": 380,
  "free_gens_remaining": 3,
  "total_paid_credits": 400,
  "total_used_credits": 20
}
```

#### GET /credits/history?page=1&limit=20
```json
// Response 200
{
  "items": [
    {
      "id": "txn_abc123",
      "type": "purchase",
      "amount": 400,
      "description": "$5 Credit Pack",
      "created_at": "2026-05-09T10:00:00Z"
    },
    {
      "id": "txn_def456",
      "type": "consumption",
      "amount": -4,
      "description": "Image generation",
      "generation_id": "gen_xyz789",
      "created_at": "2026-05-09T10:05:00Z"
    }
  ],
  "total": 15,
  "page": 1,
  "limit": 20
}
```

### 2.3 Generation Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/generations` | Submit image generation task | Yes |
| `GET` | `/generations` | List user's generations (paginated) | Yes |
| `GET` | `/generations/{id}` | Get generation status + result | Yes |
| `DELETE` | `/generations/{id}` | Cancel a pending generation | Yes |

#### POST /generations
```json
// Request
{
  "prompt": "A cat wearing sunglasses on a beach",
  "image_url": "https://...",        // optional: source image for editing
  "params": {                          // optional: model-specific params
    "strength": 0.7,
    "guidance_scale": 7.5,
    "width": 512,
    "height": 512
  }
}

// Response 202 (task accepted, processing)
{
  "id": "gen_xyz789",
  "status": "processing",
  "credits_charged": 4,
  "credits_remaining": 376,
  "created_at": "2026-05-09T10:05:00Z",
  "poll_interval_ms": 2000            // client should poll after this many ms
}
```

#### GET /generations/{id}
```json
// Response 200 (completed)
{
  "id": "gen_xyz789",
  "status": "completed",
  "prompt": "A cat wearing sunglasses on a beach",
  "result_url": "https://storage.example.com/images/gen_xyz789.png",
  "result_expires_at": "2026-05-16T10:05:00Z",  // 7-day expiry for stored images
  "credits_charged": 4,
  "created_at": "2026-05-09T10:05:00Z",
  "completed_at": "2026-05-09T10:05:12Z"
}

// Response 200 (failed)
{
  "id": "gen_xyz789",
  "status": "failed",
  "error": "Generation failed: content policy violation",
  "credits_charged": 0,              // credits refunded on failure
  "credits_remaining": 380,
  "created_at": "2026-05-09T10:05:00Z",
  "failed_at": "2026-05-09T10:05:15Z"
}
```

#### GET /generations?page=1&limit=20&status=completed
```json
// Response 200
{
  "items": [
    {
      "id": "gen_xyz789",
      "status": "completed",
      "prompt": "A cat wearing sunglasses...",
      "result_url": "https://storage.example.com/images/gen_xyz789.png",
      "result_expires_at": "2026-05-16T10:05:00Z",
      "created_at": "2026-05-09T10:05:00Z"
    }
  ],
  "total": 42,
  "page": 1,
  "limit": 20
}
```

### 2.4 Payment Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `GET` | `/payments/products` | List available credit packages | No |
| `POST` | `/payments/checkout` | Create Midtrans Snap transaction | Yes |
| `GET` | `/payments/history` | User's purchase history | Yes |

#### GET /payments/products
```json
// Response 200
{
  "products": [
    {
      "id": "prod_starter",
      "name": "Starter Pack",
      "price_idr": 75000,
      "price_display": "Rp75.000",
      "credits": 400,
      "generations": 100,
      "popular": false
    },
    {
      "id": "prod_pro",
      "name": "Pro Pack",
      "price_idr": 150000,
      "price_display": "Rp150.000",
      "credits": 1000,
      "generations": 250,
      "popular": true
    }
  ]
}
```

#### POST /payments/checkout
```json
// Request
{
  "product_id": "prod_starter"
}

// Response 200
{
  "snap_token": "SNAP-abc123def456",
  "order_id": "PAY-2xKj9mNp",
  "redirect_url": "https://app.midtrans.com/snap/v1/transactions/..."
}
```

### 2.5 Webhook Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/webhooks/midtrans` | Midtrans payment notification | Signature verification |
| `POST` | `/webhooks/kieai` | kie.ai callback (if supported) | API key |

#### POST /webhooks/midtrans
Handled transaction_status values:
- `capture` + `fraud_status=accept` — payment successful, add credits
- `settlement` — payment settled (for bank transfers), add credits
- `pending` — payment pending (bank transfer waiting), log and wait
- `deny` — payment denied, mark as failed
- `cancel` — payment cancelled, mark as failed
- `expire` — payment expired, mark as expired
- `capture` + `fraud_status=challenge` — flagged for review, do not add credits yet

#### POST /webhooks/kieai (future: if kie.ai adds webhook support)
- Generation completed notification
- Generation failed notification

### 2.6 Health / System Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `GET` | `/health` | Health check | No |
| `GET` | `/health/detailed` | DB + external service status | Admin |

---

## 3. Database Schema

### Design Notes
- MVP uses SQLite; schema is PostgreSQL-compatible
- All timestamps are UTC, stored as ISO 8601 strings
- UUIDs use a `usr_`, `gen_`, `txn_` prefix for type identification
- Foreign keys use ON DELETE CASCADE where appropriate

### 3.1 users

```sql
CREATE TABLE users (
    id              TEXT PRIMARY KEY,              -- e.g., "usr_2xKj9mNp"
    email           TEXT NOT NULL UNIQUE,
    email_verified  INTEGER NOT NULL DEFAULT 0,    -- boolean: 0/1
    password_hash   TEXT NOT NULL,                  -- bcrypt
    name            TEXT,                           -- display name (optional)
    credits         INTEGER NOT NULL DEFAULT 0,     -- paid credits balance
    free_gens_remaining INTEGER NOT NULL DEFAULT 5, -- free generation counter
    stripe_customer_id  TEXT,                       -- Midtrans/Stipe customer ID (future)
    is_active      INTEGER NOT NULL DEFAULT 1,     -- soft delete / ban
    created_at     TEXT NOT NULL DEFAULT (datetime('now')),  -- ISO 8601
    updated_at     TEXT NOT NULL DEFAULT (datetime('now')),

    CONSTRAINT credits_non_negative CHECK (credits >= 0),
    CONSTRAINT free_gens_non_negative CHECK (free_gens_remaining >= 0)
);

CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe_customer ON users(stripe_customer_id);
```

### 3.2 refresh_tokens

```sql
CREATE TABLE refresh_tokens (
    id          TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  TEXT NOT NULL,                      -- SHA-256 of refresh token
    device_info TEXT,                               -- User-Agent or device label
    expires_at  TEXT NOT NULL,                      -- ISO 8601
    created_at  TEXT NOT NULL DEFAULT (datetime('now')),
    revoked_at  TEXT,                               -- set when revoked

    CONSTRAINT not_expired CHECK (expires_at > datetime('now'))
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);
```

### 3.3 generations

```sql
CREATE TABLE generations (
    id              TEXT PRIMARY KEY,              -- e.g., "gen_xyz789"
    user_id         TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    prompt          TEXT NOT NULL,                  -- user's text prompt
    source_image_url TEXT,                          -- original image URL (for edits)
    params_json     TEXT,                           -- JSON: model params (strength, etc.)
    status          TEXT NOT NULL DEFAULT 'processing',  -- processing|completed|failed|canceled
    kie_task_id     TEXT,                           -- kie.ai task ID
    result_url      TEXT,                           -- stored result image URL
    result_expires_at TEXT,                         -- when stored image expires
    error_message   TEXT,                           -- failure reason if failed
    credits_charged INTEGER NOT NULL DEFAULT 0,    -- credits deducted
    credits_refunded INTEGER NOT NULL DEFAULT 0,   -- credits returned on failure
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    completed_at    TEXT,
    failed_at       TEXT,

    CONSTRAINT status_valid CHECK (status IN ('processing', 'completed', 'failed', 'canceled'))
);

CREATE INDEX idx_generations_user ON generations(user_id, created_at DESC);
CREATE INDEX idx_generations_status ON generations(status);
CREATE INDEX idx_generations_kie_task ON generations(kie_task_id);
```

### 3.4 credit_transactions

```sql
CREATE TABLE credit_transactions (
    id              TEXT PRIMARY KEY,              -- e.g., "txn_abc123"
    user_id         TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            TEXT NOT NULL,                  -- purchase|consumption|refund|free_grant
    amount          INTEGER NOT NULL,               -- positive for credit, negative for debit
    balance_after   INTEGER NOT NULL,               -- credits balance after transaction
    description     TEXT NOT NULL,                   -- e.g., "Starter Pack (Rp75.000)", "Image generation"
    reference_id    TEXT,                            -- FK to generation or payment_session
    reference_type  TEXT,                            -- generation|payment_session|signup
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),

    CONSTRAINT type_valid CHECK (type IN ('purchase', 'consumption', 'refund', 'free_grant')),
    CONSTRAINT reference_type_valid CHECK (reference_type IN ('generation', 'payment_session', 'signup', 'admin_adjustment'))
);

CREATE INDEX idx_credit_transactions_user ON credit_transactions(user_id, created_at DESC);
CREATE INDEX idx_credit_transactions_reference ON credit_transactions(reference_id);
```

### 3.5 payment_sessions

```sql
CREATE TABLE payment_sessions (
    id                  TEXT PRIMARY KEY,
    user_id             TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    order_id            TEXT NOT NULL UNIQUE,          -- e.g., "PAY-2xKj9mNp" (sent to Midtrans)
    gateway             TEXT NOT NULL DEFAULT 'midtrans',  -- midtrans|stripe (future)
    snap_token          TEXT,                           -- Midtrans Snap token
    product_id          TEXT NOT NULL,                  -- e.g., "prod_starter"
    amount_idr          INTEGER NOT NULL,               -- e.g., 75000 (in IDR, no decimals)
    credits_to_add      INTEGER NOT NULL,               -- e.g., 400
    payment_type        TEXT,                           -- qris|gopay|bank_transfer|etc (from Midtrans)
    status              TEXT NOT NULL DEFAULT 'pending',  -- pending|completed|failed|expired|challenge
    created_at          TEXT NOT NULL DEFAULT (datetime('now')),
    completed_at        TEXT,

    CONSTRAINT session_status_valid CHECK (status IN ('pending', 'completed', 'failed', 'expired', 'challenge'))
);

CREATE INDEX idx_payment_sessions_user ON payment_sessions(user_id, created_at DESC);
CREATE INDEX idx_payment_sessions_order ON payment_sessions(order_id);
```

### 3.6 rate_limits

```sql
CREATE TABLE rate_limits (
    id          TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    action      TEXT NOT NULL,                      -- generation|login|register
    ip_address  TEXT,
    timestamp   TEXT NOT NULL DEFAULT (datetime('now')),

    CONSTRAINT action_valid CHECK (action IN ('generation', 'login', 'register'))
);

CREATE INDEX idx_rate_limits_user_action ON rate_limits(user_id, action, timestamp);
CREATE INDEX idx_rate_limits_ip_action ON rate_limits(ip_address, action, timestamp);
```

### SQLite to PostgreSQL Migration Notes

When migrating to PostgreSQL:
- `INTEGER` boolean fields -> `BOOLEAN`
- `TEXT` timestamp fields -> `TIMESTAMPTZ`
- `TEXT` JSON fields -> `JSONB`
- `TEXT` id fields -> `UUID` (or keep TEXT with prefix pattern)
- Add `SERIAL` auto-increment for internal sequential IDs if desired
- SQLite `datetime('now')` -> PostgreSQL `NOW()` or `DEFAULT NOW()`
- Add `COMMENT ON` for table/column documentation

---

## 4. Auth Strategy

### 4.1 Overview
- **JWT access tokens** (short-lived: 15 minutes) + **JWT refresh tokens** (long-lived: 7 days)
- Access tokens stored in memory (React state/context)
- Refresh tokens stored in HTTP-only cookies (Secure, SameSite=Strict)
- Refresh token rotation: every refresh issues a new refresh token, old one is revoked

### 4.2 Password Security
- Hashing: `bcrypt` with cost factor 12
- Password requirements: min 8 chars, at least 1 uppercase, 1 lowercase, 1 digit
- Validation on server-side before hashing

### 4.3 JWT Payload

**Access Token:**
```json
{
  "sub": "usr_2xKj9mNp",
  "email": "user@example.com",
  "type": "access",
  "iat": 1715246400,
  "exp": 1715247300         // 15 min
}
```

**Refresh Token:**
```json
{
  "sub": "usr_2xKj9mNp",
  "type": "refresh",
  "jti": "rt_abc123",       // unique ID for revocation tracking
  "iat": 1715246400,
  "exp": 1715851200         // 7 days
}
```

### 4.4 Auth Flow

```
Registration:
  1. POST /auth/register {email, password, name?}
  2. Server validates input, hashes password, creates user with 20 credits (5 free gens)
  3. Logs "free_grant" credit_transaction
  4. Returns user object (NO automatic login — user must login separately)

Login:
  1. POST /auth/login {email, password}
  2. Server validates credentials, creates access + refresh tokens
  3. Stores refresh token hash in DB
  4. Returns access token in body, sets refresh token in HTTP-only cookie

Token Refresh:
  1. POST /auth/refresh with refresh token from cookie
  2. Server validates token, checks revocation in DB
  3. Issues new access + refresh tokens (rotation)
  4. Revokes old refresh token in DB
  5. Returns new tokens

Logout:
  1. POST /auth/logout
  2. Server revokes refresh token in DB
  3. Clears HTTP-only cookie

Middleware (every authenticated endpoint):
  1. Extract Authorization: Bearer <token> header
  2. Verify JWT signature and expiration
  3. Load user from DB
  4. Check is_active flag
  5. Attach user to request context
```

### 4.5 Environment Variables for Auth
```
JWT_SECRET=<random-256-bit-hex>
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=7
BCRYPT_COST=12
```

---

## 5. Image Generation Flow

### 5.1 Complete Flow Diagram

```
User                React SPA           FastAPI Backend        kie.ai API
  |                     |                      |                     |
  |  "Generate image"   |                      |                     |
  |-------------------->|                      |                     |
  |                     | POST /generations    |                     |
  |                     | {prompt, params}     |                     |
  |                     |--------------------->|                     |
  |                     |                      |                     |
  |                     |                      | 1. Check auth        |
  |                     |                      | 2. Check credits     |
  |                     |                      | 3. Deduct 4 credits  |
  |                     |                      | 4. Create generation |
  |                     |                      |    record (processing)|
  |                     |                      |                     |
  |                     |                      | POST /tasks         |
  |                     |                      | {prompt, image_url} |
  |                     |                      |-------------------->|
  |                     |                      |                     |
  |                     |                      | {task_id: "kie_abc"}|
  |                     |                      |<--------------------|
  |                     |                      |                     |
  |                     |                      | Store kie_task_id   |
  |                     |                      |                     |
  |                     | 202 {id, status:     |                     |
  |                     |    "processing",     |                     |
  |                     |    poll_interval}    |                     |
  |                     |<---------------------|                     |
  |                     |                      |                     |
  |  Show spinner       |                      |                     |
  |<--------------------|                      |                     |
  |                     |                      |                     |
  |  (poll every 2s)   |                      |                     |
  |-------------------->|                      |                     |
  |                     | GET /generations/{id} |                     |
  |                     |--------------------->|                     |
  |                     |                      |                     |
  |                     |                      | [Background worker] |
  |                     |                      | GET /tasks/kie_abc  |
  |                     |                      |-------------------->|
  |                     |                      |                     |
  |                     |                      | {status: "running"} |
  |                     |                      |<--------------------|
  |                     |                      |                     |
  |                     |                      | ... poll again ...  |
  |                     |                      |                     |
  |                     |                      | GET /tasks/kie_abc  |
  |                     |                      |-------------------->|
  |                     |                      |                     |
  |                     |                      | {status: "completed"|
  |                     |                      |  output: [{url}]}   |
  |                     |                      |<--------------------|
  |                     |                      |                     |
  |                     |                      | 1. Download image   |
  |                     |                      | 2. Store to local/S3 |
  |                     |                      | 3. Update generation |
  |                     |                      |    status=completed  |
  |                     |                      | 4. Store result_url  |
  |                     |                      |                     |
  |                     | GET /generations/{id} |                     |
  |                     |--------------------->|                     |
  |                     |                      |                     |
  |                     | 200 {status:         |                     |
  |                     |   "completed",       |                     |
  |                     |   result_url: "..."}  |                     |
  |                     |<---------------------|                     |
  |                     |                      |                     |
  |  Show result image  |                      |                     |
  |<--------------------|                      |                     |
```

### 5.2 Step-by-Step Backend Logic

```
POST /generations handler:
  1. Authenticate user (JWT middleware)
  2. Validate request body (prompt required, params optional)
  3. Determine credit source:
     a. If free_gens_remaining > 0: use free gen (no credit deduction)
     b. Else if credits >= 4: deduct 4 credits
     c. Else: return 402 Payment Required
  4. Create generation record with status="processing"
  5. Record credit transaction (consumption or free_grant consumption)
  6. Call kie.ai POST /tasks with prompt + params
  7. Store kie_task_id in generation record
  8. Return 202 with generation ID and poll interval

Background Worker (runs every 3 seconds):
  1. Query generations WHERE status = 'processing' AND created_at > now() - 5 min
  2. For each: GET /tasks/{kie_task_id} from kie.ai
  3. If status = "completed":
     a. Download result image
     b. Save to ./storage/images/{gen_id}.png (MVP) or S3 (production)
     c. Update generation: status="completed", result_url, completed_at
  4. If status = "failed":
     a. Update generation: status="failed", error_message, failed_at
     b. Refund credits: add credits back, record credit_transaction (refund)
     c. If was free gen: increment free_gens_remaining back
  5. If status = "running": skip, poll again next cycle

Generation Timeout:
  - If generation is "processing" for > 5 minutes, mark as "failed"
  - Refund credits as above
```

### 5.3 Free vs Paid Credit Logic

```
on_submit_generation(user):
  if user.free_gens_remaining > 0:
    user.free_gens_remaining -= 1
    record transaction: type="consumption", amount=0, description="Free generation"
    # No credit deduction needed — free gens don't cost credits
  else if user.credits >= 4:
    user.credits -= 4
    record transaction: type="consumption", amount=-4, description="Image generation"
  else:
    return Error 402

on_generation_failed(user, generation):
  if generation.used_free_gen:
    user.free_gens_remaining += 1
    record transaction: type="refund", amount=0, description="Free gen refund"
  else:
    user.credits += generation.credits_charged
    record transaction: type="refund", amount=generation.credits_charged, description="Failed gen refund"
```

### 5.4 kie.ai API Integration Details

```
Base URL: https://api.kie.ai/v1 (confirm actual URL)

POST /tasks
  Headers: Authorization: Bearer {KIE_API_KEY}
  Body: {
    "model": "nano-banana-edit",
    "prompt": "...",
    "image_url": "...",        // optional for edit mode
    "params": { ... }           // model-specific parameters
  }
  Response: { "task_id": "kie_abc123" }

GET /tasks/{task_id}
  Headers: Authorization: Bearer {KIE_API_KEY}
  Response (processing): { "status": "running" }
  Response (completed): {
    "status": "completed",
    "output": [{ "url": "https://..." }]
  }
  Response (failed): {
    "status": "failed",
    "error": "..."
  }
```

---

## 6. Payment Flow

### 6.0 Payment Strategy

**Primary: Midtrans (Indonesian market)**
- Registration: Perorangan (KTP only, no PT/NPWP required for Starter Pack)
- Payment methods: QRIS (0.7%), GoPay (2%), Virtual Account (Rp4K flat), DANA, ShopeePay
- Integration: Midtrans Snap (hosted checkout page, mobile-optimized)
- Webhook: Midtrans notification endpoint (HTTP POST with signature verification)

**Future: Stripe (international cards)**
- For international customers paying in USD
- Apple Pay / Google Pay
- Added when PT/business entity is established or using international Stripe account

**Pricing in IDR (primary currency):**
- Starter Pack: Rp75,000 → 400 credits (100 generations)
- Pro Pack: Rp150,000 → 1,000 credits (250 generations)
- 5 free generations for new users (no payment required)

### 6.1 Purchase Flow Diagram

```
User                React SPA           FastAPI Backend        Midtrans
  |                     |                      |                   |
  |  "Buy Rp75K pack"   |                      |                   |
  |-------------------->|                      |                   |
  |                     | POST /payments/checkout                  |
  |                     | {product_id: "prod_starter"}            |
  |                     |--------------------->|                   |
  |                     |                      |                   |
  |                     |                      | 1. Verify product
  |                     |                      | 2. Create Snap    |
  |                     |                      |    transaction    |
  |                     |                      |                   |
  |                     |                      | snap.transactions.
  |                     |                      | create(token, {
  |                     |                      |   transaction_details:
  |                     |                      |   { order_id, gross_amount }
  |                     |                      |   item_details: [...]
  |                     |                      |   customer_details: {...}
  |                     |                      | })
  |                     |                      |------------------>|
  |                     |                      |                   |
  |                     |                      | { token }         |
  |                     |                      |<------------------|
  |                     |                      |                   |
  |                     |                      | Store payment_session in DB
  |                     |                      |                   |
  |                     | { snap_token }      |                   |
  |                     |<---------------------|                   |
  |                     |                      |                   |
  |                     | snap.pay( snap_token ) — Midtrans Snap popup
  |                     |                      |                   |
  |  [Midtrans Snap UI] |                      |                   |
  |  - QRIS QR code     |                      |                   |
  |  - GoPay option     |                      |                   |
  |  - VA bank transfer |                      |                   |
  |  - Other e-wallets  |                      |                   |
  |                     |                      |                   |
  |  User completes payment on Midtrans Snap   |
  |                     |                      |                   |
  |                     |                      |  [Midtrans notification]
  |                     |                      |  POST /webhooks/midtrans
  |                     |                      |  {transaction_status: "capture",
  |                     |                      |   payment_type: "qris",
  |                     |                      |   order_id: "PAY-...",
  |                     |                      |   gross_amount: "75000"}
  |                     |                      |<------------------|
  |                     |                      |                   |
  |                     |                      | 1. Verify signature (SHA512)
  |                     |                      | 2. Find payment_session in DB
  |                     |                      | 3. Verify status is "pending"
  |                     |                      | 4. Add credits to user
  |                     |                      | 5. Record credit_transaction
  |                     |                      | 6. Mark session completed
  |                     |                      |                   |
  |                     |                      | 200 OK            |
  |                     |                      |------------------>|
  |                     |                      |                   |
  |  (Frontend polls /credits/balance)         |                   |
  |  "Credits updated"  |                      |                   |
  |<--------------------|                      |                   |
```

### 6.2 Midtrans Snap Configuration

```python
# Product catalog (defined in code, not in Midtrans dashboard)
PRODUCTS = {
    "prod_starter": {
        "name": "Starter Pack",
        "price_idr": 75000,
        "credits": 400,
        "generations": 100,
    },
    "prod_pro": {
        "name": "Pro Pack",
        "price_idr": 150000,
        "credits": 1000,
        "generations": 250,
    },
}

# Midtrans Snap transaction creation
snap_params = {
    "transaction_details": {
        "order_id": f"PAY-{uuid_short}",       # unique per transaction
        "gross_amount": product["price_idr"],    # in IDR, integer
    },
    "item_details": [{
        "id": product_id,
        "price": product["price_idr"],
        "quantity": 1,
        "name": product["name"],
    }],
    "customer_details": {
        "first_name": user.name or user.email.split("@")[0],
        "email": user.email,
    },
    "callbacks": {
        "finish": f"{FRONTEND_URL}/billing?status=success",
        "error": f"{FRONTEND_URL}/billing?status=error",
        "unfinish": f"{FRONTEND_URL}/billing?status=pending",
    },
}
```

### 6.3 Midtrans Webhook Handler Logic

```
POST /webhooks/midtrans:
  1. Read raw request body (JSON)
  2. Verify signature:
     - Compute SHA512(order_id + status_code + gross_amount + SERVER_KEY)
     - Compare with signature_key from notification body
     - If mismatch: return 403 (reject)
  3. Parse notification:
     - order_id: "PAY-{short_id}"
     - transaction_status: "capture" | "settlement" | "pending" | "deny" | "cancel" | "expire" | "refund"
     - payment_type: "qris" | "gopay" | "bank_transfer" | etc.
     - fraud_status: "accept" | "challenge"
  4. Handle successful payment (transaction_status in ["capture", "settlement"] AND fraud_status == "accept"):
     a. Find payment_session in our DB by order_id
     b. Verify session is still "pending" (idempotency)
     c. Add credits to user: user.credits += session.credits_to_add
     d. Record credit_transaction: type="purchase", amount=session.credits_to_add
     e. Update payment_session: status="completed", completed_at=now, payment_type=notification.payment_type
     f. Free gens remain unchanged (independent of paid credits)
  5. Handle failed/expired payment (transaction_status in ["deny", "cancel", "expire"]):
     a. Update payment_session: status="expired" or "failed"
  6. Handle challenge (transaction_status == "capture" AND fraud_status == "challenge"):
     a. Log for manual review
     b. Do NOT add credits yet — wait for acceptance
  7. Return 200 OK immediately (Midtrans retries on non-200)
```

### 6.4 Idempotency

The payment_sessions table tracks payment state. The webhook handler checks `status = 'pending'` before adding credits. If the webhook fires twice, the second invocation finds `status = 'completed'` and skips.

Midtrans may send multiple notifications for the same transaction (e.g., "pending" → "capture" → "settlement"). Always process based on the latest status and check DB state before updating.

---

## 7. Free Tier Logic

### 7.1 Rules
- Every new user gets **5 free generations** (tracked in `free_gens_remaining`)
- Free gens are consumed **before** paid credits
- Free gens do NOT cost credits — they are a separate counter
- If a free gen fails, `free_gens_remaining` is incremented back
- Free gens never expire — they remain until used
- Buying credits does NOT affect free_gens_remaining

### 7.2 Enforcement Logic (Pseudocode)

```python
def check_generation_eligibility(user: User) -> tuple[bool, str, str]:
    """
    Returns (eligible, credit_source, error_message)
    credit_source: "free" | "paid" | "none"
    """
    if user.free_gens_remaining > 0:
        return True, "free", ""
    elif user.credits >= 4:
        return True, "paid", ""
    else:
        return False, "none", "Insufficient credits. Purchase more to continue."

def deduct_generation(user: User, credit_source: str) -> int:
    """
    Deducts the appropriate resource. Returns credits_charged.
    """
    if credit_source == "free":
        user.free_gens_remaining -= 1
        return 0  # no credits charged
    else:  # "paid"
        user.credits -= 4
        return 4

def refund_generation(user: User, generation: Generation):
    """
    Refunds the resource if generation fails.
    """
    if generation.credits_charged == 0:
        # This was a free gen
        user.free_gens_remaining += 1
    else:
        user.credits += generation.credits_charged
```

### 7.3 Database Enforcement

The `users` table has `CHECK (free_gens_remaining >= 0)` and `CHECK (credits >= 0)` constraints. These prevent negative values at the database level even if there's a race condition in application logic.

### 7.4 Free Tier vs Paid Credits Interaction

```
Scenario: User has 3 free gens remaining and 400 paid credits

Generation #1: free_gens_remaining 3->2, credits unchanged (400)
Generation #2: free_gens_remaining 2->1, credits unchanged (400)
Generation #3: free_gens_remaining 1->0, credits unchanged (400)
Generation #4: free_gens_remaining 0, credits 400->396
Generation #5: free_gens_remaining 0, credits 396->392
...
Generation #100: free_gens_remaining 0, credits 4->0
Generation #101: 402 Payment Required
```

---

## 8. File Structure

### 8.1 Backend (FastAPI)

```
saas-image-gen/
  backend/
    app/
      __init__.py
      main.py                     # FastAPI app factory, lifespan events
      config.py                   # Settings from env vars (pydantic-settings)
      dependencies.py             # DI: get_db, get_current_user, get_redis

      auth/
        __init__.py
        router.py                 # /auth/* endpoints
        service.py                # Auth business logic
        schemas.py                # Pydantic request/response models
        security.py               # JWT encode/decode, password hashing

      credits/
        __init__.py
        router.py                 # /credits/* endpoints
        service.py                # Credit balance, transaction logic
        schemas.py                # Pydantic models

      generations/
        __init__.py
        router.py                 # /generations/* endpoints
        service.py                # Generation orchestration logic
        schemas.py                # Pydantic models
        worker.py                 # Background polling worker for kie.ai

      payments/
        __init__.py
        router.py                 # /payments/* endpoints
        service.py                # Midtrans Snap integration logic
        schemas.py                # Pydantic models
        products.py               # Product catalog definitions (IDR prices, credits)

      webhooks/
        __init__.py
        router.py                 # /webhooks/* endpoints
        midtrans_handler.py        # Midtrans notification handler

      models/
        __init__.py
        user.py                   # SQLAlchemy User model
        generation.py              # SQLAlchemy Generation model
        credit_transaction.py      # SQLAlchemy CreditTransaction model
        payment_session.py          # SQLAlchemy PaymentSession model
        refresh_token.py           # SQLAlchemy RefreshToken model
        rate_limit.py              # SQLAlchemy RateLimit model

      db/
        __init__.py
        session.py                # Engine, SessionLocal, Base
        migrations/               # Alembic migration files
          versions/
          env.py
          alembic.ini

      middleware/
        __init__.py
        auth.py                   # JWT verification middleware
        rate_limit.py              # Rate limiting middleware

      utils/
        __init__.py
        id_gen.py                 # Prefixed ID generation (usr_, gen_, txn_)
        storage.py                 # Image storage abstraction (local/S3)

    tests/
      __init__.py
      conftest.py                 # Fixtures: test client, test DB, mock user
      test_auth.py
      test_credits.py
      test_generations.py
      test_payments.py
      test_webhooks.py
      test_worker.py

    alembic.ini                   # Root alembic config
    pyproject.toml                # Dependencies, scripts
    Dockerfile                    # Multi-stage build
    docker-compose.yml            # Backend + worker + DB
    .env.example                  # Template for environment variables
    .gitignore
```

### 8.2 Frontend (React)

```
saas-image-gen/
  frontend/
    public/
      favicon.ico
      index.html

    src/
      main.tsx                    # React entry point
      App.tsx                     # Router setup

      api/
        client.ts                 # Axios instance with interceptors (auth headers, refresh)
        auth.ts                   # Auth API calls
        credits.ts                # Credits API calls
        generations.ts            # Generations API calls
        payments.ts               # Payments API calls (Midtrans Snap)

      components/
        Layout/
          Header.tsx
          Footer.tsx
          Sidebar.tsx
        Auth/
          LoginForm.tsx
          RegisterForm.tsx
        Editor/
          ImageEditor.tsx          # Main editor component
          PromptInput.tsx
          GenerationCard.tsx
          GenerationList.tsx
        Credits/
          CreditBalance.tsx
          CreditHistory.tsx
          PurchaseModal.tsx              # Midtrans Snap checkout flow
        Common/
          Button.tsx
          Modal.tsx
          Spinner.tsx
          Toast.tsx

      pages/
        LoginPage.tsx
        RegisterPage.tsx
        DashboardPage.tsx          # Main editor + generations
        BillingPage.tsx            # Purchase credits
        HistoryPage.tsx           # Generation history

      hooks/
        useAuth.ts                # Auth state + actions
        useCredits.ts             # Credit balance polling
        useGenerations.ts          # Generation CRUD + polling
        usePolling.ts              # Generic polling hook

      context/
        AuthContext.tsx            # Auth provider + state
        ToastContext.tsx           # Toast notification provider

      types/
        api.ts                    # API response types
        models.ts                  # Domain model types

      utils/
        constants.ts              # API URLs, credit costs, IDR formatting
        formatters.ts             # Date, IDR currency formatting (Rp75.000 format)

    package.json
    tsconfig.json
    vite.config.ts                # Vite config
    tailwind.config.js
    index.html                     # Includes Midtrans Snap JS SDK script
    .env.example
    .gitignore
```

### 8.3 Shared/Root

```
saas-image-gen/
  docker-compose.yml             # Full stack: backend + worker + frontend + DB
  .github/
    workflows/
      ci.yml                     # Lint, test, build
      deploy.yml                 # Deploy to production
  README.md
  .gitignore
  .env.example
```

---

## 9. Deployment Strategy

### 9.1 MVP Deployment (Phase 1)

| Component | Platform | Reasoning |
|---|---|---|
| **Frontend** | Vercel | Free tier, automatic HTTPS, global CDN, easy React deployment |
| **Backend API** | VPS (Hetzner/Railway) | Need persistent process for background worker, WebSocket potential |
| **Database** | SQLite on VPS | MVP simplicity, single-server deployment |
| **Image Storage** | Local filesystem on VPS | MVP simplicity, move to S3 later |
| **Midtrans Webhooks** | Midtrans -> VPS endpoint | Needs public HTTPS URL |

### 9.2 Production Deployment (Phase 2)

| Component | Platform | Reasoning |
|---|---|---|
| **Frontend** | Vercel | Same as MVP |
| **Backend API** | Docker on VPS (or Railway/Render) | Containerized for consistency, easy scaling |
| **Database** | PostgreSQL on managed service | Reliability, backups, concurrent access |
| **Image Storage** | Cloudflare R2 or AWS S3 | Durability, CDN, no egress fees (R2) |
| **Background Worker** | Docker on same VPS | Shares DB connection, simple architecture |
| **Monitoring** | Sentry + Uptime Robot | Error tracking + uptime monitoring |

### 9.3 Docker Compose (Development & Production)

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build: ./backend
    ports:
      - "8000:8000"
    env_file: .env
    environment:
      - DATABASE_URL=sqlite:///./data/app.db
      - STORAGE_BACKEND=local
      - STORAGE_PATH=/app/data/images
    volumes:
      - app_data:/app/data
    depends_on:
      - worker

  worker:
    build: ./backend
    command: python -m app.generations.worker
    env_file: .env
    environment:
      - DATABASE_URL=sqlite:///./data/app.db
      - STORAGE_BACKEND=local
      - STORAGE_PATH=/app/data/images
    volumes:
      - app_data:/app/data

  # Uncomment for PostgreSQL migration
  # db:
  #   image: postgres:16
  #   environment:
  #     POSTGRES_DB: saas_image_gen
  #     POSTGRES_USER: app
  #     POSTGRES_PASSWORD: ${DB_PASSWORD}
  #   volumes:
  #     - pg_data:/var/lib/postgresql/data

volumes:
  app_data:
  # pg_data:
```

### 9.4 Environment Variables

```bash
# .env.example

# Application
APP_NAME=SaasImageGen
APP_ENV=development          # development|staging|production
APP_URL=http://localhost:8000
FRONTEND_URL=http://localhost:5173
SECRET_KEY=<random-256-bit-hex>

# Database
DATABASE_URL=sqlite:///./data/app.db
# DATABASE_URL=postgresql://app:password@localhost:5432/saas_image_gen

# JWT
JWT_SECRET=<random-256-bit-hex>
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=7

# Midtrans (Primary — Indonesian payments)
MIDTRANS_SERVER_KEY=SB-Mid-server-...
MIDTRANS_CLIENT_KEY=SB-Mid-client-...
MIDTRANS_IS_PRODUCTION=false   # true for production
MIDTRANS_SNAP_URL=https://app.sandbox.midtrans.com/snap/v1/transactions  # sandbox
# MIDTRANS_SNAP_URL=https://app.midtrans.com/snap/v1/transactions       # production

# Stripe (Future — International cards)
# STRIPE_SECRET_KEY=sk_test_...
# STRIPE_PUBLISHABLE_KEY=pk_test_...
# STRIPE_WEBHOOK_SECRET=whsec_...

# kie.ai
KIE_API_KEY=kie_...
KIE_API_BASE_URL=https://api.kie.ai/v1
KIE_POLL_INTERVAL_SECONDS=3
KIE_MAX_POLL_DURATION_SECONDS=300

# Storage
STORAGE_BACKEND=local         # local|s3
STORAGE_PATH=./data/images
# S3_BUCKET=my-image-gen-results
# S3_REGION=us-east-1
# AWS_ACCESS_KEY_ID=...
# AWS_SECRET_ACCESS_KEY=...

# Rate Limiting
RATE_LIMIT_GENERATIONS_PER_MINUTE=10
RATE_LIMIT_LOGIN_PER_MINUTE=5
RATE_LIMIT_REGISTER_PER_HOUR=3
```

---

## 10. Security Considerations

### 10.1 Rate Limiting

```
Endpoint Category       | Limit              | Key
------------------------|--------------------|--------------------
POST /auth/register     | 3 per hour         | IP address
POST /auth/login       | 5 per minute       | IP address
POST /auth/refresh      | 20 per minute      | User ID
POST /generations       | 10 per minute      | User ID + IP
GET /generations/{id}   | 30 per minute      | User ID
POST /payments/checkout | 5 per minute      | User ID
All other endpoints     | 60 per minute      | IP address
```

Implementation: sliding window counter stored in `rate_limits` table (MVP). For production, migrate to Redis.

### 10.2 API Key Management

```
SECRETS HANDLING:
  - ALL secrets in environment variables, NEVER in source code
  - .env file in .gitignore
  - Production secrets in server environment or Docker secrets
  - Use pydantic-settings for validated config loading

KEY ROTATION:
  - JWT_SECRET: rotate by accepting old+new during transition period
  - MIDTRANS_SERVER_KEY: rotate via Midtrans dashboard (both sandbox & production)
  - KIE_API_KEY: rotate via kie.ai dashboard

NEVER:
  - Hardcode API keys
  - Commit .env files
  - Log secrets or tokens
  - Return internal errors with stack traces in production
```

### 10.3 Input Validation

```python
# All inputs validated via Pydantic models with strict constraints

class GenerationRequest(BaseModel):
    prompt: str = Field(
        ...,
        min_length=1,
        max_length=2000,
        description="Image generation prompt"
    )
    image_url: Optional[HttpUrl] = Field(
        None,
        description="Source image URL for editing"
    )
    params: Optional[GenerationParams] = Field(
        None,
        description="Model-specific parameters"
    )

class GenerationParams(BaseModel):
    strength: float = Field(0.7, ge=0.0, le=1.0)
    guidance_scale: float = Field(7.5, ge=1.0, le=20.0)
    width: int = Field(512, ge=256, le=1024, multiple_of=64)
    height: int = Field(512, ge=256, le=1024, multiple_of=64)

class RegisterRequest(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=128)
    name: Optional[str] = Field(None, max_length=100)
```

### 10.4 Additional Security Measures

```
1. CORS: Whitelist only the frontend domain in production
   - Allow: FRONTEND_URL
   - Never: *

2. HTTPS: Enforce on all endpoints
   - VPS: Use Caddy or Nginx with Let's Encrypt
   - Vercel: Automatic HTTPS

3. SQL Injection: SQLAlchemy ORM with parameterized queries
   - Never use raw SQL strings with f-strings
   - Always use model.query() or session.execute() with bound params

4. XSS: React escapes by default, but sanitize any user HTML
   - DOMPurify for any dangerouslySetInnerHTML usage

5. CSRF: JWT in Authorization header is immune to CSRF
   - Refresh token in HTTP-only cookie: use SameSite=Strict

6. File Upload Security (if accepting images):
   - Validate Content-Type header matches actual file content (magic bytes)
   - Limit file size (max 10MB)
   - Store images outside webroot or use signed URLs
   - Strip EXIF metadata from uploaded images

7. Content Policy:
   - Validate prompts against a content filter before sending to kie.ai
   - If kie.ai rejects a prompt, return appropriate error to user
   - Log rejected prompts for review (without PII)

8. Midtrans Signature Verification:
   - Compute SHA512(order_id + status_code + gross_amount + SERVER_KEY)
   - Compare with `signature_key` in notification body
   - Reject any notification with invalid signature (return 403)
   - Always verify before processing any payment notification

9. Response Headers:
   - X-Content-Type-Options: nosniff
   - X-Frame-Options: DENY
   - Content-Security-Policy: default-src 'self'
   - Strict-Transport-Security: max-age=31536000; includeSubDomains

9. Logging:
   - Log all auth events (login, register, failed login)
   - Log all credit transactions
   - Log all generation requests (prompt hash, not full prompt for privacy)
   - Never log passwords, tokens, or API keys

10. Dependency Security:
    - Run pip-audit or safety check in CI
    - Pin all dependencies in requirements.txt or pyproject.toml
    - Keep Midtrans SDK and other security-critical deps updated
```

---

## Appendix A: Credit Transaction Types

| Type | Amount | Description | Trigger |
|---|---|---|---|
| `free_grant` | +20 | Welcome bonus (5 free gens) | User registration |
| `purchase` | +400 or +1000 | Credit pack purchase | Midtrans webhook |
| `consumption` | -4 or 0 | Generation cost (0 if free gen) | Generation submitted |
| `refund` | +4 or 0 | Failed generation refund | Generation failed/timeout |

## Appendix B: Generation Status Machine

```
  [created]
      |
      v
  processing  ----timeout---->  failed (+ refund)
      |                           ^
      |                           |
  kie.ai running                  |
      |                           |
      v                           |
  completed                   kie.ai error
                                  |
                                  v
                              failed (+ refund)

  Any state --user cancel--> canceled (if still processing, attempt cancel at kie.ai)
```

## Appendix C: API Error Response Format

```json
{
  "error": {
    "code": "INSUFFICIENT_CREDITS",
    "message": "You need at least 4 credits to generate an image. Purchase more credits to continue.",
    "details": {
      "required_credits": 4,
      "current_credits": 2
    }
  }
}
```

### Error Codes

| HTTP | Code | Description |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Request body or params invalid |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 402 | `INSUFFICIENT_CREDITS` | Not enough credits for generation |
| 403 | `FORBIDDEN` | Account inactive or banned |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `EMAIL_EXISTS` | Email already registered |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Unexpected server error |
| 502 | `UPSTREAM_ERROR` | kie.ai API error |
| 504 | `UPSTREAM_TIMEOUT` | kie.ai API timeout |

## Appendix D: Migration Plan (SQLite -> PostgreSQL)

```
Phase 1 (MVP): SQLite
  - Single file, no external DB dependency
  - Suitable for <10,000 concurrent users
  - Use WAL mode for better concurrent read performance
  - pragma: journal_mode=WAL, busy_timeout=5000

Phase 2 (Growth): PostgreSQL
  - Trigger: >5,000 DAU or concurrent write contention
  - Migration steps:
    1. Update DATABASE_URL env var
    2. Update SQLAlchemy dialect from sqlite to postgresql
    3. Run Alembic migration to create PostgreSQL schema
    4. Migrate data using sqlite3 -> CSV -> COPY
    5. Test with shadow traffic
    6. Switch over with maintenance window
  - PostgreSQL-specific changes:
    - Replace INTEGER booleans with BOOLEAN
    - Replace TEXT timestamps with TIMESTAMPTZ
    - Replace TEXT JSON with JSONB
    - Add proper indexes for production queries
    - Enable connection pooling (pgbouncer)
```

## Appendix E: Midtrans Registration & IDR Pricing

### E.1 Midtrans Starter Pack Registration (Perorangan / Individual)

**Requirements for Individual (no PT needed):**
- KTP (Indonesian ID card) — the only document needed
- No NPWP required for Starter Pack (QRIS, GoPay, Virtual Account only)
- No business entity (PT/CV) required
- Bank account in your personal name (matches KTP)
- NPWP pribadi only needed if adding card payments later

**Registration flow:**
1. Sign up at midtrans.com
2. Choose "Perorangan" (Individual) registration
3. Upload KTP
4. Verify bank account
5. Activate Starter Pack payment methods: QRIS, GoPay, Virtual Account
6. Get Sandbox keys for development, Production keys after verification

**Sandbox testing:**
- Use `SB-Mid-server-...` and `SB-Mid-client-...` keys
- Snap URL: `https://app.sandbox.midtrans.com/snap/v1/transactions`
- Test payment methods available in Midtrans sandbox dashboard

### E.2 IDR Pricing & Payment Methods

| Product | IDR Price | Credits | Generations | QRIS Fee | Net (QRIS) |
|---------|-----------|---------|-------------|----------|------------|
| Starter Pack | Rp75,000 | 400 | 100 | Rp525 (0.7%) | Rp74,475 |
| Pro Pack | Rp150,000 | 1,000 | 250 | Rp1,050 (0.7%) | Rp148,950 |

| Payment Method | Fee at Rp75K | Fee at Rp150K |
|---------------|-------------|---------------|
| QRIS | 0.7% = Rp525 | 0.7% = Rp1,050 |
| GoPay | 2% = Rp1,500 | 2% = Rp3,000 |
| Virtual Account | Rp4,000 flat | Rp4,000 flat |
| DANA | 1.5% = Rp1,125 | 1.5% = Rp2,250 |
| ShopeePay | 2% = Rp1,500 | 2% = Rp3,000 |

### E.3 Frontend: Midtrans Snap Integration

```html
<!-- Include in index.html -->
<script type="text/javascript"
        src="https://app.sandbox.midtrans.com/snap/snap.js"
        data-client-key="SB-Mid-client-xxx"></script>
<!-- Production: https://app.midtrans.com/snap/snap.js -->
```

```typescript
// After getting snap_token from POST /payments/checkout
window.snap.pay(snap_token, {
  onSuccess: function(result) {
    // Payment successful — poll /credits/balance to update UI
    console.log(result.status_message);
  },
  onPending: function(result) {
    // Payment pending (e.g., bank transfer waiting)
    console.log("Waiting for payment confirmation");
  },
  onError: function(result) {
    // Payment failed
    console.log("Payment error");
  },
  onClose: function() {
    // User closed the popup without completing payment
    console.log("Payment popup closed");
  }
});
```