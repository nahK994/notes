# 🔒 Part 2: GIL, Threading & The Reality of Concurrency

---

## 🧠 1. Why This Part Matters

Part 1-এ তুমি শিখেছো:

```
asyncio → event loop → non-blocking I/O → single thread
```

এখন স্বাভাবিক প্রশ্ন উঠবে:

> তাহলে thread দরকার কেন? 🤔
> আর GIL নামের এই "lock" জিনিসটা কী, এবং কেন exist করে?

### 🔍 এই পার্টে কী শিখবো:

- CPU-bound vs I/O-bound কাজের পার্থক্য
- GIL কী এবং কেন Python-এ এটা আছে
- Thread কখন কাজে আসে, কখন আসে না
- Thread Pool কী এবং কীভাবে ব্যবহার করতে হয়
- Asyncio + Thread Pool একসাথে কীভাবে কাজ করে
- কোন situation-এ কোন tool ব্যবহার করবে

👉 এই পার্টে অনেক **illusion ভাঙবে** — threading সম্পর্কে যা ভাবতে, তার অনেকটাই সত্য না।

---

## ⚙️ 2. CPU-bound vs I/O-bound (Foundation)

এটা বোঝা জরুরি কারণ পুরো threading + asyncio সিদ্ধান্তটাই এই দুটো concept-এর উপর নির্ভর করে।

---

### 🔥 CPU-bound কাজ কী?

CPU-bound মানে হলো এমন কাজ যেখানে **CPU সারাক্ষণ কাজ করছে** — থামার সুযোগ নেই, অপেক্ষা নেই।

```python
def cpu_heavy():
    for i in range(10**8):  # ১০ কোটি iteration
        pass
```

**Characteristics:**
```
- CPU usage high (প্রায় ১০০%)
- কোনো waiting নেই
- pure computation — শুধু হিসাব করছে
```

**বাস্তব উদাহরণ:**
- বড় সংখ্যার গাণিতিক calculation
- Image/video processing
- Machine learning model training
- Data compression/encryption
- JSON/XML parsing (বড় ফাইলের)

---

### 🌐 I/O-bound কাজ কী?

I/O-bound মানে এমন কাজ যেখানে **CPU বেশিরভাগ সময় অপেক্ষা করছে** — কোনো external resource-এর জন্য।

```python
def io_task():
    time.sleep(5)      # অপেক্ষা করছে
    # অথবা:
    requests.get(url)  # network response-এর জন্য অপেক্ষা
    db.query(sql)      # database-এর জন্য অপেক্ষা
    file.read()        # disk-এর জন্য অপেক্ষা
```

**বাস্তব উদাহরণ:**
- Database query
- REST API call
- File read/write
- Network socket
- Cache (Redis) access

---

### 🎯 Core Difference

```
CPU-bound → CPU সময় দরকার (CPU কাজ করছে)
I/O-bound → CPU অপেক্ষা করছে (I/O শেষ হওয়ার জন্য)
```

### 🔍 কেন এই পার্থক্যটা এত গুরুত্বপূর্ণ?

কারণ এই দুই ধরনের কাজে **threading ভিন্নভাবে আচরণ করে**:

```
I/O-bound → threading কাজে আসে ✅
CPU-bound → threading কাজে আসে না, বরং slower হতে পারে ❌
```

কেন? কারণ GIL 😈 — এটা নিয়ে এখনই বিস্তারিত আসছে।

---

## 🔒 3. GIL (Global Interpreter Lock)

GIL হলো Python-এর সবচেয়ে বেশি আলোচিত এবং সবচেয়ে বেশি ভুল বোঝা concept।

---

### 🧠 Simple Definition

> **GIL হলো একটা mutex lock, যেটা ensure করে যে CPython interpreter-এ এক সময়ে মাত্র একটাই thread Python bytecode execute করতে পারবে।**

সহজ ভাষায়: যতগুলো thread-ই থাকুক, Python-এর code একটার বেশি thread একসাথে চালাতে পারবে না।

---

### ❗ গুরুত্বপূর্ণ: এটা শুধু CPython-এ আছে

