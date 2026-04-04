পুরো document টা দিচ্ছি — শুধু "FD inherit হয়" অংশটা বিস্তারিত করে, বাকি সব হুবহু আগের মতো।

---

# 🐚 tiny-shell — সম্পূর্ণ ভূমিকা
## Part 1: Concept

---

## ১. Shell কী? কেন লাগে?

তুমি Linux চালু করলে একটা কালো স্ক্রিন আসে। সেখানে টাইপ করো `ls`, Enter চাপো — file list দেখায়। এই কালো স্ক্রিনটাই **Shell**।

Shell হলো তোমার এবং OS-এর মাঝখানের **দোভাষী**।

```
তুমি                Shell              OS (Kernel)
────                 ─────              ───────────
"ls টাইপ করলাম" ──► বুঝলো, translate ──► "file list দাও"
                     করলো              ──► দিলো
                ◄─── দেখালো ◄───────────
```

Shell ছাড়া OS-এর সাথে কথা বলতে হলে machine code লিখতে হতো। Shell সেই কাজটা সহজ করে দেয়।

**Shell কোনো magic না।** এটা নিজেও একটা সাধারণ program — ঠিক `ls` বা `grep`-এর মতোই। পার্থক্য একটাই: এটা **অন্য program চালানোর** কাজ করে।

### Shell আসলে কী করে?

Shell মূলত একটা **loop**:

```
┌─────────────────────────────────────────────┐
│                SHELL LOOP                   │
│                                             │
│  1. Prompt দেখাও  ──►  "tiny-shell> "      │
│         │                                   │
│         ▼                                   │
│  2. Input পড়ো   ──►  "ls -la"             │
│         │                                   │
│         ▼                                   │
│  3. Parse করো   ──►  cmd="ls", args=["-la"] │
│         │                                   │
│         ▼                                   │
│  4. Execute করো ──►  নতুন process বানাও   │
│         │                                   │
│         ▼                                   │
│  5. Wait করো    ──►  process শেষ হওয়া পর্যন্ত│
│         │                                   │
│         └──────────────────────────────────►│
│                  (আবার শুরু থেকে)           │
└─────────────────────────────────────────────┘
```

এটাকে বলে **REPL** — Read, Eval, Print, Loop।

---

## ২. OS কী? কীভাবে কাজ করে?

**Operating System (OS)** হলো সেই software যে hardware-কে manage করে এবং তোমার program-গুলোকে চালানোর সুযোগ দেয়।

ভাবো একটা হোটেল:

```
হোটেলের analogy:
┌──────────────────────────────────────────────────┐
│  Guest (তুমি / তোমার program)                    │
│  "আমাকে একটা room দাও, খাবার দাও"               │
├──────────────────────────────────────────────────┤
│  Reception / Manager (OS / Kernel)               │
│  সব resource manage করে, fair share দেয়         │
├──────────────────────────────────────────────────┤
│  হোটেলের Infrastructure (Hardware)              │
│  Room (RAM), Kitchen (CPU), Storage (Disk)       │
└──────────────────────────────────────────────────┘
```

OS ছাড়া প্রতিটা program-কে নিজেই RAM manage করতে হতো, নিজেই disk-এ লিখতে হতো — chaos হতো।

### OS-এর স্তর

```
┌─────────────────────────────────────────────┐
│              USER SPACE                     │
│                                             │
│   tiny-shell    ls    grep    firefox       │
│   (তোমার program-রা এখানে চলে)             │
│                                             │
├─────────────────────────────────────────────┤
│         SYSTEM CALL INTERFACE               │
│                                             │
│   ← এটাই একমাত্র দরজা kernel-এ ঢোকার →    │
│   User space থেকে kernel-এ যাওয়ার          │
│   নিয়মকানুন এখানে                          │
│                                             │
├─────────────────────────────────────────────┤
│              KERNEL SPACE                   │
│                                             │
│  Process   File      Memory   Network       │
│  Manager   System    Manager  Stack         │
│  (কে চলবে) (disk     (RAM     (internet)   │
│            manage)   manage)               │
│                                             │
├─────────────────────────────────────────────┤
│               HARDWARE                      │
│   CPU    RAM    SSD/HDD    Keyboard    NIC  │
└─────────────────────────────────────────────┘
```

**User Space**: তোমার সব program এখানে চলে। এরা সরাসরি hardware ছুঁতে পারে না।

