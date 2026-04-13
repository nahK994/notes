## OS এবং Kernel: তোমার Computer-এর অদৃশ্য শাসক

একটা প্রশ্ন দিয়ে শুরু করি।

তোমার computer-এ এই মুহূর্তে হয়তো browser চলছে, music player চলছে, text editor চলছে। সবাই একই RAM ব্যবহার করছে, একই CPU ব্যবহার করছে। কিন্তু কেউ কারো জায়গায় হাত দিচ্ছে না, কেউ কাউকে থামিয়ে দিচ্ছে না।

এটা কীভাবে সম্ভব?

কেউ একজন মাঝখানে থেকে সব সামলাচ্ছে। সেই কেউটার নাম **Operating System**।

---

### OS না থাকলে কী হতো?

ধরো OS নেই। তুমি একটা program লিখলে। এই program-কে নিজেই ঠিক করতে হবে — RAM-এর কোন address-এ data রাখবো। নিজেই disk-এ byte-by-byte লিখতে হবে। নিজেই keyboard থেকে input পড়তে হবে।

এখন দুটো program একসাথে চালাও। দুজনেই RAM-এর একই address-এ লিখতে চাইছে। কেউ থামাবে না, কেউ referee নেই।

সব ভেঙে পড়বে।

OS হলো সেই referee — hardware-এর উপর বসে সবকিছু manage করে, প্রতিটা program-কে তার প্রাপ্য resource দেয়, আর কেউ যেন অন্যের জায়গায় হাত না দেয় সেটা নিশ্চিত করে।

একটা রূপক দিয়ে ভাবো:

```
┌──────────────────────────────────────────────────┐
│  Guest (তোমার program)                           │
│  "আমাকে একটা room দাও, খাবার দাও"               │
├──────────────────────────────────────────────────┤
│  Hotel Manager (OS / Kernel)                     │
│  সব resource manage করে, fair share দেয়         │
├──────────────────────────────────────────────────┤
│  হোটেলের Infrastructure (Hardware)              │
│  Room (RAM), Kitchen (CPU), Storage (Disk)       │
└──────────────────────────────────────────────────┘
```

Guest জানে না kitchen কীভাবে চলে। সে শুধু order দেয়। Manager মাঝখানে থেকে সব coordinate করে।

তোমার program ঠিক এভাবেই OS-কে বলে — "আমাকে memory দাও।" OS দেয়। Program জানেও না RAM-এর কোন physical address-এ সেটা গেল। জানার দরকারও নেই।

---

### OS-এর স্তরগুলো

OS একটা monolithic block না। এটা কয়েকটা স্তরে সাজানো — প্রতিটার আলাদা দায়িত্ব, আলাদা ক্ষমতা।

```
┌─────────────────────────────────────────────────┐
│                  USER SPACE                     │
│                                                 │
│      tiny-shell    ls    grep    firefox        │
│      (তোমার program-রা এখানে চলে)              │
│                                                 │
├─────────────────────────────────────────────────┤
│           SYSTEM CALL INTERFACE                 │
│                                                 │
│      ← user space থেকে kernel-এ ঢোকার          │
│            একমাত্র দরজা →                      │
│                                                 │
├─────────────────────────────────────────────────┤
│                 KERNEL SPACE                    │
│                                                 │
│   Process     File       Memory     Network     │
│   Manager     System     Manager    Stack       │
│   (কে চলবে)  (disk      (RAM       (internet)  │
│               manage)    manage)               │
│                                                 │
├─────────────────────────────────────────────────┤
│                  HARDWARE                       │
│      CPU    RAM    SSD/HDD    Keyboard    NIC   │
└─────────────────────────────────────────────────┘
```

#### User Space

তোমার লেখা সব program এখানে চলে। `tiny-shell`, `ls`, `firefox` — সবাই user space-এর বাসিন্দা।

এখানকার program-গুলোর একটা কঠিন নিয়ম আছে — এরা সরাসরি hardware ছুঁতে পারে না। RAM-এর কোনো address সরাসরি পড়তে পারে না, disk-এ সরাসরি লিখতে পারে না।

