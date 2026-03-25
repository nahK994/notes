# Module 2 — Mini Challenge: Solution

## 🧩 Problem Setup

```
1টা async worker
5টা request একসাথে আসে
প্রতিটা request:
    → DB await  = 1s
    → API await = 1s
```

---

## প্রশ্ন ১ — Total time কত?

**উত্তর: ~2 seconds**

### কীভাবে?

```
t=0s  → 5টা request একসাথে start হলো
        → সবাই DB await-এ গেলো (coroutine pause, GIL ছাড়লো)
        → event loop সবার জন্য I/O OS-এর কাছে register করলো

t=1s  → সবার DB done হলো (প্রায় একসাথে)
        → সবাই API await-এ গেলো (আবার pause)

t=2s  → সবার API done → সবাই complete ✅
```

কারণটা পরিষ্কার — প্রতিটা `await`-এ coroutine pause হয়, event loop অন্য request-কে সুযোগ দেয়। তাই ৫টা request-এর waiting time একে অপরের সাথে **overlap** করে। ৫ গুণ সময় লাগে না, সবার অপেক্ষাটা একসাথে হয়।

---

## প্রশ্ন ২ — যদি sync হতো?

```
Req1 → DB(1s) + API(1s) = 2s   [0s  → 2s]
Req2 → DB(1s) + API(1s) = 2s   [2s  → 4s]
Req3 → DB(1s) + API(1s) = 2s   [4s  → 6s]
Req4 → DB(1s) + API(1s) = 2s   [6s  → 8s]
Req5 → DB(1s) + API(1s) = 2s   [8s  → 10s]
```

**Total = 10 seconds**

Sync-এ একটা request শেষ না হওয়া পর্যন্ত পরেরটা শুরুই হতে পারে না। CPU প্রতিটা wait-এর সময় বসে থাকে — ওই সময়টা সম্পূর্ণ নষ্ট।

---

## প্রশ্ন ৩ — Performance gain কোথায়?

**উত্তর: Waiting time overlap হয়েছে।**

```
Sync model:
Req1: [DB wait]─[API wait]
Req2:                     [DB wait]─[API wait]
Req3:                                         [DB wait]─[API wait]
      |──────────────── 6s (3 request) ──────────────────────────|

Async model:
Req1: [DB wait]─[API wait]
Req2: [DB wait]─[API wait]   ← একই সময়ে
Req3: [DB wait]─[API wait]   ← একই সময়ে
      |──────── ~2s ─────────|
```

> **Core Insight:**
> ```
> async = idle time reuse
> ```
> CPU যখন একটার জন্য অপেক্ষা করছে, সেই সময়টায় অন্যটাকে এগিয়ে দেওয়া হচ্ছে।
> Async কোনো magic না — শুধু অপেক্ষার সময়গুলো চালাকভাবে ব্যবহার করা।

---
---

# Module 3 — Gunicorn: The Process Orchestrator

## 3.1 Gunicorn কী?

Gunicorn হলো Python web application-এর জন্য একটা **process manager** — সে নিজে HTTP parse করে না, application-ও চালায় না। সে শুধু worker process তৈরি করে, সেগুলো বাঁচিয়ে রাখে, আর incoming request সেগুলোর মধ্যে ভাগ করে দেয়।

একটা orchestra-র conductor-এর মতো ভাবো — সে নিজে কোনো instrument বাজায় না, কিন্তু সব musician (worker) কখন কীভাবে বাজাবে সেটা সে নিয়ন্ত্রণ করে।

### ❗ Common Misconception

```
Gunicorn = web server ❌

Gunicorn = process manager ✅
    → worker spawn করে
    → worker monitor করে (crash করলে restart)
    → request worker-দের মধ্যে distribute করে
```

Nginx বা Caddy যেখানে actual HTTP connection handle করে (TLS, static files, reverse proxy), Gunicorn সেখানে **শুধু Python process manage করে।** Production-এ সাধারণত `Nginx → Gunicorn → App` এই chain থাকে।

