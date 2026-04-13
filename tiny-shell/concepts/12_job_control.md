# Process Group, Job Control এবং Terminal-এর অদৃশ্য শাসন

তুমি terminal-এ `ls | grep .go | wc -l` চালালে তিনটা আলাদা process জন্ম নেয়। কিন্তু একটা প্রশ্ন কখনো মাথায় এসেছে?

তুমি যখন Ctrl+C চাপো — তিনটার মধ্যে কোনটা মরে? একটা? সবগুলো? কে সিদ্ধান্ত নেয়?

উত্তরটা লুকিয়ে আছে Linux-এর একটা elegant design-এ — **Process Group**।

---

## Process Group: দলবদ্ধ জীবন

প্রতিটা process শুধু একা থাকে না। Linux প্রতিটা process-কে একটা **group**-এ রাখে। Group-এর একটা ID থাকে — **PGID (Process Group ID)**।

```
তুমি লিখলে: ls | grep .go | wc -l

Shell একটা নতুন group বানালো:
PGID: 1235

┌─────────────────────────────────────┐
│         Process Group 1235          │
│                                     │
│  ls        grep        wc           │
│  PID:1235  PID:1236    PID:1237     │
│                                     │
└─────────────────────────────────────┘

Shell নিজে আলাদা group-এ:
┌─────────────────────────────────────┐
│         Process Group 890           │
│                                     │
│  tiny-shell                         │
│  PID:890                            │
│                                     │
└─────────────────────────────────────┘
```

Group বানানোর কারণ একটাই — **একসাথে signal পাঠানো যায়।** পুরো group-কে একটা signal দিলে সবাই পায়। Ctrl+C চাপলে shell পুরো foreground group-কে SIGINT পাঠায় — `ls`, `grep`, `wc` তিনজনই একসাথে মরে। তোমাকে আলাদা করে প্রতিটাকে kill করতে হয় না।

---

## Terminal: সবার মালিক

Process Group বোঝার পরে আরেকটা জিনিস বোঝা দরকার — **Terminal**।

Terminal শুধু একটা কালো বাক্স না যেখানে text দেখায়। সে একজন **controller** — সে ঠিক করে দেয় কোন process group এই মুহূর্তে keyboard পাবে, কোন process group screen-এ লিখতে পারবে।

```
Terminal (/dev/pts/0)
│
│ Foreground Group ← শুধু এই group keyboard পায়
│ ┌──────────────────────────────┐
│ │  ls   │   grep   │   wc     │
│ │ 1235  │   1236   │  1237    │
│ └──────────────────────────────┘
│
│ Background Groups ← keyboard নেই, চুপচাপ চলছে
│ ┌──────────────┐   ┌──────────────┐
│ │ sleep 100 &  │   │ python app & │
│ │ PID: 1400    │   │ PID: 1500    │
│ └──────────────┘   └──────────────┘
```

যেকোনো মুহূর্তে terminal-এ শুধু **একটাই foreground group** থাকতে পারে। বাকি সব background-এ। Foreground group keyboard পায়, Ctrl+C-তে signal পায়, screen-এ লিখতে পারে অবাধে।

Background group keyboard পেতে চাইলে? Linux তাকে **SIGTTIN** signal দিয়ে থামিয়ে দেয়। জোর করে keyboard নেওয়া যায় না।

---

## fg এবং bg: গ্রুপ বদলানোর খেলা

এখন মজার অংশ।

তুমি `sleep 100 &` চালালে sleep background-এ চলে গেল। তুমি `jobs` টাইপ করলে দেখবে:

```
[1]  Running    sleep 100 &
```

`[1]` হলো **job number** — shell নিজে assign করে। PID না, job number। তুমি এই number দিয়ে job control করো।

```
fg %1    ← job 1 কে foreground-এ আনো
bg %1    ← job 1 কে background-এ পাঠাও
kill %1  ← job 1 কে মেরে দাও
```

ভেতরে কী হয়?

```
fg %1 চালালে:

Shell kernel-কে বললো:
"sleep-এর group-কে terminal-এর
 foreground group বানাও"

tcsetpgrp(terminal, sleep_pgid)
         │
         ▼
Terminal এখন sleep-এর দিকে তাকিয়ে
keyboard input sleep পাচ্ছে
Ctrl+C চাপলে sleep মরবে, shell না
```

```
Ctrl+Z চাপলে:

Terminal পুরো foreground group-কে
SIGTSTP পাঠালো

sleep থেমে গেল (STOPPED state)
Shell আবার foreground-এ এলো

jobs দেখালো:
[1]  Stopped    sleep 100
```