বাধা মনে হচ্ছে? এটাই আসলে সুরক্ষা। তোমার browser যদি সরাসরি disk-এ লিখতে পারতো, যেকোনো malicious website তোমার পুরো system এক নিমেষে মুছে দিতে পারতো।

#### Kernel Space

OS-এর মূল অংশ এখানে থাকে। Kernel সরাসরি hardware-এর সাথে কথা বলে — CPU-কে বলে কোন process কতক্ষণ চলবে, RAM-এর কোথায় কী রাখা হবে, disk থেকে কীভাবে data পড়া হবে।

এখানে একটাই বিপদ — এই জায়গায় bug হলে পুরো system crash করে। User space-এ তোমার program crash করলে শুধু সেই program মরে, বাকি সব ঠিক থাকে। Kernel-এ crash হলে সব একসাথে মরে। Linux-এ এই মহাপ্রলয়ের নাম **kernel panic**।

#### System Call Interface

এটাই সবচেয়ে গুরুত্বপূর্ণ সীমানা।

User space থেকে kernel-এ যাওয়ার **একমাত্র উপায়** হলো এই interface দিয়ে। File খুলতে চাও? Kernel-কে বলতে হবে। নতুন process বানাতে চাও? Kernel-কে বলতে হবে। এই "বলার" পদ্ধতিকে বলে **System Call**।

এটা একটা controlled gateway। Bank-এর teller window-র মতো — তুমি সরাসরি vault-এ ঢুকতে পারবে না। নির্দিষ্ট window দিয়ে request করতে হবে, teller যাচাই করবে, তারপর কাজ হবে।

---

### Kernel আসলে কী কী manage করে?

Kernel-এর কাজকে চারটা বড় ভাগে ভাগ করা যায়।

**Process Manager** — কোন process কখন CPU পাবে সেটা সে ঠিক করে। তোমার computer-এ এই মুহূর্তে হয়তো ৩০০টা process চলছে, কিন্তু CPU core আছে হয়তো ৮টা। Kernel millisecond-এ millisecond-এ switch করে সবাইকে CPU-র ভাগ দেয় — তাই মনে হয় সব একসাথে চলছে, আসলে চলে না।

**Memory Manager** — প্রতিটা process-কে আলাদা memory দেয়। কেউ যেন অন্যের memory না পড়তে পারে সেটা hardware-স্তরে নিশ্চিত করে। একটা process crash করলে তার memory নিজে থেকে free করে দেয়।

**File System** — disk-এ data কীভাবে সাজানো থাকবে, কীভাবে পড়া-লেখা হবে — সব kernel জানে। তুমি শুধু বলো "এই file খোলো", kernel বাকি সব করে।

**Network Stack** — internet-এ data পাঠানো-নেওয়ার পুরো কাজ kernel করে। তুমি শুধু "এই data পাঠাও" বললে kernel TCP/IP-এর পুরো জটিল খেলা একা সামলায়।

---

### Shell চালাতে OS কীভাবে কাজ করে?

`ls` টাইপ করলে মনে হয় সরাসরি হয়ে যাচ্ছে। আসলে প্রতিটা ধাপে kernel-এর কাছে যেতে হচ্ছে:

```
[User Space]
  tiny-shell → "ls চালাতে হবে"
       │
       │ syscall: fork()
       ▼
[Kernel Space]
  "ঠিক আছে, নতুন process বানাচ্ছি"
       │
       ▼
[User Space]
  child process → syscall: exec("/bin/ls")
       │
[Kernel Space]
  "/bin/ls disk থেকে RAM-এ তুললাম, চালু করলাম"
       │
       ▼
[User Space]
  ls চলছে, output দিচ্ছে
       │
       │ syscall: exit()
       ▼
[Kernel Space]
  "process শেষ, RAM free করলাম"
```

Shell নিজে কিছু করে না। প্রতিটা গুরুত্বপূর্ণ কাজে সে kernel-এর কাছে যায়, kernel করে দেয়।

---

এখানে একটা কথা মনে গেঁথে নাও।

Shell নিজে powerful না। Shell powerful কারণ সে kernel-এর বিশাল শক্তিকে তোমার কাছে সহজে পৌঁছে দেয়।

**Kernel হলো আসল engine। Shell হলো steering wheel।**