**Kernel Space**: OS-এর মূল অংশ। সে hardware-কে সরাসরি control করে। এখানে bug হলে পুরো system crash করে।

**System Call Interface**: User space থেকে kernel-এ যাওয়ার একমাত্র উপায়। এটা একটা controlled gateway — যেকোনো কিছু করতে দেয় না।

### Shell চালাতে OS কীভাবে সাহায্য করে?

```
তুমি terminal-এ "ls" টাইপ করলে:

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

প্রতিটা গুরুত্বপূর্ণ কাজে kernel-এর কাছে যেতে হয়।

---

## ৩. Process কী?

### Program vs Process

```
DISK-এ:                    RAM-এ (চলার সময়):
────────                   ──────────────────
/bin/ls                    Process (PID 1234)
  │                          │
  │ একটা file,               │ জীবন্ত, চলছে
  │ ঘুমাচ্ছে,               │ CPU খাচ্ছে
  │ কিছু করছে না            │ memory দখল করেছে
  │
  └── recipe বইয়ের মতো     └── রান্না হচ্ছে মতো
```

**Program** = recipe (disk-এ পড়ে থাকে)
**Process** = সেই recipe অনুযায়ী রান্না (RAM-এ চলছে)

একই recipe দিয়ে একসাথে অনেক রান্না হতে পারে। তুমি ৫টা terminal-এ `ls` চালালে ৫টা আলাদা process — প্রত্যেকের আলাদা PID, আলাদা memory।

### কেন Process লাগে?

Process না থাকলে একটা program চললে অন্যটা চলতে পারত না। Process-এর কারণে:

```
একই সময়ে:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ firefox  │  │ terminal │  │ music    │
│ PID:1001 │  │ PID:1002 │  │ PID:1003 │
│          │  │          │  │          │
│ নিজের   │  │ নিজের   │  │ নিজের   │
│ memory   │  │ memory   │  │ memory   │
└──────────┘  └──────────┘  └──────────┘
     ↑               ↑             ↑
     └───────────────┴─────────────┘
              CPU share করছে
         (এত দ্রুত switch করে যে
          মনে হয় একসাথে চলছে)
```

প্রতিটা process আলাদা — একজনের crash অন্যজনকে সাধারণত নামায় না।

### Process-এর ভেতরে কী থাকে?

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
│  │ (তুমি যখন নতুন data তৈরি করো) │    │
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

**Code Segment**: program-এর instructions। Read-only — চলার সময় বদলায় না।

**Stack**: function call হলে এখানে জমা হয়। `main()` → `execute()` → `fork()` — এই chain এখানে থাকে। function শেষ হলে stack থেকে মুছে যায়।

**Heap**: তুমি যখন নতুন data তৈরি করো (যেমন নতুন string, slice) সেটা এখানে জমা হয়। Stack-এর মতো automatically মুছে যায় না।

**File Descriptors**: process বাইরের জগতের সাথে যেভাবে কথা বলে। এটা নিচে আলাদা section-এ বিস্তারিত আসবে।

**Environment Variables**: PATH, HOME ইত্যাদি। Child process এগুলো parent থেকে copy করে পায়।

### Process-এর জীবনচক্র

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

**RUNNING**: CPU-তে এই মুহূর্তে চলছে।

**RUNNABLE**: চলার জন্য ready, কিন্তু CPU busy — অপেক্ষায় আছে। Linux millisecond-এ switch করে তাই আমরা বুঝতে পারি না।

**WAITING**: কোনো কিছুর জন্য অপেক্ষা করছে। যেমন `ls` disk থেকে file list পড়ছে — সেই সময় CPU ছেড়ে WAITING-এ চলে যায়। disk পড়া শেষ হলে আবার RUNNABLE। এই সময়টায় অন্য process CPU পায় — এভাবেই Linux একসাথে হাজার process চালায়।

### 🧟 Zombie Process — মৃত কিন্তু তালিকায় আছে

এটা অনেকের কাছে confusing। সহজ করে বলি।

বাস্তব জীবনের analogy: ধরো তোমার বন্ধু তোমাকে বললো "আমি কাজ শেষ করে তোমাকে result জানাবো।" বন্ধু কাজ শেষ করলো, কিন্তু তুমি এখনো জিজ্ঞেস করোনি। সে technically কাজ শেষ, কিন্তু result দেওয়ার জন্য অপেক্ষায় দাঁড়িয়ে আছে।

Process-এর ক্ষেত্রে:

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

Zombie process খুব কম resource নেয় — শুধু process table-এ একটা row। কিন্তু অনেক zombie জমলে process table ভরে যায়, নতুন process বানানো যায় না।

**Solution**: Parent সবসময় `wait()` করবে — child শেষ হলে তার exit code নেবে। তাহলে zombie থাকবে না। Milestone 7-এ এটা handle করব।

### PID এবং PPID

```
প্রতিটা process-এর দুটো ID:

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

