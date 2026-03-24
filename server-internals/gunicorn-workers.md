# 🐾 Module 4: Gunicorn Worker Zoo — সব Worker, সব Scenario

---

## Module 4-এর আগের Mini Challenge-এর উত্তর

আগের module শেষে প্রশ্ন ছিল:

> FastAPI app, Gunicorn, 1 worker, 5টা request — প্রতিটায় 1 সেকেন্ড DB await + 0.1 সেকেন্ড calculation।

**Case A — Sync worker, মোট সময়:** 5.5 সেকেন্ড।
Sequential। প্রতিটা request 1.1s নেয়, পরেরটা আগেরটা শেষ না হলে শুরু হয় না।

**Case B — Gthread (threads=5), মোট সময়:** প্রায় 1.1 সেকেন্ড।
5টা thread একসাথে 5টা request নেয়। DB wait-এর সময় অন্য thread-ও wait করছে — কিন্তু সবার wait overlap করে, তাই মোট সময় একটা request-এর মতোই।

**Case C — UvicornWorker async, মোট সময়:** প্রায় 1.1 সেকেন্ড।
Event loop-এ সব coroutine একসাথে DB await করে। Case B-র মতো সময়, কিন্তু 5টা OS thread-এর বদলে 5টা lightweight coroutine — memory অনেক কম।

**Case D — UvicornWorker কিন্তু DB driver blocking, মোট সময়:** 5.5 সেকেন্ড।
Async worker থাকলেও blocking DB driver পুরো event loop আটকে দেয়। Case A-র মতোই performance। Async লেখা, কিন্তু সুবিধা শূন্য।

**Key insight:** Case B এবং C-র সময় একই, কিন্তু C-তে 500 concurrent request-এ threads লাগবে 500টা (প্রচুর memory), আর C-তে লাগবে 500টা coroutine (অনেক কম memory)। Scale-এ গিয়ে পার্থক্যটা স্পষ্ট হয়।

---

## এই Module-এ কী দেখব?

Gunicorn-এ worker type বদলানো মানে শুধু একটা flag পরিবর্তন না — পুরো execution model বদলে যায়। Gunicorn-এ বেশ কয়েকটা worker type আছে, প্রতিটা ভিন্ন technology-র উপর দাঁড়ানো।

কিন্তু সেগুলো বোঝার আগে একটা fundamental প্রশ্নের উত্তর দরকার — Greenlet কী? কারণ Gunicorn-এর অনেক worker Greenlet-এর উপর নির্মিত।

---

# 🧬 4.1 Greenlet — Worker বোঝার আগে এটা বোঝো

Python-এ concurrency-র তিনটা মূল উপায় আছে: **process**, **thread**, এবং **greenlet**। Greenlet-টা অনেকের কাছে কম পরিচিত, কিন্তু Gunicorn-এর বড় একটা অংশ এর উপর নির্ভরশীল।

**OS Thread বনাম Greenlet — মূল পার্থক্য:**

OS thread তৈরি করলে Operating System সেটা manage করে। OS জানে এই thread আছে, OS scheduler এটাকে CPU-তে schedule করে, OS এটার জন্য stack memory allocate করে (সাধারণত 1-8 MB)। Context switch মানে OS একটা thread-কে pause করে অন্যটাকে CPU-তে তোলে — এই কাজে কিছুটা overhead আছে।

Greenlet সম্পূর্ণ আলাদা। এটা OS-এর কাছে invisible। OS জানেও না যে multiple greenlet চলছে। Python-এর user space-এ, application level-এ, একটা scheduler আছে যেটা greenlet-গুলো manage করে। একটা greenlet চলছে, যখন সে বলে "আমি থামছি," তখন scheduler অন্য greenlet-কে চালু করে।

```
OS Thread:
OS ← জানে → Thread 1, Thread 2, Thread 3
OS scheduler context switch করে
প্রতিটা thread ~1-8 MB stack

Greenlet:
OS ← জানে না → (শুধু একটা OS thread দেখে)
Python-level scheduler greenlet switch করে
প্রতিটা greenlet ~few KB
```

**এর ফলে:**
- Greenlet তৈরি করা OS thread-এর চেয়ে অনেক সস্তা
- Greenlet switch করা OS context switch-এর চেয়ে অনেক দ্রুত
- একটা process-এ হাজারো greenlet চলতে পারে