---

## 3.2 Architecture

```
Client Request
      ↓
   Nginx (reverse proxy, optional)
      ↓
  Gunicorn Master Process
      ↓        ↓        ↓
  Worker 1  Worker 2  Worker 3
      ↓        ↓        ↓
   App       App       App
      ↓        ↓        ↓
  Response  Response  Response
```

### দুটো মূল component:

**Master Process**
- Worker তৈরি করে (spawn)
- Worker-দের monitor করে — crash করলে নতুন spawn করে
- Signal handle করে (graceful shutdown, reload)
- নিজে কোনো request handle করে না

**Worker Process**
- Actual request handle করে
- Worker-এর ভেতরে কী হবে সেটা নির্ভর করে **worker type**-এর উপর

---

## 3.3 Worker Classification — আসল ছবিটা

এটাই সবচেয়ে বেশি confusing অংশ। Worker type বোঝার আগে একটা প্রশ্ন: **"একটা worker একসাথে কতটা request handle করতে পারে?"** — এই প্রশ্নের উত্তরই worker classification।

```
Gunicorn Worker Types
│
├── Sync Workers          → একটা request at a time
│   └── sync (default)
│
├── Thread-based Workers  → একাধিক thread, OS-managed concurrency
│   └── gthread
│
└── Async Workers         → event loop বা greenlet, cooperative concurrency
    ├── gevent            (greenlet-based, monkey patching)
    ├── eventlet          (greenlet-based, monkey patching)
    └── UvicornWorker     (asyncio-based, modern)
```

**gthread একটা thread-based worker — async worker না।**

এটা OS thread ব্যবহার করে, event loop না। একটু পরে বিস্তারিত দেখবো।

---

## 3.4 প্রতিটা Worker Type — ভেতর থেকে দেখো

### 🔵 Sync Worker (default)

**কীভাবে কাজ করে:**

```
Worker Process
└── Single Thread
    └── Request A চলছে...
        └── DB query (blocking — thread আটকে)
        └── API call (blocking — thread আটকে)
        └── Response
    └── এখন Request B নেবে (A শেষ হওয়ার পর)
```

**Request flow:**
```
Request আসলো → Worker নিলো → process করলো (blocking) → Response → পরের Request
```

**মূল কথা:** একটাই thread, একটাই request। যতক্ষণ request শেষ না হয়, worker অপেক্ষায়।

**কখন ব্যবহার করবো:**
```
✅ Simple CRUD app যেখানে load বেশি না
✅ App-এ blocking library আছে, async করা সম্ভব না
✅ Debug করা সহজ লাগবে (sequential behavior)
❌ High concurrency দরকার হলে
❌ I/O-heavy app (DB/API call বেশি) হলে
```

---

### 🟡 gthread Worker (Thread-based)

**কীভাবে কাজ করে:**

```
Worker Process
└── Thread Pool (--threads N দিয়ে control করো)
    ├── Thread 1 → Request A (DB wait-এ আছে, GIL ছেড়েছে)
    ├── Thread 2 → Request B (চলছে, GIL নিয়েছে)
    └── Thread 3 → Request C (API wait-এ আছে, GIL ছেড়েছে)
```

**Request flow:**
```
Request আসলো → Thread Pool-এর idle thread নিলো → process করলো → Response
              (কোনো thread idle না থাকলে queue-তে অপেক্ষা)
```

**gthread কি async?** না।

gthread OS thread ব্যবহার করে — event loop না। কিন্তু I/O-bound কাজে (DB, API call) GIL release হয়, তাই multiple thread effectively concurrent হতে পারে। এটাকে বলে **thread-based concurrency** — cooperative নয়, OS-managed।

```
Sync worker:  1 worker → 1 request (always)
gthread:      1 worker → N request (thread count পর্যন্ত, I/O-bound হলে)
```

