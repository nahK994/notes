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

একটা coroutine তৈরি হয়ে শেষ হওয়া পর্যন্ত কয়েকটা stage পার করে। একটু সতর্কভাবে দেখো:

```
Created → Scheduled → Running → Paused → Running → Done
                                   ↑          ↓
                               (await)    (resume হলে আবার Running)
```

**Created:** `coro = task()` — object তৈরি হলো, এখনো চলেনি।

**Scheduled:** `asyncio.create_task(coro)` — event loop-এর ready queue-তে গেলো। এখনো চলেনি, কিন্তু চলার জন্য line-এ দাঁড়িয়েছে।

**Running:** Event loop তুলে নিলো, চালাচ্ছে।

**Paused:** `await` পেলো → নিজে থেকে pause, control event loop-এ। (I/O-এর জন্য অপেক্ষা করছে)

**Running (আবার):** যার জন্য অপেক্ষা করছিলো সেটা শেষ → event loop আবার তুলে নিলো → continue।

**Done:** শেষ — result বা exception নিয়ে বের হলো।

> **কেন "Resumed" না, "Running" বলছি?**
>
> "Resumed" মানে হলো paused থেকে running-এ গেলো — এটা একটা transition, আলাদা state না। একটা coroutine pause হয় `await`-এ, তারপর যখন I/O শেষ হয়, event loop সেটাকে আবার ready queue-তে রাখে এবং পরে চালায়। এই "চালানো" মানেই সে আবার **Running** state-এ — আগের মতোই। তারপর যদি আরেকটা `await` পায়, আবার Paused। এই cycle চলতে থাকে।
>
> ```python
> async def example():
>     print("A")          # Running
>     await sleep(1)      # Paused (I/O wait)
>     print("B")          # Running আবার (resume হয়েছে, কিন্তু state = Running)
>     await sleep(1)      # Paused আবার
>     print("C")          # Running আবার
>                         # Done
> ```
>
> `await` যতবার আসবে, ততবার Paused → Running cycle হবে।

---

### OS-এর সাহায্য — epoll/kqueue

#### epoll/kqueue কী?

ধরো তুমি ১০,০০০ network connection একসাথে monitor করতে চাও। তুমি জানতে চাও — কোনটায় data এসেছে?

**সহজ কিন্তু বাজে উপায়:** প্রতিটা connection একে একে check করো।

```
for conn in connections:      # ১০,০০০ বার ঘুরছো
    if conn.has_data():
        handle(conn)
```

১০,০০০ connection-এর মধ্যে মাত্র ২টায় data আছে — কিন্তু তুমি ১০,০০০ বার check করছো। এটাকে বলে **polling**, এবং এটা ভয়ঙ্কর wasteful।

**OS-এর smart উপায় — epoll (Linux) / kqueue (macOS):**

তুমি OS-কে বলো: "এই ১০,০০০ connection watch করো। যেটায় data আসবে, আমাকে জানাবে।"

OS নিজে efficiently watch করে। Data আসলে শুধু ওই connection-এর কথা জানায়। তুমি বসে বসে সব check করছো না।

```
তুমি (event loop):        OS (epoll/kqueue):
"এগুলো দেখো"  ────────→  [efficiently monitoring করছে]
                          [data আসলে...]
"কোনটায় data?" ←────────  "connection #4872 আর #9031"
শুধু এই দুটো handle করো
```

epoll/kqueue হলো OS-এর একটা **event notification system**। এটা kernel-এর ভেতরে থাকে এবং হাজার হাজার file descriptor (connection, file, socket) efficiently monitor করতে পারে।

#### Asyncio-এর সাথে সম্পর্ক

Asyncio সরাসরি epoll/kqueue ব্যবহার করে। কোনো extra thread নেই।

```
asyncio event loop flow:

1. তুমি লিখলে: await client.get("https://api.example.com")

2. asyncio → OS-কে বলে:
   "এই socket-এ data আসলে আমাকে জানাবে"
   (epoll/kqueue-এ register করলো)

3. এই coroutine pause হলো।
   Event loop অন্য ready coroutine চালাচ্ছে।

4. Network response এলো → OS kernel detect করলো

5. OS → asyncio-কে notify করলো:
   "ওই socket ready"

6. asyncio → সেই coroutine-কে ready queue-তে দিলো

7. সেই coroutine resume হলো, response পেলো
```

Diagram হিসেবে:

```
┌─────────────────────────────────────────────────┐
│              Python Process                      │
│                                                  │
│  ┌──────────────────────────────────┐            │
│  │         Event Loop               │            │
│  │                                  │            │
│  │  Ready Queue:                    │            │
│  │  [coro_A] [coro_C]               │            │
│  │                                  │            │
│  │  Waiting (I/O):                  │            │
│  │  [coro_B → socket_42]            │            │
│  │  [coro_D → socket_87]            │            │
│  └──────────┬───────────────────────┘            │
│             │ epoll_wait() call                  │
│             ↓                                    │
└─────────────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────┐
│              OS Kernel                           │
│                                                  │
│  epoll instance:                                 │
│  watching [socket_42, socket_87, ...]            │
│                                                  │
│  "socket_42 ready!" → notify event loop         │
└─────────────────────────────────────────────────┘
```

এটাই কারণ asyncio কোনো background thread ছাড়া হাজার হাজার connection handle করতে পারে — OS নিজেই efficient notification দিচ্ছে।

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

#### সমাধান: `run_in_executor` দিয়ে thread/process pool-এ পাঠাও

CPU-heavy কাজকে async code-এর ভেতর থেকে আলাদা thread বা process-এ পাঠিয়ে দেওয়া যায় — event loop block না করে।

**Thread pool দিয়ে (I/O-bound blocking কাজ, যেমন পুরনো library যেটা async support করে না):**

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor()

async def handler():
    loop = asyncio.get_event_loop()

    # process_image() blocking কিন্তু এটাকে thread-এ পাঠিয়ে দিলাম
    result = await loop.run_in_executor(executor, process_image, image_data)
    return result
```

```
Event Loop (main thread):
  handler coroutine চলছে
  → run_in_executor call
  → process_image() → thread pool-এর thread-এ গেলো
  → coroutine pause (await)
  → event loop অন্য coroutines চালাচ্ছে ✅

Thread Pool:
  Thread #1: process_image() চলছে (blocking, কিন্তু আলাদা thread-এ)
  → শেষ হলে → event loop-কে notify
  → coroutine resume
```

**Process pool দিয়ে (CPU-bound কাজ, যেমন image processing, ML inference):**

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

process_executor = ProcessPoolExecutor()

async def handler():
    loop = asyncio.get_event_loop()

    # process_image() CPU-heavy → আলাদা process-এ পাঠাও
    result = await loop.run_in_executor(process_executor, process_image, image_data)
    return result
```

Thread pool vs Process pool:

| | Thread Pool | Process Pool |
|---|---|---|
| কখন ব্যবহার করবে? | Blocking I/O (requests, পুরনো sync library) | CPU-heavy কাজ (image/video processing, ML) |
| Memory share? | হ্যাঁ (same process) | না (আলাদা process) |
| GIL এর সমস্যা? | আছে (CPU bound-এ কাজ করবে না) | নেই (আলাদা process) |
| Overhead | কম | বেশি (process তৈরিতে সময় লাগে) |

সহজ নিয়ম:
```
Blocking legacy library → ThreadPoolExecutor
CPU-heavy কাজ          → ProcessPoolExecutor
```

### ফাঁদ ৩: async ≠ parallel

```
async ≠ thread       (thread একাধিক, async single thread)
async ≠ parallel     (parallel মানে multiple CPU, async single thread)
async = event-driven (event হলে react করো)
```

Async-এ যেকোনো এক মুহূর্তে মাত্র একটা coroutine execute হচ্ছে। দুটো CPU-heavy কাজ async দিয়ে "parallel" করা যাবে না।

---

## 🧵 Part 5: Python Threading — কখন, কেন, কীভাবে

### Python Thread আর OS Thread — এরা কি আলাদা?

এটা অনেকে গুলিয়ে ফেলে। সহজ করে বলি।

**OS Thread** হলো operating system-এর তৈরি execution unit। OS নিজে schedule করে, নিজে switch করে।

**Python Thread** হলো Python-এর `threading` module দিয়ে তৈরি thread। কিন্তু এরা আসলে OS thread-ই — Python শুধু একটা convenient wrapper দিয়েছে।

```python
import threading

def worker():
    print("আমি একটা thread")

t = threading.Thread(target=worker)
t.start()  # এটা আসলে OS-কে বলছে একটা নতুন OS thread তৈরি করতে
```

