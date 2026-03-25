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
        → সবাই DB await-এ গেলো (pause, GIL ছাড়লো)
        → event loop idle নেই, সবার জন্য I/O register করলো OS-এর কাছে

t=1s  → সবার DB done হলো (প্রায় একসাথে)
        → সবাই API await-এ গেলো (আবার pause)

t=2s  → সবার API done → সবাই complete ✅
```

কারণটা পরিষ্কার — প্রতিটা `await`-এ coroutine pause হয়, event loop অন্য request-কে সুযোগ দেয়। তাই ৫টা request-এর waiting time একে অপরের সাথে **overlap** করে। ৫ গুণ সময় লাগে না, সবার অপেক্ষাটা একসাথে হয়।

---

## প্রশ্ন ২ — যদি sync হতো?

```
Req1 → DB(1s) + API(1s) = 2s   [0s → 2s]
Req2 → DB(1s) + API(1s) = 2s   [2s → 4s]
Req3 → DB(1s) + API(1s) = 2s   [4s → 6s]
Req4 → DB(1s) + API(1s) = 2s   [6s → 8s]
Req5 → DB(1s) + API(1s) = 2s   [8s → 10s]
```

**Total = 10 seconds**

Sync-এ একটা request শেষ না হওয়া পর্যন্ত পরেরটা শুরুই হতে পারে না। CPU প্রতিটা `sleep`/`wait`-এর সময় বসে থাকে — ওই সময়টা সম্পূর্ণ নষ্ট।

---

## প্রশ্ন ৩ — Performance gain কোথায়?

**উত্তর: Waiting time overlap হয়েছে।**

```
Sync model:
Req1: [DB wait]─[API wait]
Req2:                     [DB wait]─[API wait]
Req3:                                         [DB wait]─[API wait]
      |────────────── 6s (3 request) ──────────────────|

Async model:
Req1: [DB wait]─[API wait]
Req2: [DB wait]─[API wait]   ← একই সময়ে
Req3: [DB wait]─[API wait]   ← একই সময়ে
      |────── ~2s ──────|
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

Nginx বা Caddy যেখানে actual HTTP connection handle করে (TLS, static files, reverse proxy), Gunicorn সেখানে **শুধু Python process manage করে।** Production-এ সাধারণত Nginx → Gunicorn → App এই chain থাকে।

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

## 3.3 Request Flow — Worker Type অনুযায়ী

Worker type পরিবর্তন করলে আচরণ আমূল বদলে যায়। এটাই Gunicorn-এর সবচেয়ে গুরুত্বপূর্ণ concept।

### Sync Worker (default)

```
Request আসলো
    ↓
Worker নিলো
    ↓
DB query (blocking — worker আটকে আছে)
    ↓
API call (blocking — worker আটকে আছে)
    ↓
Response পাঠালো
    ↓
Worker এখন নতুন request নিতে পারবে
```

**মূল কথা:** Sync worker একটার পর একটা request handle করে। যতক্ষণ একটা request শেষ না হয়, worker সেখানেই আটকে।

### Async Worker (uvicorn/gevent)

```
Request আসলো
    ↓
Worker-এর event loop নিলো
    ↓
DB query শুরু → await → event loop অন্য request নিলো
    ↓
API call শুরু → await → আবার অন্য request
    ↓
DB/API complete → resume → Response পাঠালো
```

**মূল কথা:** একটা async worker একসাথে অনেক request handle করতে পারে — যতক্ষণ প্রতিটা `await` করছে, ততক্ষণ অন্যরা চলতে পারে।

---

## 3.4 Worker Types — বিস্তারিত

| Worker Type | কীভাবে কাজ করে | কখন ব্যবহার করবো |
|---|---|---|
| `sync` (default) | ১ worker = ১ request at a time | Simple app, blocking code |
| `gthread` | ১ worker = multiple thread | Blocking I/O, thread-safe code |
| `gevent` | ১ worker = event loop (greenlet) | I/O-heavy, legacy async |
| `uvicorn.workers.UvicornWorker` | ১ worker = asyncio event loop | FastAPI, Starlette, modern async app |

### Sync Worker ভেতরে:

```
Worker Process
└── একটামাত্র thread
    └── একটাই request handle করছে
    └── blocking করলে পুরো worker আটকে
```

### gthread Worker ভেতরে:

```
Worker Process
└── Thread Pool (--threads N)
    ├── Thread 1 → Request A (DB wait-এ আছে)
    ├── Thread 2 → Request B (চলছে)
    └── Thread 3 → Request C (API wait-এ আছে)
```

GIL-এর কারণে CPU-bound কাজে একাধিক thread একসাথে চলতে পারে না, কিন্তু I/O-bound কাজে (DB, API call) GIL release হয়, তাই threads effectively concurrent।

### UvicornWorker ভেতরে:

```
Worker Process
└── asyncio Event Loop
    ├── Coroutine A → await DB → paused
    ├── Coroutine B → await API → paused
    ├── Coroutine C → running
    └── ... হাজারো coroutine possible
```

Thread overhead নেই, GIL-এর সমস্যা নেই (I/O-bound-এ), এবং একটা worker অনেক বেশি concurrent request handle করতে পারে।

---

## 3.5 Scaling Strategy

### Workers বাড়ানো — Parallelism

```bash
gunicorn -w 4 app:app
```

