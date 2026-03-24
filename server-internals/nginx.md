# 🌐 Module 6: Nginx — Gatekeeper, Traffic Controller, এবং Shield

---

## Module 5-এর Boss Fight Challenge-এর উত্তর

আগের challenge ছিল:

> 1 Gunicorn instance, 2 worker, 500 concurrent user।
> প্রতিটা request: PostgreSQL query (100ms async) + payment API call (400ms async) + মাঝে মাঝে PDF generation (200ms CPU)।

**প্রশ্ন ১ — কোন worker?**

UvicornWorker। কারণ workload-এর বেশিরভাগ I/O-bound — DB এবং payment API-এ মোট 500ms wait। Async দিয়ে এই wait-এর সময় অন্য request চলতে পারে।

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 2 myapp:app
```

**প্রশ্ন ২ — PDF generation কীভাবে handle করবে?**

PDF generation CPU-heavy। Async handler-এ সরাসরি করলে event loop সেই 200ms block থাকবে। সমাধান হলো separate process pool-এ offload:

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

pool = ProcessPoolExecutor(max_workers=4)

@app.post("/invoice")
async def generate_invoice(data: InvoiceData):
    payment = await payment_api.charge(data)         # async, ঠিক আছে
    db_record = await db.save(payment)               # async, ঠিক আছে
    loop = asyncio.get_event_loop()
    pdf = await loop.run_in_executor(pool, build_pdf, db_record)  # CPU কাজ আলাদা process-এ
    return pdf
```

**প্রশ্ন ৩ — Bottleneck কোথায়?**

তিনটা জায়গায়:
- Payment API (400ms) সবচেয়ে slow — এটা external, control নেই
- PDF generation CPU block করতে পারে যদি offload না করা হয়
- মাত্র 2 worker — CPU-bound peak-এ কম পড়তে পারে

**প্রশ্ন ৪ — 2000 concurrent user-এ scale করতে হলে?**

- Worker বাড়াও: `-w 4` (CPU core অনুযায়ী)
- Multiple server instance + load balancer (Nginx)
- Payment API-এর response cache করো (Redis) যেখানে সম্ভব
- PDF generation আলাদা microservice বা queue-এ নিয়ে যাও

এই শেষ পয়েন্টটাই — multiple server + load balancer — আজকের module-এর বিষয়। শুধু একটা Gunicorn দিয়ে কতটুকু যাওয়া যাবে? কখন Nginx দরকার? কেন?

---

## কেন Nginx শিখছি?

Development-এ `uvicorn myapp:app` বা `gunicorn myapp:app` চালালেই কাজ চলে। কিন্তু production-এ শুধু Gunicorn রাখলে অনেক সমস্যা হয়। Gunicorn একটা Python process — সে connection buffering করতে পারে না ভালোভাবে, SSL handle করতে পারে না, static file serve করতে পারে না efficiently। এসব কাজের জন্য দরকার একটা dedicated, high-performance server যেটা Gunicorn-এর সামনে দাঁড়িয়ে থাকবে।

সেটাই Nginx।

---

# 🧠 6.1 Nginx কী — এবং এটা কী না

Nginx (উচ্চারণ: "engine-x") একটা C-তে লেখা web server। এটা তৈরি হয়েছিল ২০০৪ সালে একটা নির্দিষ্ট সমস্যা সমাধান করতে — "C10K problem।" মানে: একটা server-এ একসাথে ১০,০০০ connection handle করা।

তখনকার Apache server thread-based ছিল — প্রতিটা connection-এ একটা thread। ১০,০০০ connection মানে ১০,০০০ thread, যেটা impractical। Nginx event-driven architecture দিয়ে তৈরি হলো — অনেকটা Python-এর asyncio-র মতো, কিন্তু C-তে, অনেক বেশি efficient।

**Nginx কী কী করতে পারে:**
- Web server (HTML, CSS, JS, images directly serve করা)
- Reverse proxy (অন্য server-এ request forward করা)
- Load balancer (multiple server-এ traffic ভাগ করা)
- SSL termination (HTTPS handle করা)
- Rate limiting
- Request caching

**Nginx কী করতে পারে না:**
- Python code execute করা
- Application logic বোঝা
- Database query করা

সহজ ভাষায়: Nginx network-এর কাজ করে, Gunicorn Python-এর কাজ করে।

---

# ⚙️ 6.2 Reverse Proxy — সবচেয়ে গুরুত্বপূর্ণ Role

Production-এ Nginx সবচেয়ে বেশি ব্যবহার হয় reverse proxy হিসেবে। এটা বোঝাটা জরুরি।

**Forward proxy বনাম Reverse proxy:**

Forward proxy client-এর হয়ে কাজ করে। তুমি যখন VPN বা office network proxy use করো — সেটা forward proxy। Client → Proxy → Internet।