**GIL-এর সাথে সম্পর্ক:**
```
CPU-bound কাজ:
    → GIL release হয় না
    → threads একসাথে চলতে পারে না
    → gthread এখানে sync-এর চেয়ে ভালো না, বরং overhead বাড়ে

I/O-bound কাজ (DB query, API call, file read):
    → I/O শুরু হলে GIL release হয়
    → অন্য thread চলতে পারে
    → gthread এখানে কাজ করে ✅
```

**কখন ব্যবহার করবো:**
```
✅ App blocking library ব্যবহার করছে (যেমন psycopg2, requests)
✅ App-কে async-এ migrate করা এখনো সম্ভব না
✅ I/O-bound কাজ আছে, কিন্তু async লেখা হয়নি
❌ CPU-bound কাজে (GIL-এর কারণে কোনো benefit নেই)
❌ App async হলে (UvicornWorker অনেক বেশি efficient)
```

---

### 🟠 gevent / eventlet Worker (Greenlet-based Async)

**কীভাবে কাজ করে:**

```
Worker Process
└── gevent Event Loop (libev/libuv based)
    ├── Greenlet A → DB query → yield → paused
    ├── Greenlet B → চলছে
    └── Greenlet C → API call → yield → paused
```

Greenlet হলো Python-এর একটা lightweight coroutine — asyncio coroutine-এর মতো, কিন্তু ভিন্ন implementation।

**Monkey Patching — এটাই gevent-এর বড় trick:**

```python
# gevent worker চালু হলে এটা automatically হয়:
import gevent.monkey
gevent.monkey.patch_all()

# এরপর থেকে:
time.sleep()        → আসলে gevent sleep (non-blocking)
socket.connect()    → আসলে gevent socket (non-blocking)
requests.get()      → আসলে gevent-patched (non-blocking)
```

অর্থাৎ তোমার blocking code না বদলেও gevent সেগুলোকে non-blocking করে দেয়। এটা powerful, কিন্তু কিছু library-তে ঠিকঠাক কাজ নাও করতে পারে।

**কখন ব্যবহার করবো:**
```
✅ Legacy app (Django, Flask) যেটা async করা কঠিন
✅ Monkey patching-এ কোনো সমস্যা নেই
✅ I/O-heavy কিন্তু async rewrite করার সময় নেই
❌ New project হলে — UvicornWorker ব্যবহার করো
❌ Library monkey patching support না করলে
```

---

### 🟢 UvicornWorker (asyncio-based Async — Modern)

**কীভাবে কাজ করে:**

```
Worker Process
└── asyncio Event Loop (Python built-in)
    ├── Coroutine A → await DB → paused
    ├── Coroutine B → await API → paused
    ├── Coroutine C → running
    └── ... হাজারো coroutine possible
```

এটা Python-এর native `asyncio` ব্যবহার করে। gevent-এর মতো monkey patching নেই — তোমাকে explicitly `async/await` দিয়ে code লিখতে হবে।

**গুরুত্বপূর্ণ পার্থক্য gevent-এর সাথে:**

```
gevent:        monkey patching → blocking code চলে (automatically non-blocking হয়)
UvicornWorker: explicit async  → তোমাকে async code লিখতে হবে
               blocking code রাখলে পুরো event loop block হবে ⚠️
```

**কখন ব্যবহার করবো:**
```
✅ FastAPI, Starlette, Litestar — এই frameworks-এর জন্য এটাই standard
✅ New async Python app
✅ High concurrency I/O-heavy app (হাজারো concurrent request)
✅ WebSocket, SSE, long-polling দরকার হলে
❌ Blocking library আছে এবং async alternative নেই
❌ App sync দিয়ে লেখা (gevent বা gthread বেশি সহজ হবে)
```

---

## 3.5 পাশাপাশি তুলনা