```
Python Process
┌────────────────────────────────────┐
│                                    │
│  threading.Thread(target=worker)   │
│           ↓                        │
│      OS Thread তৈরি হলো           │
│      (OS-এর কাছে গেলো)            │
│                                    │
└────────────────────────────────────┘
          ↓
OS Kernel: "নতুন thread দাও"
OS: [OS Thread তৈরি করলো, schedule করবে]
```

তাহলে Python Thread = OS Thread? হ্যাঁ, exactly। Python শুধু সেটাকে তৈরি করার আর manage করার একটা Python-friendly API দিয়েছে।

---

### GIL — Python Threading-এর বড় মাথাব্যথা

Python-এ একটা বিশেষ জিনিস আছে: **GIL (Global Interpreter Lock)**।

GIL মানে হলো — যেকোনো এক মুহূর্তে একটাই Python thread Python bytecode execute করতে পারবে। তুমি ১০টা thread তৈরি করলেও CPU-তে একসাথে একটাই চলবে।

```
Thread 1: [চলছে] → GIL ছাড়লো → অপেক্ষা
Thread 2:            → GIL নিলো → [চলছে] → GIL ছাড়লো
Thread 3:                           → GIL নিলো → [চলছে]
...
```

**কেন GIL আছে?** Python-এর memory management (reference counting) thread-safe না। GIL দিয়ে race condition থেকে বাঁচানো হয়। এটা Python-এর একটা ঐতিহাসিক design decision।

**মানে কী?** CPU-bound কাজে Python threading কোনো কাজে আসে না। ৪টা thread দিয়ে ৪x speed পাবে না — পাবে মোটামুটি 1x (কারণ একসাথে একটাই চলছে)।

কিন্তু I/O-bound কাজে GIL release হয়। Thread যখন I/O-এর জন্য অপেক্ষা করছে (network, file), তখন সে GIL ছেড়ে দেয় এবং অন্য thread চলতে পারে।

---

### Thread Pool কী?

Thread তৈরি করা expensive। প্রতিটা request-এ নতুন thread তৈরি করে শেষে delete করলে অনেক overhead।

**Thread Pool** হলো আগে থেকে কিছু thread তৈরি করে রাখো, কাজ এলে সেগুলো দাও, কাজ শেষে ফেরত নাও — delete করো না।

```
Thread Pool (size=4):
┌────────────────────────────────────────┐
│  Thread 1: [idle]                      │
│  Thread 2: [idle]                      │
│  Thread 3: [idle]                      │
│  Thread 4: [idle]                      │
└────────────────────────────────────────┘

কাজ এলো → Thread 1 নিলো → কাজ করলো → আবার idle
আরেকটা কাজ এলো → Thread 2 নিলো → ...

কোনো thread idle না থাকলে → কাজ queue-তে অপেক্ষা করে
```

Python-এ `ThreadPoolExecutor`:

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=4) as executor:
    future1 = executor.submit(fetch_data, url1)  # Thread 1-এ দিলো
    future2 = executor.submit(fetch_data, url2)  # Thread 2-এ দিলো
    future3 = executor.submit(fetch_data, url3)  # Thread 3-এ দিলো

    result1 = future1.result()  # অপেক্ষা করো
    result2 = future2.result()
    result3 = future3.result()
```

---

### Threading vs Asyncio — কখন কোনটা?

এটাই সবচেয়ে গুরুত্বপূর্ণ প্রশ্ন। চলো কয়েকটা situation দিয়ে দেখি।

---

#### Situation 1: পুরনো sync library (যেমন `requests`) দিয়ে অনেক API call

`requests` library async না। এটা blocking। তুমি async করতে পারবে না।

**Threading দিয়ে:**

```python
import threading
import requests

results = []
lock = threading.Lock()

def fetch(url):
    data = requests.get(url).json()  # blocking, কিন্তু আলাদা thread-এ
    with lock:
        results.append(data)

threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
```

```
Thread 1: requests.get(url1) ─── waiting for network ───→ got data
Thread 2: requests.get(url2) ─── waiting for network ───→ got data
Thread 3: requests.get(url3) ─── waiting for network ───→ got data
         [GIL release হয় I/O wait-এ, তাই সত্যিকারের concurrent]
মোট: ~১ সেকেন্ড (সব একসাথে wait করছে)
```

**Asyncio দিয়ে (async library `httpx` ব্যবহার করে):**

```python
import asyncio
import httpx

async def fetch(client, url):
    return await client.get(url)