Reverse proxy server-এর হয়ে কাজ করে। Client জানেও না পেছনে কতগুলো server আছে। Client → Nginx → (Gunicorn 1, Gunicorn 2, ...)।

**কেন reverse proxy দরকার:**

ধরো তোমার app চলছে `localhost:8000`-এ। তুমি সরাসরি এই port internet-এ expose করলে:
- Gunicorn সব connection নিজে handle করবে — Python-এ, slow
- Slow client (যেমন 3G-তে user) connection ধরে রাখবে, Gunicorn worker blocked থাকবে
- SSL নেই, HTTPS নেই
- কোনো protection নেই

Nginx সামনে থাকলে:
- Nginx (C, event-driven) সব TCP connection নেয়
- Request পুরো আসার পর Gunicorn-এ forward করে (buffering)
- Gunicorn শুধু complete request পায়, slow network-এর সাথে লড়াই করে না
- Nginx SSL terminate করে — Gunicorn plain HTTP পায়

---

# 🔄 6.3 একটা Request কীভাবে Journey করে

পুরো flow টা step-by-step বোঝা দরকার। একটু বিস্তারিত দেখি।

**Step 1 — Browser TCP connection করে Nginx-এ:**

User `https://example.com/api/users` type করলে browser port 443-এ TCP connection করে। এই connection Nginx নেয়। Gunicorn এখনও জানে না কিছু।

**Step 2 — Nginx SSL terminate করে:**

HTTPS connection মানে data encrypted। Nginx SSL certificate ব্যবহার করে decrypt করে। এখন থেকে Nginx এবং Gunicorn-এর মধ্যে plain HTTP চলে (internal network, তাই safe)।

**Step 3 — Nginx request buffer করে:**

Client ধীরে ধীরে request body পাঠাচ্ছে (slow network)। Nginx পুরো body আসা পর্যন্ত buffer করে। তারপর একটা পরিপূর্ণ HTTP request Gunicorn-এ forward করে।

**Step 4 — Nginx request route করে:**

`/static/` path? সরাসরি disk থেকে serve, Gunicorn-এ যাবেই না।
`/api/` path? Gunicorn-এ forward।

**Step 5 — Gunicorn request process করে:**

Gunicorn worker Python code চালায়, database query করে, response তৈরি করে।

**Step 6 — Response ফিরে যায়:**

Gunicorn → Nginx → Client। Nginx response-ও buffer করতে পারে এবং client-এর speed অনুযায়ী পাঠায়।

এই পুরো process-এ Gunicorn শুধু step 5-এ জড়িত। বাকি সব Nginx করছে।

---

# 📦 6.4 Static File Serving — Nginx-এর সহজ কিন্তু বড় সুবিধা

তোমার Django বা FastAPI app-এ CSS, JavaScript, images আছে। এগুলো serve করার দুটো উপায়:

**উপায় ১ — Python দিয়ে (খারাপ):**

```python
# Django settings.py
STATIC_ROOT = '/var/www/static/'
# তারপর Django-ই static file serve করে
```

প্রতিটা CSS বা image request Django Python process-এর মধ্যে দিয়ে যাবে। Python file পড়বে, HTTP response বানাবে। এটা অনেক slow এবং unnecessarily Gunicorn worker ব্যবহার করে।

**উপায় ২ — Nginx দিয়ে (ভালো):**

```nginx
location /static/ {
    root /var/www/app;
    expires 30d;
    add_header Cache-Control "public";
}
```

Nginx সরাসরি disk থেকে file পড়ে client-এ পাঠায়। Python involve হয় না। C-তে লেখা, অনেক দ্রুত। একটা CSS file serve করতে Gunicorn worker জড়িত হওয়া সম্পূর্ণ অপ্রয়োজনীয় — Nginx এটা অনেক বেশি efficiently করতে পারে।

---

# 🔒 6.5 Security — Nginx-এর Shield Role

Gunicorn সরাসরি internet-এ expose করা unsafe। কেন?

**Gunicorn production-ready security নেই:**
- DDoS mitigation নেই
- Rate limiting নেই
- Malformed HTTP request ভালোভাবে handle করে না
- Request size limit নেই by default

**Nginx এসব handle করে:**

```nginx
# প্রতি IP থেকে সেকেন্ডে ১০টার বেশি request না
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req zone=api burst=20 nodelay;

# বড় request body block করো
client_max_body_size 10m;

# Timeout set করো
proxy_connect_timeout 10s;
proxy_read_timeout 30s;
```

Nginx-এ এই rules দিলে Gunicorn পর্যন্ত malicious বা excessive request পৌঁছায় না। Gunicorn শুধু legitimate request handle করে।

