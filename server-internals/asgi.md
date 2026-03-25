# ASGI: যে দারোয়ান একসাথে হাজারজনের সাথে কথা বলতে পারে

> *"WSGI জানত একটা কাজ শেষ করে পরের কাজ ধরতে। ASGI জানে একটা কাজ অপেক্ষায় থাকলে সেই ফাঁকে অন্য কাজ সেরে ফেলতে।"*

---

## আগের পর্বের উত্তর — সংখ্যাগুলো মিলিয়ে নাও

আগের পর্বে একটা scenario দিয়েছিলাম:

> **2টা sync worker, প্রতিটা request 3 সেকেন্ড — 10টা request একসাথে এলে কী হবে?**

চলো হিসেব করি।

**একসাথে সর্বোচ্চ কতটা request?**

Worker সংখ্যা = 2। মানে যেকোনো মুহূর্তে সর্বোচ্চ 2টা request চলতে পারে। তৃতীয় request আসলে সে queue-তে দাঁড়াবে, যতক্ষণ না কোনো worker free হয়।

**10টা শেষ হতে কত সময়?**

2 জন করে কাজ করছে, তাই batch-এ ভাবো:

```text
Batch 1 → Request 1 ও 2  → 3s
Batch 2 → Request 3 ও 4  → 3s
Batch 3 → Request 5 ও 6  → 3s
Batch 4 → Request 7 ও 8  → 3s
Batch 5 → Request 9 ও 10 → 3s

মোট: 5 × 3 = 15 সেকেন্ড
```

**Bottleneck কোথায়?**

এখানে একটু গভীরে ভাবো। প্রতিটা request 3 সেকেন্ড নিচ্ছে — কিন্তু সেই 3 সেকেন্ড কি সত্যিই কিছু করছে? বেশিরভাগ সময় worker DB-এর জবাবের জন্য বা external API-এর জন্য বসে আছে। CPU idle, কিন্তু worker occupied।

Worker বাড়ালে কি সমাধান? হ্যাঁ, কিছুটা — কিন্তু প্রতিটা worker আলাদা OS process, আলাদা memory। 100 worker দিলে memory 100 গুণ বাড়বে। এই approach-এর একটা physical ceiling আছে।

**আসল সমাধান হলো:** Worker-কে idle বসে না থেকে সেই waiting সময়ে অন্য কাজ করতে দাও। এটাই ASGI করে।

---

## সমস্যাটা আরেকটু পরিষ্কার করি

Real-world web app-এ একটা request কী কী করে? ধরো একটা e-commerce order API:

```text
1. User authenticate করো    → DB query    → 20ms wait
2. Product stock check করো  → DB query    → 15ms wait
3. Payment gateway call করো → External API → 300ms wait
4. Order save করো           → DB query    → 25ms wait
5. Email notification পাঠাও → SMTP server  → 50ms wait
```

মোট সময় ধরো 410ms। কিন্তু এই 410ms-এর মধ্যে CPU আসলে কতক্ষণ কাজ করছে? হয়তো 10ms। বাকি 400ms সে অপেক্ষা করছে — network response-এর জন্য, DB-এর জন্য।

WSGI worker-এ এই 400ms সম্পূর্ণ নষ্ট। ASGI event loop-এ এই 400ms-এ অন্য কয়েকশো request এগিয়ে যেতে পারে।

---

## ASGI কী এবং কেন এলো

**ASGI = Asynchronous Server Gateway Interface**

WSGI যেমন একটা contract ছিল — "server এবং app এই নিয়মে কথা বলবে" — ASGI-ও তেমনই একটা contract, কিন্তু async-aware।

WSGI-র model ছিল খুব simple: request এলো → handle করো → response দাও → connection বন্ধ। এই model-এ connection মানেই একটা transaction — শুরু আছে, শেষ আছে।

ASGI-র model আলাদা: connection এলো → এই connection-এ multiple message আসতে পারে → multiple message পাঠানো যায় → connection যতক্ষণ খোলা ততক্ষণ চলতে পারে।

এই conceptual shift-টা ছোট মনে হলেও এটাই WebSocket, Server-Sent Events, HTTP/2 streaming — সব কিছু সম্ভব করে।

