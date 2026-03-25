# Ground Zero: HTTP, Process, Thread, এবং Async — সব কিছুর ভিত্তি

> *"যে বাড়ির ভিত্তি দেখেনি, সে দেয়ালের ফাটল বুঝতে পারবে না।"*

---

## কেন এই পর্বটা গুরুত্বপূর্ণ?

FastAPI বা Django দিয়ে API লেখা শুরু করলে মনে হয় সব বুঝেছি। Route লিখলাম, DB query দিলাম, response পাঠালাম — কাজ হচ্ছে।

কিন্তু যেদিন production-এ হাজারো user একসাথে ঢুকল, server হঠাৎ slow হয়ে গেল, latency বেড়ে গেল, কিছু request timeout করল — সেদিন বুঝলাম কাজ করানো আর বোঝা এক জিনিস না।

এই পর্বে আমরা একদম নিচ থেকে শুরু করব। HTTP request আসলে কী? Socket কে? Process আর Thread-এর পার্থক্য কী? Event loop কীভাবে কাজ করে? এই জিনিসগুলো মাথায় পরিষ্কার থাকলে performance debugging, concurrency bug, scaling decision — সবকিছু অনেক সহজ হয়ে যায়।

---

## HTTP — তারের উপর দিয়ে কী যায়?

সবকিছুর শুরু এখানে। Browser থেকে বা কোনো client থেকে যখন request পাঠানো হয়, সেটা আসলে একটা **plain text message**।

```
GET /users?id=10 HTTP/1.1
Host: example.com
User-Agent: curl/7.68.0
Accept: application/json
```

কোনো binary নয়, কোনো magic নয় — শুধু structured text। এই text-এ চারটা জিনিস থাকে:

**Method** — আমি কী করতে চাই?
- `GET` = data চাই
- `POST` = data পাঠাচ্ছি
- `PUT` = কিছু update করছি
- `DELETE` = কিছু মুছছি

**Path** — কোথায় যেতে চাই? `/users`, `/api/orders/42`

**Headers** — extra context। Authentication token কোথায়? Content কোন format-এ? Client কে?

**Body** — POST বা PUT-এ actual data। GET-এ সাধারণত থাকে না।

Server-ও একইভাবে response পাঠায়:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 21

{"name": "Shomi"}
```

Status code (`200 OK`, `404 Not Found`, `500 Internal Server Error`), headers, তারপর body।

### HTTP-এর দুটো চরিত্র — মনে রাখার মতো

**Stateless:** প্রতিটা request server-এর কাছে সম্পূর্ণ নতুন। আগের request-এ কী হয়েছিল, server সেটা মনে রাখে না। তুমি login করলে, পরের request-এ আবার prove করতে হবে তুমি কে — তাই JWT token বা session cookie পাঠাতে হয়।

এটা performance-এ সুবিধা দেয়: server-কে user-এর "state" মেমোরিতে ধরে রাখতে হয় না। যেকোনো server যেকোনো request handle করতে পারে।

**Connection reuse:** HTTP/1.1-এ একটাই TCP connection বারবার ব্যবহার হতে পারে (keep-alive)। প্রতিটা request-এ নতুন connection তৈরির overhead বাঁচে। HTTP/2-এ আরো একধাপ এগিয়ে — একটা connection-এ একসাথে অনেক request চলতে পারে (multiplexing)।

---

## Socket — দরজাটা আসলে কোথায়?

HTTP text যায় কীভাবে? তারের মধ্যে দিয়ে যায় **TCP connection**-এ, আর সেটার entry point হলো **socket**।

Socket হলো OS-এর একটা abstraction — একটা endpoint যেখানে network data আসে এবং যায়। তোমার server যখন port 8000-এ listen করছে, সে আসলে একটা socket তৈরি করেছে এবং সেই socket-এ নতুন connection-এর জন্য অপেক্ষা করছে।

```
Client                           Server
  │                                 │
  │──── TCP connection establish ───▶│  ← Socket এখানে connection accept করে
  │                                 │
  │──── raw bytes পাঠাল ───────────▶│  ← Server bytes পড়ে HTTP parse করে
  │                                 │
  │◀─── raw bytes পেল ──────────────│  ← Server HTTP response bytes পাঠাল
  │                                 │
