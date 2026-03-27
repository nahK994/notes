# 🧵 Multitasking, Concurrency, আর Python-এর Asyncio

---

## 🤔 শুরুটা হোক একটা প্রশ্ন দিয়ে

ধরো তুমি একটা web app বানিয়েছো। কেউ request করলে তুমি তিনটা জায়গা থেকে data আনো — Weather API, News API, আর Stock API। প্রতিটা API response দিতে ১ সেকেন্ড নেয়।

তোমার app কতক্ষণ নেবে?

সহজ উত্তর: ৩ সেকেন্ড। একটার পর একটা।

কিন্তু একটু ভাবো — তিনটা API-কে তো তুমি একসাথে জিজ্ঞেস করতে পারতে। তাহলে ১ সেকেন্ডেই শেষ হতো। তুমি ২ সেকেন্ড শুধু অপেক্ষায় নষ্ট করলে।

এই "অপেক্ষার সময়টাকে কাজে লাগানো" — এটাই multitasking-এর মূল কথা। আর Python-এ এটা করার নাম **asyncio**।

কিন্তু asyncio বোঝার আগে, একটু পিছিয়ে যাই। কারণ multitasking একটা পুরনো সমস্যা — এবং OS থেকে শুরু করে Python পর্যন্ত সবাই এটা আলাদাভাবে solve করেছে।

---

## 🖥️ Part 1: Multitasking — সমস্যাটা কোথায়?

### কম্পিউটার আসলে একসাথে কী করতে পারে?

তুমি এখন হয়তো browser চালাচ্ছো, music শুনছো, আর background-এ antivirus চলছে। মনে হচ্ছে সব একসাথে হচ্ছে।

কিন্তু যদি তোমার একটাই CPU core থাকে — সে literally একসাথে একটাই কাজ করতে পারে। তাহলে এই "সব একসাথে" ব্যাপারটা হয় কীভাবে?

উত্তর হলো — OS খুব দ্রুত একটা থেকে আরেকটায় switch করছে। এত দ্রুত যে তোমার কাছে একসাথে মনে হচ্ছে। এটাকে বলে **time-slicing**।

```
CPU এর reality:
  Browser   → [চলছে 10ms]
  Music     → [চলছে 10ms]
  Antivirus → [চলছে 10ms]
  Browser   → [চলছে 10ms]
  ...
```

চোখে দেখলে মনে হবে সব একসাথে — কিন্তু আসলে পালা করে।

এই সমস্যাটা solve করতে OS দুটো mechanism দিয়েছে: **Process** আর **Thread**।

---

### Process — সম্পূর্ণ আলাদা বাড়ি

যখন তুমি Chrome খোলো, OS একটা **process** তৈরি করে। Process মানে হলো একটা সম্পূর্ণ বিচ্ছিন্ন execution environment — নিজের memory, নিজের resources, সব আলাদা।

Chrome আর Spotify দুটো আলাদা process। তারা একে অপরের memory দেখতে পায় না, কথা বলতে পারে না সরাসরি।

```
Process A (Chrome)     Process B (Spotify)
┌────────────────┐     ┌────────────────┐
│ নিজের memory   │     │ নিজের memory   │
│ নিজের stack    │     │ নিজের stack    │
│ নিজের heap     │     │ নিজের heap     │
└────────────────┘     └────────────────┘
    ↕ OS পাহারা দেয়, কেউ কাউকে ছুঁতে পারবে না
```

**সুবিধা:** একটা crash করলে আরেকটা মরবে না।
**অসুবিধা:** তৈরি করতে expensive, memory বেশি খায়, কথা বলানো কঠিন।

---

### Thread — একই বাড়ির আলাদা ঘর

Process-এর ভেতরেই ছোট ছোট execution unit তৈরি করা যায় — এগুলোকে বলে **Thread**।

Chrome-এর ভেতরে একটা thread হয়তো page render করছে, আরেকটা JavaScript চালাচ্ছে, আরেকটা network request করছে। তারা একই memory share করে, তাই কথা বলা সহজ।

