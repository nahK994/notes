# ০৩. Process

## Program vs Process

তুমি কি কখনো ভেবেছ — `ls` কি একটা জিনিস, নাকি দুটো?

`/bin/ls` disk-এ পড়ে আছে। নিরীহ একটা file। কিছু করছে না, কাউকে বিরক্ত করছে না। তুমি terminal-এ `ls` টাইপ করার আগ পর্যন্ত সে ঘুমিয়েই থাকে।

তুমি Enter চাপলে কী হয়? সেই ঘুমন্ত file-টা হঠাৎ জীবন্ত হয়ে ওঠে — RAM-এ জায়গা নেয়, CPU-র সময় চায়, screen-এ কথা বলে। এই জীবন্ত সত্তাটাই **Process**।

```
DISK-এ:                    RAM-এ (চলার সময়):
────────                   ──────────────────
/bin/ls                    Process (PID 1234)
  │                          │
  │ একটা file,               │ জীবন্ত, চলছে
  │ ঘুমাচ্ছে                 │ CPU খাচ্ছে
  │ কিছু করছে না             │ memory দখল করেছে
  │
  └── recipe বইয়ের মতো     └── রান্না হচ্ছে মতো
```

**Program** = recipe (disk-এ পড়ে থাকে)
**Process** = সেই recipe অনুযায়ী রান্না (RAM-এ চলছে)

একটা সুন্দর দিক হলো — একই recipe দিয়ে একসাথে অনেক রান্না হতে পারে। তুমি ৫টা terminal-এ `ls` চালালে ৫টা আলাদা process — প্রত্যেকের আলাদা PID, আলাদা memory, কিন্তু একই `/bin/ls` file থেকে জন্ম।

---

## কেন Process লাগে?

Process ছাড়া multitasking সম্ভব না।

Process না থাকলে একটা program চললে অন্যটা চলতে পারত না। তোমাকে firefox বন্ধ করে তারপর music player খুলতে হতো। browser-এ কিছু download হওয়ার সময় পুরো computer আটকে থাকত।

Process-এর কারণে এটা সম্ভব:

```
একই সময়ে:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ firefox  │  │ terminal │  │ music    │
│ PID:1001 │  │ PID:1002 │  │ PID:1003 │
│          │  │          │  │          │
│ নিজের    │  │ নিজের    │  │ নিজের    │
│ memory   │  │ memory   │  │ memory   │
└──────────┘  └──────────┘  └──────────┘
     ↑               ↑             ↑
     └───────────────┴─────────────┘
              CPU share করছে
         (এত দ্রুত switch করে যে
          মনে হয় একসাথে চলছে)
```

প্রতিটা process আলাদা দ্বীপের মতো — নিজের memory, নিজের state। একজনের crash অন্যজনকে সাধারণত টেনে নামায় না। Firefox হ্যাং করলে তোমার music থামে না।

---

## Process-এর ভেতরে কী থাকে?

Process মানে শুধু "চলছে" না। Kernel প্রতিটা process-এর জন্য RAM-এ একটা সাজানো বাড়ি বানিয়ে দেয়। সেই বাড়িতে কয়েকটা আলাদা ঘর:

```
Process (PID: 1234) — RAM-এ যা থাকে:
┌─────────────────────────────────────────┐
│                                         │
│  Code Segment                           │
│  ┌─────────────────────────────────┐    │
│  │ program-এর instructions        │    │
│  │ (machine code, read-only)       │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Stack                          ↕ grows │
│  ┌─────────────────────────────────┐    │
│  │ function call history           │    │
│  │ local variables                 │    │
│  │ main() → runShell() → execute() │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Heap                           ↕ grows │
│  ┌─────────────────────────────────┐    │
│  │ dynamic memory                  │    │
│  │ (তুমি যখন নতুন data তৈরি করো)  │    │
│  └─────────────────────────────────┘    │
│                                         │
│  File Descriptors                       │
│  ┌─────────────────────────────────┐    │
│  │ FD 0 → keyboard                 │    │
│  │ FD 1 → screen                   │    │
│  │ FD 2 → screen (error)           │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Environment Variables                  │
│  ┌─────────────────────────────────┐    │
│  │ PATH=/usr/bin:/bin              │    │
│  │ HOME=/home/shomi                │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

**Code Segment** — program-এর instructions। Read-only, চলার সময় বদলায় না। একই `/bin/ls` থেকে জন্ম নেওয়া ৫টা process এই অংশটা share করতে পারে।

**Stack** — function call হলে এখানে জমা হয়। `main()` → `runShell()` → `execute()` — এই chain এখানে থাকে। function শেষ হলে stack থেকে মুছে যায়। এটা স্বয়ংক্রিয় — তোমাকে কিছু করতে হয় না।

**Heap** — তুমি যখন নতুন data তৈরি করো, যেমন নতুন string বা slice, সেটা এখানে জমা হয়। Stack-এর মতো automatically মুছে যায় না। Go-তে garbage collector এটা manage করে, C-তে তোমাকে নিজেই `free()` করতে হয়।

**File Descriptors** — process বাইরের জগতের সাথে যেভাবে কথা বলে। পরের chapter-এ এটা নিয়ে বিস্তারিত আলোচনা হবে।

**Environment Variables** — `PATH`, `HOME` ইত্যাদি। Child process এগুলো parent থেকে copy করে পায়। তাই তুমি terminal-এ যে `PATH` set করো, সেখান থেকে চালানো সব command সেটা পায়।

---

## Process-এর জীবনচক্র

প্রতিটা process জন্মায়, কাজ করে, মরে। কিন্তু মাঝখানে বেশ কিছু state আছে:

```
                    fork() — জন্ম
                        │
                        ▼
                   ┌─────────┐
                   │ CREATED │ ← সবে তৈরি হলো
                   └────┬────┘
                        │ ready হলো
                        ▼
                   ┌──────────┐
          ┌───────►│ RUNNABLE │ ← চলার জন্য ready
          │        └────┬─────┘   কিন্তু CPU পাচ্ছে না
          │             │ CPU পেলো
          │             ▼
          │        ┌─────────┐
          │        │ RUNNING │ ← এখন সত্যিই চলছে
          │        └────┬────┘
          │             │
          │      ┌──────┴──────┐
          │      │             │
          │      ▼             ▼
          │  ┌─────────┐  ┌────────┐
          └──│ WAITING │  │ ZOMBIE │
             └────┬────┘  └───┬────┘
                  │           │
            I/O শেষে    parent wait()
            RUNNABLE-এ  করলে
            ফিরে যায়        │
                             ▼
                         ┌──────┐
                         │ DEAD │ ← সম্পূর্ণ মুছে গেছে
                         └──────┘