**কিন্তু একটা গুরুত্বপূর্ণ সীমা আছে:** Greenlet cooperative। মানে একটা greenlet নিজে না বললে scheduler তাকে থামাতে পারে না। OS thread preemptive — OS যেকোনো সময় যেকোনো thread থামাতে পারে।

এই cooperative nature-টাই gevent এবং eventlet-এর কাজের ভিত্তি। এবং এটাই তাদের সীমাবদ্ধতাও।

এখন প্রতিটা Gunicorn worker দেখা যাক।

---

# 🪵 4.2 Sync Worker — একটাই কাজ, একটাই সময়

**`-k sync` (default)**

এটা Gunicorn-এর default worker। আলাদাভাবে specify না করলে এটাই চলে।

**কীভাবে কাজ করে:**

প্রতিটা worker process একটাই request handle করে। Request আসলে worker নেয়, শুরু থেকে শেষ পর্যন্ত handle করে, response পাঠায়, তারপর পরের request নেয়। মাঝখানে যদি DB query-তে 500ms wait হয়, worker সেই 500ms বসে থাকে।

```bash
gunicorn -w 4 myapp:app
# 4টা worker process, প্রতিটা একটা request
# maximum concurrent: 4
```

**Worker সংখ্যার সূত্র:**

Gunicorn-এর official recommendation হলো `(2 × CPU_CORES) + 1`। 4 core machine-এ 9টা worker। কিন্তু এটা starting point — app-এর workload অনুযায়ী adjust করতে হয়।

**কখন ভালো:**
CPU-heavy কাজে sync worker সবচেয়ে সঠিক। Image processing, video transcoding, ML inference — এখানে async দিয়ে কিছু পাওয়ার নেই, কারণ wait নেই, শুধু computation। Multiple sync worker মানে multiple CPU core সত্যিকার অর্থে parallel-এ কাজ করছে।

**কখন খারাপ:**
I/O-heavy workload-এ sync worker waste হয়। প্রতিটা DB query-তে worker idle বসে থাকে। 100 concurrent user মানে 100টা worker দরকার, যেটা impractical।

---

# 🧵 4.3 Gthread Worker — Thread দিয়ে Concurrency

**`-k gthread`**

Gthread একটা hybrid approach। Process-এর মতো isolation, কিন্তু প্রতিটা process-এর ভেতরে multiple thread। একটা thread blocked থাকলে অন্য thread কাজ করতে পারে।

**কীভাবে কাজ করে:**

Application code একবার load হয় per process। সেই process-এর ভেতরে thread pool থাকে। Request আসলে pool থেকে একটা thread নেওয়া হয়, request শেষ হলে thread pool-এ ফিরে যায়।

```bash
gunicorn -k gthread --threads 4 -w 2 myapp:app
# 2 process × 4 thread = 8 concurrent request
```

**GIL-এর বাস্তবতা:**

Python-এর GIL (Global Interpreter Lock) একটা সময়ে একটাই thread-কে Python bytecode চালাতে দেয়। তাহলে multiple thread-এ কী লাভ?

লাভ আছে I/O-তে। যখন একটা thread DB query করে OS-এর network stack-কে call দেয়, তখন GIL release হয়। OS network response-এর অপেক্ষায় থাকে, এই সময়ে Python runtime অন্য thread-কে চালাতে পারে। DB response আসলে আবার GIL নিয়ে Python code চালায়।

CPU-heavy কাজে GIL release হয় না (Python bytecode চলছে), তাই gthread সেখানে কাজ করে না।

**Sync-এর চেয়ে কতটা ভালো:**

Sync worker-এ 100 concurrent request মানে 100 process (প্রচুর memory)। Gthread-এ `--threads 20` দিলে 5 process-এই 100 concurrent handle হয় — অনেক কম memory।

**কখন ভালো:**
- Legacy Django app যেটা sync ORM ব্যবহার করে
- Mixed workload — কিছু I/O, কিছু computation
- Async ecosystem-এ migrate না করে concurrency বাড়াতে চাইলে

---

# 🌿 4.4 Gevent Worker — Green Thread-এর Magic

**`-k gevent`**

