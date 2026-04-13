## Foreground এবং Background: একসাথে অনেক কাজ

একটা পরিচিত দৃশ্য।

তুমি terminal-এ একটা বড় file download করছো। Command দিলে, cursor থেমে গেলো, progress দেখাচ্ছে — কিন্তু আর কিছু করতে পারছো না। Terminal যেন জমে গেছে। Download শেষ না হওয়া পর্যন্ত অপেক্ষা করা ছাড়া উপায় নেই।

এটা **foreground** task।

এখন একই command-এর শেষে `&` লাগাও। Download শুরু হলো, কিন্তু prompt ফিরে এলো। তুমি অন্য কাজ করতে পারছো — file edit করো, আরেকটা command চালাও, যা খুশি করো। Download পেছনে চলছে।

এটা **background** task।

একটা `&` চিহ্ন। এটুকুই পার্থক্য।

---

### Foreground Task আসলে কী?

সাধারণত Shell একটা command চালালে সে `wait()` করে বসে থাকে — child process শেষ না হওয়া পর্যন্ত নতুন prompt দেয় না।

```
তুমি: "sleep 10" লিখলে

Shell
  │
  ├─ fork() → sleep-child জন্মালো
  │
  └─ wait() ← এখানে আটকে আছে
       │
       │  10 সেকেন্ড...
       │
       │  sleep শেষ হলো
       ▼
  prompt ফিরে এলো
```

এই সময়টায় Shell ঘুমিয়ে। তুমি কিছু টাইপ করলেও সে শুনছে না। Child শেষ না হওয়া পর্যন্ত সে উঠবে না।

এটাই foreground-এর মূল কথা — **Shell অপেক্ষা করে।**

---

### Background Task আসলে কী?

`sleep 10 &` লিখলে Shell একটাই কাজ এড়িয়ে যায় — `wait()` করে না।

```
তুমি: "sleep 10 &" লিখলে

Shell
  │
  ├─ fork() → sleep-child জন্মালো
  │
  ├─ [1] 1234 ← job number আর PID দেখালো
  │
  └─ wait() করলো না, সাথে সাথে prompt দিলো

তুমি: অন্য কাজ করো
sleep: পেছনে চলছে...
```

Child process চলছে। Shell-ও চলছে। দুজন একসাথে, কেউ কাউকে আটকাচ্ছে না।

কিন্তু এখানে একটা সমস্যা আছে।

`wait()` না করলে child শেষ হলে zombie হয়ে যাবে। Shell তাই সম্পূর্ণ `wait()` এড়ায় না — সে মাঝে মাঝে উঁকি দিয়ে দেখে কোনো background child শেষ হয়েছে কিনা। শেষ হলে তার exit code নিয়ে নেয়, zombie সাফ করে।

তুমি পরের command দিলে বা Enter চাপলে Shell সেই মুহূর্তে এই কাজটা করে:

```
$ [Enter চাপলে]

[1]+ Done    sleep 10
$
```

---

### Job: Shell-এর হিসাবের খাতা

Background-এ task পাঠালে Shell একটা হিসাব রাখে। এই হিসাবের প্রতিটা entry-কে বলে **Job**।

```bash
$ sleep 100 &
[1] 1234

$ sleep 200 &
[2] 1235

$ sleep 300 &
[3] 1236
```

`[1]`, `[2]`, `[3]` হলো job number। `1234`, `1235`, `1236` হলো PID।

`jobs` command দিলে সব দেখা যায়:

```bash
$ jobs
[1]   Running    sleep 100 &
[2]   Running    sleep 200 &
[3]-  Running    sleep 300 &
```

Shell এই তালিকাটা নিজে maintain করে। প্রতিটা job-এর PID, status, command — সব মনে রাখে।

---

### Foreground থেকে Background, Background থেকে Foreground

Job গুলো fixed না। চলতে চলতে বদলানো যায়।

**Foreground task-কে background-এ পাঠাও:**

ধরো একটা command চালিয়েছো, কিন্তু ভুলে `&` দাওনি। এখন terminal আটকে আছে।

```
Ctrl+Z চাপো:

^Z
[1]+ Stopped    sleep 100
$
```

`Ctrl+Z` মানে — *"এই process-কে থামাও, আমাকে terminal ফিরিয়ে দাও।"* Process মরে যায় না, **pause** হয়ে যায়। তুমি terminal ফিরে পাও।

