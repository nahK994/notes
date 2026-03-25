# WSGI: Python Web-এর পুরনো দারোয়ান — যাকে না বুঝলে Async বোঝা যাবে না

> *"একটা পুরনো সিস্টেম বোঝার সবচেয়ে ভালো উপায় হলো জানা — কোন সমস্যা সমাধান করতে সে এসেছিল।"*

---

## শুরুতে একটা প্রশ্ন

ধরো তুমি একটা ক্যাফে খুলেছ। একজনই কাউন্টারে দাঁড়িয়ে আছে — সে order নেয়, বানায়, serve করে। একজন customer এলে ঠিক আছে। কিন্তু দশজন একসাথে এলে?

প্রথম জন order দিল। বাকি নয়জন দাঁড়িয়ে অপেক্ষা করছে।

এটাই WSGI-র পৃথিবী। এবং এই সীমাবদ্ধতা বোঝার মধ্যেই লুকিয়ে আছে async, ASGI, এবং modern web architecture-এর পুরো যুক্তি।

---

## আগের পর্বের শিক্ষা — Blocking System কী করে

আগের চ্যালেঞ্জে আমরা দেখেছিলাম একটা blocking server-এ কী হয়:

- **১টি worker**, প্রতিটি request **৫ সেকেন্ড** লাগে
- একসাথে মাত্র **১টি** request handle হয়
- ১০টা request একসাথে এলে শেষ user-কে অপেক্ষা করতে হয় **৫০ সেকেন্ড**

```text
Request 1  → [████████] 5s  → done
Request 2  →           [████████] 5s  → done
Request 3  →                     [████████] 5s  → done
...
Request 10 →                                       [████████] 5s → done
                                                                    ↑
                                                               এই user 50s অপেক্ষা করেছে
```

প্রথম user পেল ৫ সেকেন্ডে। শেষ user পেল ৫০ সেকেন্ডে। User experience? ভয়াবহ।

এই সমস্যার সমাধান খুঁজতে গিয়েই concurrency, multiple workers, এবং async-এর দরকার পড়ে। কিন্তু তার আগে বুঝতে হবে — Python-এর web ecosystem আসলে কীভাবে কাজ করে। এখানেই আসে **WSGI**।

---

## WSGI কেন এলো? — ইতিহাসটা জানা দরকার

২০০০-এর দশকের শুরুতে Python-এ web development ছিল বিশৃঙ্খল। Django তার নিজস্ব উপায়ে server-এর সাথে কথা বলত, Flask অন্যভাবে, Pylons আবার অন্যভাবে। প্রতিটি framework প্রতিটি server-এর জন্য আলাদা adapter লিখতে হত।

মানে হলো: তুমি যদি Apache-এ Django চালাও, তাহলে এক ধরনের glue code। Nginx-এ চালালে আরেক ধরনের। নতুন framework লিখলে প্রতিটি server-এর জন্য আলাদা আলাদা কাজ।

**WSGI (Web Server Gateway Interface)** এলো ২০০৩ সালে, PEP 333-এ, এই chaos থামাতে।

WSGI বলল একটাই কথা:

> *"সবাই একটা common contract follow করো। Server জানবে app-কে কীভাবে call করতে হয়। App জানবে server থেকে কী পাবে। মাঝখানে আর কোনো অনুবাদকের দরকার নেই।"*

এই idea-টা এত সফল হয়েছিল যে আজও Django, Flask সহ অনেক framework-এর ভিত্তি WSGI।

---

## WSGI App আসলে কী?

WSGI-র সবচেয়ে সুন্দর দিক হলো এর simplicity। একটা WSGI-compliant app মানে শুধু একটা Python function (বা callable) যেটা দুটো argument নেয়:

```python
def app(environ, start_response):
    status = "200 OK"
    headers = [("Content-Type", "text/plain")]

    start_response(status, headers)
    return [b"Hello, WSGI World!"]
```

এটুকুই। এই তিনটা জিনিস দিয়েই পুরো WSGI দুনিয়া চলে।

### `environ` — Request-এর সব তথ্য

`environ` হলো একটা Python dictionary যেখানে HTTP request-এর সব তথ্য আছে:

```python
environ = {
    "REQUEST_METHOD": "GET",          # GET, POST, PUT...
    "PATH_INFO": "/api/users",        # URL path
    "QUERY_STRING": "page=2",         # URL-এর ?-এর পরের অংশ
    "HTTP_HOST": "example.com",       # Host header
    "wsgi.input": <socket>,           # Request body
    # ... আরো অনেক কিছু
}
```

Server এই dictionary তৈরি করে app-কে দেয়। App এখান থেকে যা দরকার নেয়।

### `start_response` — Response শুরু করা

এটা একটা callable যেটা দিয়ে app server-কে বলে: "এই status code দাও, এই headers পাঠাও।"

