## Signal: OS-এর জরুরি বার্তাবাহক

একটা দৃশ্য কল্পনা করো।

তুমি terminal-এ একটা command চালিয়েছো। সে চলছে। হঠাৎ মনে হলো — এটা আর দরকার নেই। Ctrl+C চাপলে। সাথে সাথে থেমে গেলো।

কিন্তু এটা কীভাবে হলো?

তুমি keyboard-এ কিছু টাইপ করোনি। Process-এর কাছে কোনো message পাঠাওনি। কোনো function call করোনি। শুধু দুটো key একসাথে চাপলে — আর একটা চলতে থাকা process মুহূর্তের মধ্যে থেমে গেলো।

এই অদ্ভুত জিনিসটার নাম **Signal**।

---

### Signal কী?

Signal হলো OS-এর notification system। একটা process-কে হঠাৎ করে কিছু জানানোর উপায় — সে এই মুহূর্তে যাই করুক না কেন।

চিঠির সাথে তুলনা করা যায়। কিন্তু সাধারণ চিঠি না — জরুরি তার। তুমি কী করছো সেটা দেখে না, কখন পড়বে সেটা জিজ্ঞেস করে না। সরাসরি এসে পড়ে।

```
Signal পাঠানো হলে:

Process চলছে...         Signal এলো!
  কোড চালাচ্ছে    ──►   সব থামলো
  কিছু গণনা করছে        Signal handle করলো
  loop-এ আছে            আবার চললো (বা মরলো)
```

Signal সংখ্যায় মাত্র কয়েকটা — প্রতিটার একটা নম্বর আর একটা নাম। Process যখন signal পায়, তখন তিনটা কাজের একটা করতে পারে:

```
১. Default behavior  → OS যা করার কথা সেটা করো (বেশিরভাগ ক্ষেত্রে মরো)
২. Handle করো        → নিজে সিদ্ধান্ত নাও কী করবে
৩. Ignore করো        → না দেখার ভান করো
```

কিন্তু দুটো signal আছে যেগুলো ignore বা handle করা যায় না। সেটা পরে আসছে।

---

### Shell-এ যে Signal গুলো লাগবে

```
┌─────────┬────────┬──────────────────────────────────────────┐
│  নাম    │ নম্বর │ কী করে                                   │
├─────────┼────────┼──────────────────────────────────────────┤
│ SIGINT  │   2    │ Interrupt — Ctrl+C চাপলে আসে            │
│ SIGTSTP │   20   │ Stop — Ctrl+Z চাপলে আসে                 │
│ SIGCONT │   18   │ Continue — থামা process চালু করো         │
│ SIGKILL │   9    │ Kill — জোর করে মারো, কোনো কথা নেই      │
│ SIGTERM │   15   │ Terminate — ভদ্রভাবে মরার অনুরোধ         │
│ SIGCHLD │   17   │ Child মরলে parent-কে জানাও              │
│ SIGPIPE │   13   │ Pipe ভেঙে গেলে জানাও                    │
│ SIGHUP  │   1    │ Terminal বন্ধ হলে জানাও                  │
└─────────┴────────┴──────────────────────────────────────────┘
```

একে একে দেখি।

---

### SIGINT — "থামো"

**কীভাবে আসে:** Ctrl+C

**Default behavior:** Process মরে যায়

তুমি যখন Ctrl+C চাপো, terminal driver সেটা ধরে এবং foreground process-এ SIGINT পাঠায়। Process-টা সেই মুহূর্তে যা-ই করুক — loop চালাক, গণনা করুক — সব থামিয়ে মরে যায়।

```
$ ping google.com
PING google.com: 56 data bytes
64 bytes from 142.250.1.1: time=12ms
64 bytes from 142.250.1.1: time=11ms
^C                          ← Ctrl+C চাপলাম
--- google.com ping statistics ---
2 packets transmitted
```

কিন্তু `ping` এখানে একটু চালাক। সে SIGINT handle করে — মরার আগে statistics দেখায়। Default behavior হলে কোনো statistics ছাড়াই মরতো।

Shell নিজেও SIGINT handle করে। তুমি Shell-এ Ctrl+C চাপলে Shell মরে না — সে foreground child-এ SIGINT পাঠায়, নিজে বেঁচে থাকে।

```
Ctrl+C চাপলে:

তুমি → Ctrl+C
Terminal → SIGINT
Shell → নিজে handle করলো, মরলো না
       foreground child-এ SIGINT পাঠালো
       child মরলো
Shell → নতুন prompt দেখালো
```

---

### SIGTSTP — "ঘুমাও"

