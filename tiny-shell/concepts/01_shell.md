# Shell: অপারেটিং সিস্টেমের সাথে কথা বলার ভাষা

কল্পনা করো তুমি একটা বিদেশী দেশে গেছো। ভাষা জানো না। তুমি যা চাও সেটা বলতে পারছো না, ওরা বুঝছে না। এখন যদি একজন দোভাষী থাকতো — তুমি বাংলায় বলো, সে অনুবাদ করে দেয় — কাজটা কত সহজ হয়ে যেতো।

**Shell হলো সেই দোভাষী।**

তুমি মানুষের ভাষায় বলো `ls`, shell সেটাকে OS-এর ভাষায় অনুবাদ করে পাঠায়, OS ফাইলের তালিকা দেয়, shell সেটা তোমার স্ক্রিনে দেখায়। Shell না থাকলে? Machine code লিখতে হতো।

```
তুমি → "ls টাইপ করলাম"
  ↓
Shell → বুঝলো, অনুবাদ করলো
  ↓
OS (Kernel) → "file list দাও" → দিলো
  ↓
Shell → তোমার স্ক্রিনে দেখালো
```

একটা গুরুত্বপূর্ণ কথা — **Shell কোনো magic না।** এটা নিজেও একটা সাধারণ প্রোগ্রাম, ঠিক `ls` বা `grep`-এর মতোই। পার্থক্য একটাই: এটা **অন্য প্রোগ্রাম চালানোর** কাজ করে।

---

## Shell আসলে ভেতরে কী করে?

Shell-এর কাজ বুঝতে হলে একটা ধারণা মাথায় রাখো — Shell একটা **অবিরাম চলতে থাকা loop।**

তুমি command দাও, সে চালায়, আবার তোমার দিকে তাকিয়ে থাকে। এই চক্রের একটা নাম আছে: **REPL — Read, Eval, Print, Loop।**

```
┌─────────────────────────────────────────────────────┐
│                    SHELL LOOP                       │
│                                                     │
│  ┌─────────┐                                        │
│  │  READ   │  ← prompt দেখাও, input পড়ো             │
│  └────┬────┘    "$ ls -la | grep .go"               │
│       │                                             │
│       ▼                                             │
│  ┌─────────┐                                        │
│  │  PARSE  │  ← input ভাঙো, মানে বোঝো               │
│  └────┬────┘    cmd="ls", args=["-la"]              │
│       │         pipe আছে, grep আছে                 │
│       ▼                                             │
│  ┌─────────┐                                        │
│  │ EXECUTE │  ← process তৈরি করো, চালাও             │
│  └────┬────┘    fork → exec → wait                  │
│       │                                             │
│       ▼                                             │
│  ┌─────────┐                                        │
│  │  LOOP   │  ← শেষ হলে আবার শুরু থেকে               │
│  └────┬────┘                                        │
│       └────────────────────────────────────────────►│
└─────────────────────────────────────────────────────┘
```

প্রতিটা ধাপে Shell বেশ কিছু কাজ করে:

```
Shell-এর দায়িত্ব:
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Input পড়া       → keyboard থেকে line নেওয়া          │
│                                                      │
│  Parse করা       → "ls -la | grep .go" বুঝে           │
│                    দুটো command চেনা, pipe চেনা       │
│                                                      │
│  Built-in চেক    → cd, exit নিজে handle করা          │
│                    (এগুলো child process-এ হয় না)     │
│                                                      │
│  Process চালানো  → fork + exec + wait                │
│                                                      │
│  Pipe লাগানো     → দুই process-এর মাঝে data path      │
│                                                      │
│  Signal handle   → Ctrl+C, Ctrl+Z ধরা                │
│                                                      │
│  Job track করা   → background process মনে রাখা       │
│                                                      │
└──────────────────────────────────────────────────────┘
```

একটু থামো। **Built-in command** মানে কী?