এখন `bg` দিয়ে সেটাকে background-এ চালু করো:

```bash
$ bg %1
[1]+ sleep 100 &
```

Process আবার চলছে — এবার background-এ।

**Background task-কে foreground-এ আনো:**

```bash
$ fg %1
sleep 100
```

এখন `sleep 100` foreground-এ। Terminal আবার আটকে গেলো — Shell আবার `wait()` করছে।

```
foreground ←──────────────────────── background
                fg %[job]

foreground ──── Ctrl+Z ──── bg %[job] ──► background
```

---

### Ctrl+C আর Ctrl+Z-এর পার্থক্য

এই দুটো একই মনে হয়, কিন্তু সম্পূর্ণ আলাদা:

```
Ctrl+C → process-কে মেরে ফেলো (SIGINT signal)
          process চিরতরে শেষ

Ctrl+Z → process-কে ঘুম পাড়াও (SIGTSTP signal)
          process থামলো, মরলো না
          পরে fg বা bg দিয়ে তুলতে পারবে
```

একটা উদাহরণ:

```
$ vim important_file.txt
[vim চলছে, তুমি কিছু edit করেছো]

Ctrl+Z চাপলে:
^Z
[1]+ Stopped    vim important_file.txt
$

[তুমি অন্য কাজ করলে]

$ fg %1
vim important_file.txt
[vim ফিরে এলো, সব edit ঠিক আছে]
```

Vim-কে মারলে সব হারিয়ে যেতো। Ctrl+Z দিয়ে pause করলে পরে ঠিক যেখানে ছিলে সেখানে ফিরে পাবে।

---

### Signal: Shell কীভাবে Process-এর সাথে কথা বলে

Ctrl+C, Ctrl+Z — এগুলো আসলে **Signal**। OS-এর একটা notification system।

Shell এই signal গুলো foreground process-এ পাঠায়:

```
তুমি Ctrl+C চাপলে:

Keyboard
  │
  │ SIGINT signal
  ▼
Shell
  │
  │ SIGINT → foreground process-এ পাঠালো
  ▼
Process পেলো → default behavior: মরে গেলো
```

কিছু সাধারণ signal:

```
SIGINT  (Ctrl+C) → থামো, বন্ধ হয়ে যাও
SIGTSTP (Ctrl+Z) → ঘুমাও, pause হও
SIGKILL          → এখনই মরো, কোনো কথা নেই
SIGTERM          → মরার আগে cleanup করার সুযোগ পাও
SIGCONT          → ঘুম থেকে ওঠো, চালু হও
```

`bg` command দিলে Shell আসলে paused process-এ `SIGCONT` signal পাঠায় — *"ওঠো, background-এ চালু হও।"*

---

### Kill: জোর করে থামানো

কোনো process যদি Ctrl+C-তে না মরে — হ্যাং করে গেছে, respond করছে না — তখন:

```bash
$ jobs
[1]  Running    some-stuck-process &

$ kill %1          # SIGTERM পাঠাও, ভদ্রভাবে বলো মরতে
$ kill -9 %1       # SIGKILL পাঠাও, জোর করে মারো
```

`SIGTERM` পেলে process নিজে cleanup করে মরতে পারে। `SIGKILL` পেলে Kernel জোর করে মেরে দেয় — process কিছু করার সুযোগ পায় না।

PID দিয়েও kill করা যায়:

```bash
$ kill 1234
$ kill -9 1234
```

---

### পুরো ছবিটা একসাথে

```
┌─────────────────────────────────────────────────────┐
│                      SHELL                          │
│                                                     │
│  Foreground:          Background Jobs:              │
│  ┌──────────┐         ┌─────────────────────────┐  │
│  │ running  │         │ [1] sleep 100  Running  │  │
│  │ process  │         │ [2] wget ...   Running  │  │
│  │          │         │ [3] vim        Stopped  │  │
│  └──────────┘         └─────────────────────────┘  │
│       │                          │                  │
│  Ctrl+Z → Stopped           fg %N → foreground      │
│  Ctrl+C → Killed            bg %N → background      │
│                             kill %N → signal        │
└─────────────────────────────────────────────────────┘
```

---

Foreground আর background-এর পার্থক্য আসলে একটাই — Shell `wait()` করে কিনা।

করলে foreground। না করলে background।

এই একটা সিদ্ধান্তের উপর দাঁড়িয়ে আছে terminal-এ multitasking-এর পুরো দুনিয়া।