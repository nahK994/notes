# 🏗️ Module 8: Full Stack Architectures — Real-world Blueprints

---

## Module 6-এর Mini Challenge-এর উত্তর

আগের challenge ছিল:

> 1000 concurrent users, 3 Gunicorn instance (প্রতিটায় 2 async worker), সামনে Nginx।

**প্রশ্ন ১ — Request distribution কীভাবে হবে?**

Nginx default round-robin ব্যবহার করে। মানে একটার পর একটা instance-এ request পাঠায়। 1000 request আসলে তিনটা instance প্রায় সমান ভাগ পাবে।

```
          1000 users
               │
             Nginx
           ┌───┴───┐
           │       │
          ┌┴┐     ┌┴┐     ┌──┐
          │A│     │B│     │C │
          └─┘     └─┘     └──┘
         ~333    ~333     ~333
        requests requests requests
```

কিন্তু "333 request per instance" মানে 333টা sequential না। প্রতিটা instance-এ 2টা async worker, প্রতিটা worker-এ event loop। মোট concurrency:

```
3 instance × 2 worker × N coroutines/worker = হাজারো concurrent request
```

**প্রশ্ন ২ — Nginx না থাকলে কী সমস্যা হতো?**

তিনটা বড় সমস্যা:

- **Load balancing নেই:** সব 1000 request একটাই instance-এ যেত, বাকি দুটো idle।
- **Slow client problem:** 3G-তে user ধীরে request body পাঠাচ্ছে — Gunicorn worker সেই পুরো সময় আটকে থাকত। Nginx এটা buffer করে তারপর একসাথে forward করে।
- **Static file inefficiency:** CSS, JS, image প্রতিটা Python process-এ যেত, যেখানে Nginx disk থেকে milliseconds-এ serve করতে পারত।

**প্রশ্ন ৩ — Bottleneck কোথায়?**

চারটা জায়গায় নজর রাখতে হবে: external API latency (সবচেয়ে সম্ভাব্য — control নেই), DB query time, worker count limit, এবং event loop-এ কোনো accidental blocking call।

---

## কেন এই Module?

আগের modules-এ individual pieces দেখলাম — HTTP, WSGI, ASGI, Worker types, Nginx। এখন সেগুলো একসাথে জুড়তে হবে।

Real production system বানাতে গেলে শুধু জানলে হয় না "async ভালো" বা "Nginx দরকার" — জানতে হয় কোন workload-এ কোন combination সবচেয়ে sense করে। এই module-এ ছয়টা real-world architecture pattern দেখব, প্রতিটার জন্য কখন, কেন, এবং কোথায় সমস্যা হতে পারে।

---

## Architecture বোঝার Framework

যেকোনো web system-এর architecture মূলত কয়েকটা layer-এর composition:

```
┌──────────────────────────────────┐
│             Client               │  Browser, mobile app, API consumer
├──────────────────────────────────┤
│             Nginx                │  Connection, routing, SSL, static files
├──────────────────────────────────┤
│           App Server             │  Gunicorn / Uvicorn — process management
├──────────────────────────────────┤
│             Worker               │  Sync / thread / async event loop
├──────────────────────────────────┤
│          Application             │  Django / FastAPI — business logic
├──────────────────────────────────┤
│       DB / Cache / APIs          │  PostgreSQL, Redis, external services
└──────────────────────────────────┘
```

Architecture সিদ্ধান্ত মানে প্রতিটা layer কী দিয়ে পূরণ করবে — এবং সেটা workload-এর সাথে মেলে কিনা।

---

## Case 1: Traditional Django App

### কখন এই architecture?

E-commerce site, admin dashboard, CMS — যেখানে standard CRUD, moderate traffic, real-time feature নেই।

### Stack

```
Client
  │
  ▼
Nginx                           ← connection, SSL, static files
  │
  ▼
Gunicorn  [sync / gthread]      ← process/thread management
  │
  ▼
Django (WSGI)                   ← views, ORM, business logic
  │
  ▼
PostgreSQL
```

### কীভাবে কাজ করে