Gevent Greenlet-এর উপর নির্মিত। এটা Gunicorn-এর সবচেয়ে "magical" worker — পুরনো sync code-কে async-like করে দেয়।

**কীভাবে কাজ করে:**

Gevent worker start হওয়ার সময় Python-এর standard library-র blocking functions গুলো replace করে দেয় non-blocking version দিয়ে। এটাকে বলে **monkey patching**।

```python
# Gevent এটা automatically করে:
import socket
socket.socket = gevent_non_blocking_socket  # replace!

import time
time.sleep = gevent_non_blocking_sleep      # replace!
```

ফলে তোমার code যখন `time.sleep(2)` call করে, সে আসলে gevent-এর non-blocking version call করছে। Gevent তখন বলে — "এই greenlet 2 সেকেন্ড wait করুক, আমি অন্য greenlet চালাই।"

তোমার code পরিবর্তন করতে হয়নি, কিন্তু সে async-এর মতো behave করছে।

```bash
gunicorn -k gevent --worker-connections 1000 -w 4 myapp:app
# --worker-connections = একটা worker-এ সর্বোচ্চ concurrent greenlet
```

**Monkey patching-এর সীমা:**

Gevent শুধু সেই libraries-গুলো patch করতে পারে যেগুলো Python-এর standard socket বা I/O mechanism use করে। কিছু C extension libraries আছে যেগুলো directly OS call করে — gevent সেগুলো patch করতে পারে না। এই libraries-গুলো gevent-এ blocking-ই থাকে।

এটাই gevent-এর সবচেয়ে বড় debugging nightmare — কখন blocking কখন non-blocking সেটা code দেখে বোঝা যায় না।

**CPU-bound কাজ gevent-এ বিপজ্জনক:**

Greenlet cooperative। CPU-heavy কাজ চলার সময় greenlet নিজে yield না করলে অন্য কেউ চলতে পারে না। `for i in range(10_000_000): result += i` এই loop চলার সময় পুরো gevent worker আটকে থাকবে।

**কখন ভালো:**
- পুরনো Django বা Flask app যেটা async করা সম্ভব না
- I/O-bound workload কিন্তু modern async library নেই
- Quick win দরকার, বড় migration সম্ভব না

---

# ⚡ 4.5 Eventlet Worker — Gevent-এর পুরনো ভাই

**`-k eventlet`**

Eventlet Gevent-এর মতোই — green thread, monkey patching, cooperative multitasking। দুটো আলাদা library, কিন্তু same concept।

```bash
gunicorn -k eventlet -w 4 myapp:app
```

**কেন এখন ব্যবহার করা উচিত না:**

Gunicorn-এর official documentation-এ এখন লেখা আছে — **Eventlet deprecated, Gunicorn 26.0-তে remove হবে।** কারণ eventlet project আর actively maintained না।

যদি কোনো পুরনো codebase-এ eventlet দেখো, সেটা gevent বা gthread-এ migrate করা উচিত। নতুন project-এ eventlet ব্যবহার করবে না।

---

# 🌪️ 4.6 Tornado Worker — Framework-Specific Worker

**`-k tornado`**

Tornado একটা Python async web framework যার নিজস্ব IOLoop আছে। Tornado worker specifically Tornado framework-এর জন্য তৈরি।

```bash
pip install gunicorn[tornado]
gunicorn -k tornado -w 4 myapp:app
```

**কীভাবে কাজ করে:**

Tornado worker প্রতিটা worker process-এ Tornado-এর নিজস্ব event loop চালায়। Tornado app-এর async handler গুলো সেই loop-এ run করে।

**বর্তমান relevance:**

Tornado-র নিজস্ব production server আছে (`tornado.httpserver`), এবং এটা ASGI-compatible। Gunicorn-এর tornado worker ব্যবহার করার চেয়ে Tornado-র নিজের server বা Uvicorn দিয়ে ASGI mode-এ চালানো বেশি modern approach।

নতুন project-এ tornado worker-এর তেমন use case নেই। পুরনো Tornado-based app maintain করতে গেলে দেখা যেতে পারে।

---

# 🚫 4.7 Gaiohttp Worker — Deprecated, শুধু জানার জন্য

**`-k gaiohttp` (Deprecated — ব্যবহার করবে না)**

