# ⚡ Module 2: ASGI — The Modern Switchboard

---

## Module 1-এর Mini Challenge-এর উত্তর

আগের module শেষে প্রশ্ন ছিল:

> 2 sync worker, প্রতিটা request 3 সেকেন্ড — 10টা request একসাথে আসলে কী হবে?

**প্রশ্ন ১ — Maximum concurrent request কতটা?**

2টা worker মানে একসাথে মাত্র 2টা request চলতে পারবে। 3য় request আসলে কোনো worker free হওয়ার জন্য অপেক্ষা করবে।

**প্রশ্ন ২ — 10টা request শেষ হতে কত সময়?**

এটা batch-এ ভাবো:
- Batch 1: request 1 এবং 2 → 3 সেকেন্ড লাগে
- Batch 2: request 3 এবং 4 → আরও 3 সেকেন্ড
- Batch 3: request 5 এবং 6 → আরও 3 সেকেন্ড
- Batch 4: request 7 এবং 8 → আরও 3 সেকেন্ড
- Batch 5: request 9 এবং 10 → আরও 3 সেকেন্ড

মোট = 5 × 3 = **15 সেকেন্ড**।

**প্রশ্ন ৩ — Bottleneck কোথায়?**

Bottleneck হলো worker সংখ্যা। প্রতিটা worker সেই 3 সেকেন্ড busy থাকে — কিন্তু সেই 3 সেকেন্ডের বেশিরভাগ সময়ই সম্ভবত DB বা API-এর জন্য wait। CPU আসলে idle, কিন্তু worker occupied। এই idle time টাই waste।

Fix করার সহজ উপায় হলো worker বাড়ানো — কিন্তু প্রতিটা worker আলাদা process, আলাদা memory। তুমি 10টা worker দিলে memory 10 গুণ বেড়ে যাবে। এই approach-এর একটা physical limit আছে।

**আসল সমাধান কী?** Worker-কে I/O wait-এর সময় অন্য কাজ করতে দাও। সেটাই ASGI করে — এই module-এ সেটাই দেখব।

---

## কেন ASGI এলো?

WSGI-র সমস্যাটা আমরা দেখলাম — I/O wait-এর সময় worker আটকে থাকে। কিন্তু এই সমস্যাটা কতটা বড়?

Real-world web app-এ একটা request কী করে? User authenticate হয় — DB query। কিছু data fetch হয় — আরও DB query। হয়তো একটা external payment API call হয়। হয়তো cache check হয়। এই প্রতিটা step-এ app অপেক্ষা করছে — network response-এর জন্য, DB-এর জন্য। Actual computation হয়তো মোট সময়ের 10%।

WSGI-তে এই 90% waiting time-এ worker কিছু করতে পারে না। ASGI এই model-টাই বদলে দেয়।

---

# 🧠 2.1 ASGI কী?

**ASGI = Asynchronous Server Gateway Interface**

ASGI হলো WSGI-র successor — একই ধারণা, কিন্তু async-aware। এটাও একটা specification বা contract, যেটা define করে Python async web server এবং Python async application কীভাবে কথা বলবে।

WSGI থেকে সবচেয়ে বড় পার্থক্য হলো conceptual model:

- **WSGI বলে:** "একটা request এলো, handle করো, response দাও, শেষ।"
- **ASGI বলে:** "একটা connection এলো। এই connection-এ multiple message আসতে পারে, তুমি multiple message পাঠাতে পারো, connection যতক্ষণ খোলা থাকে।"

এই পার্থক্যটা ছোট মনে হলেও এটাই WebSocket, streaming, long-polling সব কিছু সম্ভব করে।

---

# ⚔️ 2.2 WSGI বনাম ASGI — পাশাপাশি তুলনা

| বিষয় | WSGI | ASGI |
|------|------|------|
| Execution model | Synchronous | Asynchronous |
| Request model | One-shot (req → res) | Ongoing conversation |
| WebSocket | ❌ সম্ভব না | ✔️ built-in |
| Concurrency | Process বা thread বাড়াও | Event loop, একই worker-এ |
| Streaming | Limited, sync | Native, async |
| Idle time | Worker blocked | Worker অন্য কাজ করে |

---

# ⚙️ 2.3 ASGI App-এর Structure

WSGI app ছিল একটা সাধারণ function — দুটো argument নাও, response return করো। ASGI app একটু আলাদা:

```python
async def app(scope, receive, send):
    ...
```

তিনটা argument। প্রতিটার আলাদা দায়িত্ব আছে।