একটা request আসলে Nginx নেয়, Gunicorn-এ forward করে। Gunicorn একটা worker process সেটা handle করে — Django view চলে, ORM দিয়ে DB query হয়, response ফেরে। Worker পুরো সময় occupied।

Concurrency আসে worker এবং thread count থেকে:

```
Gunicorn: -w 4 --threads 4
= 4 process × 4 thread = 16 concurrent request
```

DB query-তে wait করার সময় thread switch হয় — GIL I/O-তে release হয়।

### Gunicorn config

```bash
gunicorn \
  -k gthread \
  --threads 4 \
  -w 4 \
  --bind 127.0.0.1:8000 \
  myproject.wsgi:application
```

### কোথায় ভালো, কোথায় সমস্যা

```
✅ ভালো                         ❌ সমস্যা
──────────────────────────────────────────────────
Simple CRUD app                 High concurrency (1000+ users)
Admin panel, CMS                WebSocket / real-time feature
Low–medium traffic              I/O wait বেশি হলে workers idle
Existing Django codebase        External API-heavy workload
```

---

## Case 2: FastAPI High-Concurrency API

### কখন এই architecture?

Microservice, payment gateway, external API aggregator, high-traffic REST API — যেখানে প্রতিটা request অনেক I/O করে এবং concurrent user অনেক বেশি।

### Stack

```
Client
  │
  ▼
Nginx                                 ← connection, SSL, rate limiting
  │
  ▼
Gunicorn  [UvicornWorker]             ← process management + async execution
  │
  ▼
FastAPI (ASGI, async views)           ← route handlers, dependency injection
  │
  ▼
asyncpg / httpx / aioredis            ← async DB, HTTP client, cache
  │
  ▼
PostgreSQL + Redis + External APIs
```

### কীভাবে কাজ করে

প্রতিটা Gunicorn worker-এ Uvicorn-এর asyncio event loop চলে। `await` আসলে event loop সেই coroutine pause করে অন্যটা চালায়:

```
t=0ms   → Req A শুরু, DB await করে     → pause
t=0ms   → Req B শুরু, API await করে    → pause
t=0ms   → Req C শুরু, DB await করে     → pause
t=50ms  → Req A-র DB response → resume → response পাঠায়
t=50ms  → Req C-র DB response → resume → response পাঠায়
t=200ms → Req B-র API response → resume → response পাঠায়

Total: ~200ms (sync হলে 3× বেশি লাগত)
```

### Async vs Blocking library — অবশ্যই জানো

```
✅ Async (ব্যবহার করো)      ❌ Blocking (async context-এ ব্যবহার করো না)
────────────────────────────────────────────────────────────────────────
asyncpg / psycopg3          psycopg2
httpx / aiohttp             requests
aioredis                    redis-py sync mode
aiofiles                    open() সরাসরি
```

একটাও blocking library থাকলে সেই request-এ পুরো event loop আটকে যাবে।

### Gunicorn config

```bash
gunicorn \
  -k uvicorn.workers.UvicornWorker \
  -w 4 \
  --bind 127.0.0.1:8000 \
  myapp:app
```

Worker count = CPU core সংখ্যা। Async হওয়ায় thread বাড়ানোর দরকার নেই।

### কোথায় ভালো, কোথায় সমস্যা

```
✅ ভালো                         ❌ সমস্যা
────────────────────────────────────────────────────────────
High concurrency (1000+ users)  Blocking library → collapse
I/O-heavy workload              CPU-heavy task → event loop block
External API calls              Legacy sync codebase-এ কঠিন
Microservices                   Team async-এ comfortable না হলে
```

---

## Case 3: Pure ASGI Stack — Nginx + Uvicorn সরাসরি

### কখন এই architecture?

Container-based deployment (Docker, Kubernetes), lightweight microservice — যেখানে process management Kubernetes নিজেই করে।

### Stack

```
Client
  │
  ▼
Nginx  (বা cloud load balancer)   ← SSL, routing
  │
  ▼
Uvicorn  [Gunicorn ছাড়া]          ← async event loop, single process
  │
  ▼
FastAPI / Django (ASGI mode)
  │
  ▼
DB / Cache
```