Python 3-এর asyncio এবং aiohttp library-র জন্য তৈরি হয়েছিল। কিন্তু Gunicorn 19.8-এ deprecated হয়েছে এবং পরে remove হয়েছে।

Alternative হলো `aiohttp.worker.GunicornWebWorker` — aiohttp প্যাকেজের নিজস্ব Gunicorn-compatible worker।

```bash
# ❌ পুরনো, কাজ করে না
gunicorn -k gaiohttp myapp:app

# ✔️ aiohttp app-এর জন্য সঠিক approach
pip install aiohttp
gunicorn -k aiohttp.worker.GunicornWebWorker myapp:app
```

---

# ⚡ 4.8 UvicornWorker — Modern Async (Third-party)

**`-k uvicorn.workers.UvicornWorker`**

এটা Gunicorn-এর built-in worker না — Uvicorn প্যাকেজের সাথে আসে। কিন্তু এটাই আজকের সবচেয়ে popular modern async setup।

```bash
pip install uvicorn
gunicorn -k uvicorn.workers.UvicornWorker -w 4 myapp:app
```

**কীভাবে কাজ করে:**

প্রতিটা worker process-এ Uvicorn-এর asyncio event loop চলে। ASGI protocol সম্পূর্ণভাবে support করে। FastAPI, Starlette, Django (ASGI mode) — সব কিছু এই worker-এ চলে।

**Gunicorn + UvicornWorker কেন?**

শুধু Uvicorn দিয়েও app run করা যায়। তাহলে Gunicorn কেন?

Gunicorn দেয় process management — worker crash হলে auto-restart, graceful reload (code deploy করার সময় connection drop ছাড়া restart), signal handling (SIGTERM, SIGHUP), worker count dynamically বাড়ানো-কমানো। Uvicorn শুধু request handle করে, process management করে না।

Production-এ এই combination standard:
```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 4 --timeout 60 myapp:app
```

**সবচেয়ে বড় শর্ত:**

পুরো code stack async হতে হবে — async DB driver, async HTTP client, async cache। একটা blocking call পুরো event loop আটকে দেয়।

---

# 📊 4.9 সব Worker পাশাপাশি — Quick Reference

| Worker | Technology | Concurrency Model | Built-in? | Status |
|--------|-----------|-------------------|-----------|--------|
| sync | OS process | 1 request / worker | ✅ | Active, default |
| gthread | OS thread | N thread / worker | ✅ | Active |
| gevent | Greenlet | Cooperative, 1000s/worker | ✅ | Active |
| eventlet | Greenlet | Cooperative | ✅ | **Deprecated** |
| tornado | Tornado IOLoop | Async per worker | ✅ | Legacy |
| gaiohttp | aiohttp asyncio | Async per worker | ❌ Removed | **Removed** |
| UvicornWorker | asyncio event loop | Async per worker | ❌ Third-party | Active, recommended |

---

# 🏗️ 4.10 Real-World Scenarios — বিস্তারিত

এখন scenario-গুলো real production situation হিসেবে দেখি — শুধু "কোন worker" না, কেন সেই সিদ্ধান্ত নেওয়া হলো, trade-off কী, এবং কীভাবে scale হবে সেটা সহ।

---

## Scenario A: ই-কমার্স Django App — Moderate Traffic

**Context:**
একটা Django-based e-commerce site। Product listing, cart, checkout — সব standard CRUD। Database হলো PostgreSQL, ORM হলো Django-র built-in (sync)। Traffic সাধারণত 200-400 concurrent user, peak-এ 800 পর্যন্ত।

**কী কী I/O হচ্ছে:**
- Product page: 3-4টা DB query (products, images, related items)
- Checkout: DB read + payment gateway API call + DB write
- Cart: Redis cache read/write

সব I/O sync। Django ORM sync, Redis library sync।

**Worker choice: Gthread**

```bash
gunicorn -k gthread --threads 4 -w 4 myapp.wsgi:application
# 4 process × 4 thread = 16 concurrent request
```

**কেন Gthread:**

Django ORM-কে async করতে গেলে পুরো codebase পরিবর্তন করতে হবে — বিশাল effort। কিন্তু gthread দিয়ে DB wait-এর সময় অন্য thread কাজ করতে পারে। প্রতিটা DB query ~50ms হলে, thread সেই 50ms অন্য request serve করতে পারছে।