```python
start_response("200 OK", [
    ("Content-Type", "application/json"),
    ("X-Custom-Header", "value"),
])
```

### Return value — Response body

Function যা return করে সেটা একটা iterable of bytes। Server এটা client-কে পাঠায়।

**মূল বিষয়:** পুরো flow-টা synchronous। App call হলো, কাজ করল, return করল। Server পরবর্তী request নিল। কোনো await নেই, কোনো callback নেই।

---

## Request কীভাবে যাতায়াত করে?

WSGI-র পুরো journey এভাবে হয়:

```text
Browser
  │
  │ HTTP Request
  ▼
Gunicorn (Web Server)
  │
  │ environ তৈরি করে
  │ একটি Worker-কে assign করে
  ▼
Worker Process
  │
  │ app(environ, start_response) call করে
  ▼
তোমার Python App (Django/Flask)
  │
  │ কাজ করে (DB query, logic...)
  │ start_response() call করে
  │ response body return করে
  ▼
Worker Process
  │
  │ response নিয়ে Gunicorn-কে দেয়
  ▼
Gunicorn
  │
  │ HTTP Response তৈরি করে
  ▼
Browser
```

**গুরুত্বপূর্ণ:** এই পুরো সময়টা — app যখন DB query করছে, file পড়ছে, external API-তে request পাঠাচ্ছে — **worker অপেক্ষা করছে।** অন্য কোনো request নিতে পারছে না।

---

## Blocking-এর ভয়াবহতা — একটা concrete উদাহরণ

```python
def view(request):
    # এই লাইনটা 5 সেকেন্ড লাগে
    data = requests.get("https://external-api.com/data")
    return JsonResponse(data.json())
```

এই ৫ সেকেন্ডে worker কী করছে? **কিছুই না।** শুধু বসে আছে, network-এর জবাবের অপেক্ষায়। কিন্তু সে occupied, তাই অন্য কোনো user-এর request নিতে পারছে না।

এটা অনেকটা এরকম: একজন দোকানদার কাউকে বলল "একটু বসো, আমি গুদামঘর থেকে জিনিস আনছি।" তারপর সে গুদামঘরে গেল, দরজা বন্ধ করে বসে রইল — বের হবে না যতক্ষণ না জিনিস আসে। বাইরে আরো ১০ জন customer দাঁড়িয়ে, কিন্তু দোকান ফাঁকা।

> **Insight:** Waiting = Wasted Capacity। System কিছু করছে না, কিন্তু blocked।

---

## Gunicorn দিয়ে WSGI-র Concurrency

WSGI নিজে concurrency দেয় না। কিন্তু Gunicorn-এর মতো server দুটো উপায়ে এই সমস্যা কমায়:

### উপায় ১ — Multiple Workers (Process)

```bash
gunicorn -w 4 myapp:app
```

৪টা আলাদা Python process চলবে। প্রতিটা একটা করে request handle করবে। মানে একসাথে ৪টা request।

```text
Worker 1 → [Request A চলছে...]
Worker 2 → [Request B চলছে...]
Worker 3 → [Request C চলছে...]
Worker 4 → [Request D চলছে...]
Worker 5 → নেই, Request E অপেক্ষায়...
```

সহজ, কিন্তু সীমাবদ্ধ। ১০০০ concurrent user হলে ১০০০ worker লাগবে? প্রতিটা worker মানে একটা পুরো Python process — ৫০-১০০MB RAM। ১০০০ worker = ৫০-১০০ GB RAM। অসম্ভব।

### উপায় ২ — Threads (gthread)

```bash
gunicorn -k gthread --threads 4 -w 2 myapp:app
```

প্রতিটি worker-এ ৪টা thread। ২ worker × ৪ thread = ৮টা concurrent request।

Thread-রা একই process-এর memory share করে, তাই process-এর মতো ভারী না। কিন্তু Python-এর GIL (Global Interpreter Lock) আছে — একসময় একটাই thread CPU-তে কাজ করতে পারে। I/O-এর সময় GIL ছাড়া হয়, তাই I/O-heavy app-এ threads ভালো কাজ করে।

---

## WSGI-তে Streaming — সীমিত সম্ভাবনা

WSGI-তে response একটা iterable return করা যায়, তাই theoretically streaming possible:

```python
def app(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])

    def generate():
        for i in range(5):
            time.sleep(1)
            yield f"Line {i}\n".encode()

    return generate()
```

কিন্তু এখানে সমস্যা হলো: generator যখন `time.sleep(1)` করছে, worker তখনও blocked। পরের request নিতে পারছে না। তাই WSGI-তে streaming আছে কিন্তু সেটা দিয়ে real-time, high-concurrency কাজ করা কঠিন।

---

## WSGI-র দেয়াল — যেখানে সে থামে

