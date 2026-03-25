# Async Python Performance: Load Testing থেকে Tuning পর্যন্ত

> *"Fast code লেখা একটা দক্ষতা। কিন্তু slow কোড কেন slow — সেটা বোঝা একটা শিল্প।"*

---

## ভূমিকা: আগের সমস্যাটা মনে আছে?

আমরা আগের পর্বে একটা মাইক্রো-ড্রামা দেখেছিলাম — একটাই async worker, তিনটা request একসাথে এসেছে, প্রতিটায় একটা DB call আর একটা CPU task।

সেই সমস্যার solution দিয়েই আজকের যাত্রা শুরু।

---

## Part 1 — আগের চ্যালেঞ্জের সমাধান

### পরিস্থিতি

- **1টি async worker** (single event loop)
- **3টি request** একসাথে এসেছে
- প্রতিটি request করে:
  - একটি DB call → `await` → **1 সেকেন্ড**
  - একটি CPU-heavy task → blocking → **1 সেকেন্ড**

---

### প্রশ্ন ১: মোট সময় কত লাগবে?

উত্তর: **~4 সেকেন্ড**

কিন্তু কেন? এটা বুঝতে হলে timeline দেখতে হবে।

```text
t=0s → তিনটা request শুরু হলো, সবাই DB-এর জন্য await করছে
t=1s → DB response এলো, তিনটাই ready

# এখন CPU task — এটা blocking, তাই একটার পর একটা:
t=1s → t=2s  : Request 1-এর CPU task চলছে
t=2s → t=3s  : Request 2-এর CPU task চলছে
t=3s → t=4s  : Request 3-এর CPU task চলছে
```

**সারসংক্ষেপ:** DB-এর 1s সবাই একসাথে করতে পারে (I/O overlap), কিন্তু CPU-এর 3×1s কেউ ভাগ করতে পারে না।

মোট = **1s (I/O) + 3s (CPU serial) = 4s**

---

### প্রশ্ন ২: Event loop কোথায় block হয়?

**CPU task-এ।**

Async programming-এর একটা মূল নিয়ম আছে — event loop তখনই অন্য কাজ করতে পারে যখন সে `await` দেখে। CPU task-এ কোনো `await` নেই, তাই event loop আটকে থাকে। একটা শেষ না হওয়া পর্যন্ত পরেরটা শুরু হতে পারে না।

---

### প্রশ্ন ৩: Fix কী?

CPU কাজকে event loop থেকে **offload** করতে হবে — thread বা process-এ।

**Option A — Thread (দ্রুত সমাধান):**

```python
await asyncio.to_thread(cpu_task)
```

এটা সহজ এবং অনেক ক্ষেত্রে যথেষ্ট। তবে Python-এর GIL (Global Interpreter Lock) থাকায় pure CPU কাজে thread সত্যিকারের parallel হয় না।

**Option B — Process Pool (CPU-heavy কাজের জন্য আদর্শ):**

```python
loop = asyncio.get_event_loop()
await loop.run_in_executor(process_pool, cpu_task)
```

আলাদা process মানে আলাদা Python interpreter — GIL-এর বাধা নেই।

---

### প্রশ্ন ৪: Fix করলে নতুন সময় কত?

**~2 সেকেন্ড**

```text
t=0s → তিনটাই DB-এর জন্য await
t=1s → DB শেষ, CPU tasks offload হলো (প্রায় parallel)
t=2s → সব শেষ
```

> **মূল শিক্ষা:**
> `async` handles *waiting* (I/O).
> `process/thread` handles *computing* (CPU).
> দুটো একসাথে ব্যবহার করতে শিখলেই আসল শক্তি আসে।

---

---

## Part 2 — Load Testing ও Tuning: Performance Alchemy

এতক্ষণ আমরা *কীভাবে কাজ করে* সেটা বুঝেছি। এখন শিখব — **system কীভাবে shape করতে হয়।**

---

## ২.১ Performance মানে আসলে কী?

Performance পরিমাপ করা হয় তিনটা জিনিস দিয়ে:

| Metric | মানে |
|---|---|
| **Latency** | একটি request শেষ হতে কত সময় লাগে |
| **Throughput** | প্রতি সেকেন্ডে কতটি request handle করা যায় (RPS) |
| **Concurrency** | একই সময়ে কতটি request "in-flight" আছে |

এই তিনটা পরস্পর সম্পর্কিত। Concurrency বাড়ালে throughput বাড়ে ঠিকই — কিন্তু যদি system সেটা handle করতে না পারে, queue জমে যায় এবং latency বেড়ে যায়। শুধু concurrency বাড়ানোই সমাধান নয়।

---

## ২.২ Gunicorn Worker Tuning

Gunicorn দিয়ে Python WSGI/ASGI app serve করার সময় তিন ধরনের concurrency আছে:

### Workers (Process)

```bash
gunicorn app:app -w 4
```