```bash
# নিজে দেখো:
echo $$          # তোমার shell-এর PID
echo $PPID       # তোমার shell-এর parent PID
pstree -p $$     # পুরো গাছ দেখো
```

---

## ৪. File Descriptor (FD)

এটা OS-এর অন্যতম সুন্দর idea। ভালো করে বোঝা দরকার কারণ Pipe এবং I/O Redirection সম্পূর্ণ এর উপর নির্ভরশীল।

### FD কী?

Process যখন বাইরের জগতের সাথে কথা বলে — screen-এ লেখে, keyboard পড়ে, file খোলে — সে একটা **ছোট্ট নম্বর** দিয়ে কাজ করে। এই নম্বরটাই **File Descriptor**।

Kernel একটা table রাখে — এই নম্বর আসলে কোথায় যাচ্ছে সেটা সে জানে:

```
Kernel-এর FD Table (process 1234-এর জন্য):
┌──────────────────────────────────────────────────┐
│ FD  │ আসলে কোথায় যাচ্ছে                        │
├─────┼──────────────────────────────────────────── │
│  0  │ /dev/pts/0 (তোমার terminal keyboard)       │
│  1  │ /dev/pts/0 (তোমার terminal screen)         │
│  2  │ /dev/pts/0 (তোমার terminal screen)         │
│  3  │ /home/shomi/notes.txt (তুমি খুলেছ)        │
│  4  │ (খালি)                                    │
└──────────────────────────────────────────────────┘
```

প্রতিটা process জন্মের সময়েই তিনটা FD পায়:

```
Process (যেকোনো)
├── FD 0 ──► stdin  ← keyboard থেকে input আসে
├── FD 1 ──► stdout ← screen-এ output যায়
└── FD 2 ──► stderr ← screen-এ error যায়
```

### FD নিয়ে সবচেয়ে গুরুত্বপূর্ণ কথা

**FD মাত্র একটা নম্বর। Kernel জানে এই নম্বর কোথায় point করে। তুমি সেই point পরিবর্তন করতে পারো।**

এটাই pipe এবং redirection-এর পুরো রহস্য।

```
সাধারণ অবস্থায়:
FD 1 ──► terminal screen

তুমি যদি FD 1 কে বদলে file-এ point করাও:
FD 1 ──► myfile.txt

তখন program যা লিখবে FD 1-এ,
সেটা screen-এ না গিয়ে file-এ যাবে।
Program জানেও না — সে শুধু FD 1-এ লিখছে।

এটাই "ls > output.txt" এর পেছনের কাজ।
```

### একটা উদাহরণ দিয়ে বুঝি

`fmt.Println("hello")` লিখলে Go আসলে কী করে?

```
Go কোড:
fmt.Println("hello")
        │
        │ Go runtime translate করে
        ▼
Syscall:
write(fd=1, data="hello\n", length=6)
  │
  │ Kernel FD table দেখে:
  │ "FD 1 মানে terminal screen"
  ▼
Terminal screen-এ "hello" দেখায়
```

FD 1 যদি file-এ point করত, তাহলে "hello" file-এ যেত। Program-এর কোড একই থাকত।

### File খুললে নতুন FD তৈরি হয়

```go
f, _ := os.Open("notes.txt")
// f.Fd() = 3  ← kernel দিলো পরবর্তী খালি নম্বর

// এখন FD table:
// FD 0 → keyboard
// FD 1 → screen
// FD 2 → screen
// FD 3 → notes.txt  ← নতুন
```

File বন্ধ করলে FD 3 আবার খালি হয়।

### FD inherit হয় — Pipe-এর ভিত্তি

এই অংশটা একটু মনোযোগ দিয়ে পড়ো। Milestone 4-এ Pipe বানানোর সময় এটা না বুঝলে কিছুই বুঝবে না।

Child process জন্মের সময় parent-এর FD table-এর **হুবহু copy** পায়। মানে parent যা যা খুলে রেখেছে — keyboard, screen, যেকোনো file — child সেগুলো সব পেয়ে যায়।

