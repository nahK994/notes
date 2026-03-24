# 🐾 Module 4: Gunicorn Worker Zoo

---

## Module 2-এর Mini Challenge-এর উত্তর

আগের module শেষে প্রশ্ন ছিল:

> 1 Uvicorn worker, async code, 5টা request একসাথে — প্রতিটায় 1 সেকেন্ড DB + 1 সেকেন্ড API await।

**প্রশ্ন ১ — সব 5টা request শেষ হতে কত সময়?**

প্রায় **2 সেকেন্ড**।

কেন? কারণ সব 5টা request প্রায় একসাথে শুরু হয়। প্রতিটা DB query await করার সময় event loop বাকিগুলোকেও DB query-তে পাঠিয়ে দেয়। 1 সেকেন্ড পর সবার DB response আসে, সবাই API call করে await করে। আরও 1 সেকেন্ড পর সবার API response আসে। মোট ≈ 2 সেকেন্ড।

```
t=0s    → 5টা request শুরু, সবাই DB query করে await
t=1s    → সবার DB response আসে, সবাই API call করে await
t=2s    → সবার API response আসে, সবাই শেষ
```

**প্রশ্ন ২ — WSGI sync worker-এ কত সময় লাগত?**

Sequential হতো, তাই **10 সেকেন্ড**।
Request 1 শেষ (2s) → Request 2 শেষ (2s) → ... → Request 5 শেষ (2s) = 5 × 2 = 10s।

**প্রশ্ন ৩ — Performance gain কোথা থেকে এলো?**

I/O wait-এর সময় থেকে। প্রতিটা request মোট 2 সেকেন্ডের মধ্যে 2 সেকেন্ডই wait করছিল — DB-র জন্য 1 সেকেন্ড, API-র জন্য 1 সেকেন্ড। CPU সেই সময় কিছুই করছিল না।

WSGI-তে worker এই idle সময়েও occupied ছিল। ASGI-তে event loop এই idle সময়ে বাকি request গুলোর কাজ এগিয়ে নিয়ে গেছে।

---

এখন একটা প্রশ্ন স্বাভাবিকভাবেই আসে — ASGI এবং event loop বুঝলাম, কিন্তু production-এ Gunicorn ব্যবহার করলে worker কীভাবে configure করব? কোন worker type বেছে নেব? এই সিদ্ধান্তটাই Module 4-এর বিষয়।

---

## কেন Worker Type এত গুরুত্বপূর্ণ?

Gunicorn নিজে কোনো request process করে না। সে শুধু একজন manager — request আসলে কোন worker-কে দেবে, worker crash করলে restart দেবে, graceful shutdown করবে। আসল কাজটা করে worker।

এখন worker কীভাবে কাজ করবে — blocking না non-blocking, thread-based না event loop-based — এটা নির্ভর করে তুমি কোন worker type বেছে নিয়েছ। একই Gunicorn-এ worker type বদলালে পুরো execution model বদলে যায়।

এটা ঠিক যেন একটা restaurant-এ manager একই থাকে, কিন্তু staff কীভাবে কাজ করে সেটা বদলে দিলে restaurant-এর পুরো serving capacity বদলে যায়।

---

# 🪵 4.1 Sync Worker — The Baseline

এটাই Gunicorn-এর default worker। কোনো `-k` flag না দিলে এটাই চলে।

**কীভাবে কাজ করে:**

একটা sync worker মানে একটা Python process। এই process একটাই request নেয়, শেষ করে, তারপর পরেরটা নেয়। সম্পূর্ণ sequential।

```
Worker 1: [Request A শুরু → process → DB wait → response] [Request B শুরু → ...]
```

Request A-এর DB wait-এর সময় Worker 1 আটকে আছে। নতুন request নিতে পারছে না।

**কখন ভালো:**

Sync worker-এর সুবিধা হলো সরলতা। Debug করা সহজ, race condition নেই, কোড যেভাবে লেখা আছে সেভাবেই চলে। Traffic কম থাকলে এবং কাজটা CPU-heavy হলে (image processing, heavy calculation) — এখানে async দিয়ে কিছু লাভ নেই, sync worker-ই সঠিক।

**কখন সমস্যা:**

High traffic এবং I/O-heavy app-এ sync worker-এর concurrency সীমাবদ্ধতা ধরা পড়ে। Worker বাড়ালে memory বাড়ে, কিন্তু প্রতিটা worker তখনও I/O wait-এ idle।

```bash
gunicorn -w 4 myapp:app   # 4টা sync worker
```

---

# 🧵 4.2 Gthread Worker — Thread দিয়ে Concurrency