```
Process (Chrome)
┌──────────────────────────────────┐
│  Thread 1      Thread 2          │
│  (Renderer)    (Network)         │
│      ↕              ↕            │
│      shared memory (heap)        │
└──────────────────────────────────┘
```

**সুবিধা:** হালকা, তৈরি করা সহজ, memory share করে।
**অসুবিধা:** shared memory মানেই বিপদ — দুটো thread একই জায়গায় একসাথে লিখলে সমস্যা হয়। এটাকে বলে **race condition**।

OS thread manage করে নিজের মতো — যেকোনো সময় একটা thread থামিয়ে আরেকটায় যেতে পারে। Thread-এর কোনো মতামত নেই। এটাকে বলে **preemptive multitasking**।

---

### তাহলে সমস্যা কোথায়?

Process আর Thread — দুটোই OS-এর tool। কিন্তু Python-এ এসে একটা নতুন সমস্যা দেখা দিলো।

ধরো তুমি ১০,০০০ API call করতে চাও concurrently। Thread দিয়ে করলে ১০,০০০ thread তৈরি করতে হবে।

সমস্যা: একটা thread ~১-৮ MB memory নেয়। ১০,০০০ thread = ~৮ GB RAM। আর প্রতিটা thread-এর মধ্যে switch করতে OS-এর সময় লাগে (context switch overhead)।

এই situation-এ Thread কাজের না।

আর এখানেই Python বললো — "আমি নিজেই manage করি।"

---

## 🔀 Part 2: Concurrency vs Parallelism — দুটো আলাদা জিনিস

Asyncio বোঝার আগে এই দুটো concept পরিষ্কার হওয়া দরকার। মানুষ প্রায়ই গুলিয়ে ফেলে।

### একটা রান্নাঘরের উদাহরণ

**Scenario:** তোমাকে চা, ডিম সিদ্ধ, আর toast বানাতে হবে।

**Parallelism — দুজন chef:**

```
Chef A: চা বানাচ্ছে     ────────────────→ done
Chef B: ডিম সিদ্ধ করছে ────────────────→ done
Chef C: toast করছে      ────────────────→ done
         [সবাই literally একই সময়ে কাজ করছে]
```

Parallelism মানে literally একই মুহূর্তে একাধিক কাজ — আলাদা আলাদা CPU core-এ, আলাদা আলাদা worker-এ।

**Concurrency — একজন smart chef:**

```
Chef: কেতলিতে পানি দিলো → [পানি গরম হচ্ছে, অপেক্ষা না করে]
      ডিম বসালো        → [ডিম সিদ্ধ হচ্ছে, অপেক্ষা না করে]
      bread দিলো       → [toast হচ্ছে]
      এখন পানির দিকে গেলো → চা বানালো
      ডিমের দিকে গেলো   → নামালো
      toast নামালো
```

Concurrency মানে একজনই কাজ করছে — কিন্তু smart ভাবে। অপেক্ষার সময়টায় অন্য কাজ সারছে। যেকোনো এক মুহূর্তে একটাই কাজ হচ্ছে, কিন্তু সব কাজ progress করছে।

### সহজ সংজ্ঞা:

| Concept | মানে | উদাহরণ |
|---|---|---|
| **Parallelism** | একই সময়ে multiple কাজ, multiple worker | multiprocessing, multiple CPU core |
| **Concurrency** | multiple কাজ progress করছে, কিন্তু যেকোনো সময়ে একটাই চলছে | asyncio, single thread |

### Python-এ কোনটা কখন?

```
CPU-heavy কাজ (ভারি calculation, image processing)
    → Parallelism দরকার
    → multiprocessing ব্যবহার করো (আলাদা CPU core)

I/O-heavy কাজ (API call, database, file read)
    → Concurrency যথেষ্ট
    → asyncio ব্যবহার করো (single thread, smart scheduling)
```

কেন? কারণ API call করার সময় CPU আসলে কিছুই করছে না — শুধু অপেক্ষা করছে। এই অপেক্ষার সময়টাকে কাজে লাগানোর জন্য আলাদা CPU লাগে না, শুধু smart scheduling দরকার।

---

## ⚙️ Part 3: Python-এর Concurrency — Asyncio

এখন মূল জায়গায় আসি।