```
fork() করার আগে:

Shell (parent, PID: 100)
├── FD 0 → /dev/pts/0  (keyboard)
├── FD 1 → /dev/pts/0  (screen)
└── FD 2 → /dev/pts/0  (screen)

fork() করার পরে:

Shell (parent, PID: 100)    Child (PID: 101)
├── FD 0 → keyboard    →    ├── FD 0 → keyboard  (copy)
├── FD 1 → screen      →    ├── FD 1 → screen    (copy)
└── FD 2 → screen      →    └── FD 2 → screen    (copy)
```

তাই `ls` terminal-এ output দিতে পারে — shell-এর FD 1 copy করে পেয়েছে, সেখানে লিখছে।

এখন প্রশ্ন হলো — **Pipe কীভাবে কাজ করে?**

`ls | grep .go` চালালে আমরা চাই ls-এর output সরাসরি grep-এর input হোক। কিন্তু ls তো FD 1-এ লেখে — সেটা screen-এ যায়। grep তো FD 0 থেকে পড়ে — সেটা keyboard থেকে আসে। তাহলে?

**উত্তর: fork()-এর আগেই FD বদলে দাও।**

```
পরিকল্পনা:

Step 1: একটা pipe বানাও
        pipe তৈরি হলে দুটো FD পাওয়া যায়:
        FD 3 = pipe-এর read end  (এখান থেকে পড়া যাবে)
        FD 4 = pipe-এর write end (এখানে লেখা যাবে)

Step 2: ls-এর জন্য fork() করার আগে
        ls-এর FD 1 (stdout) বদলে দাও:
        FD 1 → pipe write end (FD 4)
        এখন ls যা লিখবে সব pipe-এ যাবে, screen-এ না

Step 3: grep-এর জন্য fork() করার আগে
        grep-এর FD 0 (stdin) বদলে দাও:
        FD 0 → pipe read end (FD 3)
        এখন grep যা পড়বে সব pipe থেকে আসবে, keyboard থেকে না

Step 4: দুই process চালাও
        ls লেখে → pipe-এ জমা হয় → grep পড়ে → screen-এ আসে
```

ছবিতে দেখো:

```
fork() + FD বদলানোর পরে:

ls (child 1)                grep (child 2)
────────────                ─────────────
FD 0 → keyboard             FD 0 → pipe read  ← বদলানো হয়েছে
FD 1 → pipe write ← বদলানো FD 1 → screen
FD 2 → screen               FD 2 → screen

ls লেখে FD 1-এ             grep পড়ে FD 0 থেকে
    │                           ▲
    │                           │
    └──────── pipe ─────────────┘
              (kernel-এর memory buffer)

ls জানে না সে pipe-এ লিখছে।
grep জানে না সে pipe থেকে পড়ছে।
দুজনেই ভাবছে সাধারণ FD — কিন্তু মাঝখানে pipe।
```

**এটাই Unix-এর সবচেয়ে সুন্দর design।** Program-গুলো নিজেরা জানে না তারা কার সাথে কথা বলছে। Shell মাঝখান থেকে FD বদলে দিয়ে সব connect করে দেয়। Program-এর কোড একটুও বদলাতে হয় না।

exec() করার পরেও এই বদলানো FD গুলো থাকে। তাই:
- fork() → FD 1 বদলাও → exec("ls") → ls চলে, pipe-এ লেখে ✓
- fork() → FD 0 বদলাও → exec("grep") → grep চলে, pipe থেকে পড়ে ✓

Milestone 4-এ এই পুরো জিনিসটা code-এ লিখব। তখন এই ছবিটা মাথায় থাকলে code দেখে সব বুঝতে পারবে।

---

## ৫. Syscall — Kernel-এর সাথে কথা বলার ভাষা

User space থেকে kernel-এর কাছে request পাঠানোর নিয়মকানুনকে **System Call (syscall)** বলে।

### কেন syscall দরকার?

```
সরাসরি hardware access দিলে কী হতো?

firefox:  "আমাকে সব RAM দাও"          ← chaos!
ls:       "disk-এ যা খুশি লিখব"        ← dangerous!
malware:  "keyboard সব data চুরি করব"  ← security breach!

Kernel মাঝখানে থেকে সব control করে:
"তোমাকে এতটুকু RAM দিলাম"
"এই folder-এ লিখতে পারবে"
"keyboard input শুধু তোমার window পাবে"
```

### Shell-এ যে syscall গুলো লাগবে