```
Master
├── Worker 1  ← আলাদা process, আলাদা Python interpreter
├── Worker 2  ← আলাদা process, আলাদা Python interpreter
├── Worker 3  ← আলাদা process, আলাদা Python interpreter
└── Worker 4  ← আলাদা process, আলাদা Python interpreter
```

- প্রতিটা worker আলাদা OS process → true parallelism (GIL আটকাতে পারে না)
- ৪টা CPU core থাকলে ৪টা request literally একসাথে চলতে পারে
- কিন্তু প্রতিটা worker আলাদা memory নেয় → **memory expensive**

**Rule of thumb:** `workers = (2 × CPU cores) + 1`

### Threads বাড়ানো — Concurrency (I/O-bound-এ)

```bash
gunicorn -k gthread --threads 4 app:app
```

```
Worker Process (১টা)
└── 4টা thread
    ├── Thread 1 → Request A
    ├── Thread 2 → Request B
    ├── Thread 3 → Request C
    └── Thread 4 → Request D
```

- একটা worker-এই বেশি concurrency
- Memory কম লাগে (thread share করে memory)
- কিন্তু GIL-এর কারণে CPU-bound কাজে কোনো benefit নেই

### Async Worker — High Concurrency I/O-bound-এর জন্য

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 2 app:app
```

```
Master
├── Worker 1 (event loop) → ১০০০+ concurrent coroutine
└── Worker 2 (event loop) → ১০০০+ concurrent coroutine
```

- ২টা worker, কিন্তু প্রতিটা হাজারো request concurrently handle করতে পারে
- I/O-heavy app-এর জন্য সবচেয়ে efficient
- **Requirement:** app-কে async হতে হবে (FastAPI, Starlette, etc.)

---

## 3.6 Trade-offs

| Strategy | সুবিধা | সমস্যা | কখন ব্যবহার |
|---|---|---|---|
| Workers বাড়ানো | True parallel, CPU-bound-এও কাজ করে | Memory heavy, প্রতি worker = আলাদা process | CPU-bound app, বা high isolation দরকার হলে |
| Threads বাড়ানো | Lightweight, একই memory share | GIL limitation, CPU-bound-এ কোনো gain নেই | Blocking I/O-heavy app |
| Async worker | অনেক বেশি concurrent, memory efficient | App-কে async লিখতে হবে, blocking code থাকলে সব আটকে | I/O-heavy modern async app |

---

## 3.7 Common Mistake — "Workers বাড়ালেই সব ঠিক হয়"

```bash
gunicorn -w 32 app:app   # ❌ এটা করো না
```

**কেন এটা ভুল?**

```
প্রতিটা worker → আলাদা Python process → আলাদা memory
32 workers × 100MB (app memory) = 3.2 GB RAM শুধু idle worker-এর জন্য!

আর CPU core যদি ৪টা থাকে:
32 worker কিন্তু ৪টার বেশি একসাথে চলতে পারবে না
→ context switching overhead বাড়বে
→ performance কমবে
```

**সঠিক approach:**
```
bottleneck কোথায় সেটা আগে বোঝো:
    I/O-bound? → async worker অথবা gthread
    CPU-bound? → workers = CPU core count এর কাছাকাছি রাখো
    Memory tight? → workers কম, threads বেশি
```

---

## 3.8 Bottlenecks — কোথায় কোথায় আটকে যায়

### CPU-bound bottleneck

```
app-এ heavy calculation আছে (ML inference, image processing, etc.)
→ workers বাড়িয়ে যতটুকু CPU core আছে ততটুকু parallel করা যাবে
→ এর বেশি workers দিলে context switching overhead বাড়বে
→ Async worker এখানে কোনো সাহায্য করে না (CPU কাজ করছে, অপেক্ষা নেই)
```

### GIL bottleneck (thread contention)

```
gthread ব্যবহার করছো + CPU-heavy কাজ আছে
→ একাধিক thread GIL-এর জন্য fight করবে
→ context switch overhead বাড়বে
→ single thread-এর চেয়ে slow হতে পারে
→ Solution: CPU-bound কাজ → workers বাড়াও (process = আলাদা GIL)
```

### Event loop block

```
async worker ব্যবহার করছো, কিন্তু কোথাও blocking code আছে:
    time.sleep(5)           ← পুরো event loop আটকে
    requests.get(url)       ← blocking HTTP call
    open("file").read()     ← blocking file I/O

→ ওই worker-এর সব pending request আটকে যাবে
→ Solution: run_in_executor দিয়ে thread pool-এ offload করো
           অথবা async library ব্যবহার করো (httpx, aiofiles, etc.)
```

---

## 3.9 Final Mental Model

```
Gunicorn = Manager (কে কী করবে সেটা ঠিক করে)
Worker   = Execution Engine (কাজটা আসলে করে)

Engine বদলালে behaviour বদলায়:
    sync engine      → sequential, simple, limited
    gthread engine   → concurrent threads, GIL-bounded
    async engine     → event loop, high concurrency, non-blocking
```

**Gunicorn নিজে কোনো performance magic করে না** — সে শুধু সঠিক engine-কে সঠিকভাবে চালু রাখে।

---
---

# ⚔️ Module 3 — Mini Challenge

## Scenario

```
Gunicorn
└── 2 sync worker
    └── প্রতিটা request = 2 seconds (blocking)
    └── 6টা request একসাথে আসলো
```

## প্রশ্ন

**1. Total time কত লাগবে?**

**2. যদি `gthread` দাও (threads=2), কী change হবে?**

**3. যদি async worker দাও, better performance পেতে কী assumption লাগবে?**
