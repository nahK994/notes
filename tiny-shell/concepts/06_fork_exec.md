## Fork-Exec Model: একটা Process কীভাবে জন্ম নেয়

তুমি কখনো ভেবেছো — `ls` টাইপ করলে Shell আসলে কী করে?

সহজ উত্তর মনে হয়: "Shell `ls` চালায়।" কিন্তু এটা সত্যি না। Shell নিজে কখনো `ls` চালায় না। সে একটা সন্তান তৈরি করে, সেই সন্তানকে `ls` বানিয়ে দেয়, তারপর অপেক্ষা করে।

এই পুরো প্রক্রিয়াটার নাম **Fork-Exec Model**। Unix-এর সবচেয়ে মৌলিক এবং সুন্দর idea গুলোর একটা।

তিনটা ধাপে হয় — `fork()`, `exec()`, `wait()`।

---

### একটা রূপক দিয়ে শুরু করি

কল্পনা করো তোমার একটা দোকান আছে। একজন customer এলো বললো — "আমাকে একটা কেক বানিয়ে দাও।"

তুমি দোকান ফেলে রেখে নিজে গিয়ে কেক বানাতে পারবে না — দোকান বন্ধ হয়ে যাবে। তাই তুমি একজন কর্মীকে ডাকলে, তাকে রেসিপি দিলে, সে রান্নাঘরে গেলো। তুমি দোকানে বসে রইলে, অপেক্ষা করলে। কেক তৈরি হলে কর্মী ফিরলো, তুমি আবার স্বাভাবিক কাজে ফিরলে।

Shell ঠিক এটাই করে।

- **তুমি** = Shell
- **কর্মী** = Child process
- **কেক বানানো** = `ls` চালানো
- **কর্মী পাঠানো** = `fork()`
- **রেসিপি দেওয়া** = `exec()`
- **অপেক্ষা করা** = `wait()`

---

### ধাপ ১: fork() — নিজের হুবহু copy তৈরি করো

`fork()` করার মানে হলো — OS-কে বলা: *"আমার একটা হুবহু copy তৈরি করো।"*

```
fork() করার আগে:

Shell Process (PID: 100)
├── Code:  shell-এর code
├── Stack: shell-এর variables
├── FD 0 → keyboard
├── FD 1 → screen
├── FD 2 → screen
└── CWD:  /home/user
```

`fork()` call করার পর OS দুটো process চালায় — একটা parent, একটা child। Child হলো parent-এর photocopy:

```
fork() করার পরে:

Shell (PID: 100)               Child (PID: 101)
├── Code:  shell-এর code  →   ├── Code:  shell-এর code  ← copy
├── Stack: shell-এর data  →   ├── Stack: shell-এর data  ← copy
├── FD 0 → keyboard        →   ├── FD 0 → keyboard        ← copy
├── FD 1 → screen          →   ├── FD 1 → screen          ← copy
└── CWD:  /home/user       →   └── CWD:  /home/user       ← copy
```

দুটো process এখন identical — memory, FD, working directory সব এক। পার্থক্য শুধু PID-এ।

কিন্তু এখন প্রশ্ন আসে — দুটো process তো একই code চালাচ্ছে। তাহলে কে বুঝবে "আমি parent" আর কে বুঝবে "আমি child"?

এখানে `fork()`-এর একটা চমৎকার trick আছে। **`fork()` দুটো জায়গায় return করে — একবার parent-এ, একবার child-এ।** কিন্তু দুটো জায়গায় দুটো আলাদা value দেয়:

```
Parent পায়: child-এর PID  (যেমন 101) — মানে "আমি parent"
Child পায়:  0              — মানে "আমি child"
```

Go-তে দেখলে:

```go
pid, _ := syscall.ForkExec(...)

if pid > 0 {
    // আমি parent — child-এর PID পেলাম
    // এখন wait করবো
} else if pid == 0 {
    // আমি child — 0 পেলাম
    // এখন exec করবো
}
```

একটা function call, দুটো জায়গায় return — এটাই `fork()`-এর সবচেয়ে confusing এবং সবচেয়ে elegant অংশ।

---

### ধাপ ২: exec() — নিজেকে সম্পূর্ণ বদলে ফেলো

Child এখন shell-এর copy হয়ে বসে আছে। তার কাজ `ls` চালানো। কিন্তু সে তো shell — `ls` না।

`exec("/bin/ls")` call করলে OS child-এর code আর data সম্পূর্ণ মুছে দেয়, `/bin/ls`-এর code সেখানে বসিয়ে দেয়, এবং সেটা চালু করে।

```
exec() করার আগে (child):        exec() করার পরে (same child):

Child (PID: 101)                 Child (PID: 101)
├── Code:  shell-এর code   →    ├── Code:  ls-এর code    ← বদলে গেছে
├── Stack: shell-এর data   →    ├── Stack: নতুন, খালি    ← বদলে গেছে
├── FD 0 → keyboard         →    ├── FD 0 → keyboard      ← একই!
├── FD 1 → screen           →    ├── FD 1 → screen        ← একই!
└── CWD:  /home/user        →    └── CWD:  /home/user     ← একই!
```