### Gunicorn কেন লাগে না এখানে?

Gunicorn-এর মূল কাজ ছিল process management — worker crash হলে restart, graceful reload। Kubernetes pod এই কাজটা নিজেই করে। Container crash হলে Kubernetes নতুন pod তোলে।

```bash
# Dockerfile CMD-এ এভাবে চালাও
uvicorn myapp:app --host 0.0.0.0 --port 8000 --workers 1
```

### কোথায় ভালো, কোথায় সমস্যা

```
✅ ভালো                         ❌ সমস্যা
────────────────────────────────────────────────────────────
Docker / Kubernetes deploy      Bare metal-এ process management manual
Lightweight microservice        Graceful reload নেই built-in
Cloud-native setup              Horizontal scaling manually করতে হয়
CI/CD pipeline simple
```

---

## Case 4: Hybrid Django — Async এবং Legacy একসাথে

### কখন এই architecture?

পুরনো Django app যেটা mostly sync, কিন্তু নতুন feature async করতে হচ্ছে। Full migration এখনই সম্ভব না।

### Stack

```
Client
  │
  ▼
Nginx
  │
  ▼
Gunicorn  [UvicornWorker]       ← ASGI mode, দুটো ধরনই handle করে
  │
  ▼
Django (ASGI mode)
  ├── async def view()          → asyncpg, httpx  (সত্যিকারের async)
  └── def view()                → sync ORM         (thread pool-এ চলে)
```

### Sync view ASGI-তে কীভাবে চলে?

Django ASGI mode-এ sync view automatically একটা thread pool-এ চলে। Event loop block হয় না — কিন্তু thread ব্যবহার হয়।

```python
# নতুন endpoint — async লেখো
async def user_list(request):
    users = await User.objects.filter(active=True).avalues()
    return JsonResponse(list(users), safe=False)

# পুরনো view — sync রাখো, Django সামলাবে
def legacy_report(request):
    data = generate_report()   # পুরনো sync code
    return HttpResponse(data)
```

### Migration strategy

```
নতুন endpoint লিখছ?         → async view + async library
পুরনো view আছে?             → sync রাখো, thread pool handle করবে
async view-এ sync DB call?  → করো না, event loop block হবে
```

---

## Case 5: CPU-Heavy Service

### কখন এই architecture?

Image processing, video transcoding, ML inference, PDF generation — যেখানে প্রতিটা request অনেক CPU নেয়।

### Stack

```
Client
  │
  ▼
Nginx
  │
  ▼
Gunicorn  [sync worker, multiple processes]   ← process-based parallelism
  │
  ▼
Python App
  │
  ▼
CPU-intensive library  (Pillow, OpenCV, NumPy, PyTorch)
```

### কেন async এখানে কাজ করে না

Async-এর সুবিধা I/O wait-এর সময় অন্য কাজ করা। CPU-bound কাজে কোনো wait নেই, শুধু calculation। `await` করার কিছু নেই।

```
Async worker + CPU-heavy task:

t=0s  → Request A: image resize শুরু (2s CPU)
         পুরো event loop আটকে গেল।
t=2s  → Request A শেষ। এবার Request B শুরু।
         মোট: sequential, কোনো gain নেই।


Sync worker × 4 processes + CPU-heavy task:

Worker 1 (core 1): Request A — resize
Worker 2 (core 2): Request B — resize   ← একসাথে
Worker 3 (core 3): Request C — resize   ← একসাথে
Worker 4 (core 4): Request D — resize   ← একসাথে

সত্যিকারের parallel execution।
```

### Async app-এ CPU task offload করতে হলে

```python
from concurrent.futures import ProcessPoolExecutor
import asyncio

cpu_pool = ProcessPoolExecutor(max_workers=4)

@app.post("/resize")
async def resize_image(file: UploadFile):
    image_data = await file.read()              # async I/O — ঠিক আছে
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(        # CPU কাজ আলাদা process-এ
        cpu_pool, do_heavy_resize, image_data
    )
    return result
```