**CPython** হলো Python-এর standard, official implementation — যেটা আমরা সাধারণত `python` command দিয়ে চালাই। GIL শুধু এখানেই আছে।

```
CPython  → GIL আছে ✅ (সবচেয়ে বেশি ব্যবহৃত)
PyPy     → GIL আছে (তবে ভিন্নভাবে handled)
Jython   → GIL নেই (Java-based Python)
IronPython → GIL নেই (.NET-based Python)
```

> ⚠️ **Note (2024):** Python 3.13 থেকে experimental "free-threaded" (no-GIL) mode আসছে, কিন্তু এখনো production-ready না। তাই এখনো GIL বোঝা জরুরি।

---

### 🔍 GIL কেন আছে? (The Real Story)

Python internals — বিশেষত CPython-এর memory management — **thread-safe না**।

#### সমস্যার মূলে: Reference Counting

Python প্রতিটা object-এর জন্য একটা **reference count** রাখে — কতবার ওই object-টা refer করা হচ্ছে তার সংখ্যা।

```python
a = [1, 2, 3]   # ref count = 1
b = a            # ref count = 2
del a            # ref count = 1
del b            # ref count = 0 → object মুছে যায়
```

এখন দুটো thread যদি একই সময়ে একটা object-এর ref count পরিবর্তন করার চেষ্টা করে:

```
Thread A: ref_count = ref_count + 1  (পড়লো: 5, যোগ করবে)
Thread B: ref_count = ref_count + 1  (পড়লো: 5, যোগ করবে)
Thread A: ref_count = 6              (লিখলো)
Thread B: ref_count = 6              (লিখলো — কিন্তু হওয়া উচিত ছিল 7!)
```

এটাই **race condition** — একটা update হারিয়ে গেছে। এর ফলে:
```
race condition  💥  → ভুল হিসাব
memory corruption 💀 → object আছে কিন্তু মনে করছে নেই, বা উল্টো
```

#### সমাধান হিসেবে GIL:

```
GIL → শুধু ১টা thread Python bytecode execute করতে পারবে
    → ref count race condition হওয়ার সুযোগ নেই
    → memory safe
```

---

### 🧠 GIL Visual

```
Thread A → GIL নিলো → Python code চালাচ্ছে
Thread B → অপেক্ষা করছে (GIL-এর জন্য)
Thread C → অপেক্ষা করছে (GIL-এর জন্য)

Thread A শেষ করলো বা I/O তে গেলো → GIL ছেড়ে দিলো

Thread B → GIL নিলো → Python code চালাচ্ছে
Thread A → অপেক্ষা করছে
Thread C → অপেক্ষা করছে
```

---

### 😂 MEME IDEA

```
"৩টা thread ready to run"
→ "GIL: one at a time please 😌"
```

---

### 🔍 GIL কখন release হয়?

এটা অনেকে জানে না — GIL সবসময় hold করা থাকে না। কিছু নির্দিষ্ট সময়ে release হয়:

```
1. I/O operation (file read, network call, sleep) → GIL release হয়
2. প্রতি N bytecode instruction পর (sys.getswitchinterval() = 5ms default)
3. C extension যদি manually release করে (যেমন NumPy)
```

এই কারণেই I/O-bound কাজে threading কাজ করে — thread A যখন network-এর জন্য অপেক্ষা করছে, তখন GIL ছেড়ে দেয়, এবং thread B চলতে পারে।

---

## ⚡ 4. Threading Reality (No Sugarcoat)

### ❌ Myth (যা সবাই ভাবে)

> "Thread দিলে সব কাজ parallel হয়ে যাবে, program fast হবে।"

### ✅ Reality

```
Python Threading:
    → concurrency (time slicing) — হ্যাঁ
    → true parallelism (CPU-bound) — না, GIL-এর কারণে
```

### 🔁 আসলে কীভাবে কাজ করে:

```
Thread A → GIL নিলো → কিছুক্ষণ চললো → GIL ছাড়লো
Thread B → GIL নিলো → কিছুক্ষণ চললো → GIL ছাড়লো
Thread A → GIL নিলো → ...
```