**কেন UvicornWorker না:**
Django async-ORM দিয়ে rewrite করতে হবে। পুরো codebase migration অনেক বড় risk।

**কেন Gevent না:**
Gevent সম্ভব, কিন্তু monkey patching-এর hidden complexity আছে। Gthread এখানে simpler এবং predictable।

**Scale করতে হলে:**
Thread count বাড়াও (`--threads 8`), তারপর worker count (`-w 8`)। Memory monitor করো — প্রতিটা Django worker ~150-300 MB নিতে পারে। DB connection pool size-ও বাড়াতে হবে (threads × workers = max connections)।

---

## Scenario B: FastAPI Microservice — High Concurrency API

**Context:**
একটা payment processing microservice। প্রতিটা request করে:
- PostgreSQL থেকে user data fetch (asyncpg — 30ms)
- External payment gateway API call (httpx — 200-500ms, variable)
- Redis-এ transaction log (aioredis — 5ms)
- PostgreSQL-এ transaction save (asyncpg — 20ms)

Traffic: 2000+ concurrent request। Response time critical।

**Worker choice: UvicornWorker**

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 4 --timeout 30 myapp:app
# 4 CPU core machine, 4 worker, প্রতিটায় async event loop
```

**কেন UvicornWorker:**

প্রতিটা request ~250-600ms নেয়, কিন্তু এই সময়ের বেশিরভাগ I/O wait। Payment gateway-র সেই 200-500ms-এ worker idle বসে থাকলে waste। Async দিয়ে সেই সময়ে অন্য request চলে।

2000 concurrent request-এ:
- Sync: 2000 worker লাগত (impractical)
- Gthread: 2000 thread লাগত (প্রচুর memory)
- UvicornWorker: 4 worker, প্রতিটায় ~500 coroutine (practical)

**Critical requirement:**
সব library async হতে হবে। `asyncpg` (না `psycopg2`), `httpx` বা `aiohttp` (না `requests`), `aioredis` (না `redis`)। একটাও sync library থাকলে পুরো gain নষ্ট।

**Scale করতে হলে:**
Worker count CPU core-এর সমান রাখো। Horizontal scaling — multiple server, load balancer (Nginx/HAProxy)। Payment gateway-র rate limit হলে request queuing (Celery বা RQ) যোগ করো।

---

## Scenario C: Image Processing API

**Context:**
একটা API যেখানে user image upload করে এবং resize + watermark করে পায়। প্রতিটা image processing ~800ms-1.5s CPU নেয় (Pillow library)। Traffic 50-100 concurrent।

**Worker choice: Sync Worker, multiple processes**

```bash
gunicorn -w 8 myapp:app  # 4 core machine → 8 worker
```

**কেন Sync:**

এখানে কোনো I/O wait নেই বলতে গেলে — image processing pure CPU। Async দিলে কোনো gain নেই কারণ কোনো `await` point নেই। Multiple sync process মানে multiple CPU core সত্যিকার অর্থে parallel-এ কাজ করছে।

**কেন Async bad হবে এখানে:**
UvicornWorker-এ async handler-এ `do_heavy_processing()` call করলে event loop সেই 800ms-1.5s block থাকবে। 4 worker থাকলে একসাথে 4টার বেশি request handle হবে না, এবং বাকি সব request সেই পুরো সময় wait করবে।

**Async দিয়েও করা সম্ভব, কিন্তু তখন offload করতে হবে:**
```python
# ProcessPoolExecutor-এ CPU কাজ offload করো
loop = asyncio.get_event_loop()
result = await loop.run_in_executor(executor, process_image, data)
```
কিন্তু এই complexity-র চেয়ে simple sync worker-ই এই use case-এ সহজ এবং কার্যকর।

**Scale করতে হলে:**
CPU bound হওয়ায় একটা machine-এ worker সংখ্যা core-এর বেশি করলে লাভ নেই — context switching বাড়বে। Horizontal scaling দরকার — আরও server। অথবা dedicated image processing service (Celery worker) আলাদা করো।

---

## Scenario D: Real-time Chat Application

**Context:**
একটা customer support chat system। WebSocket connection। Thousands of users একসাথে connected, প্রতিটা connection long-lived। Messages real-time forward হয়।

**Worker choice: UvicornWorker (ASGI mandatory)**

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 4 myapp:app
```