---

# ⚖️ 6.6 Load Balancing — Traffic ভাগ করো Multiple Server-এ

একটা Gunicorn instance-এ maximum কতটুকু load নেওয়া যাবে তার একটা limit আছে। Traffic বাড়লে multiple server instance চালাতে হয়। Nginx এগুলোর মধ্যে traffic ভাগ করে দেয়।

```nginx
upstream gunicorn_servers {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

server {
    location / {
        proxy_pass http://gunicorn_servers;
    }
}
```

এই config দিলে Nginx automatically চারটা Gunicorn instance-এ request ভাগ করবে।

**Load balancing strategies:**

Nginx-এর default হলো round-robin — একটার পর একটা server-এ পাঠায়। কিন্তু আরও কয়েকটা strategy আছে:

```nginx
upstream backend {
    least_conn;     # সবচেয়ে কম active connection যার কাছে আছে তাকে দাও
    # অথবা
    ip_hash;        # একই client সবসময় একই server-এ যাবে (session affinity)
    # অথবা
    least_time header;  # সবচেয়ে কম response time যার, তাকে দাও

    server 127.0.0.1:8000 weight=3;  # এই server বেশি powerful, বেশি traffic দাও
    server 127.0.0.1:8001 weight=1;
    server 127.0.0.1:8002 backup;    # শুধু অন্যরা down হলে ব্যবহার হবে
}
```

**Health check:**

```nginx
upstream backend {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;

    # একটা server fail করলে automatically বাদ দাও
    server 127.0.0.1:8002 max_fails=3 fail_timeout=30s;
}
```

কোনো Gunicorn instance crash করলে Nginx automatically সেটাকে rotation থেকে বের করে দেয় এবং অন্যগুলোতে traffic পাঠায়।

---

# ⚙️ 6.7 Nginx Configuration — বিস্তারিত

Nginx config দেখতে ভয় লাগে, কিন্তু structure বুঝলে সহজ।

```nginx
# /etc/nginx/sites-available/myapp

# HTTP থেকে HTTPS-এ redirect করো
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

# Main server block
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL certificate (Let's Encrypt সাধারণত এখানে থাকে)
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Static files — Python involve হবে না
    location /static/ {
        root /var/www/myapp;
        expires 30d;
        gzip on;
        gzip_types text/css application/javascript;
    }

    # Media files (user uploads)
    location /media/ {
        root /var/www/myapp;
    }

    # বাকি সব Gunicorn-এ যাবে
    location / {
        proxy_pass http://127.0.0.1:8000;

        # Important headers — Gunicorn-এ real client info পাঠাও
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 60s;

        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
}
```

**`proxy_set_header` কেন দরকার:**

Nginx request forward করলে Gunicorn দেখে request আসছে `127.0.0.1` থেকে (Nginx-এর IP)। Real client-এর IP জানা যাচ্ছে না। `X-Real-IP` এবং `X-Forwarded-For` headers দিলে Gunicorn real client IP পায়। Django-তে `REMOTE_ADDR` বা FastAPI-তে `request.client.host` সঠিক IP দেবে।

---

# 🆚 6.8 Nginx বনাম Gunicorn — কে কী করে

এই দুটো compete করে না, এরা দুটো আলাদা layer-এ কাজ করে।

| কাজ | Nginx | Gunicorn |
|-----|-------|----------|
| TCP connection accept করা | ✅ হাজারো | ⚠️ সীমিত |
| SSL/TLS terminate করা | ✅ | ❌ |
| Static file serve করা | ✅ milliseconds-এ | ❌ (বা অনেক slow) |
| Request routing (`/api` vs `/static`) | ✅ | ❌ |
| Rate limiting | ✅ built-in | ❌ |
| Load balancing | ✅ | সীমিত |
| Python code চালানো | ❌ | ✅ |
| Django/FastAPI app চালানো | ❌ | ✅ |
| Database query করা | ❌ | ✅ (app-এর মাধ্যমে) |

Analogy: Nginx হলো airport-এর security, immigration, এবং boarding gate system। Gunicorn হলো airplane-এর crew। Security জানে কে ঢুকতে পারবে, কোন gate-এ যাবে — কিন্তু উড়তে জানে না। Crew উড়তে জানে — কিন্তু সে পুরো airport manage করতে পারে না।

---

# 🏗️ 6.9 Real Architectures — Small থেকে Large

**Architecture 1 — Development (Nginx নেই):**

```
Browser → Gunicorn:8000 → Django/FastAPI
```

Development-এ এটাই যথেষ্ট। Traffic নেই, SSL লাগে না, static file performance matter করে না।

---

**Architecture 2 — Small Production (Single Server):**

```
Browser → Nginx:443 → Gunicorn:8000 → App
               ↓
          /static/ files (disk থেকে সরাসরি)
```