দেখতে parallel মনে হয়, আসলে দ্রুত switching হচ্ছে।

### 🔍 Analogy:

এটা যেন একটা office-এ একটাই computer আছে। ৩ জন কর্মী আছে। একজন কিছুক্ষণ কাজ করে, তারপর উঠে দেয়, আরেকজন বসে। দেখতে মনে হয় সবাই কাজ করছে, কিন্তু আসলে একটার বেশি একসাথে কাজ করছে না।

---

## 🧠 5. GIL + CPU-bound কাজ = বিপদ

```python
import threading

def cpu_heavy():
    for i in range(10**8):
        pass

# Single thread
cpu_heavy()  # ধরো সময় লাগে ৫ সেকেন্ড

# দুটো thread দিলে কি ২.৫ সেকেন্ড লাগবে?
t1 = threading.Thread(target=cpu_heavy)
t2 = threading.Thread(target=cpu_heavy)
t1.start(); t2.start()
t1.join(); t2.join()
# বাস্তবে: ৫+ সেকেন্ড লাগে — single thread-এর চেয়ে বেশি বা সমান!
```

### 💣 কেন এটা হয়?

```
Thread A → CPU কাজ করছে → GIL hold করছে
Thread B → GIL চাইছে → বারবার চেক করছে (busy waiting)

এই GIL fight এর কারণে:
    → context switching overhead বাড়ে
    → CPU cache miss হয়
    → performance আসলে কমে যায়
```

### 🎯 Conclusion

```
CPU-bound → threading useless, কখনো কখনো আরো slow
CPU-bound → multiprocessing ব্যবহার করো (পরের পার্টে)
```

---

## 🌐 6. GIL + I/O-bound কাজ = ঠিকঠাক

```python
import threading, time

def io_task():
    time.sleep(5)   # ৫ সেকেন্ড অপেক্ষা

# Single thread: ৫ সেকেন্ড
# দুটো thread দিলে?
t1 = threading.Thread(target=io_task)
t2 = threading.Thread(target=io_task)
t1.start(); t2.start()
t1.join(); t2.join()
# বাস্তবে: ~৫ সেকেন্ড! (১০ না)
```

### 🔍 কেন এটা কাজ করে?

```
Thread A → time.sleep(5) শুরু করলো
         → I/O operation → GIL release করলো!
Thread B → GIL পেলো → sleep শুরু করলো
         → দুটোই একসাথে ঘুমাচ্ছে

উভয়ের sleep শেষ হয় ~একই সময়ে → total: ~৫ সেকেন্ড
```

### 🎯 Conclusion

```
I/O-bound → threading কাজে আসে
কারণ: I/O-তে GIL release হয়, অন্য thread চলতে পারে
```

---

## 🧵 7. OS Thread vs Python Thread

### 🧠 Key Truth

Python-এর `threading.Thread` আসলে **OS-level thread** তৈরি করে — কোনো fake বা lightweight thread না।

```
threading.Thread() → OS Thread তৈরি → CPU schedule করে
```

### 🔍 এটা জানা দরকার কেন?

OS thread তৈরি করা মানে:
- Memory খরচ: প্রতিটা thread-এর নিজস্ব stack ~১-৮ MB
- OS-কে জানাতে হয় (system call)
- Context switch করার সময় OS-কে সব register, stack save করতে হয়

এই overhead অনেক। তাই প্রতিটা request-এর জন্য নতুন thread তৈরি করা ব্যয়বহুল।

### ⚠️ Paradox

```
OS thread → theoretically parallel হতে পারে
GIL → Python layer-এ সেই parallelism আটকে দেয় (CPU-bound-এর ক্ষেত্রে)
```

তাই Python-এ thread আছে, OS সেগুলো আলাদা CPU core-এ পাঠাতে পারে, কিন্তু GIL-এর কারণে Python code একসাথে একটাই চলে।

---

## 🧵 8. Thread Pool (Deep Dive)

প্রতিটা request-এর জন্য নতুন thread তৈরি করা ব্যয়বহুল। **Thread Pool** এই সমস্যার সমাধান।