async def main():
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(*[fetch(client, url) for url in urls])
```

```
Coroutine 1: await client.get(url1) → paused
Coroutine 2: await client.get(url2) → paused
Coroutine 3: await client.get(url3) → paused
[সব একসাথে OS-এর epoll দিয়ে monitor হচ্ছে]
মোট: ~১ সেকেন্ড
```

এই situation-এ দুটোই কাজ করে। কিন্তু:
- Threading: memory বেশি খায়, race condition সম্ভব
- Asyncio: memory কম, race condition নেই, কিন্তু async library লাগে

**১০,০০০ request হলে?**

```
Threading:  ১০,০০০ thread = ~৮ GB RAM, OS overwhelmed ❌
Asyncio:    ১০,০০০ coroutine = ~কয়েক MB RAM           ✅
```

---

#### Situation 2: CPU-heavy কাজ — image resize

```python
from PIL import Image  # sync, CPU-heavy

def resize_image(path):
    img = Image.open(path)
    img = img.resize((800, 600))
    img.save(path + "_resized.jpg")
```

**Asyncio দিয়ে চেষ্টা করলে (ভুল!):**

```python
async def handle_images(paths):
    for path in paths:
        resize_image(path)  # ⚠️ CPU-heavy, কোনো await নেই
```

```
Event Loop:
  resize_image(path1) → CPU ব্যস্ত, ২ সেকেন্ড
  [এই ২ সেকেন্ড event loop জমে আছে]
  resize_image(path2) → আরো ২ সেকেন্ড
  [মোট: sequential-এর মতোই]
```

Asyncio এখানে কোনো কাজে আসবে না — CPU-bound কাজে কোনো I/O wait নেই, তাই pause করার সুযোগ নেই।

**Threading দিয়ে চেষ্টা করলে (আংশিক কাজ):**

```python
from concurrent.futures import ThreadPoolExecutor
import threading

with ThreadPoolExecutor(max_workers=4) as ex:
    futures = [ex.submit(resize_image, path) for path in paths]
    for f in futures:
        f.result()
```

```
Thread 1: resize_image(path1) [CPU: 2s]
Thread 2: resize_image(path2) [CPU: 2s]
Thread 3: resize_image(path3) [CPU: 2s]
Thread 4: resize_image(path4) [CPU: 2s]

GIL কারণে: একটাই Python thread একসাথে চলছে
[মোট: ~৮ সেকেন্ড, sequential-এর মতো!]  ❌
```

GIL-এর কারণে CPU-bound কাজে threading কাজ করে না।

**সঠিক সমাধান — multiprocessing:**

```python
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor(max_workers=4) as ex:
    futures = [ex.submit(resize_image, path) for path in paths]
    for f in futures:
        f.result()
```

```
Process 1 (Core 1): resize_image(path1) [CPU: 2s]
Process 2 (Core 2): resize_image(path2) [CPU: 2s]
Process 3 (Core 3): resize_image(path3) [CPU: 2s]
Process 4 (Core 4): resize_image(path4) [CPU: 2s]

[আলাদা process = আলাদা GIL = সত্যিকারের parallel]
মোট: ~২ সেকেন্ড ✅
```

---

#### Situation 3: Async code-এ blocking legacy function

তুমি FastAPI লিখছো (async), কিন্তু একটা পুরনো sync function ব্যবহার করতে হচ্ছে।

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
from fastapi import FastAPI

app = FastAPI()
executor = ThreadPoolExecutor(max_workers=10)

def old_sync_function(data):
    # পুরনো blocking code, পরিবর্তন করা যাচ্ছে না
    time.sleep(1)  # simulate DB call
    return {"result": data}

@app.get("/process")
async def handler(data: str):
    loop = asyncio.get_event_loop()
    # blocking function-কে thread pool-এ পাঠাও
    result = await loop.run_in_executor(executor, old_sync_function, data)
    return result
```

```
Event Loop (main thread):
  request এলো
  → run_in_executor → old_sync_function থ্রেডে পাঠালো
  → await করলো (coroutine paused)
  → অন্য requests handle করছে ✅

Thread Pool:
  Thread 1: old_sync_function(data) চলছে [blocking OK, আলাদা thread-এ]
  → শেষ → event loop-কে notify
  → coroutine resume → response পাঠালো
```

---

#### সারাংশ: Threading vs Asyncio — কখন কোনটা