---

### `scope` — Connection-এর পরিচয়পত্র

`scope` হলো একটা dictionary যেটা এই connection সম্পর্কে সব static তথ্য রাখে। Connection তৈরি হওয়ার সময় এটা একবার set হয়, পরে আর বদলায় না।

```python
# HTTP request-এর জন্য scope দেখতে এরকম
{
    "type": "http",
    "method": "GET",
    "path": "/users",
    "headers": [...],
    "query_string": b"id=10"
}

# WebSocket connection-এর জন্য type হবে "websocket"
{
    "type": "websocket",
    "path": "/ws/chat"
}
```

`type` field দিয়ে app বুঝতে পারে এটা HTTP request নাকি WebSocket connection নাকি অন্য কিছু।

---

### `receive` — Client কী বলছে শোনো

`receive` একটা async callable। এটাকে `await` করলে client-এর পাঠানো পরবর্তী message পাওয়া যায়।

HTTP-তে এটা দিয়ে request body পড়া হয়। WebSocket-এ এটা দিয়ে client-এর পাঠানো প্রতিটা message পড়া হয়।

```python
message = await receive()
# message = {"type": "http.request", "body": b"...", "more_body": False}
```

`await` করা হচ্ছে মানে — "message না আসা পর্যন্ত এখানে wait করো, কিন্তু event loop কে block করো না।" এই সময়ে অন্য request চলতে পারে।

---

### `send` — Client-কে কিছু বলো

`send`-ও একটা async callable। এটা দিয়ে client-কে data পাঠানো হয়। HTTP-তে প্রথমে response header পাঠাতে হয়, তারপর body।

```python
await send({
    "type": "http.response.start",
    "status": 200,
    "headers": [(b"content-type", b"text/plain")],
})

await send({
    "type": "http.response.body",
    "body": b"Hello, ASGI",
})
```

দুটো আলাদা `send` call করতে হচ্ছে — এটা WSGI থেকে আলাদা। এই design-টার কারণ হলো response আংশিকভাবেও পাঠানো যায়। Header পাঠিয়ে body পরে পাঠানো যায়, বা body chunks-এ পাঠানো যায়।

---

### একটা Complete Minimal ASGI App

```python
async def app(scope, receive, send):
    if scope["type"] == "http":
        # request body পড়ো (এখানে দরকার না হলেও)
        await receive()

        # response header পাঠাও
        await send({
            "type": "http.response.start",
            "status": 200,
            "headers": [(b"content-type", b"text/plain")],
        })

        # response body পাঠাও
        await send({
            "type": "http.response.body",
            "body": b"Hello, ASGI",
        })
```

এটাই একটা পূর্ণাঙ্গ ASGI app। FastAPI বা Django Channels ভেতরে ভেতরে এটাই করছে, শুধু অনেক বেশি feature-সহ।

---

# ⚡ 2.4 Event Loop — ASGI-র প্রাণ

ASGI কাজ করে event loop-এর উপর। Module 0-তে event loop-এর ধারণা দেখেছিলাম। এখন দেখো এটা ASGI-তে কীভাবে কাজ করে।

ধরো 3টা request একসাথে এলো, প্রতিটাতে একটা DB query আছে যেটা 100ms নেয়:

```
t=0ms   → Request A শুরু, DB query দিল, await করল → pause
t=0ms   → Request B শুরু, DB query দিল, await করল → pause
t=0ms   → Request C শুরু, DB query দিল, await করল → pause
t=100ms → Request A-র DB response এলো, resume হলো, response পাঠাল
t=100ms → Request B-র DB response এলো, resume হলো, response পাঠাল
t=100ms → Request C-র DB response এলো, resume হলো, response পাঠাল
```

মোট সময় ≈ 100ms। WSGI-তে একই কাজ sequential হলে লাগত 300ms।

Event loop এই পুরো choreography manage করে। কোন task কখন resume হবে, কোনটা এখন run করবে — সব সে ঠিক করে। তুমি শুধু `await` লিখলেই সে বাকিটা বোঝে।

---

# 🧨 2.5 Blocking — Async-এর সবচেয়ে বড় শত্রু

ASGI-র সবচেয়ে important rule হলো — **async function-এর ভেতরে কোনো blocking call রাখা যাবে না।**

```python
# ❌ এটা করলে পুরো event loop আটকে যাবে
async def handler(scope, receive, send):
    time.sleep(5)   # blocking! event loop freeze হয়ে গেল
    ...
```

