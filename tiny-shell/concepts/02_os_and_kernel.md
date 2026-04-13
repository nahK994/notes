# ০২. OS এবং Kernel

## OS কী? কেন লাগে?

একটু কল্পনা করো।

তুমি একটা computer বানালে — CPU আছে, RAM আছে, disk আছে। এখন সরাসরি একটা program চালাতে চাইলে কী হবে?

Program-কে নিজেই বলতে হবে — "RAM-এর কোন address-এ আমার data রাখব?" Program-কে নিজেই disk-এ byte-by-byte লিখতে হবে। দুটো program একসাথে চালালে দুজনেই RAM-এর একই জায়গায় লিখতে চাইবে। কেউ থামাবে না, কেউ referee নেই। সব ভেঙে পড়বে।

**Operating System** হলো সেই referee — যে hardware-এর উপর বসে সবকিছু manage করে, প্রতিটা program-কে তার প্রাপ্য resource দেয়, আর কেউ যেন অন্যের জায়গায় হাত না দেয় সেটা নিশ্চিত করে।

ভাবো একটা হোটেল:

```
┌──────────────────────────────────────────────────┐
│  Guest (তুমি / তোমার program)                      │
│  "আমাকে একটা room দাও, খাবার দাও"                 │
├──────────────────────────────────────────────────┤
│  Reception / Manager (OS / Kernel)               │
│  সব resource manage করে, fair share দেয়          │
├──────────────────────────────────────────────────┤
│  হোটেলের Infrastructure (Hardware)                │
│  Room (RAM), Kitchen (CPU), Storage (Disk)       │
└──────────────────────────────────────────────────┘
```

Guest জানে না kitchen কীভাবে চলে। সে শুধু order দেয়। Manager মাঝখানে থেকে সব coordinate করে। ঠিক এভাবেই তোমার program OS-কে বলে "আমাকে memory দাও", OS দেয় — program জানেও না RAM-এর কোন physical address-এ সেটা গেল।

OS ছাড়া প্রতিটা program-কে নিজেই RAM manage করতে হতো, নিজেই disk-এ লিখতে হতো — chaos হতো।

---

## OS-এর স্তর

OS একটা monolithic block না। এটা কয়েকটা স্তরে সাজানো — প্রতিটা স্তরের আলাদা দায়িত্ব, আলাদা ক্ষমতা।

```
┌─────────────────────────────────────────────┐
│              USER SPACE                     │
│                                             │
│   tiny-shell    ls    grep    firefox       │
│   (তোমার program-রা এখানে চলে)                │
│                                             │
├─────────────────────────────────────────────┤
│         SYSTEM CALL INTERFACE               │
│                                             │
│   ← এটাই একমাত্র দরজা kernel-এ ঢোকার →         │
│   User space থেকে kernel-এ যাওয়ার            │
│   নিয়মকানুন এখানে                              │
│                                             │
├─────────────────────────────────────────────┤
│              KERNEL SPACE                   │
│                                             │
│  Process   File      Memory   Network       │
│  Manager   System    Manager  Stack         │
│  (কে চলবে) (disk     (RAM     (internet)    │
│            manage)   manage)                │
│                                             │
├─────────────────────────────────────────────┤
│               HARDWARE                      │
│   CPU    RAM    SSD/HDD    Keyboard    NIC  │
└─────────────────────────────────────────────┘
```

### User Space

তোমার লেখা সব program এখানে চলে। `tiny-shell`, `ls`, `firefox` — সবাই user space-এর বাসিন্দা।

এখানকার program-গুলোর একটা কঠিন সীমাবদ্ধতা আছে — এরা সরাসরি hardware ছুঁতে পারে না। RAM-এর কোনো address সরাসরি পড়তে পারে না, disk-এ সরাসরি লিখতে পারে না। চাইলেও না।

এটা বাধা মনে হতে পারে। কিন্তু এটাই সুরক্ষা। তোমার browser যদি সরাসরি disk-এ লিখতে পারত, তাহলে যেকোনো malicious website তোমার পুরো system মুছে দিতে পারত।

