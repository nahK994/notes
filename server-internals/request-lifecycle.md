# 🔬 Module 9: Request Lifecycle Deep Dive

---

## Module 8-এর Mini Challenge-এর উত্তর

আগের challenge ছিল: Chat app (WebSocket) + External API calls + Image processing — এই তিনটা feature একসাথে রাখবে নাকি আলাদা করবে?

**উত্তর: অবশ্যই আলাদা তিনটা service।**

কেন একসাথে রাখা যাবে না সেটা বোঝার জন্য execution model-এর conflict দেখো:

- Chat service-এর দরকার ASGI + async event loop। হাজারো WebSocket connection একটা worker-এ।
- API service-এরও দরকার ASGI + async। External API await-এর সময় অন্য request চলবে।
- Image processing-এর দরকার CPU এবং multiple processes। Async loop-এ এটা রাখলে CPU কাজ পুরো event loop block করবে।

তিনটার execution model আলাদা। একটা process-এ সব ঢোকালে সবচেয়ে খারাপ trade-off নিতে হবে।

**সঠিক breakdown:**

```
                    Client
                      │
                    Nginx             ← সব traffic এখানে আসে
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
    Chat Service   API Service  Image Service
    (ASGI/async)  (ASGI/async)  (sync/process)
    WebSocket      FastAPI       Gunicorn sync
    Uvicorn        Uvicorn       CPU workers
```

**Nginx routing:**

```nginx
location /ws/     { proxy_pass http://chat_service; }
location /api/    { proxy_pass http://api_service; }
location /process/{ proxy_pass http://image_service; }
```

**Key insight:** Different workload → different execution model → different service। এটাই microservice architecture-এর মূল কারণ — scaling এবং execution isolation।

---

## কেন এই Module?

এতক্ষণ আমরা জেনেছি HTTP কী, WSGI/ASGI কী, worker কীভাবে কাজ করে, Nginx কী করে। কিন্তু এই সবগুলো একসাথে কীভাবে কাজ করে — একটা request browser থেকে বের হয়ে server-এ পৌঁছায়, process হয়, এবং ফিরে আসে — এই পুরো journey টা end-to-end দেখা হয়নি।

এই module-এ সেটাই করব। প্রতিটা step-এ কী হচ্ছে, কেন হচ্ছে, সময় কোথায় যাচ্ছে, এবং কোথায় সমস্যা হতে পারে — সব।

এই mental model টা থাকলে যেকোনো performance সমস্যায় সঠিক জায়গায় খুঁজতে পারবে।

---

## Full Lifecycle — একনজরে

```
Browser
  │
  │  TCP Connection (SYN → SYN-ACK → ACK)
  │
  ▼
Nginx
  │
  ├── Static file? → Disk থেকে serve → Browser
  │
  │  HTTP proxy (plain HTTP, localhost)
  │
  ▼
Gunicorn
  │
  │  Worker selection
  │
  ▼
Worker (sync / thread / async event loop)
  │
  │  Python code execution
  │
  ▼
Application (Django / FastAPI)
  │
  ├── DB query (PostgreSQL, MongoDB...)
  ├── Cache check (Redis...)
  └── External API call (payment, SMS...)
  │
  ▼
Response তৈরি
  │
  │  (same path ফিরে যায়)
  │
  ▼
Worker → Gunicorn → Nginx → Browser
```

এটাই big picture। এখন প্রতিটা step বিস্তারিত দেখি।

---

## Step 1: TCP Connection — সবকিছুর আগে

Browser যখন `https://example.com/api/users` request করে, তার আগে একটা network-level handshake হয়। এটাকে বলে **TCP three-way handshake**।

```
Browser                           Server (Nginx)
  │                                     │
  │──── SYN ──────────────────────────▶│   "আমি connect করতে চাই"
  │                                     │
  │◀─── SYN-ACK ────────────────────── │   "ঠিক আছে, আমি ready"
  │                                     │
  │──── ACK ──────────────────────────▶│   "confirmed, শুরু করি"
  │                                     │
  │           Connection established    │
  │                                     │
  │──── HTTP Request ────────────────▶ │
```