### 🧠 Definition

> Thread Pool হলো আগে থেকে তৈরি করা কিছু worker thread-এর একটা দল, যারা task queue থেকে কাজ নিয়ে execute করে।

### 🔍 Structure

```
Thread Pool:
    [Thread 1]  ← কাজ করছে
    [Thread 2]  ← idle, কাজের জন্য অপেক্ষা করছে
    [Thread 3]  ← idle, কাজের জন্য অপেক্ষা করছে

Task Queue:
    → task_4 (pending)
    → task_5 (pending)
    → task_6 (pending)
```

### 🎯 Flow

```
নতুন task আসে
    ↓
Task Queue-তে ঢোকে
    ↓
কোনো thread idle থাকলে → সে task তুলে নেয় → execute করে
    ↓
শেষ হলে thread আবার idle হয়ে queue-র দিকে তাকায়
```

### 💡 কেন Thread Pool ব্যবহার করবো?

```
প্রতিটা request-এ নতুন thread তৈরি না করে:
    → thread তৈরির overhead বাঁচে (pool-এ আগেই তৈরি)
    → thread reuse হয়
    → max_workers দিয়ে concurrency control করা যায়
    → memory অনুমানযোগ্য (unbounded thread তৈরি হয় না)
```

### 😂 MEME IDEA

```
"প্রতিটা request-এ নতুন employee hire" vs "permanent staff pool থেকে কাজ দাও"
(chaos vs organized office)
```

---

## 🧠 9. Manual Thread vs Thread Pool

### ❌ Manual Thread (request handling-এ ব্যবহার করো না)

```python
import threading

def handle_request(req):
    # কাজ করো

# প্রতিটা request-এ নতুন thread
threading.Thread(target=handle_request, args=(req,)).start()
```

**সমস্যা:**
```
- প্রতিটা thread তৈরিতে OS call → expensive
- কোনো limit নেই → ১০০০ request = ১০০০ thread = memory শেষ
- thread-এর lifecycle manually manage করতে হয়
```

### ✅ Thread Pool (সঠিক পদ্ধতি)

```python
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=10)

# task submit করো
future = executor.submit(handle_request, req)
result = future.result()  # result নাও

# অথবা context manager দিয়ে
with ThreadPoolExecutor(max_workers=10) as executor:
    results = executor.map(handle_request, requests_list)
```

**সুবিধা:**
```
- controlled: সর্বোচ্চ ১০টা thread
- reusable: thread বারবার ব্যবহার হয়
- futures: result সহজে পাওয়া যায়
- auto cleanup: context manager দিয়ে
```

---

## ⚡ 10. Async vs Threading — Decision Table

| Task Type | Best Tool | কারণ |
|---|---|---|
| Non-blocking I/O (async library আছে) | **asyncio** | Thread overhead নেই, হাজারো concurrent |
| Blocking I/O (sync library, পরিবর্তন করা যাচ্ছে না) | **Thread Pool** | GIL I/O-তে release হয়, কাজ করে |
| CPU-bound (pure Python) | **multiprocessing** | আলাদা process = আলাদা GIL = true parallel |
| CPU-bound (NumPy/SciPy) | **Thread Pool** (বিশেষ ক্ষেত্রে) | NumPy নিজে GIL release করে |

### 🔍 Decision Flow:

```
কাজটা কি I/O-bound?
    ├─ হ্যাঁ → async library আছে? → asyncio ব্যবহার করো
    │          async library নেই? → Thread Pool ব্যবহার করো
    └─ না (CPU-bound) → multiprocessing ব্যবহার করো
```

---

## 🧠 11. Offloading — Critical Concept

Asyncio + Thread Pool একসাথে কীভাবে ব্যবহার করবো? এখানেই আসে **offloading**।

### ❗ সমস্যা

```python
async def handler():
    time.sleep(5)  # ⚠️ blocking call async function-এ!
```

এটা পুরো event loop block করে দেবে — Part 1-এ দেখেছিলাম।

