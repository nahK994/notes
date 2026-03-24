🔍 Module 9: Request Lifecycle Deep Dive
🎬 Scenario A (sync)

Client
→ Nginx
→ Gunicorn(sync worker)
→ Django
→ DB
→ Response

⏱️ blocking chain

🎬 Scenario B (async)

Client
→ Nginx
→ UvicornWorker
→ FastAPI
→ async DB / HTTP
→ Response

⏱️ interleaved execution

🎬 Scenario C (mixed)

Client
→ Nginx
→ Gunicorn + UvicornWorker
→ FastAPI
→ blocking lib → thread pool
→ Response