এই handshake-এ সময় লাগে — network latency-র উপর নির্ভর করে 1ms থেকে 100ms+ পর্যন্ত।

**Keep-alive কেন গুরুত্বপূর্ণ:**

HTTP/1.1-এ default `Connection: keep-alive`। মানে একবার TCP connection হলে সেটা বারবার reuse হয়। প্রতিটা request-এর আগে নতুন handshake লাগে না।

```
Keep-alive ছাড়া:
Request 1: [TCP handshake][HTTP req/res][Connection close]
Request 2: [TCP handshake][HTTP req/res][Connection close]
Request 3: [TCP handshake][HTTP req/res][Connection close]
                ↑ প্রতিটায় extra cost

Keep-alive সহ:
[TCP handshake]
Request 1: [HTTP req/res]
Request 2: [HTTP req/res]   ← একই connection reuse
Request 3: [HTTP req/res]
[Connection close]
```

HTTP/2-তে আরও উন্নত — একই TCP connection-এ multiple request একসাথে চলে (multiplexing)।

**HTTPS হলে আরও একটা step:**

TCP handshake-এর পরে TLS handshake হয়। Certificate exchange, encryption negotiation। Nginx এটা handle করে — Gunicorn পর্যন্ত plain HTTP আসে।

```
Browser                    Nginx
  │                          │
  │── TCP handshake ────────▶│
  │                          │
  │── TLS handshake ────────▶│   certificate check, key exchange
  │                          │
  │── Encrypted HTTP ───────▶│
  │                          │
  │          Nginx decrypt করে, plain HTTP forward করে
  │                          │──── plain HTTP ──▶ Gunicorn
```

---

## Step 2: HTTP Request Nginx-এ পৌঁছায়

TCP connection established হলে browser HTTP request পাঠায়। এটা raw bytes:

```
GET /api/users?page=2 HTTP/1.1\r\n
Host: example.com\r\n
Authorization: Bearer eyJhbGc...\r\n
Accept: application/json\r\n
\r\n
```

Nginx এই bytes পড়ে parse করে। Method, path, headers সব আলাদা করে।

**Nginx-এর প্রথম সিদ্ধান্ত — static নাকি dynamic?**

```
Request: GET /api/users
                │
                ▼
        Nginx config check
                │
    ┌───────────┴───────────┐
    │                       │
    ▼                       ▼
/static/ path?         /api/ path?
    │                       │
    ▼                       ▼
Disk থেকে file         Gunicorn-এ
সরাসরি serve           forward করো
(Python জড়িত না)
```

Static file হলে Gunicorn পর্যন্ত যায়ই না। Nginx সরাসরি disk থেকে পড়ে response দেয়। এটা অনেক দ্রুত — C-তে লেখা, event-driven, microseconds-এ।

**Slow client problem এবং Nginx buffering:**

একটা user 3G connection-এ request body পাঠাচ্ছে — ধীরে ধীরে, 5 সেকেন্ড ধরে। Nginx এই পুরো body buffer করে। শেষ হলে একসাথে Gunicorn-এ পাঠায়।

```
Without Nginx buffering:
Browser ──[ধীরে ধীরে 5 সেকেন্ড]──▶ Gunicorn worker
                                         ↑
                              5 সেকেন্ড worker আটকে আছে

With Nginx buffering:
Browser ──[ধীরে ধীরে 5 সেকেন্ড]──▶ Nginx buffer
                                         │
                                    [instant forward]
                                         │
                                         ▼
                                    Gunicorn worker
                                    (মাত্র কয়েক ms)
```

এই buffering-এর কারণে Gunicorn worker দ্রুত free হয়, পরের request নিতে পারে।

---

## Step 3: Nginx থেকে Gunicorn-এ

Nginx request Gunicorn-এ forward করে। এটা সাধারণত হয় দুটো উপায়ে:

**Unix socket (দ্রুততর):**

