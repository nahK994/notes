# অধ্যায় ৪: The Abstraction: The Process

---

## 🔰 ভূমিকা

কম্পিউটারে আমরা একসাথে অনেক প্রোগ্রাম চালাই — browser, music player, text editor সব একসাথে চলে। কিন্তু CPU তো একটাই (বা কয়েকটা)! তাহলে এত প্রোগ্রাম একসাথে কীভাবে চলে?

এর উত্তর হলো — Operating System একটি বিশেষ কৌশল ব্যবহার করে যার নাম **Virtualization**।

OS মূলত CPU-কে এমনভাবে ভাগ করে দেয় যে মনে হয় **প্রতিটি প্রোগ্রাম তার নিজস্ব CPU পাচ্ছে।** এই illusion তৈরি করতে OS যে abstraction ব্যবহার করে, তার নাম হলো **Process।**

সহজ কথায়: **Process = একটি চলমান প্রোগ্রাম।**

---

## ১. Process কী?

একটি প্রোগ্রাম যখন disk-এ থাকে, সেটা শুধু একগুচ্ছ instruction — নিষ্প্রাণ। কিন্তু যখন সেটা চালানো হয়, OS সেটাকে memory-তে নিয়ে আসে এবং চালু করে — তখন সেটা হয়ে যায় একটি **Process।**

একটি Process-এর মধ্যে থাকে:

### ক) Memory (Address Space)
Process-এর নিজস্ব memory থাকে যেখানে থাকে:
- **Code (Instructions):** প্রোগ্রামের যত instruction আছে
- **Data:** Global variable-গুলো
- **Stack:** Function call, local variable, return address
- **Heap:** Runtime-এ dynamically allocate করা memory (যেমন `malloc`)

### খ) Registers
CPU-র ভেতরে ছোট ছোট storage আছে যাকে register বলে। Process চলার সময় এগুলোতে তথ্য থাকে:

- **Program Counter (PC):** এখন কোন instruction চলছে
- **Stack Pointer:** Stack-এর বর্তমান অবস্থান
- **General Purpose Registers:** অস্থায়ী হিসাবের জন্য

### গ) I/O Information
Process কোন কোন file খুলে রেখেছে, কোন device ব্যবহার করছে — এই তথ্যও process-এর অংশ।

---

## ২. Process-এর জীবনচক্র — Machine State

একটি Process চলার সময় OS তার সম্পর্কে যা যা তথ্য রাখে, সেটাকে একসাথে বলা হয় **Machine State।**

Machine State-এর মূল উপাদান:

```
┌─────────────────────────────┐
│         Machine State        │
│                             │
│  ┌─────────┐  ┌──────────┐  │
│  │ Memory  │  │Registers │  │
│  │(Address │  │  PC, SP  │  │
│  │ Space)  │  │  etc.    │  │
│  └─────────┘  └──────────┘  │
│                             │
│  ┌──────────────────────┐   │
│  │   I/O Information    │   │
│  │  (open files, etc.)  │   │
│  └──────────────────────┘   │
└─────────────────────────────┘
```

---

## ৩. Process তৈরি হয় কীভাবে?

OS একটি program থেকে process তৈরি করে কয়েকটি ধাপে:

### ধাপ ১: Code ও Data Load করা
Program-এর code ও static data disk থেকে পড়ে memory-তে নিয়ে আসা হয়।

পুরনো OS-এ পুরো program একবারে load হতো। আধুনিক OS-এ **Lazy Loading** ব্যবহার হয় — মানে যতটুকু দরকার ততটুকুই প্রথমে load হয়, বাকিটা পরে দরকার হলে।

### ধাপ ২: Stack তৈরি করা
Program-এর জন্য stack memory তৈরি হয়। C program-এ stack ব্যবহার হয় local variable, function parameter ও return address রাখতে।

OS `main()` function-এর জন্য `argc` ও `argv` দিয়ে stack তৈরি করে।