Python বললো — Thread ছাড়াই আমি concurrency দিতে পারি। কীভাবে? **Coroutine** আর **Event Loop** দিয়ে।

### Coroutine কী? (Thread-এর সাথে তুলনা)

Thread হলো OS-এর বানানো execution unit। OS যেকোনো সময় থামাতে পারে, চালু করতে পারে।

**Coroutine** হলো Python-এর নিজস্ব execution unit। একটা function যেটা মাঝপথে pause করতে পারে এবং পরে ঠিক সেখান থেকে resume করতে পারে।

```python
# সাধারণ function — শুরু হয়, শেষ হয়, মাঝে কোনো pause নেই
def normal_function():
    print("শুরু")
    print("শেষ")

# Coroutine — মাঝপথে pause করতে পারে
async def coroutine():
    print("শুরু")
    await asyncio.sleep(1)  # এখানে pause, অন্যরা চলুক
    print("শেষ")            # পরে এখান থেকে resume
```

`async def` দিয়ে বানানো function-ই coroutine। এটাকে call করলে সাথে সাথে চলে না — একটা coroutine object তৈরি হয়।

```python
coro = coroutine()  # এখনো চলেনি, শুধু object তৈরি হয়েছে
```

| বিষয় | Thread | Coroutine |
|---|---|---|
| কে manage করে? | OS | Python / event loop |
| Switch কখন হয়? | যেকোনো সময় (OS জোর করে) | শুধু `await`-এ (নিজে থেকে) |
| Memory | ~১-৮ MB প্রতিটায় | ~কয়েক KB প্রতিটায় |
| কতটা চালানো যায়? | কয়েকশো | হাজার হাজার |
| Race condition? | সম্ভব | নেই (single thread) |

সবচেয়ে বড় পার্থক্য:

**Thread = preemptive** → OS জোর করে থামায়, coroutine-এর কোনো বলার নেই।

**Coroutine = cooperative** → নিজে `await` দিয়ে বলে "এখন অন্যজন চলুক।" না বললে সে চলতেই থাকবে।

---

### Event Loop — সিডিউলার যে সব manage করে

Coroutine নিজে নিজে চলতে পারে না। কেউ একজন লাগবে যে এদের চালাবে, pause করবে, resume করবে। সেই কেউ হলো **Event Loop**।

Event loop হলো একটা infinite loop যেটা সারাক্ষণ দেখছে — কোন coroutine এখন run করার জন্য ready আছে। সেটা run করে। যেটা I/O-এর জন্য অপেক্ষা করছে, সেটাকে সরিয়ে রাখে। I/O শেষ হলে সেটাকে আবার ready তালিকায় আনে।

```
Event Loop
    ↓
Ready Queue → execute (পরবর্তী await পর্যন্ত)
    ↓
await পেলে → pause, I/O কাজ OS-এর হাতে দাও
    ↓
OS notify করলে → সেই coroutine আবার Ready Queue-তে
    ↓
(চক্র চলতে থাকে)
```

মনে রাখো: Event loop **scheduler**, worker না। সে নিজে কাজ করে না — ঠিক করে কে কখন কাজ করবে।

একটা mental model:

```python
# Event loop এর কাজ মোটামুটি এরকম (simplified)
while True:
    task = ready_queue.pop()
    task.run()               # পরবর্তী await পর্যন্ত চালাও

    for completed in io_events:   # OS যা complete জানালো
        ready_queue.push(completed)  # আবার queue-তে দাও
```

---

### `await` — সবচেয়ে গুরুত্বপূর্ণ keyword

`await` মানে: "এই coroutine pause করো, thread না।"

```
"এই coroutine pause করো, thread না"
```

এটা বোঝাটা critical। `await` করলে:
- ✅ শুধু এই coroutine থামে
- ✅ Thread free থাকে
- ✅ Event loop অন্য coroutine চালাতে পারে
- ❌ Thread block হয় না
- ❌ পুরো program থামে না

```python
async def task():
    print("A")
    await asyncio.sleep(2)   # শুধু এই coroutine pause, thread free
    print("B")
```

Timeline:

```
t=0s: "A" print হলো
t=0s: await দেখলো → coroutine pause, control event loop-এ গেলো
t=0s: event loop এই ফাঁকে অন্য coroutine চালাতে পারছে
t=2s: sleep শেষ → OS জানালো → coroutine আবার ready queue-তে
t=2s: "B" print হলো
```

---

### Coroutine-এর জীবনচক্র

একটা coroutine তৈরি হয়ে শেষ হওয়া পর্যন্ত কয়েকটা stage পার করে:

```
Created → Scheduled → Running → Paused → Resumed → Done
```

**Created:** `coro = task()` — object তৈরি হলো, এখনো চলেনি।

**Scheduled:** `asyncio.create_task(coro)` — event loop-এর ready queue-তে গেলো। এখনো চলেনি, কিন্তু চলার জন্য line-এ দাঁড়িয়েছে।

**Running:** Event loop তুলে নিলো, চালাচ্ছে।

**Paused:** `await` পেলো → নিজে থেকে pause, control event loop-এ।

**Resumed:** যার জন্য অপেক্ষা করছিলো সেটা শেষ → আবার ready queue-তে → আবার চলবে।

**Done:** শেষ — result বা exception নিয়ে বের হলো।

---

### `await` vs `create_task` — Sequential vs Concurrent

এটাই সবচেয়ে বড় practical পার্থক্য।

**Sequential — একটার পর একটা:**

```python
result1 = await fetch_weather()   # শেষ হোক, তারপর
result2 = await fetch_news()      # এটা শুরু হবে
```

```
weather ────────────→ done
                           news ────────────→ done
[     1s     ]        [     1s     ]
মোট: ২ সেকেন্ড
```

`fetch_weather` শেষ না হওয়া পর্যন্ত `fetch_news` শুরুই হবে না।

**Concurrent — একসাথে schedule করো:**

```python
task1 = asyncio.create_task(fetch_weather())
task2 = asyncio.create_task(fetch_news())

result1 = await task1
result2 = await task2
```

```
weather ────────────→ done
news    ────────────→ done   (একই সময়ে চলছে)
[     1s     ]
মোট: ১ সেকেন্ড
```

`create_task` coroutine-কে immediately run করে না — event loop-এর ready queue-তে schedule করে দেয়। তারপর event loop দুটো task interleave করে চালায়। একটা `await`-এ থামলে অন্যটা চলে।

সহজ মনে রাখার উপায়:

```
await       → এখনই অপেক্ষা করো (sequential)
create_task → schedule করো (concurrent)
```

---

### OS-এর সাহায্য — epoll/kqueue

একটা প্রশ্ন আসতে পারে — asyncio কি আসলে background-এ thread ব্যবহার করে?

**না।** Asyncio সরাসরি OS-এর I/O mechanism ব্যবহার করে।

```
event loop → OS-কে বলে: "এই connection-এ data আসলে আমাকে জানিও"
             (OS নিজেই track করে, কোনো thread লাগে না)
event loop → এই ফাঁকে অন্য কাজ করছে
OS → "ওই connection ready" → notify
event loop → সেই coroutine-কে resume করে
```

Linux-এ এই mechanism-এর নাম **epoll**, macOS-এ **kqueue**। Python-এর asyncio এগুলো নিজে নিজে ব্যবহার করে — তোমাকে manually করতে হয় না।

---

## 💣 Part 4: ফাঁদগুলো — যেখানে নতুনরা আটকায়

### ফাঁদ ১: Blocking code — silent killer

```python
import time

async def handler():
    time.sleep(3)  # ⚠️ মারাত্মক ভুল!
```

দেখতে harmless মনে হচ্ছে — কিন্তু এটা পুরো event loop-কে ৩ সেকেন্ড জমিয়ে দেবে।

কেন?

```
event loop এই coroutine চালাচ্ছে
→ time.sleep(3) OS-কে বলে "এই thread ঘুমাক"
→ event loop ওই thread-এই চলে
→ thread ঘুমালে event loop-ও ঘুমায়
→ সমস্ত pending coroutine আটকে যায়
```

`time.sleep()` thread block করে। `asyncio.sleep()` শুধু coroutine pause করে।