```
Nginx ──[/tmp/gunicorn.sock]──▶ Gunicorn
```

Unix socket হলো OS-এর মধ্যে file-like channel। Network involve না, তাই দ্রুত।

**TCP socket (সহজ):**

```
Nginx ──[127.0.0.1:8000]──▶ Gunicorn
```

Localhost-এর মধ্যে TCP। Network stack involve হয়, একটু slow, কিন্তু পার্থক্য সাধারণত negligible।

**Nginx যে headers পাঠায়:**

```http
GET /api/users HTTP/1.0
Host: example.com
X-Real-IP: 203.0.113.10          ← real client IP
X-Forwarded-For: 203.0.113.10    ← proxy chain
X-Forwarded-Proto: https         ← original scheme
```

এই headers না থাকলে Gunicorn মনে করবে request আসছে `127.0.0.1` থেকে। Django-তে `request.META['REMOTE_ADDR']` বা FastAPI-তে `request.client.host` ভুল IP দেবে।

---

## Step 4: Gunicorn Worker Selection

Gunicorn একটা master process। সে নিজে request process করে না। তার কাজ হলো worker process manage করা এবং incoming request গুলো idle worker-দের দেওয়া।

```
Gunicorn Master Process
│
├── Worker 1  [idle]       ← Request A পাবে
├── Worker 2  [busy - Req X চলছে]
├── Worker 3  [idle]       ← Request B পাবে
└── Worker 4  [busy - Req Y চলছে]
```

**Queue তৈরি হওয়ার পরিস্থিতি:**

যদি সব worker busy থাকে এবং নতুন request আসে:

```
Gunicorn Master
│
├── Worker 1  [busy]
├── Worker 2  [busy]
├── Worker 3  [busy]
└── Worker 4  [busy]
      ↑
New Request: "আমি কোথায় যাব?"
      │
      ▼
OS-level accept queue-এ বসে অপেক্ষা করে।
queue full হলে → connection refused বা timeout।
```

এটাই worker count এত গুরুত্বপূর্ণ হওয়ার কারণ — কিন্তু শুধু worker বাড়ালেই হয় না, execution model ঠিক না হলে worker আরও দ্রুত blocked হয়।

---

## Step 5: Execution Model — এখানেই সব পার্থক্য

Worker request পেলে execution model kick in করে। এই তিনটা model আগে দেখেছি, কিন্তু এখন request lifecycle-এর মধ্যে দেখি।

### Sync Model

```
Worker receives request
        │
        ▼
   Python code শুরু
        │
        ▼
   DB query (psycopg2)
        │
   [████████ 200ms waiting ████████]    ← CPU idle, worker blocked
        │
        ▼
   External API call (requests)
        │
   [████████████ 400ms waiting ████████████]  ← CPU idle, worker blocked
        │
        ▼
   Response তৈরি (10ms)
        │
        ▼
   Worker free হলো → পরের request নেবে
```

মোট সময়: 610ms। কিন্তু এই 610ms-এর মধ্যে CPU actually কাজ করেছে মাত্র ~10ms। বাকি 600ms? Idle। Wait করছে।

Worker কিন্তু এই পুরো 610ms occupied ছিল। নতুন কোনো request নিতে পারেনি।

### Thread Model

Thread model sync-এর মতোই চলে, কিন্তু একটা worker process-এর মধ্যে multiple thread থাকে।

```
Worker (1 process, 4 threads)

Thread 1: [code][DB wait 200ms     ][API wait 400ms           ][done]
Thread 2:       [code][DB wait 200ms     ][API wait 400ms           ][done]
Thread 3:             [code][DB wait 200ms     ][API wait 400ms      ][done]
Thread 4:                   [code][DB wait 200ms     ][API wait 400ms][done]

Timeline: 0ms ─────────────────────────────────────────────── ~650ms
```

Thread 1 যখন DB wait করছে, Thread 2 তার Python code চালাচ্ছে। DB wait-এ GIL release হয়, তাই এটা কাজ করে।