`cd` বা `exit` — এগুলো আলাদা কোনো প্রোগ্রাম না। এগুলো Shell-এর নিজের ভেতরে গেঁথে রাখা command। কারণটা সহজ: `cd` যদি child process-এ চলতো, সে নিজের directory বদলাতো, তারপর মরে যেতো — তোমার shell-এর directory একই থাকতো। তাই এই বিশেষ command গুলো Shell নিজেই সরাসরি চালায়।

---

## fork → exec → wait: Shell-এর তিনটা জাদুর ধাপ

যখন তুমি `ls` লেখো, Shell আসলে তিনটা OS system call করে:

**`fork()`** — Shell নিজের একটা হুবহু copy তৈরি করে। এই copy-টাকে বলে child process।

**`exec()`** — child process নিজেকে মুছে ফেলে `ls` প্রোগ্রাম দিয়ে replace করে।

**`wait()`** — parent shell অপেক্ষা করে, `ls` শেষ না হওয়া পর্যন্ত নতুন prompt দেয় না।

```
Shell (parent)
    │
    ├── fork() ──► Shell-এর copy (child)
    │                    │
    │              exec("ls") ──► ls চলে, output দেয়
    │                    │
    └── wait() ◄── child মরে যায়
    │
    ▼
আবার prompt দেখায়
```

এই তিনটা ধাপ না বুঝলে Shell-এর কিছুই বোঝা যায় না। পরে যখন নিজে Shell বানাবে, এই তিনটা function-ই হবে তোমার মূল হাতিয়ার।

---

## Shell কত প্রকার?

Shell মূলত দুই ভাগ।

### Command-line Shell (CLI)

Text-based। তুমি টাইপ করো, সে চালায়।

| Shell        | OS                     | বৈশিষ্ট্য                               |
|--------------|------------------------|------------------------------------------|
| `bash`       | Linux, macOS (পুরনো)   | সবচেয়ে common, scripting-এ শক্তিশালী   |
| `zsh`        | macOS (Catalina+)      | plugin system, autocomplete, থিম         |
| `fish`       | Linux, macOS           | syntax highlight, সহজ scripting          |
| `sh`         | Unix, Linux            | POSIX standard, সবচেয়ে portable         |
| `PowerShell` | Windows, Linux, macOS  | object-based pipeline, .NET integration  |
| `cmd.exe`    | Windows                | পুরনো, সীমিত, batch scripting            |

### Graphical Shell (GUI)

Desktop environment — window, icon, mouse সব এর অংশ।

| Shell            | OS      |
|------------------|---------|
| Windows Explorer | Windows |
| GNOME Shell      | Linux   |
| KDE Plasma       | Linux   |
| Aqua             | macOS   |

> তুমি এতদিন যে Desktop ব্যবহার করেছো সেটাও একটা Shell — শুধু graphical।

### কোন OS-এ কোন Shell?

| Operating System | Default Shell                                       |
|------------------|-----------------------------------------------------|
| Ubuntu / Debian  | `bash`                                              |
| macOS (Catalina+)| `zsh` — Apple ২০১৯ সালে `bash` থেকে switch করেছে  |
| macOS (পুরনো)    | `bash`                                              |
| Windows 10/11    | `PowerShell`, `cmd.exe`, WSL-এ `bash`               |
| Android          | `sh` (mksh variant)                                 |
| Embedded Linux   | `sh` বা busybox sh — হালকা, কম resource             |
| FreeBSD          | `sh`                                                |

---

## Shell কেন নিজে বানাবো?

Shell ব্যবহার করা আর Shell বোঝা — এই দুটো সম্পূর্ণ আলাদা জিনিস।

যখন তুমি নিজে একটা Shell বানাবে, তুমি আসলে শিখবে:

- OS কীভাবে process তৈরি করে
- Program-রা একে অপরের সাথে কীভাবে কথা বলে (pipe)
- Keyboard-এর Ctrl+C কীভাবে একটা চলতে থাকা program থামায় (signal)
- Terminal আসলে কীভাবে কাজ করে

Shell বানানো মানে শুধু একটা tool বানানো না — এটা OS-এর ভেতরের দুনিয়াটা নিজের হাতে ছুঁয়ে দেখা।

চলো শুরু করি।

---