**কীভাবে আসে:** Ctrl+Z

**Default behavior:** Process pause হয়ে যায়

SIGINT আর SIGTSTP-এর পার্থক্যটা জীবন-মৃত্যুর:

```
SIGINT  → process মরলো    (চিরতরে)
SIGTSTP → process ঘুমালো  (জাগানো যাবে)
```

```
$ vim important_file.txt
[vim চলছে, কাজ করছো]

Ctrl+Z চাপলে:
^Z
[1]+ Stopped    vim important_file.txt
$

[তুমি অন্য কাজ করলে]

$ fg %1
[vim ফিরে এলো, সব ঠিক আছে]
```

Vim মরেনি। ঘুমিয়ে ছিলো। জাগিয়ে দিলে ঠিক যেখানে ছিলো সেখান থেকে শুরু করলো।

---

### SIGCONT — "জাগো"

**কীভাবে আসে:** `fg` বা `bg` command

**Default behavior:** Stopped process আবার চলা শুরু করে

SIGTSTP-এর উল্টো। কোনো process pause হয়ে থাকলে SIGCONT পাঠালে সে আবার চলা শুরু করে।

```
$ fg %1    → Shell SIGCONT পাঠালো stopped job-এ
$ bg %1    → Shell SIGCONT পাঠালো, background-এ চললো
```

তুমি নিজেও পাঠাতে পারো:

```bash
kill -SIGCONT 1234    # PID 1234-কে জাগাও
```

---

### SIGKILL — "এখনই মরো"

**কীভাবে আসে:** `kill -9` বা `kill -SIGKILL`

**Default behavior:** Process সাথে সাথে মরে

এটাই সবচেয়ে নিষ্ঠুর signal। আর সবচেয়ে শক্তিশালী।

SIGKILL handle করা যায় না। Ignore করা যায় না। Process এটা পেলে কিছু করার সুযোগ পায় না — Kernel সরাসরি তাকে মেরে দেয়।

```
SIGTERM পেলে process:        SIGKILL পেলে process:
  - file save করতে পারে       - কিছুই করতে পারে না
  - connection বন্ধ করতে পারে  - Kernel জোর করে মারে
  - cleanup করতে পারে         - কোনো cleanup নেই
  - তারপর মরে
```

কখন SIGKILL দরকার? যখন process হ্যাং করে গেছে, SIGTERM-এ সাড়া দিচ্ছে না।

```bash
$ kill 1234        # আগে ভদ্রভাবে বলো
$ kill -9 1234     # না শুনলে জোর করো
```

একটা কথা মনে রাখো — SIGKILL মানে process-এর কোনো cleanup হয় না। সে হয়তো একটা file-এর মাঝখানে লিখছিলো — সেই file corrupt হতে পারে। তাই SIGKILL সবার শেষ অস্ত্র।

---

### SIGTERM — "মরার আগে গুছিয়ে নাও"

**কীভাবে আসে:** `kill` command (default)

**Default behavior:** Process মরে যায়

SIGTERM হলো ভদ্র অনুরোধ। *"তুমি কি মরবে? আমি অপেক্ষা করছি।"*

Process চাইলে এটা handle করতে পারে — মরার আগে file save করো, network connection বন্ধ করো, log লেখো। তারপর নিজে থেকে exit করো।

```
Server process SIGTERM পেলে:

SIGTERM এলো
  │
  ├── নতুন request নেওয়া বন্ধ করো
  ├── চলতে থাকা request গুলো শেষ করো
  ├── database connection বন্ধ করো
  ├── log-এ লেখো "gracefully shutting down"
  └── exit(0)
```

`systemctl stop` করলে OS আগে SIGTERM পাঠায়। কিছুক্ষণ অপেক্ষা করে। তারপরও না মরলে SIGKILL।

---

### SIGCHLD — "তোমার সন্তান মরেছে"

**কীভাবে আসে:** Child process মরলে parent পায়

**Default behavior:** Ignore

এটা Shell-এর জন্য বিশেষভাবে গুরুত্বপূর্ণ।

Background job শেষ হলে Shell কীভাবে জানে? SIGCHLD দিয়ে।

```
Shell চলছে, prompt দেখাচ্ছে
    │
    │  [background-এ sleep 10 চলছিলো]
    │
    │  sleep 10 শেষ হলো → OS Shell-কে SIGCHLD পাঠালো
    │
    ▼
Shell SIGCHLD handler চললো:
  waitpid(WNOHANG) → child-এর exit code নিলো
  zombie সাফ করলো
  job table update করলো

পরের prompt-এ:
[1]+ Done    sleep 10
$
```