CPU কাজ আলাদা process-এ চলে, event loop free থাকে।

### Gunicorn config

```bash
gunicorn \
  -w 8 \
  --timeout 120 \
  --bind 127.0.0.1:8000 \
  myapp:app
```

---

## Case 6: Scaled Multi-Instance Architecture

### কখন এই architecture?

Production system যেখানে একটা server যথেষ্ট না। High traffic, high availability — একটা server down হলেও service চলবে।

### Stack

```
Client
  │
  ▼
Nginx  [Load Balancer]
  │
  ├─────────────────────────────────────┐
  ▼                                     ▼
Server 1                             Server 2
┌─────────────────────┐          ┌─────────────────────┐
│  Nginx (local)      │          │  Nginx (local)      │
│  Gunicorn ×2        │          │  Gunicorn ×2        │
│  FastAPI            │          │  FastAPI            │
└─────────────────────┘          └─────────────────────┘
  │                                     │
  └────────────────┬────────────────────┘
                   ▼
         ┌──────────────────┐
         │   PostgreSQL     │  (primary + replica)
         │   Redis Cache    │
         └──────────────────┘
```

### Stateless থাকা কেন জরুরি

একজন user-এর একটা request Server 1-এ গেল, পরেরটা Server 2-এ গেল। যদি session data locally রাখা হয়, দ্বিতীয় request-এ সেটা থাকবে না।

```
সমাধান — সব shared state centralized:

Session data    → Redis
User cache      → Redis
File uploads    → S3 / object storage  (local disk-এ না)
Database        → Central PostgreSQL
```

### Nginx load balancing config

```nginx
upstream app_servers {
    least_conn;  # সবচেয়ে কম connection যার কাছে, তাকে দাও

    server 10.0.0.1:8000 weight=2;             # শক্তিশালী server বেশি traffic
    server 10.0.0.2:8000 weight=1;
    server 10.0.0.3:8000 max_fails=3           # 3 fail হলে বাদ দাও
                          fail_timeout=30s;
}
```

---

## তিনটা Execution Model পাশাপাশি

ধরো 3টা request এলো, প্রতিটায় 100ms DB wait + 50ms processing।

**Sync (1 worker):**

```
Worker: [A: 50ms][A: wait 100ms][A: 50ms][B: 50ms][B: wait 100ms][B: 50ms]...

Timeline: 0 ──────────────────────────────────────── 600ms
Total: 3 × 200ms = 600ms
```

**Threaded (3 threads):**

```
Thread 1: [A: 50ms][A: wait 100ms][A: 50ms]
Thread 2: [B: 50ms][B: wait 100ms][B: 50ms]
Thread 3: [C: 50ms][C: wait 100ms][C: 50ms]

Timeline: 0 ─────────────────── 200ms
Total: ~200ms  (কিন্তু 3× memory)
```

**Async (1 worker, event loop):**

```
Event loop:
t=0ms    → A শুরু → DB await → pause
t=0ms    → B শুরু → DB await → pause
t=0ms    → C শুরু → DB await → pause
t=100ms  → A resume → done
t=100ms  → B resume → done
t=100ms  → C resume → done

Timeline: 0 ─────────────────── ~200ms
Total: ~200ms  (3× কম memory — coroutine, thread না)
```

Threaded এবং Async-এর সময় একই। কিন্তু 1000 concurrent request-এ:
- Threaded: 1000টা OS thread = প্রচুর memory + context switch overhead
- Async: 1000টা lightweight coroutine = অনেক কম memory, efficient

---

## সবচেয়ে বেশি হওয়া Mistakes

**Mistake 1 — Async worker + sync library:**

```python
# ❌ FastAPI + UvicornWorker-এ:
import requests
result = requests.get("https://api.example.com")  # পুরো event loop freeze

# ✅ ঠিক করো:
import httpx
async with httpx.AsyncClient() as client:
    result = await client.get("https://api.example.com")
```

**Mistake 2 — Async handler-এ CPU task সরাসরি:**