কিন্তু সীমা আছে। 4টা thread মানে একসাথে 4টা request। 100 concurrent request-এ 100টা thread — প্রতিটা OS thread ~1-8MB stack নেয়। 100 thread = সম্ভাব্য 800MB শুধু stack-এর জন্য।

### Async Model

```
Event Loop (1 thread, 1 worker)

t=0ms   → Request A শুরু, DB await → coroutine A pause
t=0ms   → Request B শুরু, DB await → coroutine B pause
t=0ms   → Request C শুরু, DB await → coroutine C pause
            │
            │  [event loop অপেক্ষা করে — কার I/O ready?]
            │
t=200ms → A-র DB response ready → coroutine A resume
t=200ms → coroutine A: API call await → pause
t=200ms → B-র DB response ready → coroutine B resume
t=200ms → coroutine B: API call await → pause
t=200ms → C-র DB response ready → coroutine C resume
t=200ms → coroutine C: API call await → pause
            │
            │  [আবার অপেক্ষা]
            │
t=600ms → A-র API response → coroutine A resume → response তৈরি → done
t=600ms → B-র API response → coroutine B resume → response তৈরি → done
t=600ms → C-র API response → coroutine C resume → response তৈরি → done
```

তিনটা request মোট সময়: ~600ms। Sync-এ লাগত 3 × 610ms = 1830ms।

এই gain কোথা থেকে এলো? I/O wait-গুলো overlap করেছে। A, B, C তিনজনেই DB-র জন্য 200ms wait করছিল — কিন্তু তিনজনের wait একসাথে হয়েছে, sequentially না।

**Event loop কীভাবে জানে I/O ready হয়েছে?**

OS-এর `epoll` (Linux) বা `kqueue` (macOS) mechanism ব্যবহার করে। Event loop OS-কে বলে "এই সব file descriptor-এর মধ্যে কোনটা ready হলে আমাকে জানিও।" OS জানায়, event loop সেই coroutine resume করে।

```
Event Loop
    │
    │ "কে ready?" (epoll_wait)
    │
    ▼
OS kernel
    │
    ├── Socket A: DB response এসেছে → ready!
    ├── Socket B: এখনও waiting
    └── Socket C: External API response এসেছে → ready!
    │
    ▼
Event Loop → coroutine A resume করো, coroutine C resume করো
```

এই পুরো mechanism-টা non-blocking — event loop কখনো stuck থাকে না, সে সবসময় "কে ready?" জিজ্ঞেস করছে এবং ready হওয়া কাজ করছে।

---

## Step 6: Application Logic

Worker execution model decide হলে Python application code চলে। এটা তোমার লেখা business logic:

```python
# FastAPI endpoint-এর ভেতরে কী হচ্ছে

@app.get("/api/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):

    # Step 1: Cache check (Redis) — ~2ms
    cached = await redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)          # cache hit → সরাসরি return

    # Step 2: DB query — ~50ms
    user = await db.execute(
        select(User).where(User.id == user_id)
    )
    user = user.scalar_one_or_none()

    if not user:
        raise HTTPException(404)

    # Step 3: Serialize
    result = user.to_dict()

    # Step 4: Cache-এ রাখো — ~2ms
    await redis.setex(f"user:{user_id}", 300, json.dumps(result))

    return result
```

এই একটা endpoint-এ:
- Redis check: 2ms (fast, local network)
- DB query: 50ms (depends on query complexity, index)
- Serialization: ~1ms (CPU, Python)
- Redis write: 2ms

Total: ~55ms। এর মধ্যে CPU actually কাজ করেছে মাত্র ~3ms। বাকি 52ms I/O wait।

Async দিয়ে এই 52ms অন্য request serve করতে পারে।

---

## Step 7: Database এবং External API — Latency-র মূল উৎস

Request-এর মোট সময়ের বেশিরভাগ এখানে যায়। Application logic নয়, Python code নয় — DB এবং external service।

**Typical latency breakdown:**