ASGI 2018 সালে Django Channels project থেকে উঠে এসেছে। Django team বুঝেছিল WSGI দিয়ে real-time feature করা আর সম্ভব না। তাই তারা একটা নতুন specification তৈরি করল, যেটা এখন Python-এর async web ecosystem-এর ভিত্তি।

---

## WSGI বনাম ASGI — একটা সরাসরি তুলনা

পার্থক্যটা concept-এ বোঝার আগে পাশাপাশি দেখো:

| বিষয় | WSGI | ASGI |
|---|---|---|
| Execution model | Synchronous — একটা শেষ হলে পরেরটা | Asynchronous — await-এ pause করে অন্যটা শুরু |
| Connection model | One-shot (req → res → বন্ধ) | Ongoing (চলতে থাকে যতক্ষণ দরকার) |
| Idle time | Worker blocked, কিছুই করে না | Event loop অন্য request নেয় |
| WebSocket | ❌ সম্ভব না | ✔️ নেটিভভাবে support করে |
| Streaming | Limited | নেটিভ, async |
| Scaling | Worker বাড়াও (process/thread) | একটা worker-এই হাজারো concurrent |

---

## ASGI App-এর গঠন — তিনটা argument কেন?

WSGI app ছিল:
```python
def app(environ, start_response):
    ...
```

ASGI app হলো:
```python
async def app(scope, receive, send):
    ...
```

তিনটা argument — প্রতিটার নিজস্ব ভূমিকা আছে। একটু মনোযোগ দিয়ে বোঝো, কারণ এই তিনটা বোঝলে ASGI-র পুরো model মাথায় ঢুকে যাবে।

---

### `scope` — Connection কে? কোথায়?

`scope` হলো connection-এর identity card। Connection establish হওয়ার মুহূর্তে একবার তৈরি হয়, পরে আর বদলায় না।

```python
# HTTP request-এর scope
{
    "type": "http",          # এটা HTTP নাকি WebSocket নাকি অন্য কিছু
    "method": "GET",         # HTTP method
    "path": "/api/users",    # URL path
    "headers": [...],        # সব headers
    "query_string": b"page=2"
}

# WebSocket-এর scope — শুধু type বদলায়
{
    "type": "websocket",
    "path": "/ws/chat"
}
```

`type` field দিয়ে app জানতে পারে এই connection কী ধরনের। একটাই app HTTP আর WebSocket দুটোই handle করতে পারে — শুধু `scope["type"]` চেক করে আলাদা logic চালাতে হবে।

---

### `receive` — Client কী বলছে?

`receive` একটা async function। এটাকে `await` করলে client-এর পাঠানো পরবর্তী message পাওয়া যায়।

```python
message = await receive()
# HTTP-তে: {"type": "http.request", "body": b"...", "more_body": False}
# WebSocket-এ: {"type": "websocket.receive", "text": "Hello!"}
```

এখানে `await` করার মানেটা বোঝো: "client message না পাঠানো পর্যন্ত এখানে থেমে থাকো — কিন্তু event loop-কে block করো না।"

এই অপেক্ষার সময়ে event loop অন্য request handle করতে পারে। Worker idle না — সে অন্য কাজ করছে।

---

### `send` — Client-কে কিছু বলো

`send`-ও async। এটা দিয়ে client-কে data পাঠানো হয়।

HTTP-তে দুটো আলাদা step আছে — আগে response-এর metadata (status code, headers), তারপর actual content:

```python
# Step 1: Response শুরু করো
await send({
    "type": "http.response.start",
    "status": 200,
    "headers": [(b"content-type", b"application/json")],
})

# Step 2: Body পাঠাও
await send({
    "type": "http.response.body",
    "body": b'{"message": "Hello"}',
})
```

কেন দুটো আলাদা? কারণ header পাঠিয়ে body পরে পাঠানো যায়, বা body একাধিক chunk-এ পাঠানো যায়। এটা streaming-কে possible করে।

---

### একটা সম্পূর্ণ Minimal ASGI App

এই তিনটা একসাথে দেখো:

```python
async def app(scope, receive, send):
    # শুধু HTTP request handle করব এখন
    if scope["type"] != "http":
        return

    # client-এর request body পড়ো
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
        "body": b"Hello from ASGI!",
    })
```