```python
# ❌ 2 সেকেন্ড event loop block থাকবে
@app.post("/resize")
async def resize(file: UploadFile):
    data = await file.read()
    result = cv2.resize(data, (800, 600))  # CPU-heavy, no await, blocks loop
    return result

# ✅ ঠিক করো — ProcessPoolExecutor-এ offload
result = await loop.run_in_executor(cpu_pool, cv2.resize, data, (800, 600))
```

**Mistake 3 — State locally রাখা, multiple server-এ deploy:**

```python
# ❌ In-memory — Server 2-এ গেলে session নেই
sessions = {}   # প্রতিটা process-এর আলাদা

# ✅ ঠিক করো — Redis-এ রাখো
await redis.set(f"session:{user_id}", token, ex=3600)
```

**Mistake 4 — Worker count বাড়িয়ে সব সমাধান:**

```bash
# ❌ Blocking code থাকলে 100 worker-ও কাজ করবে না
gunicorn -w 100 myapp:app  # 100 process = 100× memory, root cause fix হয়নি
```

---

## Architecture Decision Framework

```
1. App কী ধরনের কাজ করে?
   ├── I/O heavy (DB, external API) ──→ async route
   └── CPU heavy (computation) ────────→ process route

2. Code কি async-ready?
   ├── হ্যাঁ (async/await, async libs) → ASGI + UvicornWorker
   └── না (sync Django, legacy)
       ├── Migrate সম্ভব? → করো → ASGI
       └── না ───────────→ gthread বা gevent

3. Real-time feature দরকার?
   ├── হ্যাঁ (WebSocket, live push) → ASGI mandatory
   └── না ──────────────────────────→ WSGI-ও চলবে

4. Traffic কেমন?
   ├── Low–medium (< 500 concurrent) → single server
   └── High (500+) ─────────────────→ multi-instance + Nginx LB

5. CPU task আছে async app-এ?
   └── হ্যাঁ → ProcessPoolExecutor দিয়ে offload করো
```

---

## Final Mental Model

```
Performance = Execution Model × Code Behavior × Infrastructure

ভুল match:
Async worker   + Blocking code    = Sync-এর চেয়েও খারাপ
Sync worker    + High concurrency = Worker exhaustion
CPU task       + Async event loop = Event loop freeze
Single server  + High traffic     = Single point of failure

সঠিক match:
I/O-heavy   + Async worker  + Async libraries = High throughput
CPU-heavy   + Sync workers  + Multi-process   = True parallelism
High traffic + Multi-server + Nginx LB        = Horizontal scale
Legacy code  + gthread      ──────────────────= Safe concurrency
```

Architecture হলো layer-এর composition। প্রতিটা layer তার দায়িত্ব পালন করলে system scale করে। একটা layer ভুল হলে bottleneck সেখানে তৈরি হয় — আর সেটা আরও বেশি worker দিয়ে fix করা যায় না।

---

## ⚔️ Mini Challenge

তুমি একটা complex system বানাচ্ছো যেখানে তিনটা আলাদা feature:

- **Chat:** WebSocket-based real-time messaging, হাজারো persistent connection
- **Feed API:** External social media API থেকে data fetch, I/O-heavy, high traffic
- **Image Service:** User-uploaded image resize এবং watermark, CPU-heavy

**প্রশ্ন ১:** তিনটা feature কি একটাই service-এ রাখবে নাকি আলাদা করবে? কেন?

**প্রশ্ন ২:** প্রতিটা feature-এর জন্য কোন worker type, কোন execution model?

**প্রশ্ন ৩:** তিনটাই আলাদা service হলে Nginx কোথায় বসবে এবং কীভাবে route করবে?

**প্রশ্ন ৪:** Image service-এ async wrapper রাখতে চাইলে CPU কাজ event loop block না করে কীভাবে করবে?

**প্রশ্ন ৫:** System-এর কোথায় কোথায় Redis লাগবে এবং প্রতিটা জায়গায় কেন?

এই পাঁচটা প্রশ্নের উত্তর দিতে পারলে এই tutorial series-এর core concepts আয়ত্তে এসে গেছে।