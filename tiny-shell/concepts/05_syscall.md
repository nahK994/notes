## Syscall: Kernel-এর সাথে কথা বলার একমাত্র ভাষা

তোমার প্রোগ্রাম যখন চলে, সে আসলে একটা খাঁচার মধ্যে থাকে।

নিজের কিছু RAM আছে, নিজের কিছু FD আছে — এর বাইরে সে কিছু ছুঁতে পারে না। Disk-এ সরাসরি লিখতে পারে না। Network-এ সরাসরি packet পাঠাতে পারে না। এমনকি screen-এ কিছু দেখাতে হলেও তাকে অনুমতি নিতে হয়।

এই অনুমতি চাওয়ার প্রক্রিয়াটার নাম **System Call** — সংক্ষেপে **Syscall**।

---

### কেন এই খাঁচা দরকার?

কারণটা বুঝতে কল্পনা করো — খাঁচা না থাকলে কী হতো:

```
খাঁচা না থাকলে:

firefox:  "আমাকে সব RAM দাও"           ← পুরো system crash
ls:       "disk-এ যা খুশি লিখবো"        ← যেকোনো file নষ্ট
malware:  "keyboard-এর সব data চুরি"    ← কেউ টের পাবে না
```

Kernel মাঝখানে দাঁড়িয়ে এই বিশৃঙ্খলা ঠেকায়:

```
Kernel-এর উত্তর:

firefox-কে: "তোমার জন্য এতটুকু RAM বরাদ্দ"
ls-কে:      "শুধু এই folder-এ access পাবে"
malware-কে: "keyboard input? না।"
```

এটাকে বলে **Privilege Separation** — user space আর kernel space আলাদা রাখা। তোমার প্রোগ্রাম সবসময় user space-এ থাকে। Kernel space-এ যেতে হলে — syscall করতে হবে।

---

### Syscall আসলে কীভাবে কাজ করে?

```
তোমার প্রোগ্রাম                    Kernel
──────────────                    ──────
                                  
  write(fd=1,                     
   "hello", 6)  ──── trap ──────► permission check
                                  ↓
                                  "FD 1 মানে screen,
                                   লেখার অনুমতি আছে"
                                  ↓
                ◄────────────────  screen-এ লিখলো
  আবার চলা শুরু
```

মাঝখানের `trap` ধাপটা গুরুত্বপূর্ণ। Syscall করা মানে প্রোগ্রাম CPU-কে বলছে — *"এখন তুমি kernel-এর কোড চালাও, আমি অপেক্ষা করি।"* Kernel কাজ শেষ করে control ফিরিয়ে দেয়। প্রোগ্রাম আবার চলে।

এই কারণে Syscall একটু slow — user space থেকে kernel space-এ যাওয়া-আসার খরচ আছে। তাই ভালো প্রোগ্রাম অপ্রয়োজনীয় syscall এড়িয়ে চলে।

---

### Shell-এ যে Syscall গুলো লাগবে

Shell বানাতে গিয়ে তুমি মূলত এই syscall গুলোর সাথে পরিচিত হবে:

```
┌──────────────┬──────────────────────────────────────┐
│ Syscall      │ কী করে                               │
├──────────────┼──────────────────────────────────────┤
│ fork()       │ নিজের হুবহু copy তৈরি করে            │
│ exec()       │ নিজেকে অন্য program দিয়ে replace করে │
│ wait()       │ child শেষ হওয়ার জন্য অপেক্ষা করে     │
│ exit()       │ process বন্ধ করে, status জানায়        │
├──────────────┼──────────────────────────────────────┤
│ open()       │ file খোলে, একটা FD দেয়               │
│ read()       │ FD থেকে data পড়ে                     │
│ write()      │ FD-তে data লেখে                       │
│ close()      │ FD বন্ধ করে, table থেকে সরায়          │
├──────────────┼──────────────────────────────────────┤
│ pipe()       │ pipe তৈরি করে, দুটো FD দেয়            │
│ dup2()       │ একটা FD-কে অন্য নম্বরে copy করে      │
├──────────────┼──────────────────────────────────────┤
│ chdir()      │ current directory বদলায়               │
│ getcwd()     │ current directory কোনটা জানায়         │
├──────────────┼──────────────────────────────────────┤
│ kill()       │ process-এ signal পাঠায়                │
│ signal()     │ signal আসলে কী করবে সেটা ঠিক করে    │
└──────────────┴──────────────────────────────────────┘
```

তিনটা গ্রুপ লক্ষ্য করো:

**Process group** — `fork`, `exec`, `wait`, `exit` দিয়ে process তৈরি করবে, চালাবে, শেষ করবে।

**File group** — `open`, `read`, `write`, `close` দিয়ে file আর FD নিয়ে কাজ করবে। `dup2` দিয়ে FD redirect করবে — এটাই pipe-এর মূল কৌশল।

**Control group** — `chdir` দিয়ে `cd` implement করবে। `kill` আর `signal` দিয়ে Ctrl+C handle করবে।

---

### Go-তে Syscall লেখো না, Wrapper ব্যবহার করো

Go-তে সরাসরি syscall number লিখতে হয় না। Standard library-তে সুন্দর wrapper আছে:

```
Syscall          Go Wrapper
────────         ──────────
fork() + exec()  → os/exec package, syscall.ForkExec()
wait()           → cmd.Wait()
open()           → os.Open()
read()           → file.Read()
write()          → file.Write() অথবা fmt.Println()
close()          → file.Close()
pipe()           → os.Pipe()
dup2()           → syscall.Dup2()
chdir()          → os.Chdir()
getcwd()         → os.Getwd()
kill()           → process.Signal()
```

কিন্তু wrapper ব্যবহার করার সময়ও জানা দরকার — ভেতরে কী হচ্ছে। তাহলে কিছু ভুল হলে কোথায় খুঁজতে হবে সেটা বুঝবে।

---

### একটা কথা মনে রাখো

Syscall হলো তোমার প্রোগ্রাম আর OS-এর মধ্যে **চুক্তিপত্র**। এই চুক্তি দশকের পর দশক ধরে একই আছে। Linux-এ আজকে যে `fork()` syscall হয়, সেটা ১৯৯১ সালেও একইভাবে হতো।

Shell থেকে শুরু করে browser, database, game engine — সব কিছু শেষ পর্যন্ত এই একই syscall-এ এসে ঠেকে। ভিত্তিটা একটাই।

পরের অধ্যায়ে দেখবো `fork()` আর `exec()` মিলে কীভাবে একটা নতুন process জন্ম নেয়।