`time.sleep(5)` call করলে Python thread-ই ঘুমিয়ে পড়ে — event loop সহ। এই 5 সেকেন্ড অন্য কোনো request চলতে পারবে না। তুমি async লিখেছ, কিন্তু সুবিধা শূন্য।

```python
# ✔️ সঠিক — event loop অন্য কাজ করতে পারে
async def handler(scope, receive, send):
    await asyncio.sleep(5)   # pause, কিন্তু event loop free
    ...
```

`await asyncio.sleep(5)` বললে event loop বোঝে — "এই task 5 সেকেন্ড দরকার নেই, অন্য কাজ করো।" 5 সেকেন্ড পর এটাকে resume করো।

**এটাই async programming-এর সবচেয়ে বড় gotcha।** Async function-এ সব I/O অবশ্যই async library দিয়ে করতে হবে। Sync database driver (যেমন `psycopg2` সরাসরি), sync HTTP library (যেমন `requests`) — এগুলো async context-এ ব্যবহার করলে পুরো app blocking হয়ে যাবে। `psycopg3` বা `asyncpg`, `httpx` বা `aiohttp` — এগুলো ব্যবহার করতে হবে।

---

# 🔄 2.6 Concurrency Model — একটাই Worker, অনেক Request

এটা অনেকের কাছে অবিশ্বাস্য লাগে — একটাই worker process, একটাই thread, তবুও হাজারো concurrent request handle করা সম্ভব।

কীভাবে? কারণ WSGI-তে worker "occupied" মানে ছিল worker কিছু একটা করছে বা wait করছে — দুটোতেই worker কাউকে serve করতে পারত না। ASGI-তে worker "wait করছে" মানে সে আসলে free — event loop অন্য request নিতে পারে।

```
WSGI (2 workers, 3 requests):
Worker 1: [==Request A (wait)========] [Request C শুরু]
Worker 2: [==Request B (wait)========]
Request C: ⏳ দাঁড়িয়ে আছে...

ASGI (1 worker, 3 requests):
Worker:   [A শুরু][B শুরু][C শুরু][A wait][B wait][C wait][A resume][B resume][C resume]
সবই প্রায় একসাথে শেষ হয়
```

ASGI-তে worker শুধু তখনই truly busy যখন সে compute করছে। I/O wait মানে worker free।

এই কারণে ASGI server-এ worker count WSGI-র মতো "concurrent user সংখ্যা" দিয়ে ঠিক করতে হয় না। বরং CPU core সংখ্যা দিয়ে ঠিক করো — প্রতি core-এ একটা worker।

---

# 🌐 2.7 WebSocket — ASGI-র Superpower

WSGI দিয়ে WebSocket করা সম্ভব না কারণ WSGI-র model হলো "request এলো, response দাও, connection বন্ধ।" WebSocket-এ connection বন্ধ হয় না — client এবং server দুজনেই যেকোনো সময় message পাঠাতে পারে।

ASGI-তে এটা naturally fits। `scope["type"] == "websocket"` হলে `receive()` দিয়ে client-এর message পড়তে থাকো, `send()` দিয়ে যখন খুশি message পাঠাও।

```python
async def app(scope, receive, send):
    if scope["type"] == "websocket":
        # connection accept করো
        await send({"type": "websocket.accept"})

        # loop করে message receive করতে থাকো
        while True:
            message = await receive()

            if message["type"] == "websocket.disconnect":
                break

            # client যা পাঠাল সেটাই ফেরত পাঠাও (echo)
            await send({
                "type": "websocket.send",
                "text": message["text"]
            })
```

এই loop-টা চলতে থাকে connection বন্ধ না হওয়া পর্যন্ত। কিন্তু যেহেতু সব `await`-সহ, event loop এই connection-টাকে background-এ রেখে অন্য connection-ও handle করতে পারে।

---

# 🧬 2.8 Streaming Response

ASGI-তে response আস্তে আস্তে পাঠানো যায় — পুরো response তৈরি হওয়ার আগেই client data পেতে শুরু করে।

```python
await send({
    "type": "http.response.start",
    "status": 200,
    "headers": [(b"content-type", b"text/plain")],
})

# data আসতে আসতে পাঠাও
for chunk in large_data_generator():
    await send({
        "type": "http.response.body",
        "body": chunk,
        "more_body": True,   # এখনও শেষ হয়নি
    })

# শেষ chunk
await send({
    "type": "http.response.body",
    "body": b"",
    "more_body": False,   # এবার শেষ
})
```

