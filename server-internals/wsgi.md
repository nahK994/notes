# 🪵 Module 1: WSGI — The Old Gatekeeper

---

## Module 0-এর Mini Challenge-এর উত্তর

আগের module শেষে একটা প্রশ্ন ছিল:

> 1 worker, 1 thread, async code — এই setup কি একসাথে multiple request handle করতে পারবে?

**উত্তর: হ্যাঁ, পারবে — কিন্তু শর্ত আছে।**

Async code-এ যখনই কোনো `await` আসে, event loop সেই task-কে pause করে অন্য task চালাতে পারে। তাই একটাই worker দিয়ে অনেক request handle করা যায় — যতক্ষণ সব request I/O-bound এবং সব `await` properly লেখা।

**কিন্তু কখন পারবে না?**

যদি কোনো একটা request-এ blocking call থাকে — যেমন `time.sleep(5)` বা কোনো sync database library — তাহলে পুরো event loop সেই 5 সেকেন্ড আটকে যাবে। Single worker হওয়ায় বাকি সব request সেই সময়টা wait করবে। Async লেখা থাকলেও একটা blocking call সব কিছু নষ্ট করে দেয়।

এই concept-টা মাথায় রেখে এবার Module 1-এ ঢোকো — কারণ WSGI সম্পূর্ণ উল্টো জগৎ।

---

## কেন WSGI শিখছি?

Python web development-এর ইতিহাস বুঝতে এবং আধুনিক async framework (ASGI) কেন এলো সেটা বুঝতে WSGI জানা দরকার। এছাড়া বেশিরভাগ production Django app এখনও WSGI-তেই চলে। তাই এটা "পুরনো" হলেও irrelevant না।

---

# 🧠 1.1 WSGI কী এবং কেন এলো?

**WSGI = Web Server Gateway Interface**

২০০০-এর দশকের শুরুতে Python web development-এ একটা বড় সমস্যা ছিল। Apache, Nginx, lighttpd — বিভিন্ন web server ছিল। Django, Flask, Pylons — বিভিন্ন framework ছিল। কিন্তু কেউ কারো সাথে সরাসরি কথা বলতে পারত না। প্রতিটা framework-কে প্রতিটা server-এর জন্য আলাদাভাবে লিখতে হতো।

WSGI এলো এই chaos দূর করতে। এটা একটা **specification** — মানে একটা চুক্তি বা contract — যেটা বলে দেয় web server এবং Python application কীভাবে একে অপরের সাথে কথা বলবে।

```
[Gunicorn / uWSGI]  ←→  [WSGI interface]  ←→  [Django / Flask]
     web server              contract            application
```

এই চুক্তির ফলে যেকোনো WSGI-compatible server যেকোনো WSGI-compatible framework-এর সাথে কাজ করতে পারে। Django বানালে Gunicorn দিয়েও চালাও, uWSGI দিয়েও চালাও — framework-কে কিছু পরিবর্তন করতে হবে না।

---

# ⚙️ 1.2 WSGI App আসলে কী?

WSGI-র contract অনুযায়ী, একটা Python web application মূলত একটাই জিনিস — **একটা callable** (function বা class) যেটা দুটো argument নেয়:

```python
def app(environ, start_response):
    status = "200 OK"
    headers = [("Content-Type", "text/plain")]
    start_response(status, headers)
    return [b"Hello, WSGI"]
```

এটাই একটা complete, working WSGI app। মাত্র কয়েক লাইনে।

এখানে দুটো জিনিস বোঝা দরকার:

**`environ`:** এটা একটা Python dictionary যেখানে HTTP request-এর সব তথ্য থাকে — method (GET/POST), path, headers, query string, request body সব কিছু। Server এই dictionary তৈরি করে app-কে দেয়।

**`start_response`:** এটা একটা callback function যেটা server দেয়। App এটাকে call করে বলে দেয় — "এই response-এর status code কী হবে, headers কী হবে।" Response body return করা হয় function-এর return value হিসেবে — লক্ষ্য করো, এটা **bytes-এর iterable**, plain string না।

---

# 🔄 1.3 একটা Request কীভাবে WSGI-তে Journey করে

এটা বোঝাটা অনেক জরুরি কারণ পরে performance সমস্যা কোথায় হয় সেটা বুঝতে এই flow লাগবে।