```
┌─────────────────┬──────────────────────────────────┐
│ Syscall         │ কী করে                           │
├─────────────────┼──────────────────────────────────┤
│ fork()          │ নিজের copy তৈরি করো              │
│ exec()          │ নিজেকে অন্য program দিয়ে replace │
│ wait()          │ child শেষ হওয়ার জন্য অপেক্ষা করো │
│ exit()          │ process বন্ধ করো                  │
│ open()          │ file খোলো, FD পাও               │
│ read()          │ FD থেকে পড়ো                     │
│ write()         │ FD-তে লেখো                       │
│ close()         │ FD বন্ধ করো                      │
│ pipe()          │ pipe তৈরি করো, দুটো FD পাও       │
│ chdir()         │ current directory বদলাও           │
│ getcwd()        │ current directory জানো            │
└─────────────────┴──────────────────────────────────┘
```

Go-তে এগুলো directly লিখতে হয় না — `os.Chdir()`, `os.Pipe()` ইত্যাদি function আছে যেগুলো ভেতরে এই syscall করে।

---

## ৬. Shell আসলে কী করে?

Shell-এর কাজ মূলত চারটা ধাপের একটা চক্র:

```
┌──────────────────────────────────────────────────┐
│                  SHELL LOOP                      │
│                                                  │
│  ┌─────────┐                                     │
│  │  READ   │ ← prompt দেখাও, input পড়ো          │
│  └────┬────┘   "tiny-shell> ls -la | grep .go"   │
│       │                                          │
│       ▼                                          │
│  ┌─────────┐                                     │
│  │  PARSE  │ ← input-কে ভাঙো, মানে বোঝো         │
│  └────┬────┘   cmd="ls", args=["-la"]            │
│       │        pipe আছে, grep আছে               │
│       │                                          │
│       ▼                                          │
│  ┌─────────┐                                     │
│  │ EXECUTE │ ← process তৈরি করো, চালাও          │
│  └────┬────┘   fork → exec → wait                │
│       │                                          │
│       ▼                                          │
│  ┌─────────┐                                     │
│  │  LOOP   │ ← শেষ হলে আবার শুরু থেকে           │
│  └────┬────┘                                     │
│       │                                          │
│       └─────────────────────────────────────────►│
└──────────────────────────────────────────────────┘
```

Shell আসলে অনেক কিছু করে:

```
Shell-এর দায়িত্ব:
┌────────────────────────────────────────────────────┐
│                                                    │
│  Input পড়া      → keyboard থেকে line নেওয়া       │
│                                                    │
│  Parse করা      → "ls -la | grep .go" বুঝে        │
│                   দুটো command চেনা, pipe চেনা     │
│                                                    │
│  Built-in চেক   → cd, exit নিজে handle করা        │
│                   (এগুলো child process-এ হয় না)   │
│                                                    │
│  Process চালানো → fork + exec + wait               │
│                                                    │
│  Pipe লাগানো    → দুই process-এর মাঝে data path   │
│                                                    │
│  Signal handle  → Ctrl+C, Ctrl+Z ধরা              │
│                                                    │
│  Job track করা  → background process মনে রাখা     │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## ৭. Fork-Exec Model — সবচেয়ে গুরুত্বপূর্ণ Concept

তুমি shell-এ `ls` টাইপ করলে shell **নিজে** `ls` চালায় না। সে একটা child তৈরি করে, সেই child `ls` হয়ে যায়। এই পুরো প্রক্রিয়াটাকে **Fork-Exec Model** বলে।

তিনটা ধাপে হয়:

### ধাপ ১: fork() — নিজের copy তৈরি করো

```
fork() করার আগে:

Shell Process (PID: 100)
├── Code: shell-এর code
├── Stack: shell-এর variables
├── FD 0 → keyboard
├── FD 1 → screen
└── CWD: /home/shomi


fork() করার পরে — দুটো process:

Shell (PID: 100)          Child (PID: 101)
├── Code: shell copy  →   ├── Code: shell copy  (exact copy)
├── Stack: shell copy →   ├── Stack: shell copy (exact copy)
├── FD 0 → keyboard  →   ├── FD 0 → keyboard  (exact copy)
├── FD 1 → screen    →   ├── FD 1 → screen    (exact copy)
└── CWD: /home/shomi →   └── CWD: /home/shomi (exact copy)