WSGI ২০ বছর আগে ডিজাইন হয়েছিল। তখনকার web অনেক সহজ ছিল। আজকের প্রয়োজনে সে কিছু জায়গায় মৌলিকভাবে অক্ষম:

**WebSocket support নেই।** WebSocket হলো persistent, two-way connection — server যেকোনো সময় client-কে data push করতে পারে। Real-time chat, live notification, collaborative editing — এসবের জন্য WebSocket দরকার। WSGI-র request-response model-এ এটা সম্ভব নয়।

**Async support নেই।** WSGI function-এ `async def` বা `await` ব্যবহার করলে কিছুই হবে না — server সেটা বুঝবে না। Django বা Flask-এ async view লিখলেও যদি WSGI server ব্যবহার কর, সেই async-এর সুবিধা পাবে না।

**Long-lived connection নেই।** Server-Sent Events (SSE), live dashboard, real-time monitoring — এসব WSGI দিয়ে ভালোভাবে করা যায় না।

---

## তাহলে WSGI কখন ব্যবহার করব?

এত সীমাবদ্ধতার পরেও WSGI এখনো প্রাসঙ্গিক। কারণ সব app-এর real-time লাগে না।

**WSGI ভালো কাজ করে:**
- সাধারণ CRUD API (blog, e-commerce backend)
- Admin dashboard যেখানে concurrency কম
- CPU-heavy processing যেখানে I/O wait-এর চেয়ে computation বেশি
- Internal tools যেখানে user সংখ্যা কম এবং নিয়ন্ত্রিত

**WSGI যথেষ্ট নয়:**
- Real-time chat বা notification system
- ১০,০০০+ concurrent user-এর API
- WebSocket-dependent feature
- Async external API call-এ ভারী নির্ভরশীল system

---

## Django + WSGI — একটা বাস্তব সতর্কতা

Django এখন async view support করে:

```python
# Django async view
async def my_view(request):
    data = await some_async_db_call()
    return JsonResponse(data)
```

কিন্তু তুমি যদি এটা Gunicorn-এর WSGI mode-এ চালাও, এই `async def` কার্যত sync-এর মতোই কাজ করে। Server জানে না এটাকে async ভাবে handle করতে হবে।

Django-র async-এর পূর্ণ সুবিধা পেতে হলে **ASGI server** (যেমন Uvicorn, Daphne) দরকার। এটা নিয়ে পরের পর্বে বিস্তারিত আলোচনা হবে।

---

## Mental Model: Single-Lane Bridge

WSGI-কে একটা single-lane bridge হিসেবে ভাবো।

```text
        গাড়ির সারি
        🚗🚗🚗🚗🚗🚗
              │
              ▼
     ┌────────────────┐
     │  Single Lane   │  ← একবারে একটাই
     │    Bridge      │
     └────────────────┘
              │
              ▼
         গন্তব্য
```

একটা গাড়ি ঢুকলে, সে বের না হওয়া পর্যন্ত পরেরটা ঢুকতে পারবে না। Multiple workers মানে multiple bridge — তবু সেটা সীমিত।

আসল সমাধান হলো bridge-এর ধারণাটাই বদলে ফেলা — যেখানে একটা bridge-এই অনেক গাড়ি চলতে পারে। সেটাই ASGI এবং async-এর পৃথিবী।

---

## চ্যালেঞ্জ: নিজে হিসাব করো

**Scenario:**
- Gunicorn চলছে
- **২টি sync worker**
- প্রতিটি request সম্পূর্ণ হতে **৩ সেকেন্ড** লাগে

**প্রশ্নগুলো:**

1. একসাথে সর্বোচ্চ কতটি request handle হতে পারে?
2. ১০টি request একসাথে এলে সবার শেষে total time কত লাগবে?
3. Bottleneck ঠিক কোথায়? Worker বাড়ালে কি infinitely scale করা যাবে?

নিজে ভেবে দেখো। পরের পর্বে এই উত্তরের সাথে সাথে ASGI-র জগতে প্রবেশ করব।

---

## সারসংক্ষেপ

| বিষয় | মূল কথা |
|---|---|
| WSGI কেন এলো | Server ও framework-এর মধ্যে common standard তৈরি করতে |
| WSGI app কী | `(environ, start_response)` নেওয়া একটি callable |
| Blocking সমস্যা | I/O wait-এ worker বসে থাকে, নতুন request নিতে পারে না |
| Concurrency কীভাবে | Multiple workers (process) বা threads দিয়ে |
| WSGI-র সীমাবদ্ধতা | WebSocket নেই, async নেই, high concurrency কঠিন |
| কখন ব্যবহার করব | Simple CRUD, low-to-medium traffic, CPU-heavy task |

---

> **পরের পর্বে:** ASGI — WSGI-র উত্তরসূরি, যে async বোঝে, WebSocket চেনে, এবং হাজার হাজার connection একা সামলাতে পারে।
