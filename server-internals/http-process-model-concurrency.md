# 🪜 Module 0: Ground Zero — HTTP, Process Model, Concurrency

---

## কেন এই module টা পড়ছি?

Backend development মানে শুধু API লেখা না। যখন তুমি একটা FastAPI বা Django app বানাও, তখন আসলে অনেক কিছু একসাথে চলছে — OS processes, threads, event loops, network connections। এগুলো না বুঝলে performance problem debug করা, concurrency handle করা, বা scalable system বানানো সম্ভব না।

এই module-এ আমরা একদম নিচ থেকে শুরু করব — HTTP request টা আসলে কীভাবে আসে, সেটা কে receive করে, কীভাবে process হয়, এবং একসাথে অনেক request এলে কী হয়। এই mental model টা পরিষ্কার হলে পরের সব কিছু অনেক সহজ লাগবে।

---

# 🌐 0.1 HTTP — What Actually Travels on the Wire

সবকিছুর শুরু হয় একটা HTTP request দিয়ে। Browser বা client থেকে যখন কোনো request আসে, সেটা আসলে একটা **plain text message** যা network-এ পাঠানো হয়।

```http
GET /users?id=10 HTTP/1.1
Host: example.com
User-Agent: curl/7.68.0
```

এটা কোনো magic না, শুধুই structured text। এই text-এর মধ্যে থাকে:

- **Method** → কী করতে চাই (GET = পড়া, POST = পাঠানো, PUT = আপডেট, DELETE = মুছা)
- **Path** → কোথায় যেতে চাই (`/users`)
- **Headers** → extra metadata — authentication token, content type, ইত্যাদি
- **Body** → POST বা PUT-এ actual data থাকে

Server response-ও একই ধরনের structure:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"name": "Shomi"}
```

---

### HTTP-এর দুটো গুরুত্বপূর্ণ চরিত্র

**Stateless protocol:** HTTP নিজে কোনো state মনে রাখে না। প্রতিটা request server-এর কাছে নতুন। আগের request কী ছিল, server সেটা জানে না। এজন্যই আমরা cookies, sessions, বা JWT token use করি — state টা client বা external store-এ রাখতে হয়।

**Connection reuse:** HTTP/1.1-এ একটা TCP connection বারবার reuse হতে পারে (keep-alive)। HTTP/2-এ একটু উপরে গিয়ে একই connection-এ একসাথে multiple request যেতে পারে (multiplexing)। এটা পরে async বোঝার সময় কাজে আসবে।

---

# 🔌 0.2 Socket — The Real Entry Gate

HTTP request টা text হলেও এটা network-এ পাঠাতে হলে একটা low-level channel দরকার। সেটা হলো **TCP socket**।

```
Client → TCP connection → Server
              ↓
         raw bytes পাঠায়
              ↓
         server bytes পড়ে → HTTP parse করে
```

Server আসলে প্রথমে raw bytes receive করে, তারপর সেই bytes থেকে HTTP structure বের করে। তোমার FastAPI বা Flask এই parsing টা তোমার হয়ে করে দেয়, কিন্তু নিচে এটাই হচ্ছে।

এই layer টা বোঝা দরকার কারণ — socket কতটা দ্রুত data read করতে পারছে, সেটা পুরো request handling-এর speed-কে affect করে।

---

# ⚙️ 0.3 Process — The Isolation Unit

Request আসলে server-এ কীভাবে execute হয়? এখানে আসে **process** এর ধারণা।

Process হলো OS-এর একটা execution unit। প্রতিটা process-এর থাকে:

- নিজের আলাদা memory space
- নিজের file descriptors (open files, sockets)
- OS-level isolation

ধরো তুমি `gunicorn` দিয়ে app run করছ — এটা multiple worker process তৈরি করে। কেন?

**Crash isolation:** একটা worker crash করলে অন্যরা চলতে থাকে।  
**Parallelism:** Multiple CPU core থাকলে multiple process একসাথে কাজ করতে পারে, কারণ প্রতিটা process আলাদা core-এ run করতে পারে।

কিন্তু process-এর cost আছে — প্রতিটা process আলাদা memory নেয়, এবং OS-কে process switch করতে হলে (context switch) সেটা সময় নেয়। তাই process সংখ্যা বাড়িয়ে দিলেই performance বাড়ে না।

---

# 🧵 0.4 Thread — Lightweight Worker

Process-এর একটা সমস্যা হলো এটা heavy। একটা নতুন process তৈরি করা মানে আলাদা memory allocate করা। এই সমস্যার সমাধান হলো **thread**।

Thread হলো একটা process-এর ভেতরে আলাদা execution path। এক process-এর সব thread:

- একই memory share করে
- একই file descriptors share করে

ফলে একটা request আসলে নতুন process না তুলে নতুন thread তৈরি করে handle করা যায় — অনেক কম cost-এ।

**কিন্তু thread-এর সমস্যাও আছে:**

যেহেতু threads memory share করে, দুটো thread একই data একসাথে modify করতে গেলে **race condition** হতে পারে। কোন thread আগে লিখবে, কোনটা পরে — এটা predictable না। এজন্য synchronization (locks, semaphores) দরকার পড়ে, যেটা আবার নিজেই complexity আনে।

---

### Python-এর বিশেষ সমস্যা: GIL

Python-এ threads use করার সময় একটা extra twist আছে — **GIL (Global Interpreter Lock)**।

Python interpreter-এ একটা lock আছে যেটা নিশ্চিত করে যে এক মুহূর্তে শুধুমাত্র একটাই thread Python bytecode execute করতে পারবে।

এর consequence:

- **CPU-bound কাজে** (heavy computation) multiple thread ব্যবহার করলেও লাভ নেই — একটা thread কাজ করার সময় বাকিরা wait করে। Parallelism পাবে না।
- **I/O-bound কাজে** (database call, API call, disk read) thread useful — কারণ I/O wait-এর সময় GIL release হয়, অন্য thread কাজ করতে পারে।

এই কারণেই Python-এ CPU-heavy কাজের জন্য multiprocessing, আর I/O-heavy কাজের জন্য threading বা async ব্যবহার করা হয়।

---

# ⚡ 0.5 Event Loop — The Async Engine

Process এবং thread দুটোই concurrency-র একটা সমাধান। কিন্তু web server-এ সবচেয়ে বড় কাজটা হলো I/O wait — database থেকে data আসার অপেক্ষা, external API-এর response-এর অপেক্ষা। এই waiting-এর সময়টা thread দিয়েও handle করা যায়, কিন্তু অনেক thread তৈরি করা মানে অনেক memory।

এর চেয়ে efficient সমাধান হলো **event loop** — async programming-এর মূল engine।

Event loop একটাই thread-এ অনেকগুলো task manage করে এভাবে:

```python
while True:
    check_ready_tasks()   # কোন task এখন কাজ করার জন্য ready?
    run_task()            # সেটা একটু চালাও