দুটো process এখন identical!
```

fork() দুটো জায়গায় return করে:
- **Parent-এ**: child-এর PID return করে (101)
- **Child-এ**: 0 return করে

এই দিয়েই বোঝা যায় "আমি parent না child":

```go
pid := fork()

if pid > 0 {
    // আমি parent — child-এর PID পেলাম
    // wait করব
} else if pid == 0 {
    // আমি child — 0 পেলাম
    // exec করব
}
```

### ধাপ ২: exec() — নিজেকে replace করো

Child এখন shell-এর copy। সে `exec("/bin/ls")` করে নিজেকে `ls` দিয়ে replace করে:

```
exec() করার আগে (child):          exec() করার পরে (same child):

Child (PID: 101)                   Child (PID: 101)
├── Code: shell-এর code      →     ├── Code: ls-এর code   ← বদলে গেছে
├── Stack: shell-এর data    →     ├── Stack: নতুন, খালি  ← বদলে গেছে
├── FD 0 → keyboard          →     ├── FD 0 → keyboard    ← একই!
├── FD 1 → screen            →     ├── FD 1 → screen      ← একই!
└── CWD: /home/shomi         →     └── CWD: /home/shomi   ← একই!

PID বদলায়নি।
FD বদলায়নি — তাই ls screen-এ লিখতে পারে।
শুধু code এবং data বদলে গেছে।
```

**এটাই সবচেয়ে সুন্দর অংশ**: exec()-এর আগে child-এর FD বদলে দিলে, exec()-এর পরেও সেই FD থাকে। এভাবেই pipe কাজ করে — exec-এর আগে FD 1 কে pipe-এ point করাই, তারপর ls যা লেখে সব pipe-এ যায়।

### ধাপ ৩: wait() — parent অপেক্ষা করে

```
Shell (parent, PID: 100)              ls (child, PID: 101)
────────────────────────              ────────────────────
wait(101) call করলো                  চলছে...
    │                                 file list বানাচ্ছে
    │ ঘুমিয়ে পড়লো                    screen-এ দেখাচ্ছে
    │ (WAITING state)                     │
    │                                     │ কাজ শেষ
    │                                     │ exit(0)
    │                                     ▼
    │◄──────── kernel জানালো ────────── ZOMBIE হলো
    │          "child 101 শেষ"
    │
    ▼ wait() return করলো
Shell জেগে উঠলো
exit code পেলো (0 = সফল)
child DEAD হলো (zombie মুছে গেলো)
আবার prompt দেখালো
```

### পুরো ছবি একসাথে

```
তুমি "ls" টাইপ করলে Enter চাপলে:

    Shell (100)
        │
        │─────────────── fork() ───────────────────┐
        │                                           │
        │ (parent)                          (child — 101)
        │ wait(101)                                 │
        │ ঘুমালো                                    │
        │                               exec("/bin/ls")
        │                                           │
        │                               ls শুরু হলো
        │                               output দেখালো
        │                                           │
        │                               exit(0)
        │◄──────────────────────────────────────────┘
        │ জেগে উঠলো
        │
        ▼
    "tiny-shell ~> " দেখালো
```

---

# Part 2: কোড — Milestone 1

এখন concept জানা আছে। কোড দেখলে সব চেনা লাগবে।

---

## Project Setup

Terminal খুলে:

```bash
mkdir tiny-shell
cd tiny-shell
go mod init tiny-shell
mkdir -p cmd/shell internal/executor
```

Structure এরকম হবে:

```
tiny-shell/
├── go.mod
└── cmd/
    └── shell/
        └── main.go
└── internal/
    └── executor/
        └── executor.go
