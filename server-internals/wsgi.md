# ⚡ Module 2: ASGI — The Modern Switchboard

---

# 🧠 2.1 ASGI কী?

**ASGI = Asynchronous Server Gateway Interface**

👉 এটা WSGI-এর evolution
👉 async + event-driven protocol

---

## 🔥 Core idea

> request = একবারের ঘটনা না
> connection = ongoing conversation

---

# ⚔️ 2.2 WSGI vs ASGI (sharp contrast)

| Feature       | WSGI 🪵        | ASGI ⚡     |
| ------------- | -------------- | ---------- |
| Model         | sync           | async      |
| Request style | one-shot       | streaming  |
| WebSocket     | ❌              | ✔️         |
| Concurrency   | process/thread | event loop |

---

# ⚙️ 2.3 ASGI App Structure

একটা ASGI app:

```python
async def app(scope, receive, send):
    ...
```

---

## 🧩 তিনটা pillar

---

### 🧱 1. `scope`

👉 request metadata

```python
{
  "type": "http",
  "method": "GET",
  "path": "/users"
}
```

---

### 📥 2. `receive`

👉 client থেকে message নেয়

* request body
* websocket message

---

### 📤 3. `send`

👉 client-কে message পাঠায়

* response start
* response body

---

# 🎬 2.4 ASGI Lifecycle (HTTP)

চল step-by-step দেখি:

---

## 🔁 Flow

1. client → request পাঠায়
2. Uvicorn → app call করে
3. app → `receive()` দিয়ে body নেয়
4. app → process
5. app → `send()` দিয়ে response দেয়

---

## 🧪 Minimal ASGI app

```python
async def app(scope, receive, send):
    if scope["type"] == "http":
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

---

👉 লক্ষ্য করো:

* সবকিছু async
* message-based protocol

---

# ⚡ 2.5 Event Loop — The Heartbeat

ASGI আসলে কাজ করে event loop দিয়ে

---

## 🧠 কীভাবে?

* multiple request আসে
* task হিসেবে queue হয়
* `await` এলে pause
* অন্য task run

---

## 🎭 Mini নাটক

```python
async def handler():
    await db_call()   # pause
    await api_call()  # pause
```

👉 এই pause-টাই concurrency unlock করে

---

# 🧨 2.6 Blocking Problem (again, কিন্তু deeper)

---

## ❌ Bad code

```python
async def handler():
    time.sleep(5)
```

👉 event loop freeze

---

## ✔️ Good code

```python
async def handler():
    await asyncio.sleep(5)
```

👉 অন্য request run করতে পারে

---

## 🧠 Golden law

> async world-এ “cooperation” লাগে
> না দিলে system stuck

---

# 🔄 2.7 Concurrency Model

---

## 🧩 Key insight

* ১ worker
* ১ thread
* ১ event loop

👉 তবুও multiple request handle possible

---

## কেন?

👉 কারণ:

* request pause করে
* interleave হয়

---

## 🎯 Visual

```text
Req1 → start → await → pause
Req2 → run
Req3 → run
Req1 → resume
```

---

# 🌐 2.8 WebSocket (ASGI superpower)

WSGI পারে না
ASGI পারে 🔥

---

## 🧪 Example flow

1. client connect
2. connection open থাকে
3. message ↔ message (bidirectional)

---

👉 chat app, live notification possible

---

# 🧬 2.9 Streaming Response

---

## 🧪 Example

```python
await send({...start...})

for chunk in data:
    await send({...body...})
```

👉 response chunk by chunk যায়

---

# ⚙️ 2.10 Servers in ASGI world

---

## 🔹 Uvicorn

* fast
* asyncio-based
* FastAPI default

---

## 🔹 Daphne

* Django Channels use করে
* WebSocket heavy use-case

---

---

## 🧠 With Gunicorn

```bash
gunicorn -k uvicorn.workers.UvicornWorker
```

👉 Gunicorn = manager
👉 UvicornWorker = async execution

---

# ⚠️ 2.11 Common Misconceptions

---

## ❌ “ASGI মানেই faster”

না

👉 depends on:

* async code
* non-blocking I/O

---

## ❌ “async = parallel”

না

👉 concurrency, not parallelism

---

# 🧩 2.12 When ASGI shines

---

## ✔️ Perfect fit:

* API calls heavy (external services)
* DB heavy
* high concurrency (1000+ users)
* WebSocket apps

---

## ❌ Not ideal:

* CPU heavy কাজ
* blocking legacy code

---

# 🎯 2.13 Final Mental Model

👉 ASGI world:

```text
request → event loop → await → pause → resume → response
```

👉 scaling:

* more workers (process)
* each worker → many concurrent request

---

# ⚔️ Mini Challenge (Level 2)

ধরো:

* 1 Uvicorn worker
* async code
* 5টা request একসাথে এলো
* প্রতিটা request:

  * 1 sec DB await
  * 1 sec API await

---

## ❓ Question:

1. total time কত লাগবে সব request শেষ হতে?
2. যদি একই code sync হত, তখন কী হতো?
3. কোথায় performance gain আসলো?

---

👉 তুমি solve করো
আমি তোমার logic deep inspect করব 🔍
