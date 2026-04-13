## I/O Redirection: Program-এর চোখ-কান বদলে দাও

একটা ছোট্ট পরীক্ষা করো।

Terminal খোলো, লেখো `ls`. File-এর তালিকা screen-এ দেখাবে। এখন লেখো `ls > output.txt`. কিছুই দেখাবে না। কিন্তু `cat output.txt` করলে দেখবে — সেই একই তালিকা, এবার file-এ জমা আছে।

`ls` প্রোগ্রামটা দুইবারই একই কাজ করেছে। একই code, একই logic। কিন্তু একবার output গেছে screen-এ, আরেকবার গেছে file-এ।

`ls` কি জানে পার্থক্যটা? না। `ls` জানেই না তার output কোথায় যাচ্ছে।

তাহলে কে বদলালো? **Shell।**

এটাই I/O Redirection।

---

### মূল ধারণা: FD-এর destination বদলানো

আগের chapter-এ দেখেছো — প্রতিটা process তিনটা FD নিয়ে জন্মায়:

```
FD 0 → stdin  (keyboard থেকে পড়ে)
FD 1 → stdout (screen-এ লেখে)
FD 2 → stderr (screen-এ error লেখে)
```

I/O Redirection মানে হলো এই FD গুলোর destination বদলে দেওয়া। Program একই FD-তে লিখছে, কিন্তু সেই FD এখন অন্য কোথাও point করছে।

```
সাধারণ অবস্থায়:          Redirection-এর পরে:

FD 1 ──► screen           FD 1 ──► output.txt

program লিখছে FD 1-এ,    program লিখছে FD 1-এ,
screen-এ দেখাচ্ছে         file-এ জমছে

Program জানে না।          Program জানে না।
```

শুধু একটা নম্বর বদলে গেছে — FD 1 এখন কোথায় point করছে। বাকি সব একই।

---

### চার ধরনের Redirection

#### ১. Output Redirection ( `>` )

```bash
ls > output.txt
```

`ls`-এর stdout (FD 1) কে screen থেকে সরিয়ে `output.txt`-এ নিয়ে যাও।

```
আগে:   FD 1 ──► screen
পরে:   FD 1 ──► output.txt
```

File না থাকলে তৈরি হয়। আগে থেকে থাকলে **সম্পূর্ণ মুছে** নতুন করে লেখে। পুরনো data যায়।

#### ২. Append Redirection ( `>>` )

```bash
ls >> output.txt
```

`>` এর মতোই, কিন্তু পুরনো data মুছে না। নতুন output পুরনোর **নিচে** যোগ হয়।

```
output.txt আগে:    output.txt পরে:
main.go             main.go
README.md           README.md
                    utils.go    ← নতুন যোগ হলো
                    go.mod      ← নতুন যোগ হলো
```

Log file লেখার সময় এটা কাজে লাগে — প্রতিবার চালালে নতুন entry যোগ হয়, পুরনোটা থাকে।

#### ৩. Input Redirection ( `<` )

```bash
sort < names.txt
```

`sort`-এর stdin (FD 0) কে keyboard থেকে সরিয়ে `names.txt`-এ নিয়ে যাও।

```
আগে:   FD 0 ──► keyboard
পরে:   FD 0 ──► names.txt
```

`sort` ভাবছে সে keyboard থেকে পড়ছে। আসলে পড়ছে file থেকে। `sort` জানেও না।

#### ৪. Stderr Redirection ( `2>` )

```bash
ls /nonexistent 2> error.txt
```

FD 2 (stderr) কে file-এ নিয়ে যাও। Normal output screen-এ থাকে, শুধু error গুলো file-এ জমে।

```
FD 1 ──► screen      (normal output এখানেই থাকলো)
FD 2 ──► error.txt   (error গেলো file-এ)
```

এটা কেন দরকার? কল্পনা করো একটা বড় script রাত্রে চলছে। সকালে উঠে দেখতে চাও কী কী error হয়েছিলো — `2> errors.log` করে রাখলেই হলো।

---

### Shell কীভাবে Redirection করে?

Pipe-এর মতোই। `fork()` আর `exec()`-এর মাঝের সেই ছোট্ট ফাঁকে `dup2()` দিয়ে FD বদলে দেয়।

`ls > output.txt` চালালে Shell ধাপে ধাপে কী করে:

**ধাপ ১ — file খোলো:**