```python
# ✅ সঠিক
async def handler():
    await asyncio.sleep(3)  # thread free, event loop চলছে
```

### ফাঁদ ২: CPU-heavy কাজ async-এ

```python
async def handler():
    await fetch_data()   # ✅ ঠিক আছে — I/O, pause হবে
    process_image()      # ⚠️ বিপদ! CPU কাজ, কোনো await নেই
```

`process_image()` যদি ২ সেকেন্ড ধরে ভারি calculation করে — এই পুরো সময় event loop আটকে।

```
process_image() → কোনো await নেই → কোনো pause নেই
→ event loop control পাচ্ছে না
→ সব pending coroutine অপেক্ষায়
```

Async শুধু তখনই কাজে আসে যখন কাজের মধ্যে **অপেক্ষা** আছে — network, disk, database। Pure CPU কাজে async কোনো সাহায্য করে না।

CPU-heavy কাজের সমাধান আসবে পরের অংশে (`run_in_executor` দিয়ে thread/process pool-এ পাঠানো)।

### ফাঁদ ৩: async ≠ parallel

```
async ≠ thread       (thread একাধিক, async single thread)
async ≠ parallel     (parallel মানে multiple CPU, async single thread)
async = event-driven (event হলে react করো)
```

Async-এ যেকোনো এক মুহূর্তে মাত্র একটা coroutine execute হচ্ছে। দুটো CPU-heavy কাজ async দিয়ে "parallel" করা যাবে না।

---

## 🌐 Part 5: Real-world — কোথায় asyncio লাগে, কোথায় না

### Use Case 1: একসাথে অনেক API Call

**❌ Without asyncio:**

```python
import requests

def get_dashboard():
    weather = requests.get("https://api.weather.com/...").json()   # ১s অপেক্ষা
    news    = requests.get("https://api.news.com/...").json()      # ১s অপেক্ষা
    stocks  = requests.get("https://api.stocks.com/...").json()    # ১s অপেক্ষা
    return weather, news, stocks
# মোট: ~৩ সেকেন্ড
```

**✅ With asyncio:**

```python
import asyncio, httpx

async def get_dashboard():
    async with httpx.AsyncClient() as client:
        weather, news, stocks = await asyncio.gather(
            client.get("https://api.weather.com/..."),
            client.get("https://api.news.com/..."),
            client.get("https://api.stocks.com/..."),
        )
    return weather.json(), news.json(), stocks.json()
# মোট: ~১ সেকেন্ড
```

৩টা API থাকলে ৩x fast। ১০টা থাকলে ~১০x fast।

---

### Use Case 2: High-Traffic Web Server

প্রতি সেকেন্ডে ১০০০ request, প্রতিটা database query করে (৫০ms লাগে)।

**❌ Blocking server:**

Request 1 → DB query (৫০ms অপেক্ষা) → response
Request 2 → Request 1 শেষ না হওয়া পর্যন্ত অপেক্ষা!

১০০০ request হলে শেষেরটা অপেক্ষা করবে: ১০০০ × ৫০ms = ৫০ সেকেন্ড।

**✅ Async server (FastAPI/aiohttp):**

```python
@app.get("/user/{id}")
async def get_user(id: int):
    result = await db.fetchrow("SELECT * FROM users WHERE id=$1", id)
    return result
```

সব ১০০০ request প্রায় একসাথে DB query করে — কেউ অপেক্ষা করছে, বাকিরা চলছে। সবার response: ~৫০ms।

এটাই কারণ FastAPI এত fast।

---

### Use Case 3: Rate-limited API থেকে বড় data

১০,০০০ record নামাতে হবে, কিন্তু API-এ rate limit — সেকেন্ডে সর্বোচ্চ ১০টা request।

```python
async def fetch_all():
    semaphore = asyncio.Semaphore(10)  # একসাথে সর্বোচ্চ ১০টা

    async def fetch_one(client, record_id):
        async with semaphore:
            return await client.get(f"/api/record/{record_id}")

    async with httpx.AsyncClient() as client:
        tasks = [fetch_one(client, i) for i in range(10000)]
        results = await asyncio.gather(*tasks)
    return results
```