### ধাপ ৩: Heap তৈরি করা
Heap হলো dynamically allocated memory-র জায়গা। C-তে `malloc()` দিয়ে এখান থেকে memory নেওয়া হয় এবং `free()` দিয়ে ফেরত দেওয়া হয়।

শুরুতে heap ছোট থাকে, দরকার হলে OS বাড়িয়ে দেয়।

### ধাপ ৪: অন্যান্য কাজ
OS কিছু initialization করে — যেমন standard input (stdin), standard output (stdout), ও standard error (stderr) খুলে দেওয়া।

### ধাপ ৫: main() থেকে শুরু
সব প্রস্তুতি শেষে OS `main()` function-এ execution শুরু করে। এখন process চলছে!

---

## ৪. Process-এর অবস্থা (Process States)

একটি process সবসময় চলে না। বিভিন্ন সময়ে সে বিভিন্ন **অবস্থায়** থাকে:

### তিনটি মূল অবস্থা:

**১. Running (চলছে)**
Process এই মুহূর্তে CPU-তে execute হচ্ছে। Instruction চলছে।

**২. Ready (প্রস্তুত)**
Process চলার জন্য সম্পূর্ণ প্রস্তুত, কিন্তু OS এখনো তাকে CPU দেয়নি। অপেক্ষায় আছে।

**৩. Blocked (আটকে আছে)**
Process কিছু একটার জন্য অপেক্ষা করছে — যেমন disk থেকে data আসার জন্য বা keyboard input-এর জন্য। এই সময় CPU-তে থাকলে CPU নষ্ট হবে, তাই OS তাকে সরিয়ে দেয়।

### অবস্থার পরিবর্তন:

```
                  ┌─────────────────┐
         Scheduled│                 │Descheduled
         (OS দিল) │                 │ (OS সরিয়ে নিল)
                  ▼                 │
  ┌───────┐    ┌─────────┐    ┌─────┴───┐
  │       │───▶│ Running │───▶│  Ready  │
  │Blocked│    └─────────┘    └─────────┘
  │       │         │
  └───────┘         │ I/O শুরু হলে
       ▲             │ (Blocked হয়)
       │             ▼
       └──── I/O শেষ হলে
             (Ready হয়)
```

### বাস্তব উদাহরণ:

ধরো তুমি একটি program চালাচ্ছ যেটা file পড়ে:

1. Program শুরু হলো → **Running**
2. File পড়ার request দিল → **Blocked** (disk থেকে data আসতে সময় লাগবে)
3. এই ফাঁকে OS অন্য process চালায়
4. Disk থেকে data এলো → **Ready**
5. OS আবার এই process-কে CPU দিল → **Running**

এভাবে OS অপেক্ষার সময়টা নষ্ট না করে অন্য কাজ করে।

---

## ৫. দুটো Process একসাথে কীভাবে চলে? — বাস্তব উদাহরণ

ধরো দুটো process আছে — **P0** ও **P1**। P0 একটু পরে I/O করবে।

| সময় | P0 এর অবস্থা | P1 এর অবস্থা | কে CPU-তে |
|------|-------------|-------------|-----------|
| 0 | Running | Ready | P0 |
| 1 | Running | Ready | P0 |
| 2 | Blocked (I/O শুরু) | Running | P1 |
| 3 | Blocked | Running | P1 |
| 4 | Ready (I/O শেষ) | Running | P1 |
| 5 | Running | Ready | P0 |
| 6 | Running | Ready | P0 |

এখানে দেখো — P0 যখন I/O-তে ব্যস্ত, P1 CPU ব্যবহার করছে। কেউ বসে নেই, CPU সবসময় কাজ করছে। এটাই **CPU Virtualization-এর মূল কৌশল।**

---

## ৬. OS কীভাবে Process-এর তথ্য রাখে? — PCB

OS প্রতিটি process সম্পর্কে তথ্য রাখে একটি ডেটা স্ট্রাকচারে যার নাম **Process Control Block (PCB)** বা **Process Descriptor।**