```
Shell:
os.OpenFile("output.txt") → FD 3 পেলো

Shell-এর FD table এখন:
  FD 0 → keyboard
  FD 1 → screen
  FD 2 → screen
  FD 3 → output.txt   ← নতুন
```

**ধাপ ২ — fork() করো:**

```
Shell → fork() → ls-child জন্মালো

ls-child এখন shell-এর হুবহু copy:
  FD 0 → keyboard
  FD 1 → screen       ← এটা বদলাতে হবে
  FD 2 → screen
  FD 3 → output.txt
```

**ধাপ ৩ — fork() আর exec()-এর মাঝে dup2() করো:**

```
ls-child এখনো চলছে, exec() হয়নি।
এই মুহূর্তে dup2(3, 1) করো:

  FD 0 → keyboard
  FD 1 → output.txt   ← screen থেকে file-এ গেলো
  FD 2 → screen
  FD 3 → output.txt   ← এটা আর লাগবে না, বন্ধ করো
```

**ধাপ ৪ — exec() করো:**

```
ls-child → exec("/bin/ls")

ls চলছে। FD 1-এ লিখছে।
FD 1 এখন output.txt।
ls-এর সব output file-এ যাচ্ছে।
ls জানেও না।
```

পুরো flow:

```
Shell
  │
  ├─ output.txt খোলো ──────────────── FD 3 পেলো
  │
  ├─ fork() ──────────────────────► ls-child
  │                                    │
  │                              dup2(FD3, FD1)
  │                              FD 1 → output.txt
  │                              FD 3 → বন্ধ করো
  │                                    │
  │                              exec("/bin/ls")
  │                                    │
  │                              ls লিখছে FD 1-এ
  │                              সব output.txt-এ যাচ্ছে
  │
  └─ wait() ── ls শেষ হলে prompt দেখালো
```

---

### একসাথে দুটো Redirection

```bash
ls > output.txt 2> error.txt
```

FD 1 যাবে `output.txt`-এ, FD 2 যাবে `error.txt`-এ।

```
FD 1 ──► output.txt   (normal output)
FD 2 ──► error.txt    (শুধু error)
```

Shell দুটো file-ই আলাদা করে খোলে, আলাদা করে `dup2()` করে।

একটা বিশেষ shortcut আছে — `2>&1`:

```bash
ls > output.txt 2>&1
```

এর মানে — *"FD 2 কে FD 1 যেখানে point করছে সেখানে নিয়ে যাও।"*

```
dup2(1, 2) করো:

FD 1 ──► output.txt
FD 2 ──► output.txt   ← FD 1-এর destination copy হলো
```

এখন stdout আর stderr দুটোই একই file-এ। সব output এক জায়গায় জমবে।

---

### Redirection আর Pipe একসাথে

Redirection আর Pipe আলাদা জিনিস না। দুটোই একই কাজ করে — FD-এর destination বদলায়। তাই একসাথে ব্যবহার করা যায়:

```bash
cat server.log 2>/dev/null | grep ERROR > errors.txt
```

```
cat server.log
  │  FD 2 → /dev/null    (cat-এর error গুলো ফেলে দাও)
  │  FD 1 → pipe         (normal output pipe-এ যাক)
  │
  ▼
[ PIPE ]
  │
  ▼
grep ERROR
  │  FD 0 → pipe         (pipe থেকে পড়ো)
  │  FD 1 → errors.txt   (match হলে file-এ লেখো)
  ▼
errors.txt
```

`/dev/null` একটা বিশেষ file — এখানে যা পাঠানো হয় তা চিরতরে হারিয়ে যায়। Error দেখতে না চাইলে `2>/dev/null` করো, সব নিঃশব্দে গায়েব হয়ে যাবে।

---

### পুরো ব্যাপারটা আসলে একটাই জিনিস

Pipe আর Redirection আলাদা feature মনে হয়। কিন্তু ভেতরে দুটো একই কাজ করে:

```
Redirection:   FD 1 ──► file
Pipe:          FD 1 ──► অন্য process-এর FD 0
```

দুটো ক্ষেত্রেই Shell `fork()`-এর পরে `exec()`-এর আগে `dup2()` দিয়ে FD-এর destination বদলে দিচ্ছে। Program জানছে না কিছু। সে শুধু FD-তে লিখছে বা পড়ছে।

এটাই Unix-এর elegance। একটাই ধারণা — FD — দিয়ে keyboard, screen, file, pipe, network সব কিছু একভাবে handle করা যায়।

Program-এর কাছে সব একই রকম। শুধু একটা নম্বর। বাকিটা Kernel জানে।