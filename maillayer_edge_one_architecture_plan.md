# Maillayer on EdgeOne – Architecture & Migration Plan

> ⚠️ **DISCLAIMER**
>
> This document describes an **intentionally divergent architecture** for Maillayer.
> MongoDB is replaced with **EdgeOne D1 + EdgeOne KV**, and the system is redesigned
> for an **edge-first, stateless execution model**. This plan is not intended to be
> merged upstream.

---

## 1. Goal

Re-implement Maillayer to run fully on **EdgeOne Pages + Edge Functions**, with:

- No MongoDB
- No long-lived servers
- Stateless APIs
- Edge-native storage and job orchestration

---

## 2. Storage Strategy Overview

| Concern | Technology | Responsibility |
|------|-----------|----------------|
| Primary data | EdgeOne D1 (SQL) | Source of truth |
| Fast state / locks | EdgeOne KV | Coordination & caching |
| Authentication | Supabase Auth | Identity & OAuth |
| Email delivery | Amazon SES (API) | Outbound email |

---

## 3. MongoDB → D1 Schema Mapping

### 3.1 Users

**Mongo (conceptual):**
```js
users {
  _id,
  email,
  name,
  provider,
  createdAt
}
```

**D1 (SQL):**
```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  supabase_user_id TEXT UNIQUE,
  email TEXT NOT NULL,
  name TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

### 3.2 Campaigns

```sql
CREATE TABLE campaigns (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  name TEXT NOT NULL,
  subject TEXT,
  status TEXT CHECK(status IN ('draft','scheduled','sending','completed')),
  scheduled_at DATETIME,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY(user_id) REFERENCES users(id)
);
```

---

### 3.3 Subscribers

```sql
CREATE TABLE subscribers (
  id TEXT PRIMARY KEY,
  email TEXT NOT NULL,
  name TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

```sql
CREATE TABLE campaign_subscribers (
  campaign_id TEXT,
  subscriber_id TEXT,
  PRIMARY KEY (campaign_id, subscriber_id)
);
```

---

### 3.4 Events (opens, clicks, bounces)

```sql
CREATE TABLE email_events (
  id TEXT PRIMARY KEY,
  campaign_id TEXT,
  subscriber_id TEXT,
  event_type TEXT,
  occurred_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 4. KV vs D1 Decision Guide

| Feature | Use | Reason |
|------|-----|-------|
| Users, campaigns | D1 | Durable relational data |
| Subscriber lists | D1 | Joins & filtering |
| Send locks | KV | Fast atomic access |
| Rate limits | KV | Low-latency counters |
| Session cache | KV | Short-lived state |
| Analytics | D1 | Queryable history |

---

## 5. Edge-Safe Email Sending Workflow

### 5.1 Constraints
- No long-running workers
- No SMTP connections
- Execution time is limited

---

### 5.2 Workflow

```
User schedules campaign
        ↓
Campaign marked as `scheduled` (D1)
        ↓
Edge Function trigger (cron / request)
        ↓
Acquire send lock (KV)
        ↓
Fetch batch of subscribers (D1)
        ↓
Send emails via SES API (batch)
        ↓
Record events (D1)
        ↓
Release lock / continue next batch
```

---

### 5.3 KV Lock Example

```
key: campaign:send-lock:{campaign_id}
value: timestamp
TTL: 60s
```

Ensures:
- No double sends
- Idempotent retries

---

## 6. Middleware Abstraction (DB-Agnostic)

### 6.1 Repository Pattern

```ts
interface CampaignRepository {
  getById(id: string): Promise<Campaign>
  updateStatus(id: string, status: string): Promise<void>
}
```

### 6.2 D1 Implementation

```ts
class D1CampaignRepository implements CampaignRepository {
  constructor(private db: D1Database) {}
}
```

### 6.3 Benefits
- Database swap friendly
- Testable logic
- No framework coupling

---

## 7. Authentication Flow (Supabase)

```
Browser → Supabase OAuth
        ↓
Supabase issues JWT
        ↓
JWT sent to Edge Function
        ↓
JWT verified
        ↓
User resolved in D1
```

Store only:
- `supabase_user_id`
- email / name snapshot

---

## 8. Recommended Branching Model

```
main              → EdgeOne + D1 implementation
experiments/*     → Spikes / POCs
legacy-mongo      → Snapshot of original design
```

---

## 9. Final Notes

This architecture:
- Is edge-native
- Scales horizontally
- Avoids server management
- Trades flexibility for predictability

It is a **rewrite of assumptions**, not just a database swap.

---

## 10. Next Steps

- Implement schema migrations
- Replace Mongo middleware
- Add SES batching logic
- Introduce observability (logs + metrics)