```
┌─────────────────────────────────────────────────────────────────┐
│                    কোন situation?                               │
└─────────────────────────────────────────────────────────────────┘
          │
          ├── CPU-heavy কাজ? (calculation, image, ML)
          │         ↓
          │    multiprocessing ব্যবহার করো
          │    (আলাদা process = আলাদা CPU core = সত্যিকারের parallel)
          │
          ├── I/O-heavy কাজ, async library আছে? (httpx, asyncpg, aiofiles)
          │         ↓
          │    Asyncio ব্যবহার করো ✅
          │    (হাজারো concurrent connection, কম memory)
          │
          ├── I/O-heavy কাজ, sync library ব্যবহার করতে হবে? (requests, psycopg2)
          │         ↓
          │    ThreadPoolExecutor ব্যবহার করো
          │    (blocking call thread-এ পাঠাও)
          │    অথবা async library-তে migrate করো
          │
          └── Async app-এ CPU-heavy কাজ মেশাতে হচ্ছে?
                    ↓
               run_in_executor(ProcessPoolExecutor) ব্যবহার করো
```

| Situation | সমাধান | কারণ |
|---|---|---|
| অনেক API call (async lib) | Asyncio + `gather` | কম memory, race condition নেই |
| অনেক API call (sync lib) | ThreadPoolExecutor | I/O wait-এ GIL release হয় |
| CPU-heavy (image/ML) | ProcessPoolExecutor | GIL bypass, সত্যিকারের parallel |
| High-traffic web server | Asyncio (FastAPI/aiohttp) | হাজারো concurrent request |
| পুরনো sync code, async app-এ | `run_in_executor` | event loop block হবে না |
| সহজ script, ১-২টা কাজ | সাধারণ sync code | overhead বেশি, benefit নেই |

---

## 🌐 Part 6: Real-world — কোথায় asyncio লাগে, কোথায় না

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

## 🧠 Part 7: পুরো ছবিটা একসাথে

```
Multitasking
│
├── Parallelism (multiple worker, একই সময়ে সত্যিকারের parallel)
│   └── Python: multiprocessing (আলাদা CPU core, GIL নেই)
│
└── Concurrency (single/limited worker, smart scheduling)
    ├── Thread-based (OS manage করে, preemptive)
    │   ├── Python: threading (OS thread wrapper)
    │   ├── GIL আছে → CPU-bound-এ কাজ করে না
    │   ├── I/O wait-এ GIL release → I/O-bound-এ কিছুটা কাজের
    │   └── ThreadPoolExecutor দিয়ে pool manage করা ভালো
    │
    └── Coroutine-based (Python manage করে, cooperative)
        ├── Python: asyncio (event loop, I/O-bound-এর জন্য best)
        ├── epoll/kqueue দিয়ে OS-এর efficient notification
        └── হাজারো concurrent connection, কম memory
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

না। Async শুধু I/O-bound কাজে কাজে আসে — যেখানে অপেক্ষা আছে। CPU-heavy কাজে `ProcessPoolExecutor` দরকার।

### 6. Coroutine-এর জীবনচক্র কী?

`Created → Scheduled → Running → Paused → Running → Done`। একটা coroutine একাধিকবার Running ↔ Paused cycle করতে পারে। প্রতিটা `await`-এ Paused হয়, I/O শেষ হলে আবার Running।

### 7. Python Thread আর OS Thread কি আলাদা?

না। Python-এর `threading.Thread` আসলে OS thread-ই তৈরি করে — Python শুধু সেটার একটা convenient wrapper।

### 8. Thread Pool কেন ব্যবহার করি?

Thread তৈরি করা expensive। Pool মানে আগে থেকে thread রাখো, কাজে লাগাও, শেষে ফেরত নাও। নতুন thread তৈরির overhead বাঁচে।

### 9. run_in_executor কখন লাগে?

যখন async code-এর ভেতরে blocking/CPU-heavy কাজ করতে হয়। event loop block না করে কাজটা thread বা process pool-এ পাঠিয়ে দেওয়া যায়। I/O blocking → `ThreadPoolExecutor`, CPU-heavy → `ProcessPoolExecutor`।

---

## 🚀 Next Part Preview

👉 Next: **GIL + Threading — কেন Python-এ threading এত complicated**

- GIL (Global Interpreter Lock) কী এবং কেন Python-এ আছে
- Threading কখন কাজ করে, কখন করে না
- CPU-bound vs I/O-bound কাজে threading-এর আচরণ
- কেন GIL থাকলেও threading কিছু ক্ষেত্রে উপকারী
- এবং কখন সব ছেড়ে multiprocessing-এ যেতে হবে