```
একটা typical API request-এর সময়:

┌─────────────────────────────────────────────────────────┐
│ Component              │ Time    │ % of total            │
├────────────────────────┼─────────┼───────────────────────┤
│ TCP + TLS (new conn)   │ 50ms    │ ~8%                   │
│ Nginx processing       │ 1ms     │ ~0.2%                 │
│ Gunicorn routing       │ 0.5ms   │ ~0.1%                 │
│ Python business logic  │ 5ms     │ ~0.8%                 │
│ DB query               │ 50ms    │ ~8%                   │
│ External API call      │ 400ms   │ ~66%                  │
│ Response serialization │ 2ms     │ ~0.3%                 │
│ Network back to client │ 100ms   │ ~16%                  │
├────────────────────────┼─────────┼───────────────────────┤
│ Total                  │ ~608ms  │ 100%                  │
└─────────────────────────────────────────────────────────┘
```

Python code মোট সময়ের 1%-এর কম। External API call-এ 66%।

এই কারণেই:
- Python optimize করা generally waste of time (1% কমালেও কিছু হয় না)
- External API slow হলে async দিয়ে overlap করাই best strategy
- DB slow হলে index, query optimize করো
- Cache (Redis) দিয়ে repeated DB call বাঁচাও

**DB Latency কীভাবে বাড়ে:**

```
Query 1 (indexed): SELECT * FROM users WHERE id = 1
└── ~1ms (index hit, single row)

Query 2 (no index): SELECT * FROM users WHERE email = 'x@y.com'
└── ~50ms (full table scan, 1M rows)

Query 3 (N+1 problem):
  for user in users:            # 100 users
      await user.posts.all()   # 100 separate DB queries
└── 100 × 10ms = 1000ms 😱
```

N+1 problem হলো সবচেয়ে common DB performance killer। একটা query-র result-এর জন্য loop-এ আরও queries করা।

---

## Step 8: Response তৈরি এবং ফেরত যাওয়া

Application response তৈরি করলে সেটা একই path ধরে ফিরে যায়।

```
Application
    │  return JSONResponse({"user": {...}})
    ▼
Worker
    │  response serialize করে bytes-এ
    ▼
Gunicorn
    │  bytes Nginx-এ পাঠায়
    ▼
Nginx
    │  response headers যোগ করে (Server, Date, etc.)
    │  গছিলে response buffer করে
    │  client-এর connection speed অনুযায়ী পাঠায়
    ▼
Browser
    │  bytes parse করে
    │  JSON deserialize করে
    ▼
JavaScript code response data পায়
```

**Response buffering আবার:**

Nginx response-ও buffer করে। Gunicorn দ্রুত পুরো response Nginx-কে দিয়ে দেয়। Nginx তারপর ধীরে ধীরে client-কে পাঠায়। এই সময়ে Gunicorn worker free — নতুন request নিতে পারে।

---

## তিনটা Model একসাথে — Timeline Comparison

ধরো 3টা request, প্রতিটায়: 50ms DB wait + 300ms external API wait + 10ms Python processing।

**Sync (1 worker):**

```
Worker:

Request A: [10ms code][50ms DB][300ms API][10ms resp] = 370ms
Request B:                                             [10ms code][50ms DB][300ms API][10ms resp] = 370ms
Request C:                                                                                         [10ms code]...

Timeline: 0 ─────────────────── 370ms ───────────────── 740ms ──────── 1110ms

Total for all 3: 1110ms
```

**Threaded (1 worker, 3 threads):**

```
Thread 1 (Req A): [10ms][██50ms DB██][████████████300ms API████████████][10ms]
Thread 2 (Req B):       [10ms][██50ms DB██][████████████300ms API████████████][10ms]
Thread 3 (Req C):             [10ms][██50ms DB██][████████████300ms API██████][10ms]

Timeline: 0 ──────────────────────────────────────────────────────── ~380ms

Total for all 3: ~380ms (প্রায় একটা request-এর সমান)
```

Thread-গুলো DB এবং API wait overlap করতে পেরেছে।

**Async (1 worker, event loop):**