ধরো user browser-এ `example.com/users` টাইপ করল:

```
1. Browser → TCP connection → Gunicorn
2. Gunicorn → raw HTTP bytes পড়ে → parse করে
3. Gunicorn → environ dictionary তৈরি করে
4. Gunicorn → app(environ, start_response) call করে
5. Django view execute হয় (DB query, logic, সব কিছু)
6. View return করে response body
7. Gunicorn → response HTTP format-এ convert করে
8. Browser-এ পাঠায়
```

সবচেয়ে গুরুত্বপূর্ণ কথা হলো **step 4 থেকে step 6 পুরোটাই blocking**। Gunicorn step 4-এ app call করার পর চুপ করে বসে থাকে যতক্ষণ না step 6-এ response ফেরত আসে। এই পুরো সময়টায় সেই worker অন্য কোনো request নিতে পারে না।

---

# 🧨 1.4 Blocking Nature — WSGI-র মূল সীমাবদ্ধতা

WSGI-র design-এ async বা event loop-এর কোনো ধারণাই নেই। এটা তৈরি হয়েছিল synchronous Python-কে মাথায় রেখে। ফলে:

```python
def view(request):
    time.sleep(5)      # DB query বা external API call হোক বা এটাই হোক
    return "done"
```

এই view যখন execute হচ্ছে, সেই worker process টা পুরো 5 সেকেন্ড আটকে আছে। CPU কিছু করছে না, শুধু wait করছে। কিন্তু worker occupied — নতুন request নেওয়ার capacity নেই।

এটা ঠিক যেন একটা দোকানে একজনই কাউন্টার আছে, আর সে একজন customer-এর order process করতে গিয়ে 5 মিনিট kitchen-এ অপেক্ষা করছে। বাকি সবাই বাইরে দাঁড়িয়ে।

---

# ⚔️ 1.5 তাহলে WSGI-তে Concurrency কীভাবে হয়?

WSGI নিজে concurrency দেয় না। এই দায়িত্ব সম্পূর্ণ server-এর।

Gunicorn দুটো উপায়ে concurrency দিতে পারে:

**উপায় 1 — Multiple worker processes:**

```bash
gunicorn -w 4 myapp:app
```

এখানে 4টা আলাদা Python process তৈরি হয়। প্রতিটা process একটা request handle করতে পারে। মানে একসাথে 4টা request parallel-এ চলতে পারে।

**উপায় 2 — Threads:**

```bash
gunicorn -k gthread --threads 4 myapp:app
```

একটা worker process-এর ভেতরে 4টা thread। প্রতিটা thread একটা request নিতে পারে। GIL-এর কারণে true parallelism নেই, কিন্তু I/O wait-এর সময় অন্য thread কাজ করতে পারে।

**Key takeaway:** WSGI-তে concurrency মানে শুধু worker বা thread সংখ্যা বাড়ানো। এর বাইরে কোনো উপায় নেই। এবং এই সংখ্যার একটা practical limit আছে — প্রতিটা process memory নেয়।

---

# 🧬 1.6 WSGI-তে Streaming

WSGI technically streaming support করে। Response body হিসেবে generator return করা যায়:

```python
def app(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])

    def generate():
        yield b"Hello\n"
        yield b"World\n"

    return generate()
```

কিন্তু এটা **synchronous streaming**। Generator টা synchronously execute হয়। Real-time use case যেমন live data feed বা server-sent events-এ এটা কার্যকর না, কারণ underlying model টাই blocking।

---

# ⚠️ 1.7 WSGI-র Limitations — কেন Modern World এটায় Stuck

আধুনিক web application-এর চাহিদার সাথে WSGI কয়েকটা জায়গায় মিলতে পারে না:

**WebSocket support নেই।** WebSocket হলো persistent, two-way connection — client এবং server দুজনেই যেকোনো সময় message পাঠাতে পারে। WSGI-র request-response model-এ এটা সম্ভব না।

**Long-lived connection নেই।** HTTP long polling বা server-sent events যেখানে connection অনেকক্ষণ খোলা রাখতে হয় — WSGI এটা efficiently handle করতে পারে না কারণ connection খোলা থাকলে worker আটকে থাকে।