এটুকুই একটা পূর্ণাঙ্গ ASGI application। FastAPI বা Django Channels ভেতরে ভেতরে এই কাজটাই করছে — শুধু routing, validation, serialization-এর বিশাল infrastructure সহ।

---

## Event Loop — ASGI-র ভেতরের ইঞ্জিন

Event loop হলো সেই coordinator যে ঠিক করে কখন কোন task চলবে। এটা বোঝাটা critical কারণ ASGI-র সব সুবিধা এখান থেকেই আসে।

ধরো 3টা request একসাথে এলো। প্রতিটাতে 100ms DB query আছে:

```text
WSGI (1 worker)-তে কী হতো:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
t=0ms    → Request A শুরু
t=0-100ms → Worker A-এর জবাবের জন্য বসে আছে
t=100ms  → Response A পাঠাল
t=100ms  → Request B শুরু হলো
t=100-200ms → Worker B-এর জবাবের জন্য বসে আছে
t=200ms  → Response B পাঠাল
... মোট 300ms
```

```text
ASGI (1 worker)-তে কী হয়:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
t=0ms   → A শুরু, DB query দিল → await (pause)
t=0ms   → B শুরু, DB query দিল → await (pause)
t=0ms   → C শুরু, DB query দিল → await (pause)
t=100ms → A-র DB response এলো → resume → response পাঠাল
t=100ms → B-র DB response এলো → resume → response পাঠাল
t=100ms → C-র DB response এলো → resume → response পাঠাল
... মোট ~100ms
```

300ms থেকে 100ms। 3গুণ দ্রুত — একই hardware, একই code, শুধু execution model আলাদা।

Event loop এই পুরো choreography manage করে নিজে নিজে। তোমাকে কিছু করতে হয় না — শুধু সঠিক জায়গায় `await` লিখতে হয়।

---

## Blocking — Async-এর সবচেয়ে বড় শত্রু

ASGI-র সবচেয়ে গুরুত্বপূর্ণ নিয়ম, যেটা না মানলে পুরো সুবিধা মাটি:

**Async function-এর ভেতরে কোনো blocking call রাখা যাবে না।**

```python
# ❌ এটা ASGI app-কে কার্যত sync করে দেয়
async def handler(scope, receive, send):
    time.sleep(5)          # Python thread ঘুমিয়ে পড়ল — event loop সহ
    data = requests.get(url) # sync HTTP call — সবাই থামল
    ...
```

`time.sleep(5)` call করলে কী হয়? Python-এর যে thread-এ event loop চলছে, সেই thread-ই ঘুমিয়ে পড়ে। Event loop চলতে পারে না। ওই 5 সেকেন্ড অন্য কোনো request handle হবে না। তুমি async লিখেছ ঠিকই — কিন্তু সুবিধা শূন্য।

```python
# ✔️ সঠিক উপায়
async def handler(scope, receive, send):
    await asyncio.sleep(5)          # thread ঘুমায় না, শুধু task pause হয়
    async with httpx.AsyncClient() as client:
        data = await client.get(url) # async HTTP — event loop চলতে থাকে
    ...
```

`await asyncio.sleep(5)` event loop-কে বলে: "এই task 5 সেকেন্ড দরকার নেই, অন্য কাজ করো।" Thread ঘুমায় না — শুধু এই task pause হয়। 5 সেকেন্ড পর event loop এই task-কে আবার চালু করে।

**Practical rule:** Async code-এ সব I/O operation async library দিয়ে করতে হবে:

| Sync (❌ async context-এ ব্যবহার করো না) | Async (✔️ ব্যবহার করো) |
|---|---|
| `requests` | `httpx` বা `aiohttp` |
| `psycopg2` | `asyncpg` বা `psycopg3` |
| `time.sleep()` | `asyncio.sleep()` |
| `open()` + `read()` | `aiofiles` |

এই list-টা মাথায় রাখো। Async project-এ যখনই কোনো I/O লিখবে, এক সেকেন্ড ভাবো — এটা async নাকি sync?

---

## WebSocket — যেটা WSGI-তে সম্ভবই ছিল না

WSGI-র model হলো request-response: client পাঠাল, server জবাব দিল, connection বন্ধ। এই model-এ server নিজে থেকে client-কে কিছু পাঠাতে পারে না — client আগে জিজ্ঞেস না করলে।