```

---

## File 1: `cmd/shell/main.go`

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"

	"tiny-shell/internal/executor"
)

func main() {
	// bufio.Scanner keyboard থেকে line-by-line পড়ে
	// os.Stdin = FD 0 = keyboard
	// (Part 1-এ FD দেখেছিলাম — এটাই সেই FD 0)
	scanner := bufio.NewScanner(os.Stdin)

	// ── REPL Loop শুরু ──────────────────────────────────
	for {

		// ┌─────────────────────────────────────────────┐
		// │  READ — prompt দেখাও, input পড়ো            │
		// └─────────────────────────────────────────────┘

		// fmt.Print — newline ছাড়া print করে
		// (fmt.Println করলে prompt-এর পরে নতুন line যেত)
		fmt.Print("tiny-shell> ")

		// scanner.Scan() একটা line পড়ে, Enter চাপলে return করে
		// false return করে শুধু দুটো ক্ষেত্রে:
		//   ১. Ctrl+D চাপলে (EOF — "input শেষ" signal)
		//   ২. কোনো error হলে
		if !scanner.Scan() {
			fmt.Println("\nবাই! 👋")
			break
		}

		// ┌─────────────────────────────────────────────┐
		// │  EVAL — input পরিষ্কার করো, বোঝো           │
		// └─────────────────────────────────────────────┘

		// scanner.Text() এইমাত্র পড়া line দেয়
		// strings.TrimSpace আগে-পিছের space, tab, newline কাটে
		// "  ls -la  " → "ls -la"
		line := strings.TrimSpace(scanner.Text())

		// খালি line? কিছু না করে আবার prompt দেখাও
		if line == "" {
			continue
		}

		// "exit" টাইপ করলে shell বন্ধ করো
		// os.Exit(0) — 0 মানে "সফলভাবে শেষ"
		// এটা syscall: exit_group(0)
		if line == "exit" {
			fmt.Println("বাই! 👋")
			os.Exit(0)
		}

		// ┌─────────────────────────────────────────────┐
		// │  EXECUTE — command চালাও                    │
		// └─────────────────────────────────────────────┘

		// executor.Execute() কে দায়িত্ব দিলাম
		// সে fork-exec করবে
		if err := executor.Execute(line); err != nil {
			// error সবসময় stderr-এ লেখো (convention)
			// os.Stderr = FD 2
			fmt.Fprintln(os.Stderr, "error:", err)
		}

		// ┌─────────────────────────────────────────────┐
		// │  LOOP — Go নিজেই আবার শুরুতে যায়           │
		// └─────────────────────────────────────────────┘
	}
}
```

---

## File 2: `internal/executor/executor.go`

```go
package executor

import (
	"fmt"
	"os"
	"os/exec"
	"strings"
)

// Execute — একটা raw input string নেয়, command চালায়
func Execute(line string) error {

	// ── Step 1: Input ভাঙো ──────────────────────────────
	//
	// strings.Fields() whitespace দিয়ে split করে
	// একাধিক space হলেও ঠিকঠাক কাজ করে:
	//
	// "ls   -la   /home" → ["ls", "-la", "/home"]
	// "ls"               → ["ls"]
	// "  "               → []  (empty)
	//
	// Milestone 3-এ proper parser দিয়ে replace করব
	// এখনকে জন্য এটাই যথেষ্ট
	parts := strings.Fields(line)

	if len(parts) == 0 {
		return nil
	}

	// parts[0] = command নাম  → "ls"
	// parts[1:] = arguments   → ["-la", "/home"]
	cmdName := parts[0]
	args    := parts[1:]

	return runCommand(cmdName, args)
}

// runCommand — আসল কাজ এখানে হয়
// fork() → exec() → wait() — তিনটাই এখানে
func runCommand(name string, args []string) error {

	// ── exec.Command ─────────────────────────────────────
	//
	// এটা এখনো কিছু চালায়নি।
	// শুধু একটা "plan" তৈরি করেছে — Cmd struct।
	//
	// ভেতরে কী করে:
	//   ১. PATH দেখে "name" কোথায় আছে খোঁজে
	//      "ls" → "/bin/ls"
	//   ২. full path, arguments সব Cmd-এ রাখে
	//   ৩. এখনো OS-কে কিছু বলেনি
	cmd := exec.Command(name, args...)

	// ── stdin/stdout/stderr connect করো ─────────────────
	//
	// Child process জন্মের সময় FD গুলো কোথায় যাবে
	// সেটা এখানে ঠিক করে দিচ্ছি।
	//
	// cmd.Stdin = os.Stdin মানে:
	//   child-এর FD 0 → আমাদের terminal keyboard
	//
	// cmd.Stdout = os.Stdout মানে:
	//   child-এর FD 1 → আমাদের terminal screen
	//
	// cmd.Stderr = os.Stderr মানে:
	//   child-এর FD 2 → আমাদের terminal screen
	//
	// এটা না করলে child কিছু print করতে পারত না।
	// (Milestone 4-এ pipe করার সময় এই stdout বদলাব)
	cmd.Stdin  = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// ── cmd.Run() — এখানেই fork-exec-wait হয় ───────────
	//
	// cmd.Run() আসলে তিনটা কাজ করে:
	//
	//   ১. fork() syscall:
	//      Kernel এই process-এর exact copy বানায়
	//      Parent = shell (চলতে থাকে)
	//      Child  = shell-এর copy (এখন ls হবে)
	//
	//   ২. exec() syscall (child-এ):
	//      Child নিজেকে "/bin/ls" দিয়ে replace করে
	//      ls-এর code RAM-এ আসে, চালু হয়
	//      FD গুলো একই থাকে (তাই screen-এ দেখাতে পারে)
	//
	//   ৩. wait() syscall (parent-এ):
	//      Shell ঘুমিয়ে পড়ে
	//      ls শেষ হলে kernel shell-কে জাগায়
	//      exit code পাওয়া যায়
	//
	// এই তিনটা একসাথে হয় cmd.Run()-এ
	err := cmd.Run()

	// ── Error Handle করো ─────────────────────────────────
	if err != nil {

		// *exec.ExitError:
		//   command চলেছে কিন্তু non-zero exit code দিয়েছে
		//   মানে command নিজে বলছে "আমি সফল হইনি"
		//   যেমন: "ls /nonexistent" → exit code 1
		if exitErr, ok := err.(*exec.ExitError); ok {
			return fmt.Errorf("%s: exit status %d",
				name, exitErr.ExitCode())
		}

		// exec.ErrNotFound:
		//   PATH-এ command খুঁজে পায়নি
		//   মানে এই নামের কোনো program নেই
		//   যেমন: "blahblah" টাইপ করলে
		if err == exec.ErrNotFound {
			return fmt.Errorf("%s: command not found", name)
		}

		return err
	}

	return nil
}
```