```

তোমার FastAPI বা Flask এই low-level parsing করে দেয়। কিন্তু জানাটা দরকার কারণ socket কতটা দ্রুত data read করতে পারছে, network কতটা congested — এগুলো সরাসরি app-এর performance-কে প্রভাবিত করে।

---

## Process — OS-এর execution unit

Request এলো, socket-এ ঢুকল। এখন কে সেটা handle করবে?

এখানে আসে **process**-এর ধারণা।

Process হলো OS-এর একটা isolated execution unit। তুমি যখন Python script চালাও, একটা process তৈরি হয়। Gunicorn চালালে সে multiple process তৈরি করে।

প্রতিটা process-এর আছে:

```
┌──────────────────────────────────┐
│           Process A              │
│  ┌────────────┐ ┌─────────────┐  │
│  │   Memory   │ │   File      │  │
│  │  (Stack +  │ │ Descriptors │  │
│  │    Heap)   │ │  (Sockets,  │  │
│  └────────────┘ │   Files)    │  │
│                 └─────────────┘  │
│  নিজস্ব isolated পরিবেশ         │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│           Process B              │
│  ┌────────────┐ ┌─────────────┐  │
│  │   Memory   │ │   File      │  │
│  │ (আলাদা)   │ │ Descriptors │  │
│  └────────────┘ └─────────────┘  │
│  A-এর কিছুই দেখতে পায় না       │
└──────────────────────────────────┘
```

এই isolation-এর দুটো সুবিধা:

**Crash isolation:** Process A crash করলে Process B চলতে থাকে। Gunicorn-এর একটা worker মরলে বাকিরা request নেয়।

**True parallelism:** যদি machine-এ 4টা CPU core থাকে, 4টা process একসাথে 4টা আলাদা core-এ চলতে পারে। সত্যিকারের parallel execution।

কিন্তু process-এর cost আছে। প্রতিটা process আলাদা memory নেয় — একই Python interpreter, একই app code, সব কপি হয়। 10টা Gunicorn worker মানে app code 10 বার memory-তে লোড। Process switch করা (context switching)-ও OS-এর জন্য ব্যয়বহুল। তাই worker সংখ্যা বাড়িয়ে দিলেই performance বাড়ে না — একটা point-এর পরে উল্টো কমে।

---

## Thread — একই বাড়িতে অনেক কামরা

Process-এর memory overhead সমাধান করতে এলো **thread**।

Thread হলো একটা process-এর ভেতরে আলাদা execution path। একই process-এর সব thread:

```
┌─────────────────────────────────────────┐
│                 Process                 │
│  ┌────────────────────────────────────┐ │
│  │         Shared Memory              │ │
│  │   (সব thread এটা দেখতে পায়)       │ │
│  └────────────────────────────────────┘ │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ Thread 1 │  │ Thread 2 │  │Thread 3│ │
│  │ (Stack)  │  │ (Stack)  │  │(Stack) │ │
│  │ নিজের   │  │ নিজের   │  │নিজের  │ │
│  │ execution│  │ execution│  │exec.   │ │
│  └──────────┘  └──────────┘  └────────┘ │
└─────────────────────────────────────────┘
```

একই memory share করে, তাই নতুন process-এর মতো আলাদা করে সব কপি করতে হয় না। Thread তৈরি করা process তৈরির চেয়ে অনেক সস্তা।

কিন্তু shared memory মানেই বিপদ। দুটো thread যদি একই data একসাথে modify করে:

```
Thread 1: balance = get_balance()    # balance = 1000
Thread 2: balance = get_balance()    # balance = 1000 (এখনো পুরনো!)
Thread 1: balance -= 500             # balance = 500
Thread 2: balance -= 500             # balance = 500 (500 কাটা হলো না!)
Thread 1: save_balance(500)
Thread 2: save_balance(500)          # ভুল! 500 টাকা গায়েব হয়ে গেল
```

এটাই **race condition** — দুটো thread একসাথে কাজ করতে গিয়ে একজনের কাজ আরেকজন overwrite করল। এড়াতে Lock, Semaphore, ইত্যাদি synchronization primitive দরকার পড়ে, যেটা নিজেই complexity এবং potential deadlock আনে।

---

### Python-এর বিশেষ সমস্যা: GIL

Python-এ thread ব্যবহারে একটা অতিরিক্ত twist আছে — **GIL (Global Interpreter Lock)**।

Python interpreter-এর ভেতরে একটা global lock আছে। এই lock যে thread ধরবে, সেই thread Python bytecode execute করতে পারবে। অন্যরা পারবে না।

```
সময় →   t1    t2    t3    t4    t5    t6
Thread 1: [GIL] [   ] [GIL] [   ] [GIL] [   ]
Thread 2: [   ] [GIL] [   ] [GIL] [   ] [GIL]
           ↑ একসাথে দুজন কখনো Python চালায় না