SIGCHLD না থাকলে Shell-কে ক্রমাগত `waitpid()` দিয়ে check করতে হতো। SIGCHLD দিয়ে OS নিজেই জানিয়ে দেয়।

---

### SIGPIPE — "তুমি শূন্যে লিখছো"

**কীভাবে আসে:** Pipe-এর read end বন্ধ হলে

**Default behavior:** Process মরে যায়

`ls | head -5` চালালে কী হয়?

`head` প্রথম ৫ লাইন পড়ে exit করে। কিন্তু `ls` হয়তো তখনো লিখে চলেছে। Pipe-এর read end বন্ধ — কেউ পড়ছে না। তখন OS `ls`-কে SIGPIPE পাঠায়: *"তুমি যেখানে লিখছো সেখানে কেউ নেই।"*

```
ls ──► pipe ──► head
                 │
                 │ 5 লাইন পেলাম, exit করলাম
                 ▼
              pipe-এর read end বন্ধ

ls এখনো লিখছে → SIGPIPE → ls মরলো
```

এই কারণে `ls | head -5` তুমি লাখো file-এর directory-তে চালালেও সাথে সাথে result পাও — `ls` সব শেষ করার আগেই `head` তাকে থামিয়ে দেয়।

---

### SIGHUP — "Terminal চলে গেছে"

**কীভাবে আসে:** Terminal বন্ধ হলে

**Default behavior:** Process মরে যায়

Terminal window বন্ধ করলে সেই terminal-এর সাথে connected সব process SIGHUP পায়। *"তোমার terminal আর নেই।"*

এই কারণে SSH connection হঠাৎ কেটে গেলে চলতে থাকা command গুলো মরে যায়।

সমাধান হলো `nohup`:

```bash
nohup python train.py &
```

`nohup` মানে — *"No Hangup"* — SIGHUP ignore করো। Terminal বন্ধ হলেও process চলতে থাকবে।

```
Terminal বন্ধ হলো → SIGHUP
                       │
              nohup আছে?
              হ্যাঁ → ignore করলো, চলছে
              না  → মরে গেলো
```

---

### Signal-এর পুরো ছবি একসাথে

```
┌─────────────────────────────────────────────────────────┐
│                    Signal Sources                       │
│                                                         │
│  Keyboard        OS              অন্য Process           │
│  ─────────       ──             ──────────────          │
│  Ctrl+C → SIGINT                kill 1234               │
│  Ctrl+Z → SIGTSTP  child মরলে → SIGCHLD                │
│           pipe ভাঙলে → SIGPIPE                          │
│           terminal বন্ধ → SIGHUP                        │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │    Process     │
              │                │
              │  Signal পেলে: │
              │  ┌──────────┐  │
              │  │ Handle   │  │ ← নিজে সিদ্ধান্ত নাও
              │  ├──────────┤  │
              │  │ Default  │  │ ← OS-এর নিয়ম মানো
              │  ├──────────┤  │
              │  │ Ignore   │  │ ← না দেখার ভান করো
              │  └──────────┘  │
              │                │
              │  ⚠️ SIGKILL:   │
              │  handle/ignore │
              │  করা যায় না   │
              └────────────────┘
```

---

### Go-তে Signal Handle করা

```go
import (
    "os"
    "os/signal"
    "syscall"
)

func main() {
    // একটা channel তৈরি করো
    sigs := make(chan os.Signal, 1)

    // কোন signal গুলো ধরতে চাও বলো
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

    // অন্য কাজ করো...

    // Signal আসলে এখানে আটকাবে
    sig := <-sigs

    switch sig {
    case syscall.SIGINT:
        fmt.Println("\nCtrl+C পেলাম, থামছি...")
        // cleanup করো
    case syscall.SIGTERM:
        fmt.Println("SIGTERM পেলাম, গুছিয়ে নিচ্ছি...")
        // cleanup করো
    }
}
```

Shell-এ এটা আরো জটিল। Shell SIGINT নিজে ignore করে, কিন্তু foreground child-এ পাঠায়। SIGTSTP-এও একই কাজ। এভাবে Ctrl+C বা Ctrl+Z চাপলে Shell মরে না — শুধু child-টা থামে বা মরে।

---

Signal হলো OS-এর জরুরি ডাক। তুমি চাও বা না চাও, সে আসে।

কিন্তু কীভাবে সাড়া দেবে — সেটা তোমার হাতে। Handle করবে, ignore করবে, নাকি চুপ করে মেনে নেবে — এই সিদ্ধান্তটাই একটা ভালো program আর একটা খারাপ program-এর পার্থক্য তৈরি করে।