# যখন Server ঠিকই আছে, সমস্যা লুকিয়ে আছে DB-তে

> *"Performance সমস্যার সমাধান করতে গেলে প্রথম প্রশ্নটা হওয়া উচিত — কোথায় সময় যাচ্ছে? সেটা না জেনে কিছু বাড়ালে সমস্যা বাড়ে, কমে না।"*

---

## পরিস্থিতিটা আবার মনে করো

আগের পর্বে একটা চ্যালেঞ্জ দিয়েছিলাম। Setup ছিল এরকম:

- **FastAPI app**, async worker
- **2টি Uvicorn worker** চলছে
- **১০০০ concurrent user** থেকে request আসছে

এবং লক্ষণগুলো ছিল:

| কী দেখা যাচ্ছে | মান |
|---|---|
| p95 latency | অনেক বেশি (ধরো ৩-৪ সেকেন্ড) |
| CPU usage | মাত্র ~২০% |
| DB response time | বেশি |

এই তিনটা clue দিয়েই পুরো সমস্যা diagnose করা যায়। একটু ধীরে ধীরে বিশ্লেষণ করি।

---

## Clue পড়তে শেখো — সংখ্যাগুলো কী বলছে

Performance সমস্যায় সবচেয়ে বড় ভুল হলো তাড়াহুড়ো করে সমাধান করতে যাওয়া। আগে data পড়তে হবে।

### CPU মাত্র ২০% — এটা কী বলছে?

CPU ২০%-এ আছে মানে server মোটেও stressed না। সে আরামে বসে আছে। যদি সমস্যাটা computation-এর হতো — ভারী calculation, image processing — তাহলে CPU ৮০-৯০%-এ থাকত।

CPU কম মানে: **server কাজ করছে না, অপেক্ষা করছে।**

কিসের অপেক্ষা? এখানেই দ্বিতীয় clue-টা কাজে লাগে।

### DB latency বেশি — এটা কী বলছে?

DB-এর response আসতে দেরি হচ্ছে। প্রতিটা request যখন DB query দিচ্ছে, সে অপেক্ষা করছে। ১০০০ জন user একসাথে request করলে ১০০০টা query DB-তে গিয়ে সবাই লাইনে দাঁড়াচ্ছে।

Server idle, কিন্তু DB overwhelmed।

### p95 latency বেশি — এটা কী confirm করছে?

p95 বেশি মানে ১০০ জনের মধ্যে ৫ জন অনেক বেশি সময় অপেক্ষা করছে। এই tail latency-র সবচেয়ে common কারণ হলো — কোথাও queue জমছে। DB-র queue।

```
তিনটা clue একসাথে:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CPU কম      → Server কাজ করছে না
DB latency বেশি → DB-তে আটকাচ্ছে
p95 বেশি    → Queue জমছে কোথাও
               ↓
           Bottleneck = DB layer
```

---

## তাহলে Worker বাড়ালে কী হতো?

এটা সবচেয়ে common ভুল। "Server slow, তাই worker বাড়াই" — এই intuition অনেক ক্ষেত্রে কাজ করে না, এই ক্ষেত্রে উল্টো ক্ষতি করত।

কেন?

Worker বাড়ানো মানে আরো বেশি concurrent request handle করার capacity। মানে DB-তে আরো বেশি concurrent query যাবে। DB যখন ইতিমধ্যে চাপে আছে, তখন আরো চাপ দিলে:

```
Worker বাড়ানোর আগে:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2 worker → 200 concurrent DB query → DB stressed, latency 500ms

Worker বাড়ানোর পরে:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
4 worker → 400 concurrent DB query → DB আরো stressed, latency 1200ms
```

Symptom আরো খারাপ হয়। এটাকে বলে **treating the symptom, not the cause।**

সঠিক চিন্তাভাবনা হলো: Bottleneck কোথায়? সেটা fix করো। তারপর প্রয়োজনে worker বাড়াও।

> **মূল নিয়ম:** Traffic scale করার আগে bottleneck সরাও। নইলে বেশি traffic মানে বেশি সমস্যা।

---

## Fix Strategy — ধাপে ধাপে

Bottleneck DB-তে। তাহলে DB-কে দ্রুত করার উপায় কী? চারটা জায়গায় কাজ করতে হবে।

---

### ধাপ ১ — DB Optimization: Query এবং Index

সবার আগে দেখো DB-তে কোন query-গুলো বেশি সময় নিচ্ছে। PostgreSQL-এ slow query log চালু করে দেখো কোনটা ৫০০ms-এর বেশি নিচ্ছে।

**Index যোগ করা:**

ধরো তুমি প্রায়ই এই query চালাচ্ছ:

```sql
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';
```

যদি `user_id` বা `status` column-এ index না থাকে, DB পুরো table scan করে। ১০ লাখ row থাকলে ১০ লাখ row দেখবে। Index থাকলে সরাসরি relevant row-এ যাবে।