```

এর মানে হলো:

**CPU-bound কাজে** (ভারী calculation) multiple thread ব্যবহার করলেও কোনো লাভ নেই। একটা thread কাজ করার সময় GIL ধরে রাখে, বাকিরা অপেক্ষা করে। দুটো thread থাকলেও effectively একটাই চলছে।

**I/O-bound কাজে** (DB query, file read, API call) thread কাজ করে। কারণ I/O wait-এর সময় thread GIL ছেড়ে দেয় — অন্য thread ঢুকে কাজ করতে পারে।

এজন্যই Python-এ CPU-heavy কাজের জন্য multiprocessing (GIL-কে bypass করে আলাদা process), আর I/O-heavy কাজের জন্য threading বা async ব্যবহার করা হয়।

---

## Event Loop — একটাই thread, অনেক কাজ

Process এবং Thread — দুটোই concurrency-র সমাধান দেয়। কিন্তু দুটোরই একটা সমস্যা: waiting-এ resource নষ্ট হয়।

একটা thread DB query দিয়ে অপেক্ষা করছে — সে কিছু করছে না, কিন্তু memory দখল করে আছে। ১০,০০০ concurrent user হলে ১০,০০০ thread? সেটা feasible না।

**Event loop** এই সমস্যার মার্জিত সমাধান।

Event loop একটাই thread-এ চলে এবং অনেকগুলো task manage করে। এর মূল loop:

```
┌─────────────────────────────────────────────┐
│              Event Loop                     │
│                                             │
│  while True:                                │
│    ┌─────────────────────────────────────┐  │
│    │  ready tasks-এর list দেখো          │  │
│    └────────────────┬────────────────────┘  │
│                     │                       │
│    ┌────────────────▼────────────────────┐  │
│    │  একটা ready task নাও, চালাও        │  │
│    │  সে await করলে → pause, অন্যটা নাও │  │
│    └────────────────┬────────────────────┘  │
│                     │                       │
│    ┌────────────────▼────────────────────┐  │
│    │  I/O events check করো              │  │
│    │  কোনো task-এর I/O শেষ হলে          │  │
│    │  সেটাকে ready list-এ দাও           │  │
│    └─────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

মূল idea: কোনো task I/O-এর জন্য wait করছে মানে সে নিজেই বলে "আমি এখন অপেক্ষা করছি, তুমি অন্য কাজ করো।" Event loop তখন অন্য ready task চালায়। I/O complete হলে waiting task-কে আবার চালু করে।

একটা receptionist-এর সাথে তুলনা করো। সে একসাথে ১০টা ফোন call handle করছে:
- Call 1-কে hold-এ রাখল (file খুঁজছে)
- Call 2-এর সাথে কথা বলল
- Call 3-এর প্রশ্নের উত্তর দিল
- Call 1-এর file এলো, তার সাথে কথা বলতে ফিরল

একজন মানুষ, একসাথে অনেক কাজ progress হচ্ছে।