```
Event loop:

t=0ms:    → A শুরু (10ms code) → DB await → pause
t=10ms:   → B শুরু (10ms code) → DB await → pause
t=20ms:   → C শুরু (10ms code) → DB await → pause

t=50ms:   → A DB done → API await → pause
t=60ms:   → B DB done → API await → pause
t=70ms:   → C DB done → API await → pause

t=350ms:  → A API done → 10ms response → done ✓
t=360ms:  → B API done → 10ms response → done ✓
t=370ms:  → C API done → 10ms response → done ✓

Timeline: 0 ─────────────────────────────── ~370ms

Total for all 3: ~370ms
```

Thread-এর মতো সময়। কিন্তু 3টা OS thread-এর বদলে 3টা tiny coroutine। 1000 concurrent request-এ thread model: ~1000 OS threads (গিগাবাইট memory), async model: ~1000 coroutines (কয়েক MB)।

---

## কোথায় সময় যায় — এবং কোথায় খুঁজবে সমস্যা

এই mental model টা সবচেয়ে practical:

```
একটা request-এর lifetime:

[Network in] → [Nginx] → [Queue] → [Worker] → [App logic] → [I/O wait] → [Response] → [Network out]
      │            │          │          │            │               │
    সমস্যা:      misconfig  worker    blocking     bad query    external
    high         SSL cert   shortage  code         N+1          API slow
    latency      error      workers   in async     no index     timeout
                            full      context
```

**Bottleneck কোথায় তা কীভাবে চিনবে:**

```
Symptom: সব request slow, even simple ones
→ Check: DB slow? (query time log করো)
→ Check: External API slow? (response time measure করো)
→ Check: Event loop blocking? (CPU 100% single core হলে)

Symptom: কিছু request fast, কিছু হঠাৎ খুব slow
→ Check: N+1 query problem?
→ Check: Cache miss হচ্ছে?
→ Check: External API intermittently slow?

Symptom: Load বাড়লে আচমকা সব fail
→ Check: Worker exhausted? (all workers busy)
→ Check: DB connection pool full?
→ Check: Memory out? (too many threads)

Symptom: CPU 100%, response slow
→ Check: CPU-bound code async loop-এ?
→ Check: Serialization bottleneck? (huge JSON response)
```

---

## Failure Scenarios — কী হলে কী ঘটে

**Scenario 1: DB হঠাৎ slow হয়ে গেল**

```
Normal:           [DB 50ms]
DB slow হলে:     [████████████████ DB 2000ms ████████████████]

Sync worker-এ:
→ প্রতিটা worker 2000ms blocked
→ নতুন request queue-এ জমে
→ queue full → connection refused
→ User দেখে: 502 Bad Gateway / timeout

Async worker-এ:
→ প্রতিটা coroutine 2000ms await করে
→ কিন্তু event loop চলছে, অন্য request নিতে পারছে
→ ধীরে হলেও serve করছে (graceful degradation)
→ User দেখে: slow response, কিন্তু timeout কম
```

**Scenario 2: External API timeout**

```
External API 30 সেকেন্ড response দিচ্ছে না।

Without timeout:
→ worker/coroutine 30 সেকেন্ড ধরে অপেক্ষা
→ সব worker এভাবে আটকে গেলে: complete outage

With timeout:
@app.get("/payment")
async def pay():
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            result = await client.post(payment_api_url, ...)
    except httpx.TimeoutException:
        raise HTTPException(503, "Payment service unavailable")
→ 5 সেকেন্ডে fast fail, user দেখে meaningful error
```

**Scenario 3: Event loop block হলে**

```
Async worker, কিন্তু একটা request-এ blocking call:

t=0ms:   Req A শুরু, DB await → pause
t=0ms:   Req B শুরু → requests.get(url)  ← BLOCKING!
         ↑
         এখানে পুরো event loop আটকে গেল।
         Req A-র DB response এলেও resume হবে না।
         Req C আসলেও নেওয়া হবে না।

t=500ms: requests.get() শেষ হলো, Req B done
t=500ms: এবার Req A resume হলো (500ms delay!)
t=500ms: Req C শুরু হলো
```