```sql
-- এই index যোগ করলেই query অনেক দ্রুত হবে
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**N+1 Query সমস্যা:**

এটা সবচেয়ে common কিন্তু সবচেয়ে silent killer। ধরো তুমি ১০ জন user-এর order দেখাতে চাও:

```python
# ❌ N+1 — ১টা query users-এর জন্য, তারপর প্রতি user-এর জন্য আলাদা query
users = await db.fetch("SELECT * FROM users LIMIT 10")
for user in users:
    orders = await db.fetch("SELECT * FROM orders WHERE user_id = $1", user.id)
    # মোট = 1 + 10 = 11টা query
```

১০ জন user মানে ১১টা query। ১০০ জন user মানে ১০১টা query। ১০০০ জন user মানে ১০০১টা query। DB-তে request সংখ্যা linearly বাড়ছে।

```python
# ✔️ JOIN দিয়ে একটাই query
result = await db.fetch("""
    SELECT u.*, o.*
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    LIMIT 10
""")
# মোট = মাত্র 1টা query
```

---

### ধাপ ২ — Connection Pooling: Connection তৈরির overhead বাঁচাও

প্রতিটা DB connection তৈরি করতে সময় লাগে — TCP handshake, authentication, session setup। একটা connection তৈরি হতে ১০-৫০ms লাগতে পারে। প্রতিটা request-এ নতুন connection তৈরি করলে সেই সময়টা প্রতিবার যোগ হয়।

Connection pool মানে আগে থেকে কিছু connection তৈরি করে রাখো। Request এলে pool থেকে নাও, শেষ হলে ফেরত দাও।

```python
# asyncpg দিয়ে connection pool
import asyncpg

# App start-এ একবার pool তৈরি করো
pool = await asyncpg.create_pool(
    dsn="postgresql://user:pass@localhost/db",
    min_size=5,    # সবসময় কমপক্ষে ৫টা connection ready
    max_size=20    # সর্বোচ্চ ২০টা পর্যন্ত তৈরি হতে পারবে
)

# প্রতিটা request-এ pool থেকে নাও
async def get_user(user_id: int):
    async with pool.acquire() as conn:    # pool থেকে একটা connection নিল
        return await conn.fetchrow(
            "SELECT * FROM users WHERE id = $1", user_id
        )
    # এখানে connection আপনাআপনি pool-এ ফিরে গেল
```

```
Connection Pool ছাড়া:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Request → [নতুন connection তৈরি: 30ms] → [query: 20ms] → [connection বন্ধ]

Connection Pool সহ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Request → [pool থেকে নাও: ~1ms] → [query: 20ms] → [pool-এ ফেরত]
```

প্রতিটা request ২৯ms বাঁচছে। ১০০০ request/second হলে এটা অনেক বড় পার্থক্য।

---

### ধাপ ৩ — Async DB Driver: Blocking কল সরাও

Async FastAPI app-এ sync DB driver ব্যবহার করলে পুরো event loop block হয়ে যায়।

```python
# ❌ Sync driver — event loop block হয়
import psycopg2

async def get_user(user_id: int):
    conn = psycopg2.connect(...)       # blocking!
    cursor = conn.cursor()
    cursor.execute("SELECT ...")       # blocking!
    return cursor.fetchone()
```

`psycopg2` sync library। `cursor.execute()` চলার সময় পুরো event loop থামে। ১০০০ concurrent user থাকলে একটার DB query চলার সময় বাকি ৯৯৯ জন অপেক্ষা করছে।

```python
# ✔️ Async driver — event loop চলতে থাকে
import asyncpg

async def get_user(user_id: int):
    async with pool.acquire() as conn:
        return await conn.fetchrow(    # await — event loop free
            "SELECT * FROM users WHERE id = $1", user_id
        )
```

`await conn.fetchrow()` মানে: "DB-এর জবাবের জন্য অপেক্ষা করছি, কিন্তু event loop অন্য request handle করো।" এটাই async-এর পুরো সুবিধা।

---

### ধাপ ৪ — Caching: DB-কে আদৌ না জিজ্ঞেস করো

সবচেয়ে দ্রুত DB query হলো যেটা আদৌ DB-তে যায় না।

অনেক data আছে যেটা বারবার একই রকম থাকে — product catalog, user profile, configuration। প্রতিটা request-এ DB-তে গিয়ে এই data আনার দরকার নেই।

**Caching-এর logic:**

```
App → Cache (Redis) → DB
      ↓ hit হলে    ↓ miss হলে
      সেখান থেকেই  DB থেকে আনো
      দাও          Cache-এ রাখো
```

```python
import redis.asyncio as redis
import json

r = redis.Redis(host='localhost', port=6379)