---

## চালাও

```bash
go run cmd/shell/main.go
```

```
tiny-shell> ls
executor.go  main.go

tiny-shell> pwd
/home/shomi/tiny-shell

tiny-shell> ls -la
total 16
drwxr-xr-x  4 shomi shomi 4096 ...

tiny-shell> whoami
shomi

tiny-shell> blahblah
error: blahblah: command not found

tiny-shell> ls /nonexistent
ls: cannot access '/nonexistent': No such file or directory
error: ls: exit status 2

tiny-shell> exit
বাই! 👋
```

---

## এখন ভেতরে উঁকি দিই

Shell চালু করার পর আরেকটা terminal খোলো:

```bash
pgrep -a tiny-shell

# ধরো PID হলো 1234
ls -la /proc/1234/fd
```

দেখবে:

```
lrwxrwxrwx ... 0 -> /dev/pts/0
lrwxrwxrwx ... 1 -> /dev/pts/0
lrwxrwxrwx ... 2 -> /dev/pts/0
```

FD 0, 1, 2 তিনটাই `/dev/pts/0` — তোমার terminal। Part 1-এ যা পড়েছিলে, এখন চোখের সামনে দেখা যাচ্ছে।

এখন tiny-shell-এ `sleep 10` চালাও। সেই ১০ সেকেন্ডে অন্য terminal-এ:

```bash
pstree -p $(pgrep tiny-shell)
```

দেখবে:

```
tiny-shell(1234)───sleep(1235)
```

shell → sleep — parent থেকে child জন্ম নিয়েছে। fork-exec চোখের সামনে।

---

## কোডের flow একবার দেখি

`ls -la` টাইপ করলে কী হয়:

```
main.go:
  scanner.Scan()          ← keyboard থেকে "ls -la" পড়লো
  line = "ls -la"
  executor.Execute(line)  ← দায়িত্ব দিলো

executor.go:
  strings.Fields("ls -la")   → ["ls", "-la"]
  cmdName = "ls"
  args    = ["-la"]
  runCommand("ls", ["-la"])

  exec.Command("ls", "-la")  ← plan তৈরি
  cmd.Stdin  = terminal      ← FD 0 connect
  cmd.Stdout = terminal      ← FD 1 connect
  cmd.Stderr = terminal      ← FD 2 connect

  cmd.Run():
    fork()  → child তৈরি (PID 1235)
    exec()  → child এখন ls
    wait()  → shell ঘুমালো

  ls চললো, output দেখালো
  ls exit(0) করলো
  wait() return করলো
  shell জেগে উঠলো

main.go:
  আবার loop শুরু
  "tiny-shell> " দেখালো
```

---

## 📋 Milestone 1 Checklist

- ✅ REPL loop চলছে
- ✅ Fork-exec কাজ করছে
- ✅ Error handle হচ্ছে
- ✅ FD connect হচ্ছে
- ✅ Process tree চোখে দেখলাম