| Worker | Type | Concurrency কীভাবে? | একটা worker কতটা request? | App লেখার requirement |
|---|---|---|---|---|
| `sync` | Sequential | নেই | ১টা | Normal sync code |
| `gthread` | Thread-based | OS thread, GIL-bounded | N (thread count) | Normal sync code |
| `gevent` | Async (greenlet) | Cooperative (monkey patch) | হাজারো | Normal sync code (auto-patched) |
| `UvicornWorker` | Async (asyncio) | Cooperative (explicit) | হাজারো | async/await দিয়ে লিখতে হবে |

**"Cooperative" মানে কী?**
```
Cooperative (async workers):
    → task নিজে বলে "আমি অপেক্ষায় আছি, তুমি চলো" (yield/await)
    → কেউ জোর করে থামায় না
    → blocking code থাকলে সবাই আটকে যায়

OS-managed (gthread):
    → OS নিজে thread switch করে
    → কেউ yield না করলেও OS থামিয়ে দিতে পারে
    → blocking code থাকলে শুধু ওই thread আটকে, বাকিরা চলে
```

---

## 3.6 Scaling Strategy

### Workers বাড়ানো — Parallelism

```bash
gunicorn -w 4 app:app
```

```
Master
├── Worker 1  ← আলাদা process, আলাদা Python interpreter, আলাদা GIL
├── Worker 2  ← আলাদা process, আলাদা Python interpreter, আলাদা GIL
├── Worker 3  ← আলাদা process, আলাদা Python interpreter, আলাদা GIL
└── Worker 4  ← আলাদা process, আলাদা Python interpreter, আলাদা GIL
```

- প্রতিটা worker আলাদা OS process → true parallelism
- ৪টা CPU core থাকলে ৪টা request literally একসাথে চলতে পারে
- কিন্তু প্রতিটা worker আলাদা memory নেয় → **memory expensive**

**Rule of thumb:** `workers = (2 × CPU cores) + 1`

### gthread দিয়ে Thread বাড়ানো

```bash
gunicorn -k gthread --threads 4 app:app
```

```
Worker Process (১টা)
└── Thread Pool
    ├── Thread 1 → Request A (DB wait → GIL released)
    ├── Thread 2 → Request B (চলছে)
    ├── Thread 3 → Request C (API wait → GIL released)
    └── Thread 4 → idle
```

একটা worker-এ বেশি concurrency, memory কম।

### Async Worker — High Concurrency-র জন্য

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 2 app:app
```

```
Master
├── Worker 1 (event loop) → ১০০০+ concurrent coroutine
└── Worker 2 (event loop) → ১০০০+ concurrent coroutine
```

২টা worker, কিন্তু প্রতিটা হাজারো request concurrently handle করতে পারে। **App async হতে হবে।**

---

## 3.7 Common Mistake — "Workers বাড়ালেই সব ঠিক হয়"

```bash
gunicorn -w 32 app:app   # ❌ এটা করো না
```

**কেন এটা ভুল?**

```
প্রতিটা worker → আলাদা Python process → আলাদা memory
32 workers × 100MB (app memory) = 3.2 GB RAM শুধু idle worker-এর জন্য!

CPU core যদি ৪টা থাকে:
32 worker কিন্তু ৪টার বেশি একসাথে চলতে পারবে না
→ context switching overhead বাড়বে
→ performance কমবে
```

**সঠিক approach:**
```
bottleneck কোথায় সেটা আগে বোঝো, তারপর tool বেছে নাও।
```

---

## 3.8 Bottlenecks — কোথায় কোথায় আটকে যায়

### CPU-bound bottleneck

```
app-এ heavy calculation আছে (ML inference, image processing)
→ async worker কোনো সাহায্য করবে না (CPU কাজ করছে, অপেক্ষা নেই)
→ workers বাড়াও (আলাদা process = আলাদা GIL = true parallel)
→ workers = CPU core count-এর কাছাকাছি রাখো
```

### GIL bottleneck (gthread + CPU-heavy)

```
gthread ব্যবহার করছো + CPU-heavy কাজ আছে
→ একাধিক thread GIL-এর জন্য fight করবে
→ context switch overhead বাড়বে, performance কমবে
→ Solution: workers বাড়াও (process = আলাদা GIL)
```

### Event loop block (async worker + blocking code)

```
UvicornWorker ব্যবহার করছো, কিন্তু কোথাও blocking code আছে:
    time.sleep(5)        ← পুরো event loop freeze
    requests.get(url)    ← blocking HTTP call
    open("file").read()  ← blocking file I/O