একটাই server-এ সব। ছোট app-এর জন্য এটাই standard।

```bash
# Gunicorn চালাও (background-এ)
gunicorn -k uvicorn.workers.UvicornWorker -w 4 --bind 127.0.0.1:8000 myapp:app

# Nginx config করো এবং reload করো
sudo nginx -t  # config test
sudo nginx -s reload
```

---

**Architecture 3 — Medium Production (Multiple Workers):**

```
Browser → Nginx:443 → upstream {
                           Gunicorn:8000 (4 workers)
                           Gunicorn:8001 (4 workers)
                           Gunicorn:8002 (4 workers)
                       }
```

Multiple Gunicorn instance, Nginx load balance করছে। একটা instance restart দিতে হলে বাকিরা চলতে থাকে।

---

**Architecture 4 — Large Scale (Multiple Servers):**

```
Internet → CDN (static assets globally cached)
               ↓
         Load Balancer (cloud provider বা dedicated Nginx)
               ↓
    ┌──────────┬──────────┬──────────┐
  Server 1  Server 2  Server 3  Server 4
  (Nginx +  (Nginx +  (Nginx +  (Nginx +
  Gunicorn) Gunicorn) Gunicorn) Gunicorn)
               ↓
         Database Cluster
         Cache (Redis)
         Message Queue
```

এই scale-এ প্রতিটা server-এ নিজস্ব Nginx আছে (local load balance) এবং সামনে আরেকটা global load balancer আছে।

---

# ⚠️ 6.10 সবচেয়ে বেশি হওয়া Mistakes

**Mistake 1 — Gunicorn সরাসরি internet-এ expose করা:**

```bash
# ❌ এটা করবে না production-এ
gunicorn --bind 0.0.0.0:8000 myapp:app
```

Gunicorn সব interface-এ (0.0.0.0) listen করলে সরাসরি internet থেকে accessible। SSL নেই, rate limiting নেই, buffering নেই।

```bash
# ✅ এটা করো — শুধু localhost-এ listen করো
gunicorn --bind 127.0.0.1:8000 myapp:app
# Nginx সামনে থেকে 443-এ listen করবে
```

**Mistake 2 — Static file Python দিয়ে serve করা:**

```python
# ❌ Django development server static serve করে, production-এ এটা দিও না
python manage.py runserver  # production-এ কখনো না
```

```nginx
# ✅ Nginx-এ static config করো
location /static/ {
    root /var/www/myapp;
}
```

**Mistake 3 — `proxy_set_header` না দেওয়া:**

```nginx
# ❌ Real IP হারিয়ে যাবে
location / {
    proxy_pass http://127.0.0.1:8000;
}

# ✅ Headers পাঠাও
location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**Mistake 4 — Timeout না দেওয়া:**

Default timeout-এ Nginx অনেকক্ষণ অপেক্ষা করে। Gunicorn crash করলে বা stuck হলে user অনেকক্ষণ pending দেখবে। Timeout দিলে fast fail হয়।

---

# 🎯 Final Mental Model

System-টাকে layer হিসেবে ভাবো। প্রতিটা layer তার নিজের দায়িত্ব পালন করে:

```
Internet
   ↓
Nginx (Network layer)
   — TCP connection management
   — SSL termination
   — Static files
   — Rate limiting
   — Request routing
   — Load balancing
   ↓
Gunicorn (Process management layer)
   — Worker process management
   — Worker restart on crash
   — Graceful reload
   ↓
Worker (Execution layer)
   — Python code execution
   — Async event loop
   ↓
App (Business logic layer)
   — Route handling
   — Database queries
   — Business logic
```

এই layered architecture-এর সুবিধা হলো প্রতিটা layer independently scale এবং optimize করা যায়।

---

# ⚔️ Mini Challenge

তোমার কাছে একটা production setup আছে:

```
Internet → Nginx → 3টা Gunicorn instance (প্রতিটায় 2 async worker)
                         ↓
                    PostgreSQL database
```

Traffic: 1000 concurrent user।

**প্রশ্ন ১:** Nginx কোন load balancing strategy ব্যবহার করবে এবং কেন?

**প্রশ্ন ২:** একটা Gunicorn instance crash করলে কী হবে? Nginx কীভাবে handle করবে?

**প্রশ্ন ৩:** Nginx না থাকলে এই 1000 concurrent user-এর কী সমস্যা হতো?

**প্রশ্ন ৪:** Static files কে serve করবে এই setup-এ? কোথায় config করতে হবে?

**প্রশ্ন ৫:** তুমি যদি দেখো Gunicorn instance গুলো সব ঠিক আছে কিন্তু response slow — bottleneck কোথায় খুঁজবে?