---

## Process-এর নতুন একটা অবস্থা: STOPPED

Milestone 3-এ process lifecycle দেখেছিলে। Job control-এর কারণে আরেকটা state যোগ হয়:

```
                    fork() — জন্ম
                        │
                        ▼
                   ┌──────────┐
                   │ RUNNABLE │
                   └────┬─────┘
                        │
                        ▼
                   ┌─────────┐
          ┌───────►│ RUNNING │
          │        └────┬────┘
          │             │
          │      ┌──────┴────────────┐
          │      │                   │
          │      ▼                   ▼
          │  ┌─────────┐        ┌─────────┐
          └──│ WAITING │        │ STOPPED │◄── Ctrl+Z
             └────┬────┘        └────┬────┘
                  │                  │ fg / bg
                  │             RUNNABLE-এ
                  │             ফিরে যায়
                  ▼
              ┌────────┐
              │ ZOMBIE │
              └────┬───┘
                   │
               ┌──────┐
               │ DEAD │
               └──────┘
```

**STOPPED** মানে process মরেনি — শুধু থেমে আছে। Memory আছে, state আছে, কিন্তু CPU নিচ্ছে না। `fg` দিলে ঠিক যেখানে ছিল সেখান থেকে শুরু করে।

---

## Shell কীভাবে Job Track করে?

Shell একটা internal table রাখে — প্রতিটা background job-এর তথ্য:

```
Shell-এর Job Table:
┌─────┬───────┬──────────┬───────────────┐
│ Job │ PGID  │ State    │ Command       │
├─────┼───────┼──────────┼───────────────┤
│  1  │ 1400  │ Running  │ sleep 100     │
│  2  │ 1500  │ Stopped  │ vim file.txt  │
│  3  │ 1600  │ Running  │ python app.py │
└─────┴───────┴──────────┴───────────────┘
```

Child শেষ হলে shell কীভাবে জানে? **SIGCHLD** signal দিয়ে। Kernel প্রতিবার কোনো child-এর state বদলালে parent-কে SIGCHLD পাঠায়। Shell সেই signal ধরে table আপডেট করে।

```
sleep 100 শেষ হলো
      │
      │ Kernel → Shell-কে SIGCHLD পাঠালো
      ▼
Shell: "job 1 শেষ হয়েছে"
Job table থেকে সরিয়ে দিলো
পরের prompt-এ দেখাবে:
[1]  Done    sleep 100
```

---

## পুরো ছবি একসাথে

```
তুমি: vim file.txt চালালে

Shell:
  fork() → vim process (PID 1500, PGID 1500)
  tcsetpgrp() → vim-কে foreground বানালো
  wait() → ঘুমালো

তুমি Ctrl+Z চাপলে:
  Terminal → PGID 1500-কে SIGTSTP পাঠালো
  vim STOPPED হলো
  Shell জেগে উঠলো (foreground-এ এলো)
  Job table-এ লিখলো: [1] Stopped vim

তুমি bg %1 চাপলে:
  Shell → vim-কে SIGCONT পাঠালো
  vim আবার চলতে শুরু করলো (background-এ)
  Job table: [1] Running vim

তুমি fg %1 চাপলে:
  Shell → tcsetpgrp(terminal, 1500)
  vim foreground-এ এলো
  Shell ঘুমালো আবার
```

---

## নিজে দেখো

```bash
# tiny-shell চালু করো, তারপর:
sleep 100 &        # background-এ পাঠাও
sleep 200 &        # আরেকটা
jobs               # দুটো দেখা যাচ্ছে?

# এখন একটাকে foreground-এ আনো
fg %1              # Ctrl+Z দিয়ে আবার stop করো

# process group দেখো
ps -o pid,pgid,stat,cmd
```

`ps` output-এ দেখবে — একই pipeline-এর সব process-এর PGID একই। Shell-এর PGID আলাদা। এটাই job control-এর ভিত্তি।

---

Process group একটা সহজ idea — কিন্তু এর উপর দাঁড়িয়ে আছে shell-এর পুরো job management। Ctrl+Z, fg, bg, jobs — এই সব command-এর পেছনে আসলে kernel-এর কাছে একটাই অনুরোধ: "এই group-টাকে foreground দাও, ওই group-টাকে background-এ রাখো।"

Milestone 8-এ এটা code-এ implement করব। 🚀