```

**RUNNING** — CPU-তে এই মুহূর্তে সত্যিই চলছে। যেকোনো সময়ে একটা CPU core-এ একটাই process RUNNING হতে পারে।

**RUNNABLE** — চলার জন্য তৈরি, কিন্তু CPU busy। অপেক্ষায় দাঁড়িয়ে আছে। Linux এত দ্রুত switch করে যে আমরা টের পাই না।

**WAITING** — কোনো কিছুর জন্য অপেক্ষা করছে। `ls` যখন disk থেকে file list পড়ে, সেই সময়টায় সে CPU ছেড়ে WAITING-এ চলে যায়। disk পড়া শেষ হলে আবার RUNNABLE। এই ফাঁকে অন্য process CPU পায় — এভাবেই Linux একসাথে হাজার process চালায়।

---

## 🧟 Zombie Process — মৃত কিন্তু তালিকায় আছে

এটা অনেকের কাছে confusing। একটু গল্পের ভঙ্গিতে বলি।

ধরো তোমার বন্ধু বললো, "আমি কাজ শেষ করে তোমাকে result জানাবো।" বন্ধু কাজ শেষ করলো, কিন্তু তুমি এখনো জিজ্ঞেস করোনি। সে technically কাজ শেষ — কিন্তু result হাতে নিয়ে তোমার জন্য দাঁড়িয়ে আছে। না পুরোপুরি বেঁচে, না পুরোপুরি চলে গেছে।

Process-এ ঠিক এটাই হয়:

```
Shell (parent)              ls (child)
──────────────              ──────────
ls চালালো            →     কাজ শুরু করলো
                            কাজ শেষ করলো
                            exit(0) করলো
                              │
                              │ "আমার exit code
                              │  কেউ নিলো না"
                              │
                              ▼
                           ZOMBIE 🧟
                           (RAM-এর বেশিরভাগ
                            free হয়ে গেছে,
                            কিন্তু process table-এ
                            entry আছে exit code সহ)

Shell (parent):
wait() করলো       ←       exit code দিলো
"ls exit code 0"           DEAD — সম্পূর্ণ মুছে গেছে
```

Zombie process নিজে খুব কম resource নেয় — শুধু process table-এ একটা row। কিন্তু অনেক zombie জমলে সেই table ভরে যায়, নতুন process বানানো যায় না। পুরো system নতুন কাজ করতে পারে না।

**Solution সহজ** — parent সবসময় `wait()` করবে। child শেষ হলে তার exit code নেবে, তাহলেই zombie সাথে সাথে মুছে যায়। Milestone 7-এ এটা handle করব।

---

## PID এবং PPID

প্রতিটা process-এর দুটো পরিচয়:

```
PID  = নিজের নম্বর (Process ID)
PPID = বাবার নম্বর (Parent Process ID)

Linux boot হলে:

PID 1: systemd (সবার বাবা)
├── PID 450: login
│   └── PID 890: bash
│       └── PID 1234: tiny-shell  ← আমাদের shell
│           └── PID 1235: ls      ← shell-এর child
│               └── (কাজ করলো, মরে গেলো)
└── PID 512: sshd
```

Linux-এ প্রতিটা process-এর একটা parent আছে। এই গাছটা একদম উপরে গেলে PID 1-এ পৌঁছায় — `systemd`, যে কিনা Linux boot হওয়ার সময় kernel নিজে তৈরি করে। সে কারো child না, সবার বাবা।

নিজে দেখতে চাইলে:

```bash
echo $$          # তোমার shell-এর PID
echo $PPID       # তোমার shell-এর parent PID
pstree -p $$     # পুরো গাছ দেখো
```

`pstree` চালালে দেখবে তুমি নিজেও এই গাছের একটা পাতা — কোথাও না কোথাও `systemd` থেকে তোমার terminal পর্যন্ত একটা সরল রেখা আছে।

---

> **মনে রাখো:** Program হলো blueprint, Process হলো সেই blueprint অনুযায়ী তৈরি জীবন্ত সত্তা। Kernel সেই সত্তার জন্মদাতা, রক্ষক — এবং প্রয়োজনে মৃত্যুদূত।