প্রতিটি worker একটি আলাদা OS process। এগুলো CPU core-এ সত্যিকারের parallel কাজ করতে পারে।

**Thumb rule:**
```
workers ≈ (CPU core সংখ্যা × 2) + 1
```

এটা শুরুর জায়গা, শেষ কথা না। বেশি worker দিলে:
- Memory অনেক বেশি লাগে (প্রতিটা process আলাদা RAM নেয়)
- Context switching overhead বাড়ে

### Threads (gthread worker)

```bash
gunicorn app:app -w 2 --threads 4
```

একটি process-এর ভেতরে multiple thread। Blocking I/O handle করার জন্য ভালো।

**সতর্কতা:** অনেক বেশি thread দিলে memory বাড়ে এবং GIL-এর কারণে CPU-bound কাজে উপকার হয় না, বরং contention বাড়তে পারে।

### Async Workers (uvicorn/uvicorn workers)

```bash
gunicorn app:app -w 2 -k uvicorn.workers.UvicornWorker
```

একটি async worker হাজার হাজার concurrent connection handle করতে পারে — কারণ I/O-তে সে block করে না, `await` করে।

**মোট concurrency-র intuition:**
```
Total concurrency ≈ workers × per-worker capacity
```

---

## ২.৩ Load Testing Tools

Code লেখার পরে শুধু "মনে হচ্ছে ভালো চলছে" যথেষ্ট না। **Measure করতে হবে।**

### wrk — দ্রুত HTTP benchmark

```bash
wrk -t4 -c100 -d30s http://localhost:8000/api/endpoint
```

- `-t4` → 4টি thread
- `-c100` → 100টি concurrent connection
- `-d30s` → 30 সেকেন্ড চালাও

Output-এ পাবে: RPS, latency distribution, error count।

### locust — User behavior simulation

```python
from locust import HttpUser, task

class WebsiteUser(HttpUser):
    @task
    def view_homepage(self):
        self.client.get("/")
```

`locust -f locustfile.py` চালালে একটি web UI আসে যেখানে user সংখ্যা ধীরে ধীরে বাড়ানো যায় এবং real-time graph দেখা যায়।

**কী মাপবে?**
- Latency: p50, p95, p99
- Throughput: RPS (Requests Per Second)
- Error rate: 5xx কতটা আসছে

---

## ২.৪ Latency Percentiles — সবচেয়ে গুরুত্বপূর্ণ কনসেপ্ট

Average latency দেখে সিস্টেম ভালো মনে হতে পারে, কিন্তু সেটা প্রায়ই বিভ্রান্তিকর।

**উদাহরণ:**

| Percentile | Latency |
|---|---|
| p50 (median) | 100ms |
| p95 | 800ms |
| p99 | 2,000ms |

এই মানে হলো: 1000 জন user-এর মধ্যে 10 জন প্রতিটি request-এ 2 সেকেন্ড অপেক্ষা করছে।

> **Real insight:** Average দেখলে মনে হবে সব ঠিক আছে। কিন্তু user-রা মনে রাখে তাদের সবচেয়ে খারাপ অভিজ্ঞতা।
>
> **"Users remember p99, not p50."**

Tail latency (p95, p99) কমানো মানে সবচেয়ে বেশি কষ্ট পাওয়া user-দের জীবন সহজ করা।

---

## ২.৫ Bottleneck কোথায়? — খুঁজে বের করার পদ্ধতি

Performance কম মানেই worker বাড়াতে হবে — এই ধারণাটা ভুল। আগে বুঝতে হবে **কোথায় সময় যাচ্ছে।**

### Step-by-step debugging:

**১. CPU check করো:**
```bash
htop
```
CPU কি 90%+ এ আছে? তাহলে CPU-bound।
CPU কি 20-30% এ আছে? তাহলে অন্য কোথাও সমস্যা।

**২. Memory check:**
Memory বেশি হলে swap-এ যাচ্ছে কিনা দেখো। Swap hit হলে সব ধীর হয়ে যায়।

**৩. I/O wait দেখো:**
```bash
iostat -x 1
```
`%iowait` বেশি হলে disk I/O bottleneck।

**৪. DB latency measure করো:**
Application log বা DB slow query log দেখো। অনেক সময় সব সমস্যার মূল একটাই slow query।

---

## ২.৬ Workload অনুযায়ী Tuning Strategy

সমস্যার সমাধান নির্ভর করে workload-এর ধরনের উপর।

### CPU-bound workload (heavy computation):

```
workers বাড়াও → প্রতিটি core-এ একটি worker
threads কমাও → GIL contention এড়াও
```

### I/O-bound workload (DB, API call, file read):

```
async worker ব্যবহার করো
connection pooling দাও
thread যোগ করো (blocking I/O-র জন্য)
```

### Mixed workload:

```
async worker + thread executor (CPU কাজের জন্য offload)
```