---

## Blocking বনাম Non-blocking — সবচেয়ে গুরুত্বপূর্ণ concept

Async code লেখার সময় একটাই নিয়ম সবচেয়ে বেশি গুরুত্বপূর্ণ — **blocking vs non-blocking।**

```python
# ❌ Blocking
time.sleep(5)
```

এই line execute হলে পুরো Python thread ৫ সেকেন্ড ঘুমিয়ে পড়ে। Event loop চলে না। অন্য কোনো request handle হয় না। Server effectively frozen।

```python
# ✔️ Non-blocking
await asyncio.sleep(5)
```

এখানে `await` মানে: "আমি এখন ৫ সেকেন্ড অপেক্ষা করব — কিন্তু event loop, তুমি এই সময়ে অন্য কাজ করো। ৫ সেকেন্ড পর আমাকে resume করো।" Thread ঘুমায় না, শুধু এই task pause হয়।

```
Blocking (time.sleep):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Request A:  [sleep 5s ████████████████████] done
Request B:  অপেক্ষা করছে... সময় নষ্ট
Request C:  অপেক্ষা করছে... সময় নষ্ট

Non-blocking (await asyncio.sleep):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Request A:  শুরু → pause ─────────────── resume → done
Request B:          শুরু → pause ──── resume → done
Request C:                 শুরু → pause ── resume → done
```

> **সোনার নিয়ম:** Async code-এর ভেতরে একটাও blocking call থাকলে পুরো event loop থেমে যায়। `time.sleep()`, `requests.get()`, sync database driver — এগুলো async context-এ ব্যবহার করলে concurrency-র সব সুবিধা শেষ।

---

## Concurrency বনাম Parallelism — দুটো আলাদা জিনিস

এই দুটো শব্দ প্রায়ই একসাথে ব্যবহার হয়, কিন্তু মানে আলাদা।

**Concurrency** মানে একাধিক task progress করছে — কিন্তু exactly একই মুহূর্তে নয়। একটা কিছুটা এগোয়, pause হয়, অন্যটা এগোয়। দ্রুত switching-এর কারণে মনে হয় সব একসাথে চলছে।

**Parallelism** মানে একাধিক task সত্যিকার অর্থে একই সময়ে চলছে — আলাদা CPU core-এ।

```
Concurrency (1 CPU, interleaving):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CPU: [Task A][Task B][Task A][Task C][Task B][Task A]
      দ্রুত switch করছে, সব "progress" করছে

Parallelism (multiple CPU):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CPU 1: [Task A ──────────────────────]
CPU 2: [Task B ──────────────────────]
CPU 3: [Task C ──────────────────────]
        সত্যিকার অর্থে একসাথে
```

**রান্নাঘরের analogy:**

Concurrency = একজন chef যে একসাথে তিনটা dish manage করছে। ডাল চুলায় দিল, সেটা সিদ্ধ হওয়ার সময় মাছ কাটছে, মাছ ভাজতে দিয়ে সবজি কুটছে। হাত একটাই, কিন্তু তিনটাই এগোচ্ছে।

Parallelism = তিনজন chef, প্রত্যেকে আলাদা dish-এ একসাথে কাজ করছে। সত্যিকারের আলাদা।

Async = concurrency। Multiple process = parallelism।

---

## CPU-bound বনাম I/O-bound — কোন সমস্যায় কোন সমাধান

কোন approach কাজে লাগবে সেটা নির্ভর করে কাজের ধরনের উপর।

**CPU-bound কাজ:**
- Image/video processing
- Machine learning inference
- Heavy mathematical calculation
- Password hashing

এগুলোতে CPU সারাক্ষণ কাজ করছে — কোনো waiting নেই। Async এখানে সাহায্য করে না কারণ pause করার কোনো জায়গাই নেই।

→ সমাধান: **Multiprocessing** — আলাদা process, আলাদা CPU core, GIL-কে bypass।

**I/O-bound কাজ:**
- Database query
- External HTTP API call
- File read/write
- Cache (Redis) access