কিন্তু কখনো কখনো blocking code ব্যবহার করতেই হয় — যেমন পুরনো library যেটার async version নেই।

### ✅ Fix: `run_in_executor`

```python
import asyncio

async def handler():
    loop = asyncio.get_event_loop()
    # blocking_task-কে thread pool-এ পাঠিয়ে দাও
    result = await loop.run_in_executor(None, blocking_task)
    # None মানে default ThreadPoolExecutor ব্যবহার করো
    return result
```

### 🔍 ধাপে ধাপে কী হয়?

```
1. event loop blocking_task-কে thread pool-এ submit করে
2. একটা thread সেটা execute করতে শুরু করে
3. event loop free — অন্য coroutine চালাতে পারছে
4. thread-এর কাজ শেষ হলে result আসে
5. coroutine resume হয়
```

### 🎯 Insight

```
offload = delegation (দায়িত্ব দেওয়া)
         ≠ speedup (কাজটা faster হয় না)
```

Offloading কাজটাকে faster করে না — কিন্তু event loop-কে free রাখে, যাতে অন্য request handle করতে পারে।

### Custom Executor ব্যবহার:

```python
from concurrent.futures import ThreadPoolExecutor

# নিজের executor দিলে thread count control করা যায়
executor = ThreadPoolExecutor(max_workers=20)

async def handler():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, blocking_task)
    return result
```

---

## 💣 12. Executor Deep Dive — সীমাবদ্ধতা বোঝো

```python
await loop.run_in_executor(None, blocking_task)
```

এটা দেখতে সহজ, কিন্তু ভেতরে কী হচ্ছে সেটা বোঝা দরকার।

### Step-by-step:

```
1. blocking_task → thread pool-এ submit হলো
2. কোনো idle thread আছে? → সে কাজ নিলো
3. event loop অন্য কাজ করছে (free)
4. thread কাজ শেষ করলো → Future-এ result রাখলো
5. event loop-কে notify করলো
6. await → result ফিরে পেলো
```

### ⚠️ গুরুত্বপূর্ণ সীমাবদ্ধতা:

```
Default ThreadPoolExecutor-এ thread count সীমিত
    → Python 3.8+: min(32, os.cpu_count() + 4)
    → সাধারণত ৮-১২টা thread
```

### 💣 Real Danger:

```
scenario: ১০০০ concurrent request, প্রতিটায় blocking_task (৩০ সেকেন্ড)
thread pool size: ১০

১০টা task → ১০টা thread চালু
৯৯০টা task → queue-তে অপেক্ষা

৯৯০তম task-কে অপেক্ষা করতে হবে:
    → ৯৯০ ÷ ১০ = ৯৯ batch × ৩০ সেকেন্ড = ৪৯৫০ সেকেন্ড (~৮২ মিনিট!)
```

### 🎯 Insight:

```
thread pool ≠ infinite scalability
thread pool = bounded concurrency
```

এই কারণেই blocking code-এর জন্য asyncio-র native solution (যেমন `aiohttp`, `asyncpg`) অনেক বেশি scalable।

---

## 🧠 13. Scenario: ১০০০ API Call (Blocking Sleep ৩০s)

এটা একটা concrete example যেটা দিয়ে সব concept একসাথে বোঝা যাবে।

### ❌ Naive approach (blocking):

```python
def handler():
    time.sleep(30)   # blocking!
    return "done"

# ১০০০ request → sequential হলে
for _ in range(1000):
    handler()
# মোট সময়: ১০০০ × ৩০ = ৩০,০০০ সেকেন্ড 💀
```

### ✅ Thread Pool approach:

```python
async def handler():
    loop = asyncio.get_event_loop()
    await loop.run_in_executor(None, time.sleep, 30)
    return "done"
```

```
Flow:
১০০০ request আসলো
→ ১০টা thread চলছে (pool size = 10)
→ ৯৯০টা queue-তে অপেক্ষা করছে

Result:
→ throughput limited by thread count
→ latency: (১০০০ ÷ ১০) × ৩০ = ৩০০০ সেকেন্ড
→ আগের চেয়ে ভালো, কিন্তু আদর্শ না
```