এটা কোথায় কাজে লাগে? ধরো একটা large CSV file generate হচ্ছে বা AI model token by token response দিচ্ছে — পুরোটা memory-তে জমিয়ে তারপর পাঠানোর দরকার নেই। যত generate হচ্ছে তত পাঠাও।

---

# ⚙️ 2.9 ASGI Servers — কে কীভাবে কাজ করে

WSGI-তে Gunicorn ছিল defacto standard। ASGI-তে কয়েকটা option আছে:

**Uvicorn:** সবচেয়ে popular ASGI server। FastAPI-র default। `asyncio`-based, অনেক fast। Development এবং production দুটোতেই ব্যবহার হয়।

```bash
uvicorn myapp:app --host 0.0.0.0 --port 8000
```

**Daphne:** Django Channels-এর জন্য তৈরি। WebSocket-heavy app-এ বেশি দেখা যায়।

**Gunicorn + UvicornWorker:** Production-এ একটা common pattern হলো Gunicorn-কে process manager হিসেবে রেখে, প্রতিটা worker-কে Uvicorn দিয়ে চালানো।

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 4 myapp:app
```

এখানে Gunicorn করছে process management — worker crash করলে restart দেওয়া, graceful shutdown। আর UvicornWorker করছে actual async request handling। দুজনের কাজ আলাদা।

---

# ⚠️ 2.10 Common Misconceptions

**"ASGI মানেই faster"** — না, সবসময় না।

ASGI তখনই faster যখন app I/O-bound। প্রতিটা request DB query বা external API call করছে এবং অপেক্ষা করছে — এই ক্ষেত্রে ASGI অনেক বেশি efficient।

কিন্তু যদি app CPU-bound হয় — image processing, heavy computation — তাহলে ASGI কোনো সুবিধা দেয় না। এই ক্ষেত্রে multiprocessing দরকার।

**"Async = Parallel"** — না।

ASGI-তে একটাই event loop, একটাই thread। সব task সেই একটা thread-এই চলছে — একটার পর একটা interleave হয়ে। এটা concurrency, parallelism না। সত্যিকারের parallelism পেতে multiple worker process লাগবে।

**"Async library use করলেই সব ঠিক"** — না।

Async library ব্যবহার করলেও code-এ কোথাও CPU-heavy synchronous operation থাকলে event loop আটকাবে। `json.loads()` বা complex string processing-ও যদি খুব বড় data-তে হয়, সেটা event loop-কে block করতে পারে।

---

# 🧩 2.11 কখন ASGI ব্যবহার করবে, কখন না

**ASGI ভালো কাজ করে যখন:**
- অনেক concurrent user — হাজারো connection একসাথে
- প্রতিটা request I/O করছে (DB, external API, cache)
- WebSocket বা real-time feature দরকার
- Streaming response দরকার

**ASGI এর সুবিধা কম যখন:**
- CPU-heavy কাজ (এখানে multiprocessing দরকার)
- Legacy sync code আছে যেটা async করা সম্ভব না
- App এতটাই simple যে WSGI-র simplicity বেশি কাজের

---

# 🎯 Final Mental Model

WSGI এবং ASGI-র পার্থক্যটা এক লাইনে:

**WSGI:** Worker কাজ করুক বা wait করুক — সে occupied। Scaling মানে worker বাড়াও।

**ASGI:** Worker শুধু তখন occupied যখন সে সত্যিকার অর্থে কাজ করছে। Wait মানে free। একটা worker-এই অনেক request চলে।

```
WSGI:
request → worker নেয় → [কাজ + wait] → response → তারপর পরের request

ASGI:
request → event loop নেয় → কাজ করে → await-এ pause → অন্য request → resume → response
```

Production-এ optimum হলো দুটো মিলিয়ে — multiple Uvicorn workers (parallelism), প্রতিটা worker-এ async event loop (concurrency)।

---

# ⚔️ Mini Challenge

ধরো তোমার setup:
- 1 Uvicorn worker
- Async code, সব I/O properly awaited
- 5টা request একসাথে এলো
- প্রতিটা request করে: 1 সেকেন্ড DB query await + 1 সেকেন্ড external API await

**প্রশ্ন ১:** সব 5টা request শেষ হতে মোট কত সময় লাগবে?

**প্রশ্ন ২:** যদি একই code WSGI sync worker-এ চলত, তাহলে সময় কত হতো?

**প্রশ্ন ৩:** এই performance gain কোথা থেকে এলো — ঠিক কোন জায়গায়?

চিন্তা করো। Event loop-এর timeline মাথায় আঁকো।