PCB-তে থাকে:
- Process ID (PID)
- Process-এর বর্তমান অবস্থা (Running/Ready/Blocked)
- Program Counter ও অন্যান্য registers-এর মান
- Memory সম্পর্কিত তথ্য
- I/O সম্পর্কিত তথ্য
- এবং আরও অনেক কিছু

### xv6 OS-এ PCB দেখতে কেমন (উদাহরণ):

```c
struct proc {
    char *mem;          // process-এর memory শুরুর address
    uint sz;            // memory-র আকার
    char *kstack;       // kernel stack
    enum procstate state; // Running/Ready/Blocked/ইত্যাদি
    int pid;            // Process ID
    struct proc *parent;// parent process
    struct trapframe *tf; // register-এর তথ্য
    struct context *context; // context switch-এর জন্য
    struct file *ofile[NOFILE]; // open করা files
    char name[16];      // process-এর নাম
};
```

OS-এ সব process-এর PCB মিলিয়ে একটি **Process List** তৈরি হয়। OS এই list দেখে সিদ্ধান্ত নেয় কোন process-কে কখন CPU দেবে।

---

## ৭. Context Switch — এক Process থেকে অন্যটায় যাওয়া

OS যখন এক process থামিয়ে অন্য process চালু করে, তখন তাকে বলে **Context Switch।**

এটা কীভাবে হয়?

**ধাপ ১:** চলমান process-এর সব register-এর মান তার PCB-তে সেভ করা হয়।

**ধাপ ২:** পরবর্তী process-এর PCB থেকে তার register-এর মান CPU-তে লোড করা হয়।

**ধাপ ৩:** নতুন process চলতে শুরু করে।

এই পুরো প্রক্রিয়া এত দ্রুত হয় যে মনে হয় সব process একসাথে চলছে — যদিও আসলে একটার পর একটা চলছে।

---

## ৮. আরও কিছু Process State

শুধু Running/Ready/Blocked ছাড়াও বাস্তবে আরও কিছু অবস্থা থাকে:

**Initial (জন্ম নিচ্ছে):**
Process তৈরি হচ্ছে কিন্তু এখনো চলার জন্য সম্পূর্ণ প্রস্তুত না।

**Zombie (মরে গেছে কিন্তু পুরোপুরি না):**
Process তার কাজ শেষ করেছে কিন্তু parent এখনো তার exit status পড়েনি। এই অবস্থায় process পুরোপুরি মুছে যায় না — শুধু exit code ধরে রাখে যতক্ষণ না parent `wait()` করে।

```
     ┌─────────┐
     │ Initial │ (তৈরি হচ্ছে)
     └────┬────┘
          │
     ┌────▼────┐
     │  Ready  │◄──────────────┐
     └────┬────┘               │
          │ Scheduled          │ Descheduled
     ┌────▼────┐               │
     │ Running │───────────────┘
     └────┬────┘
          │ I/O বা event-এর অপেক্ষা
     ┌────▼────┐
     │ Blocked │
     └────┬────┘
          │ I/O বা event শেষ
          │
     ┌────▼────┐
     │  Zombie │ (কাজ শেষ, parent-এর অপেক্ষা)
     └─────────┘
```

---

## ৯. মূল শিক্ষা (Summary)

| বিষয় | মূল কথা |
|---|---|
| Process | চলমান প্রোগ্রাম। Code + Data + Stack + Heap + Registers |
| Virtualization | OS একটি CPU-কে অনেক virtual CPU-র মতো দেখায় |
| Process States | Running, Ready, Blocked (এবং Initial, Zombie) |
| PCB | OS প্রতিটি process-এর তথ্য যেখানে রাখে |
| Context Switch | এক process থেকে অন্য process-এ যাওয়া |
| Blocked কেন? | I/O বা অন্য কিছুর অপেক্ষায় থাকলে CPU নষ্ট না করতে |