Real-time chat, live notifications, collaborative editing — এসবে দরকার persistent connection যেখানে server যেকোনো সময় data push করতে পারে। এটাই WebSocket।

ASGI-তে WebSocket স্বাভাবিকভাবেই fit করে। Connection ধরে রাখো, `receive()` দিয়ে client-এর message পড়তে থাকো, `send()` দিয়ে যখন খুশি message পাঠাও:

```python
async def app(scope, receive, send):
    if scope["type"] == "websocket":

        # প্রথমে connection accept করতে হয়
        await send({"type": "websocket.accept"})

        # এবার loop — connection বন্ধ না হওয়া পর্যন্ত চলবে
        while True:
            message = await receive()

            # client disconnect করলে loop ভাঙো
            if message["type"] == "websocket.disconnect":
                break

            # client যা পাঠাল সেটা uppercase করে ফেরত দাও
            text = message.get("text", "")
            await send({
                "type": "websocket.send",
                "text": text.upper()
            })
```

এই while loop অনেকক্ষণ চলতে পারে — কিন্তু প্রতিটা `await` মুহূর্তে event loop free। হাজারটা এরকম WebSocket connection একই worker-এ চলতে পারে।

WSGI-তে এই loop-টা একটাই worker সারাদিন block করে রাখত।

---

## Streaming Response — Data তৈরি হতে হতে পাঠাও

ASGI-তে response একসাথে সব পাঠাতে হয় না। Data তৈরি হতে হতে পাঠানো যায়:

```python
await send({
    "type": "http.response.start",
    "status": 200,
    "headers": [(b"content-type", b"text/plain")],
})

# data generate হতে হতে পাঠাও
for i in range(10):
    await asyncio.sleep(0.5)  # ধরো এখানে কিছু processing হচ্ছে
    await send({
        "type": "http.response.body",
        "body": f"Line {i}\n".encode(),
        "more_body": True,    # এখনো শেষ হয়নি
    })

# শেষ করো
await send({
    "type": "http.response.body",
    "body": b"",
    "more_body": False,
})
```

`more_body: True` মানে "আরো আসছে, connection খোলা রাখো।" `more_body: False` মানে "এবার শেষ।"

এটা কোথায় কাজে লাগে? বড় CSV export, AI model-এর token-by-token output, large file download — পুরো response memory-তে load না করে stream করা যায়। Client ডেটা পেতে শুরু করে, পুরোটা আসার আগেই।

---

## ASGI Servers — কোনটা কখন?

ASGI server মানে সেই software যে HTTP connection নেয় এবং তোমার ASGI app-কে call করে।

**Uvicorn** — সবচেয়ে popular। FastAPI-র default। Fast, lightweight। Development-এ সরাসরি ব্যবহার করো।

```bash
uvicorn myapp:app --host 0.0.0.0 --port 8000 --reload
```

**Daphne** — Django Channels-এর জন্য বানানো। WebSocket-heavy Django app-এ বেশি দেখা যায়।