`asyncio.Semaphore(10)` মানে — একসাথে সর্বোচ্চ ১০টা coroutine এই block-এ ঢুকতে পারবে। বাকিরা বাইরে অপেক্ষা করবে। Rate limit মেনে চলছে, আবার ১০x concurrency-ও পাচ্ছো।

**Blocking approach:** ১০,০০০ × ১০০ms = ১০০০ সেকেন্ড (~১৭ মিনিট)
**Async approach:** ~১০০ সেকেন্ড (~১.৫ মিনিট)

---

### সংক্ষেপে: কোথায় asyncio, কোথায় না

| Situation | Asyncio দরকার? | কারণ |
|---|---|---|
| একসাথে অনেক API call | ✅ হ্যাঁ | I/O wait time বাঁচে |
| High-traffic web server | ✅ হ্যাঁ | হাজারো concurrent request |
| Database query (async driver) | ✅ হ্যাঁ | DB wait time-এ অন্য কাজ |
| একটাই API call | ❌ না | overhead বেশি, লাভ নেই |
| Image/video processing | ❌ না | CPU-bound, asyncio কাজে আসে না |
| Simple script | ❌ না | complexity বাড়বে, benefit নেই |

```
Asyncio শুধু তখনই কাজে আসে যখন তুমি অনেক কিছু "অপেক্ষা" করছো।
সেই অপেক্ষার সময়টাকে সে কাজে লাগায়।
```

---

## 🧠 Part 6: পুরো ছবিটা একসাথে

```
Multitasking
│
├── Parallelism (multiple worker, একই সময়ে সত্যিকারের parallel)
│   └── Python: multiprocessing (আলাদা CPU core, GIL নেই)
│
└── Concurrency (single worker, smart scheduling)
    ├── Thread-based (OS manage করে, preemptive)
    │   └── Python: threading (GIL আছে, I/O-bound-এ কিছুটা কাজের)
    │
    └── Coroutine-based (Python manage করে, cooperative)
        └── Python: asyncio (event loop, I/O-bound-এর জন্য best)
```

Asyncio-র চারটা মূল কাজ:

```
Event Loop:
    → run ready coroutines      (ready queue থেকে নাও, চালাও)
    → delegate I/O to OS        (I/O কাজ OS-এর হাতে দাও)
    → resume completed tasks    (OS জানালে আবার চালু করো)
    → never block itself        (নিজে কখনো block হবে না)
```

---

## ✅ Self-check

### 1. `await` কি thread block করে?

না। `await` শুধু coroutine pause করে। Thread free থাকে, event loop অন্য coroutine চালাতে পারে।

### 2. async I/O কি background-এ thread ব্যবহার করে?

না। OS-এর epoll/kqueue mechanism ব্যবহার করে। OS fd ready হলে event loop-কে notify করে — কোনো extra thread নেই।

### 3. কেন `time.sleep()` dangerous async code-এ?

`time.sleep()` পুরো **thread** ঘুম পাড়িয়ে দেয়। Event loop ওই thread-এই চলে, তাই সেও ঘুমিয়ে পড়ে। সমস্ত pending coroutine আটকে যায়। সমাধান: `await asyncio.sleep()`।

### 4. Concurrency আর Parallelism-এর পার্থক্য কী?

Concurrency: একজন worker, multiple কাজ progress করছে, অপেক্ষার সময় অন্য কাজ সারছে।
Parallelism: multiple worker, literally একই সময়ে multiple কাজ চলছে।

### 5. CPU-heavy কাজ কি async দিয়ে fast হবে?

না। Async শুধু I/O-bound কাজে কাজে আসে — যেখানে অপেক্ষা আছে। CPU-heavy কাজে multiprocessing দরকার।

---

## 🚀 Next Part Preview

👉 Next: **GIL + Threading — কেন Python-এ threading এত complicated**

- GIL (Global Interpreter Lock) কী এবং কেন Python-এ আছে
- Threading কখন কাজ করে, কখন করে না
- CPU-bound vs I/O-bound কাজে threading-এর আচরণ
- কেন GIL থাকলেও threading কিছু ক্ষেত্রে উপকারী
- এবং কখন সব ছেড়ে multiprocessing-এ যেতে হবে