একটা blocking call পুরো event loop-কে hostage করে রাখে।

---

## Latency vs Throughput — দুটো আলাদা জিনিস

এই দুটো metric গুলিয়ে ফেলা common mistake।

**Latency:** একটা single request শেষ হতে কতটুকু সময়।

**Throughput:** প্রতি সেকেন্ডে কতটা request handle হচ্ছে।

```
উদাহরণ:

Sync, 1 worker:
  Latency (single request): 200ms
  Throughput: 5 req/sec (1000ms ÷ 200ms)

Async, 1 worker, 100 concurrent:
  Latency (single request): 200ms       ← same!
  Throughput: 500 req/sec               ← much higher!
```

Async latency কমায় না। একটা request তখনও 200ms নেয়। কিন্তু wait-এর সময় অন্য request চলে, তাই প্রতি সেকেন্ডে অনেক বেশি request serve হয়।

তুমি যদি দেখো "async করলাম, কিন্তু single request speed আগের মতোই" — এটা expected। Async-এর gain আসে high concurrency-তে, single request speed-এ না।

---

## Caching — Latency কমানোর সবচেয়ে effective উপায়

Async দিয়ে I/O wait overlap করা যায়, কিন্তু সেই wait টা যদি না হয়? Cache ঠিক এটাই করে।

```
Without cache:
Request → DB query (50ms) → Response

With cache (cache hit):
Request → Redis check (2ms) → Response  ← 25x faster!

With cache (cache miss):
Request → Redis check (2ms) → DB query (50ms) → Redis write (2ms) → Response
        (পরের request cache থেকে পাবে)
```

Cache strategy:

```python
async def get_user(user_id: int):
    # 1. Cache check করো
    cache_key = f"user:{user_id}"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Cache miss → DB query
    user = await db.get(User, user_id)

    # 3. Cache-এ store করো (5 মিনিটের জন্য)
    await redis.setex(cache_key, 300, json.dumps(user.to_dict()))

    return user.to_dict()
```

**Cache invalidation** (কখন cache clear করবে):

```python
async def update_user(user_id: int, data: dict):
    await db.update(User, user_id, data)
    await redis.delete(f"user:{user_id}")  # stale cache মুছে দাও
```

---

## Final Mental Model

```
Latency  = Time to complete one request
         = Network + Nginx + Queue + Worker + I/O wait + Response

Throughput = Requests per second
           = How well you overlap the waits


Async improves throughput     (I/O overlap)
Thread improves throughput    (parallel waits)
Process improves CPU usage    (true parallel compute)
Cache improves latency        (eliminate the wait)
Index improves DB latency     (faster I/O)
CDN improves network latency  (closer to client)
```

Performance optimization-এ সবচেয়ে বড় ভুল হলো wrong layer-এ optimize করা। Python code micro-optimize করা যখন bottleneck DB-তে — সেটা waste। সঠিক layer খুঁজে সেখানে কাজ করো।

---

## ⚔️ Mini Challenge

ধরো তোমার setup:
- 1 async worker (UvicornWorker)
- 3টা request একসাথে এলো
- প্রতিটা request করে:
  - DB query: `await asyncpg.fetch(...)` → 100ms
  - CPU task: `result = compress_image(data)` → 1 সেকেন্ড (blocking, no await)

**প্রশ্ন ১:** Total time কত লাগবে সব 3টা request শেষ হতে?

Timeline টা হাতে এঁকে দেখাও — কখন কোন request চলছে, কখন pause, কোথায় block।

**প্রশ্ন ২:** `compress_image()` call-এর সময় event loop-এ ঠিক কী হচ্ছে? অন্য দুটো request কি চলতে পারছে?

**প্রশ্ন ৩:** এই সমস্যা fix করার দুটো উপায় বলো।

**প্রশ্ন ৪:** Fix করলে new total time কত হবে? কেন?

এই চারটা প্রশ্নের উত্তর দিতে পারলে async lifecycle পুরোপুরি clear।