### Kernel Space

OS-এর মূল অংশ এখানে থাকে। Kernel সরাসরি hardware-এর সাথে কথা বলে — CPU-কে বলে কোন process কতক্ষণ চলবে, RAM-এর কোথায় কী রাখা হবে, disk থেকে কীভাবে data পড়া হবে।

এখানে একটাই বিপদ — এই জায়গায় bug হলে পুরো system crash করে। User space-এ যদি তোমার program crash করে, শুধু সেই program মরে। Kernel-এ crash হলে সব মরে। Linux-এ এটাকে বলে **kernel panic**।

### System Call Interface

এটাই সবচেয়ে গুরুত্বপূর্ণ সীমানা।

User space থেকে kernel-এ যাওয়ার **একমাত্র উপায়** হলো এই interface দিয়ে। তুমি file খুলতে চাও? Kernel-কে বলতে হবে। নতুন process বানাতে চাও? Kernel-কে বলতে হবে। এই "বলার" পদ্ধতিকে বলে **System Call**।

এটা একটা controlled gateway — kernel নিজে ঠিক করে রেখেছে কোন কোন কাজের জন্য request আসতে পারে, কীভাবে আসতে পারে। যেকোনো কিছু করতে দেয় না।

ভাবো bank-এর teller window-এর মতো। তুমি সরাসরি vault-এ ঢুকতে পারবে না। নির্দিষ্ট window দিয়ে request করতে হবে, teller যাচাই করবে, তারপর কাজ হবে।

---

## Shell চালাতে OS কীভাবে সাহায্য করে?

`ls` টাইপ করলে মনে হয় সরাসরি হয়ে যাচ্ছে। আসলে প্রতিটা ধাপে kernel-এর কাছে যেতে হচ্ছে:

```
User Space:
  tiny-shell → "ls চালাতে হবে"
       │
       │ syscall: fork()
       ▼
Kernel:
  "ঠিক আছে, নতুন process বানাচ্ছি"
       │
       ▼
User Space:
  নতুন process → syscall: exec("/bin/ls")
       │
Kernel:
  "/bin/ls disk থেকে RAM-এ তুললাম, চালু করলাম"
       │
       ▼
User Space:
  ls চলছে → output দিচ্ছে
       │
       │ syscall: exit()
       ▼
Kernel:
  "process শেষ, RAM free করলাম"
```

প্রতিটা গুরুত্বপূর্ণ কাজে kernel-এর কাছে যেতে হয়। Shell নিজে কিছু করে না — সে শুধু kernel-কে বলে কী করতে হবে।

---

## একটু গভীরে — Kernel আসলে কী কী manage করে?

Kernel-এর কাজকে চারটা বড় ভাগে ভাগ করা যায়:

**Process Manager** — কোন process কখন CPU পাবে সেটা সে ঠিক করে। তোমার computer-এ এই মুহূর্তে হয়তো ৩০০টা process চলছে, কিন্তু CPU core আছে হয়তো ৮টা। Kernel millisecond-এ switch করে সবাইকে CPU-র ভাগ দেয়, তাই মনে হয় সব একসাথে চলছে।

**File System** — disk-এ data কীভাবে সাজানো থাকবে, কীভাবে পড়া-লেখা হবে সেটা kernel জানে। তুমি শুধু বলো "এই file খোলো", kernel বাকি সব করে।

**Memory Manager** — প্রতিটা process-কে আলাদা memory দেয়, কেউ যেন অন্যের memory না পড়তে পারে সেটা নিশ্চিত করে। একটা process crash করলে তার memory আবার free করে দেয়।

**Network Stack** — internet-এ data পাঠানো-নেওয়ার পুরো কাজ kernel করে। তুমি শুধু "এই data পাঠাও" বললে kernel TCP/IP সামলায়।

---

> **মনে রাখো:** Shell নিজে powerful না। Shell powerful কারণ সে kernel-এর শক্তিকে তোমার কাছে সহজে পৌঁছে দেয়। Kernel হলো আসল engine — shell হলো steering wheel।