**সবচেয়ে গুরুত্বপূর্ণ নিয়ম:** আগে measure করো, তারপর change করো। Change-এর আগে ও পরে benchmark তুলনা করো।

---

## ২.৭ DB ও External API Tuning

অনেক ক্ষেত্রেই সবচেয়ে বড় bottleneck application code-এ নয় — DB-তে।

### Connection Pooling

প্রতিটি request-এ নতুন DB connection তৈরি করা ব্যয়বহুল। Connection pool মানে connection গুলো আগে থেকে তৈরি করা থাকে এবং reuse হয়।

```python
# asyncpg দিয়ে pool:
pool = await asyncpg.create_pool(dsn, min_size=5, max_size=20)

async with pool.acquire() as conn:
    result = await conn.fetch("SELECT ...")
```

### Caching (Redis)

বারবার একই data DB থেকে আনার দরকার নেই। Redis-এ cache করো।

```python
cached = await redis.get(f"user:{user_id}")
if cached:
    return json.loads(cached)

data = await db.fetch_user(user_id)
await redis.setex(f"user:{user_id}", 300, json.dumps(data))  # 5 মিনিট cache
return data
```

### Timeout সেট করো

Hanging request পুরো system-কে আটকে রাখতে পারে।

```python
async with httpx.AsyncClient(timeout=5.0) as client:
    response = await client.get(url)
```

---

## ২.৮ সবচেয়ে বড় Tuning ভুলগুলো

এগুলো না জানলে অনেক সময় নষ্ট হয়:

**❌ Worker অন্ধভাবে বাড়ানো**
Worker বাড়ালে memory বাড়ে। Memory শেষ হলে system crash করে বা swap-এ যায়। Measure ছাড়া বাড়াবে না।

**❌ DB bottleneck ignore করা**
Application side optimize করে অনেক সময় নষ্ট হয়, কিন্তু আসল সমস্যা ছিল একটা slow SQL query।

**❌ Async code-এ blocking call রেখে দেওয়া**
`requests.get()` বা `time.sleep()` async function-এর ভেতরে ব্যবহার করলে পুরো event loop block হয়। সব সময় `httpx`, `asyncio.sleep()` ব্যবহার করো।

**❌ Load test না করে production-এ দেওয়া**
"Localhost-এ ভালো চলছে" মানে production-এ ভালো চলবে না। Load test করো।

---

## ২.৯ Scaling — Vertical বনাম Horizontal

এক পর্যায়ে একটাই machine আর যথেষ্ট না।

| | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| মানে | বড় machine কিনো (বেশি RAM, CPU) | বেশি machine যোগ করো |
| সুবিধা | সহজ, code change নেই | প্রায় unlimited scale |
| অসুবিধা | একটা limit আছে, expensive | Load balancer দরকার, complexity বাড়ে |

Real world production system-এ horizontal scaling জেতে। Load balancer (nginx, AWS ALB) সামনে রেখে multiple instance চালানো হয়।

---

## ২.১০ Final Mental Model

সব কিছু মাথায় রেখে একটা সূত্র:

```
Performance = (Concurrency × Efficiency) − Bottlenecks
```

- **Concurrency বাড়াও** — worker, thread, async
- **Efficiency বাড়াও** — connection pool, caching, fast queries
- **Bottleneck সরাও** — slow DB, blocking code, missing index

তিনটা একসাথে না করলে একটার improvement অন্যটায় আটকে যাবে।

---

## চ্যালেঞ্জ: তুমি কী করবে?

এবার একটা real scenario:

**পরিস্থিতি:**
- FastAPI app, async worker
- 2টি worker চলছে
- 1000 concurrent user

**লক্ষণ:**
- p95 latency অনেক বেশি (ধরো 3-4 সেকেন্ড)
- CPU মাত্র ~20% ব্যবহার হচ্ছে
- DB latency বেশি

**প্রশ্নগুলো ভাবো:**

1. Bottleneck কোথায়? (Hint: CPU কম, কিন্তু latency বেশি — এটা কী বলছে?)
2. Worker বাড়ালে কি এই সমস্যা solve হবে?
3. Fix করার strategy কী হবে step-by-step?
4. Caching কোথায় বসালে সবচেয়ে বেশি কাজে লাগবে?

নিজে ভাবো, তারপর answer লেখো। পরের পর্বে detailed solution আসবে।

---

## সারসংক্ষেপ

| বিষয় | মূল কথা |
|---|---|
| Async + CPU | CPU কাজ thread/process-এ offload করো |
| Worker tuning | Measure করো, তারপর adjust করো |
| Load testing | wrk বা locust দিয়ে সংখ্যা দিয়ে কথা বলো |
| Latency | p99 দেখো, average নয় |
| Bottleneck | CPU, memory, I/O, DB — একটা একটা করে rule out করো |
| Scaling | Horizontal scaling real world-এ জেতে |