এগুলোতে CPU বেশিরভাগ সময় অপেক্ষা করছে — response আসার জন্য।

→ সমাধান: **Async বা Threading** — waiting time-এ অন্য কাজ করো।

```
I/O-bound request timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[auth DB] [wait ────────] [product DB] [wait ──────] [payment API] [wait ──────────────] [save] [email]
  5ms         20ms           5ms           15ms          5ms             300ms              5ms    5ms
  
মোট ≈ 360ms — কিন্তু CPU কাজ করেছে মাত্র ~25ms!
বাকি 335ms শুধু অপেক্ষা।
```

এই 335ms-ই হলো async-এর সুযোগ।

---

## Request Handling-এর তিনটা Model

এতক্ষণের সব কিছু মিলিয়ে দেখো — request কীভাবে handle হতে পারে।

**Model 1 — Synchronous (Blocking):**

```
Request 1 আসল ─▶ [কাজ + wait + কাজ] ─▶ Response 1
                                                    ─▶ Request 2 আসল ─▶ [কাজ] ─▶ Response 2
```

সহজ, predictable। কিন্তু একটা শেষ না হলে পরেরটা শুরু হয় না।

**Model 2 — Threaded:**

```
Request 1 ─▶ Thread 1: [কাজ + wait + কাজ] ─▶ Response 1
Request 2 ─▶ Thread 2: [কাজ + wait + কাজ] ─▶ Response 2   ← একসাথে
Request 3 ─▶ Thread 3: [কাজ + wait + কাজ] ─▶ Response 3
```

Multiple request একসাথে চলে। I/O-bound কাজে কাজ করে। কিন্তু GIL-এর কারণে CPU কাজে কোনো লাভ নেই, এবং অনেক thread মানে অনেক memory।

**Model 3 — Async (Non-blocking):**

```
Event Loop (single thread):

t=0ms: Req A শুরু ─▶ DB query ─▶ await (pause)
t=0ms: Req B শুরু ─▶ DB query ─▶ await (pause)
t=0ms: Req C শুরু ─▶ DB query ─▶ await (pause)
t=20ms: A-র DB response ─▶ A resume ─▶ Response A
t=20ms: B-র DB response ─▶ B resume ─▶ Response B
t=20ms: C-র DB response ─▶ C resume ─▶ Response C
```

একটাই thread, কিন্তু waiting time-এ অন্য request চলছে। I/O-heavy workload-এ সবচেয়ে efficient।

---

## Throughput বনাম Latency — দুটো আলাদা metric

Performance নিয়ে কথা বলতে গেলে এই দুটো সবসময় আলাদা রাখতে হবে।

**Latency:** একটা single request-এর শুরু থেকে শেষ পর্যন্ত সময়।

**Throughput:** প্রতি সেকেন্ডে server কতটা request handle করতে পারে (Requests Per Second, RPS)।

Async model latency কমায় না — একটা request-এর কাজ কম হয়নি। কিন্তু waiting time-এ অন্য request চলায় **throughput অনেক বাড়ে।** একই hardware-এ অনেক বেশি user serve করা যায়।

```
উদাহরণ — প্রতিটা request 100ms:

Sync (1 worker):
  Throughput = 10 RPS (প্রতি সেকেন্ডে ১০টা, serial)

Async (1 worker, I/O-bound):
  Throughput = ~500 RPS (wait-এ অন্যরা চলে)
  
Latency দুটোতেই ~100ms — কিন্তু throughput বিশাল পার্থক্য
```

---

## Backpressure — যখন সামলানো যায় না

Async বা threaded model-এ throughput বাড়লেও একটা limit আছে। যদি incoming request-এর হার server-এর capacity-র চেয়ে বেশি হয়ে যায়:

```
স্বাভাবিক অবস্থা:
Requests: ──▶──▶──▶──▶                    ──▶──▶──▶──▶
Server:   [handle][handle][handle][handle]

Overloaded:
Requests: ──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶
Server:   [handle][handle][handle][queue বাড়ছে...]
                                          [timeout!][crash!]
```