Sync worker-এর সীমাবদ্ধতা কাটাতে সহজ পরের ধাপ হলো threading। Gthread worker-এ একটা process-এর ভেতরে multiple thread থাকে। প্রতিটা thread আলাদা request handle করে।

**কীভাবে কাজ করে:**

```
Worker (1 process):
  ├── Thread 1: [Request A — DB wait করছে]
  ├── Thread 2: [Request B — processing]
  └── Thread 3: [Request C — শুরু হলো]
```

Request A যখন DB-র জন্য wait করছে, Thread 1 blocked — কিন্তু Thread 2 এবং Thread 3 চলছে। ফলে একটা worker-এ একাধিক request simultaneously progress করতে পারে।

**GIL-এর বাস্তবতা:**

Python-এর GIL থাকায় একসাথে একটাই thread Python bytecode চালাতে পারে। তাহলে কি threading কোনো কাজে আসে না?

আসে — I/O-bound কাজে। DB query করার সময় thread OS-এর কাছে network call পাঠিয়ে block হয়ে যায়, এই সময় GIL release হয় এবং অন্য thread Python code চালাতে পারে। তাই I/O-heavy workload-এ gthread sync worker-এর তুলনায় অনেক বেশি efficient।

CPU-bound কাজে (ভারী calculation) gthread কোনো সুবিধা দেয় না — GIL-এর কারণে threads সত্যিকারের parallel চলে না।

```bash
gunicorn -k gthread --threads 4 -w 2 myapp:app
# 2 worker, প্রতি worker-এ 4 thread = মোট 8 concurrent request
```

**কখন ভালো:**

Legacy codebase যেখানে async করা সম্ভব না, কিন্তু concurrency দরকার। Sync code যেটা I/O করে — gthread দিয়ে সহজেই concurrency বাড়ানো যায়, code পরিবর্তন না করেই।

---

# ⚡ 4.3 UvicornWorker — True Async

এটা ASGI worker। Gunicorn-কে process manager হিসেবে রেখে প্রতিটা worker Uvicorn-এর event loop দিয়ে চলে।

**কীভাবে কাজ করে:**

একটা worker মানে একটা event loop। সেই event loop-এ শত শত coroutine একসাথে চলতে পারে। কোনো request await করলে event loop অন্যটা resume করে।

```
Worker (1 event loop):
  Coroutine A: await db_query()  ← pause
  Coroutine B: processing        ← running
  Coroutine C: await api_call()  ← pause
  Coroutine D: await db_query()  ← pause
  ... (আরও অনেক)
```

Thread-based approach-এ প্রতিটা concurrent request-এর জন্য একটা thread লাগে। Thread-এর memory overhead আছে। Event loop-এ হাজারো coroutine একটা thread-এই চলে — memory overhead অনেক কম।

**সবচেয়ে বড় শর্ত:**

UvicornWorker-এর পুরো সুবিধা পেতে পুরো code stack async হতে হবে। একটাও blocking call থাকলে সেই request-এর সময় পুরো event loop আটকে যায়।

- ✅ `asyncpg`, `databases` — async DB driver
- ✅ `httpx`, `aiohttp` — async HTTP client
- ❌ `psycopg2`, `requests` — blocking, event loop আটকাবে

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 4 myapp:app
# 4টা async worker, প্রতিটায় event loop
```

---

# 🌿 4.4 Gevent Worker — Green Threads-এর Magic

Gevent একটু আলাদা ধরনের solution। এটা "green threads" বা greenlet ব্যবহার করে।

**Greenlet কী?**

Greenlet হলো user-space-এ lightweight coroutine — OS thread না, Python-level cooperative multitasking। অনেকটা asyncio coroutine-এর মতো, কিন্তু পার্থক্য হলো gevent-এ তুমি `await` লিখতে হয় না।

**Monkey Patching — Gevent-এর আসল কৌশল:**

Gevent একটা কাজ করে যেটাকে বলে "monkey patching।" এটা Python-এর standard library-র blocking functions গুলো (socket, time.sleep, ইত্যাদি) চুপচাপ replace করে দেয় non-blocking version দিয়ে।

```python
# gevent চালু হলে এটা automatically হয়
import gevent.monkey
gevent.monkey.patch_all()