```

মূল idea হলো: কোনো task যদি I/O-এর জন্য wait করছে, সে নিজেই বলে "আমি এখন wait করছি, তুমি অন্য কাজ করো।" Event loop তখন অন্য কোনো ready task চালায়। I/O complete হলে আবার সেই task-কে resume করে।

এটা একজন efficient receptionist-এর মতো — সে একসাথে অনেক কল handle করে, কিন্তু কাউকে hold-এ রাখার সময় অন্যদের সাথে কথা বলে।

---

# 🧨 0.6 Blocking vs Non-blocking — The Most Critical Concept

Async programming বোঝার সবচেয়ে গুরুত্বপূর্ণ concept হলো blocking vs non-blocking।

**Blocking code:**

```python
time.sleep(5)
```

এই line execute হলে পুরো thread 5 সেকেন্ডের জন্য থেমে যায়। CPU idle বসে থাকে। আর কোনো কাজ এগোয় না।

**Non-blocking code:**

```python
await asyncio.sleep(5)
```

এখানে `await` মানে — "আমি এখন wait করছি, কিন্তু event loop, তুমি এই সময়ে অন্য task চালাও।" 5 সেকেন্ড পর যখন ready হবে, তখন আমাকে resume করো।

> **Golden rule:** Async code-এর ভেতরে একটাও blocking call থাকলে পুরো event loop আটকে যায়। `time.sleep()`, বা কোনো sync database library — এগুলো async context-এ ব্যবহার করলে concurrency-র কোনো সুবিধাই পাবে না।

---

# ⚔️ 0.7 Concurrency vs Parallelism

এই দুটো শব্দ প্রায়ই একসাথে ব্যবহার হয়, কিন্তু এরা আলাদা জিনিস।

**Concurrency** মানে multiple task "progress" করছে — কিন্তু exactly একই সময়ে নয়। একটা task কিছুটা এগোয়, তারপর pause করে, অন্যটা এগোয়। মনে হয় সব একসাথে চলছে।

→ Async এবং threads এই model follow করে।

**Parallelism** মানে multiple task সত্যিকার অর্থে একই সময়ে চলছে — আলাদা CPU core-এ।

→ Multiple processes এই model follow করে।

**Analogy:**  
Concurrency = একজন chef যে একসাথে তিনটা dish manage করছে — কিন্তু হাত একটাই, একবারে একটায় কাজ করছে।  
Parallelism = তিনজন chef, প্রত্যেকে আলাদা dish-এ একসাথে কাজ করছে।

---

# 🧬 0.8 CPU-bound vs I/O-bound

কোন সমস্যায় কোন solution use করবে, সেটা নির্ভর করে কাজটা কী ধরনের।

**CPU-bound:** কাজটা computation-heavy — image processing, video encoding, hashing, machine learning inference।  
→ এখানে waiting নেই, শুধু calculation। Solution: **multiprocessing** (GIL bypass করে আলাদা core use করো)।

**I/O-bound:** কাজটা waiting-heavy — database query, HTTP API call, disk read/write।  
→ এখানে CPU idle থাকে, শুধু response-এর জন্য অপেক্ষা। Solution: **async বা threading** (waiting time-এ অন্য কাজ করো)।

> **Rule of thumb:** বেশি wait করতে হচ্ছে? → Async। বেশি calculate করতে হচ্ছে? → Multiprocessing।

---

# 🧱 0.9 Request Handling Models

এতক্ষণ যা পড়লাম সেটা একসাথে রাখলে বোঝা যাবে request কীভাবে handle হয়। মূলত তিনটা model আছে:

**Model 1: Synchronous (Blocking)**

```
request আসে → handle করো → response পাঠাও → তারপর পরের request নাও
```

একটা request শেষ না হওয়া পর্যন্ত পরেরটা শুরু হয় না। সহজ কিন্তু slow।

**Model 2: Threaded**

```
request 1 → thread 1 handle করছে
request 2 → thread 2 handle করছে (একসাথে)
```

Multiple request parallel-ish ভাবে handle হয়। কিন্তু GIL-এর কারণে Python-এ এটা সত্যিকারের parallelism না — I/O-bound কাজে কাজ করে, CPU-bound-এ না।

**Model 3: Async (Non-blocking)**

```
request 1 আসে → শুরু করে → DB-এর জন্য await করে → pause
request 2 আসে → এখন run করে
request 1 → DB response এলে resume হয়
```

একটাই thread-এ অনেক request interleave করে handle হয়। I/O-heavy workload-এ সবচেয়ে efficient।

---

# 🧠 0.10 Throughput vs Latency

Performance নিয়ে কথা বলতে গেলে দুটো metric বোঝা দরকার:

**Latency:** একটা single request শেষ হতে কতটা সময় লাগছে।  
**Throughput:** এক সেকেন্ডে মোট কতটা request handle করতে পারছে।

Async model latency কমায় না — একটা request-এর কাজ কমে না। কিন্তু waiting time-এ অন্য request চলে বলে **throughput অনেক বেড়ে যায়**। মানে server অনেক বেশি user একসাথে serve করতে পারে।

---

# 🧨 0.11 Backpressure — যখন System Overwhelm হয়

Async বা threaded model-এ throughput বাড়লেও একটা limit আছে। যদি incoming requests-এর হার processing capacity-র চেয়ে বেশি হয়ে যায়, system overloaded হয়।

লক্ষণ:
- Response ধীরে ধীরে slow হয়
- Timeout শুরু হয়
- শেষ পর্যন্ত crash

সমাধান:
- **Queue** — excess request queue-এ রাখো, ধীরে ধীরে process করো
- **Rate limiting** — প্রতি user/IP-কে নির্দিষ্ট সংখ্যক request-এ limit করো
- **Load balancing** — multiple server-এ request ভাগ করো

এখনই deep dive না করলেও concept টা মাথায় রাখো — scalable system design-এ এটা বারবার আসবে।

---

# 🧩 0.12 Common Misconceptions

এই concepts নিয়ে কিছু popular ভুল ধারণা পরিষ্কার করে নেওয়া দরকার:

**"Async মানেই fast"** — না। Async fast কারণ এটা I/O wait-এর সময় অন্য কাজ করে। কিন্তু যদি কাজটাই CPU-heavy হয়, async কোনো সুবিধা দেবে না।

**"Threads always run in parallel"** — Python-এ না। GIL-এর কারণে CPU-bound কাজে threads parallel চলে না।

**"More workers = better performance"** — না। প্রতিটা worker memory নেয়। বেশি worker মানে বেশি context switching, যেটা উল্টো overhead তৈরি করে।

---

# 🎯 Final Mental Model

একটা request-এর lifecycle এভাবে ভাবো:

```
socket → server → worker → execution model → app code → response
```

Performance কোথায় depend করে:
- **Execution model** কী (sync / thread / async)?
- Code কি **blocking** নাকি **non-blocking**?
- কাজটা **CPU-bound** নাকি **I/O-bound**?

---

# 🧠 Self-check

Module শেষে নিজেকে এই তিনটা প্রশ্ন করো যেকোনো code দেখলে:

1. এই code কি block করছে?
2. আমি wait করছি (I/O) নাকি compute করছি (CPU)?
3. Concurrency দরকার (interleaving) নাকি parallelism (true parallel)?

এই তিনটা প্রশ্নের উত্তর জানলে backend-এর বেশিরভাগ performance সিদ্ধান্ত নিজেই নিতে পারবে।

---

# ⚔️ Mini Challenge

ধরো তোমার কাছে একটা server আছে:
- 1 worker process
- 1 thread
- async code দিয়ে লেখা

**Question:** এই single worker কি একসাথে multiple request handle করতে পারবে?

ভাবো:
- কখন পারবে — কী ধরনের request হলে?
- কখন পারবে না — কী করলে এটা fail করবে?

এই প্রশ্নের উত্তরটা নিজে বের করার চেষ্টা করো। পরের module-এ এটাই বিস্তারিত দেখব।