লক্ষণ: response ধীরে ধীরে slow হয়, timeout বাড়ে, শেষে crash।

সমাধান তিনটা:
- **Queue:** Excess request সরাসরি reject না করে queue-এ রাখো, ধীরে ধীরে process করো
- **Rate limiting:** প্রতি user বা IP-কে নির্দিষ্ট সংখ্যক request-এ limit করো
- **Load balancing:** Multiple server-এ request ভাগ করো

এখনই deep dive করছি না, কিন্তু মাথায় রাখো — যেকোনো scalable system design-এ এই concept বারবার আসবে।

---

## Common ভুল ধারণা — এখনই পরিষ্কার করো

**"Async মানেই দ্রুত"** — না। Async তখনই fast যখন কাজটা I/O-bound। CPU-heavy কাজে async কোনো সুবিধা দেয় না — একটাই thread, কিছু pause করার জায়গা নেই।

**"Python-এ threads parallel চলে"** — না, CPU-bound কাজে চলে না। GIL-এর কারণে একসময় একটাই thread Python চালাতে পারে। I/O-bound কাজে threads useful, CPU-bound-এ multiprocessing দরকার।

**"Worker বেশি দিলেই performance বাড়বে"** — না। প্রতিটা worker memory নেয়। Context switching overhead বাড়ে। একটা point-এর পর performance কমে। Measure করে সিদ্ধান্ত নিতে হবে।

**"Async library use করলেই async হয়ে গেল"** — না। Async library ব্যবহার করলেও যদি কোথাও sync blocking call থাকে, সেই জায়গায় event loop আটকে যাবে। একটা জায়গাই যথেষ্ট সব সুবিধা নষ্ট করতে।

---

## Final Mental Model — তিনটা প্রশ্ন

যেকোনো performance সমস্যার সামনে পড়লে এই তিনটা প্রশ্ন করো:

```
┌─────────────────────────────────────────────┐
│  1. Code কি block করছে?                     │
│     হ্যাঁ → async করো বা offload করো        │
│                                             │
│  2. আমি wait করছি নাকি compute করছি?        │
│     Wait (I/O) → async বা thread            │
│     Compute (CPU) → multiprocessing         │
│                                             │
│  3. Concurrency দরকার নাকি parallelism?     │
│     Interleaving যথেষ্ট → async             │
│     True parallel চাই → multiple process    │
└─────────────────────────────────────────────┘
```

এই তিনটার উত্তর জানলে backend-এর বেশিরভাগ performance decision নিজেই নিতে পারবে।

---

## চ্যালেঞ্জ: নিজে ভাবো

**Setup:**
- 1টি worker process
- 1টি thread
- Async code দিয়ে লেখা

**প্রশ্ন:**

এই single worker কি একসাথে multiple request handle করতে পারবে?

- কোন পরিস্থিতিতে পারবে?
- কোন পরিস্থিতিতে পারবে না — ঠিক কী করলে এটা fail করবে?

Hint: event loop-কে কী করলে block করা যায়? সেই উত্তরটাই বলবে কখন fail হবে।

---

## সারসংক্ষেপ

| Concept | মূল কথা |
|---|---|
| HTTP | Plain text message — method, path, headers, body |
| Socket | Network entry point — raw bytes আসে এখানে |
| Process | Isolated execution unit — crash-proof, true parallelism সম্ভব, কিন্তু heavy |
| Thread | Shared memory-তে lightweight execution — I/O-bound-এ useful, GIL CPU-bound-এ আটকায় |
| Event Loop | Single thread-এ অনেক task manage — await-এ pause, অন্যটা চলে |
| Blocking | Thread আটকে যায় — event loop freeze |
| Non-blocking | Task pause হয়, thread free — অন্য task চলে |
| Concurrency | Interleaving — একটাই thread, মনে হয় সব একসাথে |
| Parallelism | True simultaneous — আলাদা CPU core-এ |
| I/O-bound | Wait বেশি → async বা thread |
| CPU-bound | Compute বেশি → multiprocessing |