**Gunicorn + UvicornWorker** — Production-এ এটাই সবচেয়ে common pattern:

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 4 myapp:app
```

কেন দুটো? কারণ দুজনের কাজ আলাদা।

Gunicorn করছে **process management** — worker crash করলে restart দেওয়া, graceful shutdown, signal handling। এটাতে Gunicorn বছরের পর বছর ধরে battle-tested।

UvicornWorker করছে **actual async request handling** — event loop চালানো, async I/O manage করা।

একা Uvicorn production-এ চললে worker crash হলে সামলানো কঠিন। একা Gunicorn দিলে async support নেই। দুটো একসাথে দিলে দুজনের সুবিধাই পাওয়া যায়।

---

## Common ভুল ধারণা — পরিষ্কার করে নাও

এই ভুলগুলো অনেকেই করে, তাই আলাদা করে বলছি।

**"ASGI মানেই দ্রুত"** — সবসময় সত্যি না।

ASGI তখনই faster যখন app I/O-bound। যদি app CPU-intensive হয় — image processing, ML inference, heavy computation — তাহলে ASGI কোনো সাহায্য করে না। Event loop-এ একটাই thread, সে একটার বেশি CPU কাজ একসাথে করতে পারে না।

**"Async = Parallel"** — দুটো আলাদা জিনিস।

Async মানে concurrency — একটা task pause থাকলে অন্যটা চলে। Parallel মানে একই সময়ে দুটো কাজ আলাদা CPU core-এ চলছে। ASGI-তে event loop single-threaded — parallelism নয়, concurrency। সত্যিকারের parallelism পেতে multiple worker process লাগবে।

**"Async library use করলেই হলো"** — না, blocking code যেখানে আছে সেখানেই বিপদ।

`asyncpg` use করলে DB query async হলো ঠিকই — কিন্তু যদি সেই query-র result দিয়ে একটা ভারী Python loop চালাও, সেটাও event loop block করতে পারে। `await` আছে মানেই নিরাপদ, এটা মনে করা ঠিক না।

---

## Mental Model: Async কীভাবে কাজ করে

WSGI আর ASGI-র পার্থক্য এক লাইনে:

**WSGI:** Worker কাজ করুক বা wait করুক — সে occupied। Scaling মানে worker বাড়াও।

**ASGI:** Worker শুধু তখনই occupied যখন সে সত্যিকার অর্থে কিছু করছে। Wait মানে ফাঁকা — অন্য কাজ নাও।

একটা রেস্তোরাঁর analogy দিয়ে ভাবো:

**WSGI waiter:** একজন customer-এর order নিয়ে kitchen-এ গেল। রান্না না হওয়া পর্যন্ত সে kitchen-এর দরজায় দাঁড়িয়ে অপেক্ষা করছে। অন্য কোনো table serve করছে না।

**ASGI waiter:** একজন customer-এর order kitchen-এ দিয়ে এলো। রান্না হচ্ছে — এই ফাঁকে সে অন্য table-এর order নিল, অন্য কাউকে পানি দিল। Kitchen থেকে ডাক এলে আবার সেই table-এ গেল।

একই waiter — দ্বিতীয়জন অনেক বেশি কাজ করতে পারছে।

```text
WSGI:   request → [কাজ + wait + কাজ] → response → তারপর পরের request

ASGI:   request A → কাজ → await ─────────────────────── resume → response A
        request B →       কাজ → await ───────────── resume → response B
        request C →             কাজ → await ─── resume → response C
                                                    ↑
                                           সব প্রায় একসাথে শেষ
```

Production-এ optimum setup: multiple Uvicorn workers (CPU core-এ parallelism) + প্রতিটাতে async event loop (I/O-তে concurrency)। দুটো মিলিয়ে সবচেয়ে বেশি performance।

---

## চ্যালেঞ্জ: Timeline নিজে আঁকো

**Setup:**
- 1টি Uvicorn worker
- Async code, সব I/O সঠিকভাবে awaited
- 5টা request একসাথে এলো
- প্রতিটা request করে: **1 সেকেন্ড DB query** + **1 সেকেন্ড external API call**

**প্রশ্ন ১:** সব 5টা request শেষ হতে মোট কত সময় লাগবে?

**প্রশ্ন ২:** একই setup কিন্তু WSGI sync worker-এ হলে সময় কত হতো?

**প্রশ্ন ৩:** এই performance gain ঠিক কোথা থেকে এলো? মাথায় timeline আঁকো — কোন মুহূর্তে কোন request কী করছে।

Hint: DB query আর API call দুটো কি একই সময়ে চলতে পারে? এই প্রশ্নের উত্তর ভাবলে পুরো জিনিসটা clear হয়ে যাবে।

---

## সারসংক্ষেপ

| বিষয় | মূল কথা |
|---|---|
| ASGI কেন এলো | WSGI-র blocking model high concurrency আর WebSocket handle করতে পারত না |
| তিনটা argument | `scope` = connection-এর পরিচয়, `receive` = client থেকে পড়ো, `send` = client-কে লেখো |
| Event loop | await-এ task pause → অন্য task চলে → resume → একটা thread-এই হাজারো concurrency |
| Blocking-এর বিপদ | sync call event loop freeze করে — সব সুবিধা নষ্ট |
| WebSocket | persistent connection, দুদিক থেকেই message — ASGI-তে naturally fit |
| Streaming | `more_body: True` দিয়ে chunks-এ response পাঠাও |
| Production setup | Gunicorn (process management) + UvicornWorker (async handling) |