# এরপর এই code async-এর মতো behave করে
import time
time.sleep(5)  # আর blocking না, gevent handle করবে
```

ফলে তোমার পুরনো sync code — Django view, পুরনো DB library — কোনো পরিবর্তন ছাড়াই async-like concurrency পায়।

**সুবিধা এবং সমস্যা:**

সুবিধা হলো legacy code migrate না করেই concurrency বাড়ানো যায়। বড় পুরনো Django app-এ এটা অনেক সময় বাঁচায়।

সমস্যা হলো hidden complexity। Code দেখে বোঝা যায় না কোথায় context switch হচ্ছে। Bug হলে trace করা কঠিন। এবং monkey patching সব library-র সাথে compatible না — কোনো library gevent-এর patch-এর সাথে মিলতে না পারলে mysterious crash হয়।

```bash
gunicorn -k gevent -w 4 myapp:app
```

---

# 📊 4.5 চারটা Worker পাশাপাশি

| Worker | Concurrency কীভাবে | Blocking হলে কী হয় | সবচেয়ে ভালো কোথায় |
|--------|-------------------|--------------------|--------------------|
| sync | একটাই request | শুধু সেই request আটকায় | Simple app, CPU-heavy |
| gthread | Thread-এ | সেই thread আটকায় | Legacy code, mixed workload |
| gevent | Green thread-এ | অন্য greenlet চলে | Legacy async, পুরনো Django |
| uvicorn | Event loop-এ | পুরো event loop আটকায় | Modern async app, high concurrency |

একটা important pattern লক্ষ্য করো: sync এবং gthread-এ blocking হলে শুধু সেই unit (worker বা thread) আটকায়, বাকিরা চলে। UvicornWorker-এ blocking হলে পুরো event loop আটকায় — এটা আরও বিপজ্জনক। এজন্যই async world-এ blocking code এত সতর্কতার বিষয়।

---

# 🧠 4.6 Worker বাড়ালেই কি সমস্যা সমাধান হয়?

একটা common ভুল হলো — "performance কম? আরও worker দাও।"

এটা সত্যি শুধু তখন যখন bottleneck CPU। কিন্তু যদি code-এর execution model ভুল হয় — sync code দিয়ে I/O-heavy app, বা async code-এ blocking call — তাহলে worker বাড়িয়ে শুধু memory বাড়বে, performance নয়।

ধরো প্রতিটা request 2 সেকেন্ড DB wait করে। 10টা sync worker দিলে একসাথে 10টা request handle হবে, কিন্তু প্রতিটা worker সেই 2 সেকেন্ড idle বসে থাকবে। Async করলে 1টা worker-এই এই 10টা request 2 সেকেন্ডে শেষ হতে পারত।

সঠিক approach:
1. প্রথমে workload বোঝো — I/O-bound নাকি CPU-bound?
2. তারপর execution model বেছে নাও
3. তারপর worker count optimize করো

---

# 🎯 Worker বেছে নেওয়ার Framework

**ধাপ ১ — App কী ধরনের কাজ করে?**

I/O-heavy (DB, external API, cache) → async বা thread-based approach।
CPU-heavy (image processing, ML inference) → multiple process, sync worker।

**ধাপ ২ — Codebase কী async-ready?**

Modern async code (`async def`, async libraries) → UvicornWorker।
পুরনো sync code, migrate করা সম্ভব না → gthread বা gevent।

**ধাপ ৩ — Complexity কতটুকু নিতে পারব?**

Debug সহজ রাখতে চাই, traffic moderate → sync বা gthread।
Maximum performance, team async-comfortable → UvicornWorker।
Legacy code কিন্তু async চাই, trade-off accept করতে পারি → gevent।

---

# 🎯 Final Mental Model

Gunicorn হলো manager, worker হলো execution brain। Brain বদলালে পুরো system-এর behaviour বদলায়।

```
Gunicorn (manager)
    ├── sync worker    → sequential, predictable
    ├── gthread worker → thread-per-request, I/O-friendly
    ├── gevent worker  → green threads, legacy-friendly
    └── uvicorn worker → event loop, maximum concurrency
```

Production-এ সবচেয়ে common modern setup: `gunicorn -k uvicorn.workers.UvicornWorker -w {cpu_count}` — Gunicorn process management করছে, Uvicorn async execution করছে, worker count CPU core-এর সমান।

---

# ⚔️ Mini Challenge

ধরো তোমার FastAPI app-এ প্রতিটা request দুটো কাজ করে:
- একটা DB query: 1 সেকেন্ড await
- একটা ছোট calculation: 0.1 সেকেন্ড CPU time

Setup: Gunicorn, 1 worker, 5টা request একসাথে।

**Case A — Sync worker:**
মোট সময় কত? CPU কতটুকু busy থাকবে?

**Case B — Gthread worker (threads=5):**
মোট সময় কত? Sync-এর চেয়ে কীভাবে আলাদা?

**Case C — UvicornWorker, async code:**
মোট সময় কত? CPU utilization কেমন হবে?

**Case D — UvicornWorker, কিন্তু DB driver টা accidentally sync (blocking):**
এখন কী হবে? Case C-র সাথে তুলনা করো।

Case D টা সবচেয়ে গুরুত্বপূর্ণ — এটাই real-world-এ সবচেয়ে বেশি হয়।