**কেন ASGI mandatory:**

WebSocket WSGI-তে কাজ করে না। WSGI model "request এলো, response দাও, শেষ।" WebSocket connection ঘণ্টার পর ঘণ্টা open থাকতে পারে। WSGI-তে এই connection খোলা থাকলে worker occupied — হাজারো user মানে হাজারো worker, যেটা impossible।

ASGI-তে connected user মানে একটা async task যেটা background-এ wait করছে। সেই wait-এর সময় worker অন্য connection-ও serve করতে পারছে। ফলে হাজারো WebSocket connection একটা worker-এই সম্ভব।

**Scale করতে হলে:**
WebSocket state (কে কোন room-এ আছে) centralized store-এ রাখতে হবে (Redis Pub/Sub)। Multiple server-এ scale করলে server-এর মধ্যে message routing-এর জন্য message broker দরকার।

---

## Scenario E: Legacy Flask App, পুরনো Codebase

**Context:**
5 বছরের পুরনো Flask app। Hundreds of endpoints। SQLAlchemy sync ORM, `requests` library দিয়ে external API call। Async করার plan আছে কিন্তু 6 মাস লাগবে। এখনই concurrency বাড়ানো দরকার।

**Worker choice: Gevent**

```bash
pip install gevent
gunicorn -k gevent --worker-connections 500 -w 4 myapp:app
```

**কেন Gevent:**

Code পরিবর্তন না করেই concurrency পাওয়া যাচ্ছে। `requests.get()` gevent-এর monkey patching-এর কারণে blocking থাকবে না — gevent এটাকে non-blocking করে দেবে। SQLAlchemy-ও gevent-compatible।

**কী কী risk:**

Monkey patching সব সময় perfect হয় না। কোনো C extension library আছে কিনা যেটা gevent patch করতে পারে না সেটা check করতে হবে। Celery, কিছু database driver — এগুলো gevent-এর সাথে কখনো কখনো conflict করে।

Deployment-এর আগে thorough testing দরকার। আর long-term plan হলো UvicornWorker-এ migrate করা।

---

# 🎯 Final Worker Selection Guide

Worker বেছে নেওয়ার সিদ্ধান্তটা এভাবে ভাবো:

```
App কি I/O-heavy?
├── না (CPU-heavy) → Sync worker + multiple processes
└── হ্যাঁ
    └── Code কি async?
        ├── হ্যাঁ (async/await, async libraries) → UvicornWorker
        └── না (sync code, legacy)
            └── Code পরিবর্তন সম্ভব?
                ├── হ্যাঁ → Async করো → UvicornWorker
                └── না
                    └── Gevent compatibility আছে? → Gevent
                                                  → Gthread (safe fallback)
```

**Production recommended setup (2026):**
- Modern app (FastAPI/Django async): `gunicorn -k uvicorn.workers.UvicornWorker -w $(nproc)`
- Django/Flask sync: `gunicorn -k gthread --threads 4 -w $(nproc)`
- Legacy, quick fix: `gunicorn -k gevent --worker-connections 1000 -w $(nproc)`
- CPU-heavy: `gunicorn -w $(($(nproc)*2+1))`

---

# ⚔️ Mini Challenge

তোমার কাছে একটা Django REST API app আছে। এই app:
- PostgreSQL database use করে (sync ORM, psycopg2)
- প্রতিটা request গড়ে 2টা DB query করে (মোট ~80ms)
- কিছু endpoint-এ external SMS API call আছে (150ms)
- কিছু endpoint-এ PDF generation আছে (~400ms CPU)

Traffic: 300 concurrent user।

**প্রশ্ন ১:** কোন worker দিয়ে শুরু করবে এবং কেন?

**প্রশ্ন ২:** SMS API call-এর endpoint-গুলো অন্য endpoint-এর চেয়ে slow। Worker type কি এই সমস্যা solve করবে? কীভাবে?

**প্রশ্ন ৩:** PDF generation endpoint sync worker-এ কেমন behave করবে? Gthread-এ কেমন?

**প্রশ্ন ৪:** এই app-কে eventually UvicornWorker-এ migrate করতে চাইলে কোথা থেকে শুরু করবে?