→ ওই worker-এর সব pending coroutine আটকে যাবে
→ Solution: run_in_executor দিয়ে thread pool-এ offload করো
           অথবা async library ব্যবহার করো (httpx, aiofiles, asyncpg)
```

---

## 3.9 Decision Guide — কোনটা কখন?

```
আমার app কেমন?
│
├── App sync দিয়ে লেখা (blocking code, পুরনো library)
│   │
│   ├── Load বেশি না, simplicity চাই
│   │   └── sync worker + workers বাড়াও
│   │
│   ├── I/O-heavy (DB/API call বেশি), async করা কঠিন
│   │   └── gthread worker (--threads N)
│   │       (I/O-তে GIL release হয়, threads concurrent হয়)
│   │
│   └── Legacy app, I/O-heavy, rewrite করা সম্ভব না
│       └── gevent worker
│           (monkey patching দিয়ে auto non-blocking)
│
└── App async দিয়ে লেখা (FastAPI, Starlette, async/await)
    │
    ├── I/O-heavy (DB/API call, WebSocket, high concurrency)
    │   └── UvicornWorker ✅ (best choice)
    │       + workers = CPU core count (parallelism-ও পাবে)
    │
    └── CPU-heavy কাজও আছে
        └── UvicornWorker + run_in_executor
            (CPU কাজ process pool-এ offload করো)
```

---

## 3.10 Final Mental Model

```
Gunicorn = Manager      → কে কী করবে ঠিক করে, workers বাঁচিয়ে রাখে
Worker   = Engine       → কাজটা আসলে করে

Engine type বদলালে behavior বদলায়:

sync           → sequential, 1 req at a time, simple
gthread        → thread pool, I/O-bound-এ concurrent, GIL-bounded
gevent         → greenlet event loop, legacy app-এর জন্য, monkey patching
UvicornWorker  → asyncio event loop, modern async app, highest concurrency
```

**Gunicorn নিজে কোনো performance magic করে না** — সে শুধু সঠিক engine-কে সঠিকভাবে চালু রাখে। Performance আসে সঠিক worker type বেছে নেওয়া থেকে।

---

# ⚔️ Module 3 — Mini Challenge

## 🧩 Scenario

একটা FastAPI app, Gunicorn দিয়ে চালানো হচ্ছে। তিনটা আলাদা configuration-এ একই workload দেওয়া হলো।

```
App:      FastAPI
Gunicorn: 1 worker
Workload: 5 request একসাথে, প্রতিটা request = 2 seconds
```

---

### Case A — Sync Worker

```
worker:   sync (default)
request:  blocking (time.sleep(2))
count:    5
```

### Case B — gthread Worker

```
worker:   gthread
threads:  5
request:  blocking (time.sleep(2))
count:    5
```

### Case C — UvicornWorker

```
worker:   uvicorn.workers.UvicornWorker
request:  non-blocking (await asyncio.sleep(2))
count:    5
```

---

## ❓ Questions

**1.** Case A-তে total time কত?

**2.** Case B-তে total time কত? Case A-র চেয়ে কেন আলাদা (বা একই)?

**3.** Case C-তে total time কত? Case B-র চেয়ে কেন আলাদা (বা একই)?

**4.** কোন case-এ CPU utilization সবচেয়ে কম, কিন্তু throughput সবচেয়ে বেশি? কেন?

---

> Hint: তিনটা case-এ concurrency-র mechanism আলাদা। শুধু সময় না, **কীভাবে** সময়টা কাটছে সেটাও ভাবো।

---

👉 তোমার reasoning লেখো — আমি razor দিয়ে কাটবো 🔍