**Native async নেই।** তুমি Django-তে `async def view()` লিখতে পারো, কিন্তু যদি WSGI server (Gunicorn sync mode) use করো, সেই async code আসলে sync-এর মতোই চলবে। Async-এর সুবিধা পাবে না।

**I/O-heavy load-এ inefficient।** যদি প্রতিটা request external API call করে এবং 200ms wait করে — WSGI-তে সেই 200ms প্রতিটা worker blocked থাকে। 100টা concurrent request handle করতে 100টা worker লাগবে।

---

# 🧠 1.8 Django + WSGI — Real World Picture

Django অনেক আগে থেকেই WSGI-based। তুমি নতুন Django project বানালে `wsgi.py` file দেখবে।

Django এখন async view support করে (`async def`), কিন্তু এখানে একটা গুরুত্বপূর্ণ nuance আছে:

- **WSGI server (Gunicorn default) + async Django view** → async view সত্যিকারের async হিসেবে চলে না। Django internally এটাকে sync thread-এ wrap করে run করে।
- **ASGI server (Uvicorn/Daphne) + async Django view** → সত্যিকারের async, event loop-এ চলে।

মানে Django-তে async-এর পুরো benefit পেতে হলে শুধু `async def` লিখলেই হবে না — server-ও পরিবর্তন করতে হবে। এটা Module 2-এ বিস্তারিত দেখব।

---

# 🧩 1.9 তাহলে WSGI কখন ব্যবহার করবে?

WSGI পুরনো মানেই খারাপ না। অনেক situation-এ এটাই সঠিক choice:

**WSGI ভালো কাজ করে যখন:**
- App টা simple CRUD — database read/write, কোনো real-time feature নেই
- Traffic moderate, concurrent user বেশি না
- Team already Django/Flask-এ comfortable, migration cost justify করে না
- CPU-heavy task আছে (multiprocessing-এর সাথে pair করে)

**WSGI avoid করো যখন:**
- Real-time feature লাগবে (chat, live notification, WebSocket)
- Thousands of concurrent connection handle করতে হবে
- External API call heavy (প্রতিটা request অনেক I/O wait করে)

---

# 🧪 1.10 Performance Reality — সংখ্যায় বোঝো

ধরো তোমার app-এ প্রতিটা request 200ms লাগে (50ms computation + 150ms DB wait)।

**WSGI with 4 workers:**

একসাথে maximum 4টা request handle হবে। 5ম request আসলে কোনো worker free হওয়ার জন্য wait করবে। প্রতি সেকেন্ডে সর্বোচ্চ 4 ÷ 0.2 = **20 requests** handle করতে পারবে।

**ASGI (async) with 1 worker:**

সেই 150ms DB wait-এর সময় অন্য request চলতে পারে। একই machine-এ অনেক বেশি concurrent request handle হবে। প্রতি সেকেন্ডে অনেক বেশি throughput পাবে।

এই পার্থক্যটাই WSGI থেকে ASGI-তে যাওয়ার মূল কারণ।

---

# 🎯 Final Mental Model

WSGI-র পুরো জগৎটাকে এক লাইনে মনে রাখো:

> **"One request, one worker, one journey — শুরু থেকে শেষ পর্যন্ত block।"**

```
request আসে → worker নেয় → শেষ পর্যন্ত আটকে থাকে → response যায় → তারপর পরের request
```

Scaling মানে শুধু এই chain-এ আরও worker যোগ করা। আর এই approach-এর একটা physical limit আছে — memory আর CPU।

পরের module-এ দেখব কীভাবে ASGI এই পুরো model-টা পরিবর্তন করে দিল।

---

# ⚔️ Mini Challenge

ধরো তোমার setup:
- Gunicorn
- 2 sync worker
- প্রতিটা request handle করতে ঠিক 3 সেকেন্ড লাগে

**প্রশ্ন ১:** একসাথে maximum কতটা request handle হতে পারবে?

**প্রশ্ন ২:** 10টা request একসাথে আসলে সব শেষ হতে মোট কত সময় লাগবে?

**প্রশ্ন ৩:** এই system-এর bottleneck কোথায়? কীভাবে fix করবে?

নিজে চিন্তা করো। Hint: worker সংখ্যা এবং request সংখ্যার অনুপাত দিয়ে ভাবো।