## Shell কী? কেন লাগে?

তুমি Linux চালু করলে একটা কালো স্ক্রিন আসে। সেখানে টাইপ করো `ls`, Enter চাপো — file list দেখায়। এই কালো স্ক্রিনটাই **Shell**।

Shell হলো এমন একটা প্রোগ্রাম যা user আর operating system-এর kernel এর মধ্যে bridge হিসেবে কাজ করে।

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
│  1. Prompt দেখাও  ──►  "tiny-shell> "        │
│         │                                   │
│         ▼                                   │
│  2. Input পড়ো   ──►  "ls -la"               │
│         │                                   │
│         ▼                                   │
│  3. Parse করো   ──►  cmd="ls", args=["-la"] │
│         │                                   │
│         ▼                                   │
│  4. Execute করো ──►  নতুন process বানাও     │
│         │                                   │
│         ▼                                   │
│  5. Wait করো    ──►  process শেষ হওয়া পর্যন্ত │
│         │                                   │
│         └──────────────────────────────────►│
│                  (আবার শুরু থেকে)            │
└─────────────────────────────────────────────┘
```

এটাকে বলে **REPL** — Read, Eval, Print, Loop।

Shell-এর কাজ মূলত চারটা ধাপের একটা চক্র:

```
┌──────────────────────────────────────────────────┐
│                  SHELL LOOP                      │
│                                                  │
│  ┌─────────┐                                     │
│  │  READ   │ ← prompt দেখাও, input পড়ো           │
│  └────┬────┘   "tiny-shell> ls -la | grep .go"   │
│       │                                          │
│       ▼                                          │
│  ┌─────────┐                                     │
│  │  PARSE  │ ← input-কে ভাঙো, মানে বোঝো           │
│  └────┬────┘   cmd="ls", args=["-la"]            │
│       │        pipe আছে, grep আছে               │
│       │                                          │
│       ▼                                          │
│  ┌─────────┐                                     │
│  │ EXECUTE │ ← process তৈরি করো, চালাও            │
│  └────┬────┘   fork → exec → wait                │
│       │                                          │
│       ▼                                          │
│  ┌─────────┐                                     │
│  │  LOOP   │ ← শেষ হলে আবার শুরু থেকে              │
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
│  Input পড়া      → keyboard থেকে line নেওয়া          │
│                                                    │
│  Parse করা      → "ls -la | grep .go" বুঝে           │
│                   দুটো command চেনা, pipe চেনা        │
│                                                    │
│  Built-in চেক   → cd, exit নিজে handle করা          │
│                   (এগুলো child process-এ হয় না)    │
│                                                    │
│  Process চালানো → fork + exec + wait                │
│                                                    │
│  Pipe লাগানো    → দুই process-এর মাঝে data path       │
│                                                    │
│  Signal handle  → Ctrl+C, Ctrl+Z ধরা                │
│                                                    │
│  Job track করা  → background process মনে রাখা        │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## ২. Shell কত প্রকার?

Shell মূলত দুই ভাগে ভাগ করা যায়।

### Command-line Shell (CLI Shell)

Text-based। তুমি যা টাইপ করো সেটা command হিসেবে নেয়। আমরা যে tiny-shell বানাচ্ছি সেটা এই ধরনের।

| Shell       | OS                     | বৈশিষ্ট্য                                  |
|-------------|------------------------|---------------------------------------------|
| `bash`      | Linux, macOS (পুরনো)   | সবচেয়ে common, scripting-এ শক্তিশালী        |
| `zsh`       | Linux, macOS (নতুন)    | plugin system, autocomplete, থিম            |
| `fish`      | Linux, macOS           | syntax highlight, সহজ scripting             |
| `sh`        | Unix, Linux            | POSIX standard, সবচেয়ে portable            |
| `PowerShell`| Windows, Linux, macOS  | object-based pipeline, .NET integration     |
| `cmd.exe`   | Windows                | পুরনো, সীমিত, batch scripting               |

### Graphical Shell (GUI Shell)

Desktop environment হিসেবে কাজ করে — window, icon, mouse সব এর অংশ।

| Shell          | OS      |
|----------------|---------|
| Windows Explorer | Windows |
| GNOME Shell    | Linux   |
| KDE Plasma     | Linux   |
| Aqua           | macOS   |

### কোন OS-এ কোন Shell?

| Operating System   | Default Shell                                        |
|--------------------|------------------------------------------------------|
| Ubuntu / Debian    | `bash`                                               |
| macOS (Catalina+)  | `zsh` — Apple ২০১৯ সালে `bash` থেকে switch করেছে   |
| macOS (পুরনো)      | `bash`                                               |
| Windows 10/11      | `PowerShell` (নতুন), `cmd.exe` (পুরনো), WSL-এ `bash`|
| Android            | `sh` (mksh variant)                                  |
| Embedded Linux     | `sh` বা busybox sh — হালকা, কম resource লাগে        |
| FreeBSD            | `sh`                                                 |