async def get_user(user_id: int):
    cache_key = f"user:{user_id}"

    # প্রথমে cache চেক করো
    cached = await r.get(cache_key)
    if cached:
        return json.loads(cached)      # Cache hit — DB-তে যেতেই হলো না

    # Cache miss — DB থেকে আনো
    user = await db.fetchrow(
        "SELECT * FROM users WHERE id = $1", user_id
    )

    # Cache-এ রাখো, ৫ মিনিটের জন্য
    await r.setex(cache_key, 300, json.dumps(dict(user)))

    return user
```

**কোন data cache করা উচিত?**

ভালো candidate:
- যে data বারবার read হয় কিন্তু কম বদলায় (product info, user profile)
- Expensive query-র result (complex JOIN, aggregation)
- Reference data (category list, configuration)

ভালো candidate নয়:
- Real-time data (stock price, live count)
- User-specific sensitive data (যদি না carefully key করা হয়)
- যে data প্রতি request-এ বদলায়

**TTL (Time-to-Live) সেট করা জরুরি।** Cache-এ data চিরকাল রাখলে stale data দেখাবে। `setex(key, 300, value)` মানে ৩০০ সেকেন্ড (৫ মিনিট) পর cache automatically expire হবে।

---

## সব কিছু একসাথে — আগে এবং পরে

এই চারটা fix করার আগে এবং পরের পরিস্থিতি কল্পনা করো:

```
আগে (কোনো fix নেই):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
User request
    │
    ▼
App (async, কিন্তু sync DB driver!)
    │ blocking call
    ▼
DB (no index, N+1 queries, no pool)
    │ slow scan, নতুন connection তৈরি
    ▼
Response: p95 = 3-4 সেকেন্ড 😞

পরে (সব fix করা):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
User request
    │
    ▼
App (async)
    │
    ▼
Redis Cache ──── hit হলে এখান থেকেই ফেরত (1-2ms)
    │ miss হলে
    ▼
DB (indexed, batched queries, connection pool, async driver)
    │ fast, non-blocking
    ▼
Response: p95 = 50-100ms 😊
```

---

## Debugging Playbook — সমস্যা দেখলে কী করবে

এই পর্বের পুরো lesson-কে একটা decision tree-তে রাখলে:

```
p95 latency বেশি?
│
├── CPU high (80%+)?
│   └── CPU-bound bottleneck
│       → Workers/processes বাড়াও
│       → CPU-heavy কাজ background-এ পাঠাও
│
└── CPU low (20-30%)?
    └── I/O bottleneck — কোথায়?
        │
        ├── DB latency বেশি?
        │   ├── Slow queries? → Index যোগ করো, query optimize করো
        │   ├── N+1 problem? → Batch queries, JOIN ব্যবহার করো
        │   ├── No connection pool? → Pool যোগ করো
        │   ├── Sync DB driver? → Async driver ব্যবহার করো
        │   └── Same data বারবার? → Cache করো (Redis)
        │
        └── External API latency বেশি?
            ├── Timeout set করো
            ├── Response cache করো
            └── Async HTTP client ব্যবহার করো (httpx)
```

---

## সবচেয়ে গুরুত্বপূর্ণ শিক্ষা

এই পুরো scenario থেকে একটাই কথা মনে রাখো:

**সমস্যা না বুঝে সমাধান করতে যাওয়া বিপজ্জনক।**

Worker বাড়ানো intuitive মনে হয় — "server slow, আরো resource দাও।" কিন্তু এই ক্ষেত্রে সেটা উল্টো কাজ করত। Bottleneck DB-তে ছিল। আরো worker মানে DB-তে আরো চাপ।

Diagnose করার order:

```
১. Measure করো   → সংখ্যা দেখো (CPU, DB latency, p95, p99)
২. Hypothesize করো → সংখ্যাগুলো কী বলছে?
৩. Fix করো       → সবচেয়ে বড় bottleneck সরাও
৪. Measure করো   → fix কাজ করেছে কিনা verify করো
৫. Repeat        → পরের bottleneck খোঁজো
```

একটা bottleneck সরালে পরেরটা দেখা যায়। Performance optimization একটা iterative process।

---

## সারসংক্ষেপ

| সমস্যা | Diagnosis | Fix |
|---|---|---|
| CPU low + DB latency high | DB bottleneck | DB optimize করো, worker নয় |
| Worker বাড়ানো কি সাহায্য করবে? | না, আরো DB চাপ বাড়বে | Bottleneck fix করো আগে |
| Query slow | Missing index বা full table scan | Index যোগ করো |
| অনেক বেশি query | N+1 problem | JOIN বা batch query ব্যবহার করো |
| Connection overhead | প্রতি request-এ নতুন connection | Connection pool ব্যবহার করো |
| Event loop blocking | Sync DB driver | Async driver (asyncpg, psycopg3) |
| বারবার same data | Repeated DB calls | Redis cache বসাও DB-এর সামনে |

---

*পরের পর্বে: Production Observability — কীভাবে চলমান system-এর ভেতরে দেখতে হয়, metrics collect করতে হয়, এবং সমস্যা হওয়ার আগেই সেটা টের পেতে হয়।*