এখানে তিনটা জিনিস লক্ষ্য করো:

**PID বদলায়নি।** Process-টা একই, শুধু ভেতরের সব বদলে গেছে।

**FD বদলায়নি।** `ls` জানে না তার FD 1 কোথায় যাচ্ছে — সে শুধু FD 1-এ লেখে। সেটা screen-এ যাক বা pipe-এ।

**Code আর Stack সম্পূর্ণ নতুন।** `exec()` return করে না। যে process `exec()` call করেছিলো, সে আর নেই — নতুন program শুরু হয়ে গেছে।

এই তৃতীয় পয়েন্টটা গুরুত্বপূর্ণ। **`exec()` কখনো return করে না** — যদি না কোনো error হয়। সফল `exec()` মানে পুরনো code মরে গেছে।

---

### fork() আর exec() আলাদা কেন?

অনেক OS-এ `spawn()` বলে একটাই function আছে — process তৈরি করো এবং program চালাও, একসাথে।

Unix কেন দুটো আলাদা step রাখলো?

কারণ `fork()` আর `exec()`-এর মাঝখানে একটা **সোনালী মুহূর্ত** আছে।

এই মুহূর্তে child process তৈরি হয়ে গেছে, কিন্তু নতুন program এখনো শুরু হয়নি। এই ফাঁকে তুমি child-এর FD বদলে দিতে পারো। Environment variable সেট করতে পারো। File descriptor বন্ধ করতে পারো।

```
fork()
  ↓
[সোনালী মুহূর্ত]
  ├── FD 1 কে pipe-এ point করাও   ← এটাই pipe-এর কৌশল
  ├── FD 0 কে file-এ point করাও  ← এটাই redirection
  ├── environment variable দাও
  └── যা খুশি সেট করো
  ↓
exec()  ← এখন নতুন program শুরু হলো, কিন্তু আমাদের setting গুলো রয়ে গেলো
```

`spawn()` দিয়ে এটা সম্ভব হতো না। Fork-Exec model এই flexibility-র জন্যই আলাদা।

---

### ধাপ ৩: wait() — parent ঘুমায়, child কাজ করে

`fork()` করার পর parent যদি সাথে সাথে আবার prompt দেখায়, তাহলে `ls`-এর output আর Shell-এর prompt একসাথে এলোমেলো হয়ে যাবে।

তাই parent `wait()` call করে ঘুমিয়ে পড়ে। Child শেষ না হওয়া পর্যন্ত উঠবে না।

```
Shell (parent, 100)                ls (child, 101)
───────────────────                ───────────────
wait() call করলো                  চলছে...
    │                              file list বানাচ্ছে
    │ ঘুমিয়ে পড়লো                 screen-এ দেখাচ্ছে
    │ (blocked)                         │
    │                                   │ কাজ শেষ
    │                                   ↓
    │                              exit(0)
    │◄──── kernel জানালো ──────── child মরে গেলো
    │      "101 শেষ, status=0"
    ↓
Shell জেগে উঠলো
exit code পেলো (0 = সফল)
আবার prompt দেখালো
```

এখানে **Zombie process** একটা ছোট্ট detail। Child মরে গেলেও তার exit code রাখার জন্য kernel তার একটা ছোট্ট record রেখে দেয় — এটাকে zombie বলে। Parent `wait()` করে সেই exit code নিলে zombie সম্পূর্ণ মুছে যায়। Parent যদি কখনো `wait()` না করে, zombie জমতে থাকে — এটা একটা bug।

---

### পুরো ছবি একসাথে

তুমি `ls` টাইপ করে Enter চাপলে:

```
Shell (PID: 100)
    │
    ├──── fork() ──────────────────────────────┐
    │                                          │
    │ (parent)                         (child, PID: 101)
    │ wait(101)                                │
    │ ঘুমালো                   [সোনালী মুহূর্ত — FD ঠিক করো]
    │                                          │
    │                               exec("/bin/ls")
    │                                          │
    │                               ls চলছে
    │                               output দেখাচ্ছে
    │                                          │
    │                               exit(0)
    │◄─────────────────────────────────────────┘
    │
    ↓
"$ " prompt দেখালো
```

---

### একটা কথা মনে রেখো

Fork-Exec model শুধু Shell-এর জন্য না। Linux-এ প্রতিটা process এভাবেই জন্মায়। তোমার browser যখন একটা নতুন tab খোলে, Python যখন subprocess চালায়, Nginx যখন নতুন worker তৈরি করে — সবাই এই একই `fork()` + `exec()` ব্যবহার করে।

OS-এর শুরু থেকে আজ পর্যন্ত, প্রতিটা process একটা শেকড় থেকে আসে। Linux boot হওয়ার সময় প্রথম process হলো `init` (PID 1)। এরপর থেকে সব process এই `init`-এর fork — প্রজন্মের পর প্রজন্ম।

তুমি এখন যে terminal-এ কাজ করছো, সেটাও `init`-এর কোনো না কোনো বংশধর।