### ✅✅ Best approach (asyncio-native):

```python
async def handler():
    await asyncio.sleep(30)   # non-blocking!
    return "done"
```

```
Flow:
১০০০ request আসলো
→ ১০০০টাই event loop-এ schedule হলো
→ সবাই একসাথে sleep শুরু করলো
→ সবার sleep শেষ হলো ~একই সময়ে

Result:
→ মোট সময়: ~৩০ সেকেন্ড! (১০০০ request-এর জন্য)
→ কোনো thread নেই, কোনো blocking নেই
```

### 🔥 কেন এতটা ভালো?

```
thread pool approach:
    ১০টা thread → ১০টাই একসাথে → বাকিরা অপেক্ষা

asyncio approach:
    ১ টা thread → ১০০০টা coroutine → সবাই concurrently
    (sleep এ GIL release হয়, তাই event loop সবাইকে manage করতে পারে)
```

---

## 🧵 14. Threading-এর Hidden Dangers

Thread ব্যবহার করলে কিছু সমস্যা আসতে পারে যেগুলো জানা দরকার।

### ⚠️ Race Condition

```python
counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1  # এটা thread-safe না!

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()

print(counter)  # 200000 আশা করা হচ্ছে, কিন্তু কম আসবে!
```

`counter += 1` আসলে তিনটা step:
1. counter-এর value পড়ো
2. ১ যোগ করো
3. ফিরে লিখো

দুটো thread এই steps interleave করলে কিছু update হারিয়ে যায়।

**Fix:**
```python
import threading

lock = threading.Lock()
counter = 0

def increment():
    global counter
    for _ in range(100000):
        with lock:       # lock নাও
            counter += 1 # safe now
                         # lock automatically ছাড়ে
```

### ⚠️ Deadlock

```python
lock_a = threading.Lock()
lock_b = threading.Lock()

def task_1():
    with lock_a:
        time.sleep(0.1)
        with lock_b:  # lock_b-এর জন্য অপেক্ষা করছে
            pass

def task_2():
    with lock_b:
        time.sleep(0.1)
        with lock_a:  # lock_a-এর জন্য অপেক্ষা করছে
            pass
```

```
task_1 → lock_a নিলো → lock_b চাইছে
task_2 → lock_b নিলো → lock_a চাইছে
→ দুজনেই অপেক্ষা করছে → deadlock 💀
```

### 🎯 কেন asyncio-তে এই সমস্যা নেই?

Asyncio single-threaded, তাই race condition নেই। Coroutine শুধু `await`-এ switch করে — তাই কোথায় switch হবে সেটা predictable।

---

## 🧠 15. Go Comparison (Quick)

Python threading-এর সীমাবদ্ধতা বোঝার জন্য Go-এর সাথে তুলনাটা helpful।

### Go-তে Goroutine:

```go
go func() {
    time.Sleep(5 * time.Second)
}()
```

```
Goroutine:
    → Go runtime manage করে (OS thread না)
    → অনেক lightweight (~কয়েক KB)
    → হাজারো goroutine চালানো যায়
    → GIL নেই
    → true concurrent execution (GOMAXPROCS দিয়ে)
```

### Python Thread:

```
Python Thread:
    → OS thread (heavy, ~১-৮ MB)
    → GIL এর কারণে limited
    → কয়েকশো thread চালানো practical
```

### 🎯 Insight:

```
Go  → concurrency first-class citizen, built-in
Python → workaround দিয়ে সমাধান করে (async + process)
```

Python-এ asyncio অনেকটা goroutine-এর মতো কাজ করে — lightweight, single-threaded, cooperative। কিন্তু CPU parallelism-এর জন্য Python-কে multiprocessing-এ যেতে হয়।

### 😂 MEME IDEA

```
"Go: চাইলে ১০ লক্ষ goroutine spin করো"
"Python: thread নিয়ে সাবধান থেকো 😅"
```

---

## 🧠 16. `threading` Module — Quick Reference

কিছু useful class/function যেগুলো কাজে লাগে:

```python
import threading

# Basic Thread
t = threading.Thread(target=func, args=(arg1,), daemon=True)
t.start()
t.join()   # thread শেষ হওয়া পর্যন্ত অপেক্ষা

# Lock — mutual exclusion
lock = threading.Lock()
with lock:
    # critical section

# Event — thread-এর মধ্যে signal পাঠানো
event = threading.Event()
event.set()    # signal পাঠাও
event.wait()   # signal-এর জন্য অপেক্ষা করো
event.clear()  # reset করো

# Semaphore — একসাথে সর্বোচ্চ N thread
sem = threading.Semaphore(5)
with sem:
    # সর্বোচ্চ ৫টা thread এখানে থাকতে পারবে
```

**`daemon=True` কী?**
Daemon thread মানে — main program শেষ হলে এই thread-ও জোর করে বন্ধ হয়ে যাবে, জয়েন করা লাগবে না। Background task-এর জন্য ভালো।

---

## 🧠 17. Final Mental Model

```
Event Loop (asyncio):
    → non-blocking I/O-এর জন্য
    → single thread, হাজারো coroutine
    → best scalability

Thread Pool:
    → blocking I/O-এর জন্য (পুরনো library, sync code)
    → limited concurrency (thread count-bounded)
    → offloading দিয়ে asyncio-র সাথে combine করা যায়

GIL:
    → CPython-এ ১টা thread Python bytecode execute করে
    → I/O-তে release হয় (threading কাজ করে)
    → CPU-bound-এ release হয় না (threading ব্যর্থ)

Multiprocessing:
    → CPU-bound কাজের জন্য
    → আলাদা process = আলাদা GIL = true parallelism
    → (বিস্তারিত পরের পার্টে)
```

---

## 🧪 Self-check Questions — উত্তরসহ

### 1. কেন CPU-bound কাজে threading slow?

কারণ CPU-bound কাজে GIL release হয় না। দুটো thread একসাথে Python bytecode execute করতে পারে না। একটা চলে, অন্যটা GIL-এর জন্য বসে থাকে। এর উপর context switching overhead যোগ হয় — ফলে single thread-এর চেয়ে সমান বা বেশি সময় লাগে।

### 2. কেন I/O-bound কাজে threading কাজ করে?

কারণ I/O operation (sleep, network call, file read) শুরু হলে GIL release হয়। Thread A যখন network response-এর জন্য অপেক্ষা করছে, Thread B GIL নিয়ে চলতে পারে। এই overlap-এর কারণে total time কমে।

### 3. `run_in_executor` কী solve করে?

এটা blocking code-কে thread pool-এ পাঠিয়ে দেয়, যাতে asyncio event loop block না হয়। Event loop free থাকে অন্য coroutine চালাতে। Blocking code-কে async context-এ ব্যবহার করার সেতু।

### 4. Thread pool কেন limited?

Thread pool-এ thread সংখ্যা fixed (যেমন ১০টা)। ১০০০ task দিলে ১০টা চলে, ৯৯০টা queue-তে অপেক্ষা করে। Thread বাড়ালে memory ও context switching overhead বাড়ে। এটাই asyncio-র চেয়ে কম scalable।

### 5. Race condition কী এবং কীভাবে এড়াবো?

একাধিক thread একই shared data একই সময়ে পরিবর্তন করার চেষ্টা করলে race condition হয় — কিছু update হারিয়ে যায়। `threading.Lock()` দিয়ে critical section protect করে এড়ানো যায়।

---

## 🎯 Summary

```
Threading:
    → CPU parallelism দেয় না (GIL-এর কারণে)
    → I/O-bound কাজে useful

GIL:
    → CPython interpreter protect করে (reference counting)
    → I/O-তে release হয়
    → CPU-bound-এ thread useless করে দেয়

Asyncio:
    → I/O-bound-এর জন্য সবচেয়ে scalable
    → কোনো thread overhead নেই

Thread Pool:
    → blocking code-এর জন্য fallback
    → asyncio-র সাথে run_in_executor দিয়ে combine করা যায়
    → কিন্তু thread count-bounded
```
