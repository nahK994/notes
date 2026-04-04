## ভূমিকা: Shell কী এবং OS কীভাবে কাজ করে?

কোড লেখার আগে **মাথায় একটা ছবি** তৈরি করতে হবে। না হলে কোড দেখলে মনে হবে magic।

---

## 🧠 Part 0: Mental Model — OS, Process, Syscall

### OS কী করে?

তুমি যখন Linux চালাও, তখন একটাই "boss" থাকে — **Linux Kernel**। সে CPU, RAM, Disk, Network — সব নিয়ন্ত্রণ করে।

তুমি সরাসরি hardware ছুঁতে পারবে না। Kernel-এর মাধ্যমে যেতে হবে।

```
┌─────────────────────────────────────────┐
│           USER SPACE                    │
│                                         │
│   তোমার Shell   │   ls   │   grep      │
│   (tiny-shell)  │        │             │
│                                         │
├─────────────────────────────────────────┤
│         SYSTEM CALL INTERFACE           │
│   (এটাই দরজা — kernel-এ ঢোকার একমাত্র পথ) │
├─────────────────────────────────────────┤
│           KERNEL SPACE                  │
│                                         │
│   Process Mgmt │ File System │ Memory   │
│   Network      │ Signals     │ Devices  │
│                                         │
├─────────────────────────────────────────┤
│           HARDWARE                      │
│   CPU  │  RAM  │  Disk  │  Keyboard    │
└─────────────────────────────────────────┘
```

## 🔬 Process — গভীরে যাই

### Process কী — একদম সহজ ভাষায়

তোমার disk-এ `/bin/ls` একটা file পড়ে আছে। সে ঘুমাচ্ছে, কিছু করছে না। এটা হলো **program** — একটা recipe-র মতো।

তুমি যখন `ls` চালাও, Linux সেই recipe দেখে রান্না শুরু করে। রান্না যতক্ষণ চলছে, ততক্ষণ সেটা **process** — জীবন্ত, RAM-এ চলছে, CPU খাচ্ছে।

```
DISK                        RAM
──────────────              ─────────────────────────────
/bin/ls                     Process (PID: 1234)
├── machine code    ──►     ├── Code Segment
├── (static, মৃত)           │     (ls-এর instructions)
└── (কিছু করছে না)          │
                            ├── Stack
                            │     (function calls, local variables)
                            │
                            ├── Heap
                            │     (dynamic memory, malloc)
                            │
                            └── File Descriptors
                                  (কোন কোন file খোলা আছে)
```

recipe (program) একটাই, কিন্তু রান্না (process) একসাথে অনেক হতে পারে। তুমি একসাথে ৫টা terminal-এ `ls` চালালে ৫টা আলাদা process তৈরি হবে — প্রত্যেকের আলাদা PID, আলাদা RAM।

---

### Process-এর ভেতরে কী থাকে?

Linux প্রতিটা process সম্পর্কে অনেক তথ্য রাখে। এগুলো দেখা যায় `/proc` ফোল্ডারে:

```bash
# তোমার shell চালু রাখো, আরেকটা terminal-এ এটা করো:
cat /proc/$$/status
```

তুমি দেখবে:

```
Name:   bash              ← process-এর নাম
Pid:    1234              ← এই process-এর ID
PPid:   1000              ← Parent-এর PID (কে জন্ম দিয়েছে)
VmRSS:  4096 kB          ← এখন কতটুকু RAM ব্যবহার করছে
Threads: 1               ← কতটা thread চলছে
```

`$$` মানে current shell-এর PID। এটা একটা shell variable।

---

### Process-এর জীবনচক্র

একটা process জন্ম নেয়, কাজ করে, মারা যায়। মাঝখানে বিভিন্ন অবস্থায় থাকতে পারে:

```
                    fork()
                      │
                      ▼
                  ┌────────┐
                  │ CREATED │  ← সবে জন্ম হলো
                  └────┬───┘
                       │
                       ▼
                  ┌────────┐
          ┌──────►│RUNNABLE│  ← চলার জন্য ready, CPU-র জন্য অপেক্ষা
          │       └────┬───┘
          │            │  CPU পেলো
          │            ▼
          │       ┌─────────┐
          │       │ RUNNING │  ← এখন সত্যিই CPU-তে চলছে
          │       └────┬────┘
          │            │
          │     ┌──────┴───────┐
          │     │              │
          │     ▼              ▼
          │  ┌───────┐    ┌────────┐
          └──│WAITING│    │ ZOMBIE │ ← কাজ শেষ, parent এখনো
             └───┬───┘    └────┬───┘   জানে না (wait করেনি)
                 │             │
          (I/O শেষে)      parent wait()
          RUNNABLE-এ           │
          ফিরে যায়             ▼
                          ┌────────┐
                          │  DEAD  │ ← সম্পূর্ণ মুছে গেছে
                          └────────┘
```

**WAITING** অবস্থাটা গুরুত্বপূর্ণ। `ls` যখন disk থেকে file list পড়ে, সে CPU ছেড়ে দিয়ে WAITING-এ চলে যায়। disk পড়া শেষ হলে আবার RUNNABLE হয়। এই সময়ে অন্য process CPU পায়। এভাবেই Linux একসাথে হাজারটা process চালায়।

---

### PID এবং PPID — Process-এর পরিচয়

প্রতিটা process-এর দুটো গুরুত্বপূর্ণ নম্বর থাকে:

```
PID  (Process ID)  — নিজের নম্বর
PPID (Parent PID)  — কে জন্ম দিয়েছে তার নম্বর
```

Linux boot হলে প্রথম যে process চালু হয় তার নাম `systemd` বা `init`, PID = 1। এরপর প্রতিটা process কোনো না কোনো parent থেকে জন্ম নেয়। পুরো জিনিসটা একটা গাছের মতো:

```
PID 1: systemd
├── PID 450: login
│   └── PID 890: bash (তোমার terminal)
│       └── PID 1234: tiny-shell (আমাদের shell)
│           └── PID 1235: ls (shell থেকে জন্ম নিলো)
│               └── (কাজ করলো, মরে গেলো)
└── PID 512: sshd
    └── ...
```

এটা নিজে দেখতে চাইলে:

```bash
pstree -p $$
```

---

### File Descriptor — Process-এর চোখ-কান-মুখ

Process যখন বাইরের জগতের সাথে কথা বলে — file পড়ে, screen-এ লেখে, keyboard থেকে input নেয় — সে করে **File Descriptor (FD)** দিয়ে।

FD হলো একটা ছোট্ট integer number। প্রতিটা process জন্মের সময়েই ৩টা FD পায়:

```
Process (যেকোনো)
├── FD 0 ──► stdin  (keyboard, input আসে এখান থেকে)
├── FD 1 ──► stdout (screen, output যায় এখানে)
└── FD 2 ──► stderr (screen, error যায় এখানে)
```

তুমি যখন কোডে লেখো `fmt.Println("hello")`, Go আসলে kernel-কে বলে:

```
write(FD=1, "hello\n", 6)
       ↑
  stdout-এ লেখো
```

Kernel FD 1 দেখে বোঝে "screen-এ পাঠাও"। এটাই syscall।

এই FD concept-টা Milestone 4 (pipes) এবং Milestone 5 (I/O redirection)-এ আবার আসবে — সেখানে আমরা এই নম্বরগুলো নিজেরাই বদলে দেব।

---

### নিজে দেখো — Process সত্যিই কীভাবে তৈরি হয়

tiny-shell চালু করো, তারপর আলাদা terminal-এ:

```bash
# সব process দেখো tree আকারে
pstree -p | grep tiny

# tiny-shell-এর memory layout দেখো (PID বসাও)
cat /proc/1234/maps

# কোন files খোলা আছে দেখো
ls -la /proc/1234/fd
```

`/proc/1234/fd` দেখলে পাবে:
```
0 → /dev/pts/0   (keyboard)
1 → /dev/pts/0   (screen)
2 → /dev/pts/0   (error screen)
```

এই `/dev/pts/0` হলো তোমার terminal। stdin, stdout, stderr তিনটাই একই terminal-এ — তাই তুমি type করো সেটাই পড়া যায়, আবার output-ও সেই terminal-এই আসে।

---

### Syscall কী?

Kernel-কে কিছু করতে বলার ভাষা হলো **System Call (syscall)**।

যেমন:
- ফাইল খুলতে চাই → `open()` syscall
- নতুন process বানাতে চাই → `fork()` syscall  
- কোনো program চালাতে চাই → `exec()` syscall
- লেখা print করতে চাই → `write()` syscall

```
তোমার কোড                 Kernel
──────────                 ──────
fmt.Println("hi")  ──►   write(1, "hi", 2)  ──►  Screen-এ দেখায়
                           [syscall নম্বর: 1]
```

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

## 🔧 Fork-Exec Model — Shell-এর সবচেয়ে গুরুত্বপূর্ণ concept

তুমি যখন shell-এ `ls` টাইপ করো, shell কিন্তু নিজে `ls` চালায় না। সে একটা **child process** তৈরি করে, সেই child `ls` হয়ে যায়।

এই দুই ধাপে হয়:

**Step 1 — `fork()`**: নিজের exact copy তৈরি করো
```
Shell Process (PID: 100)
        │
        │  fork()
        │
   ┌────┴────┐
   │         │
Parent      Child
(PID: 100)  (PID: 101)
Shell        Shell-এর copy
চলতে থাকে   (এখন ls হয়ে যাবে)
```

**Step 2 — `exec()`**: Child নিজেকে ls দিয়ে replace করে
```
Child (PID: 101)
Shell-এর copy
      │
      │  exec("/bin/ls")
      │
      ▼
Child (PID: 101)
এখন সে ls!
(আগের shell code মুছে গেছে,
ls-এর code এসে গেছে)
```

**Step 3 — `wait()`**: Parent অপেক্ষা করে child শেষ হওয়া পর্যন্ত
```
Parent (Shell)          Child (ls)
──────────────          ──────────
wait() বলে ঘুমায়  ◄──  কাজ করে
      │                     │
      │                     │ (ls output দেখায়)
      │                     │
      │◄────── exit() ──────┘
      │
      ▼
আবার prompt দেখায়
```

এই পুরো pattern-টার নাম **fork-exec model**। Unix/Linux-এর সবকিছু এভাবে কাজ করে।

---

## 🚀 Milestone 1: Basic Shell Loop

এবার কোড লেখা শুরু। এই milestone-এ আমরা বানাব:
- Prompt দেখানো
- Input পড়া
- Command execute করা (fork-exec দিয়ে)
- Error handle করা

### Project Setup

Terminal খুলে এটা করো:

```bash
mkdir tiny-shell
cd tiny-shell
go mod init tiny-shell
mkdir -p cmd/shell internal/executor internal/builtins internal/parser
```

তোমার structure এরকম হবে:
```
tiny-shell/
├── go.mod
├── cmd/
│   └── shell/
│       └── main.go
└── internal/
    ├── executor/
    │   └── executor.go
    ├── builtins/
    │   └── builtins.go
    └── parser/
        └── parser.go
```

---

### Step 1: `main.go` — Shell-এর হৃদয়

```go
// cmd/shell/main.go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"

	"tiny-shell/internal/executor"
)

func main() {
	// bufio.Scanner দিয়ে keyboard থেকে input পড়ব
	// os.Stdin মানে হলো keyboard (standard input)
	scanner := bufio.NewScanner(os.Stdin)

	// এটাই REPL loop — infinite loop
	for {
		// ── READ ──
		// Prompt দেখাও
		// fmt.Print vs fmt.Println: Print newline দেয় না
		fmt.Print("tiny-shell> ")

		// পরের line পড়ো (Enter চাপার পর)
		// Scan() false return করে যদি input শেষ হয়ে যায়
		// (যেমন Ctrl+D চাপলে)
		if !scanner.Scan() {
			fmt.Println("\nবাই! 👋")
			break
		}

		// ── EVAL ──
		// যা পড়লাম তা নাও, আগে-পিছের whitespace কাটো
		line := strings.TrimSpace(scanner.Text())

		// খালি input হলে আবার loop-এর শুরুতে যাও
		if line == "" {
			continue
		}

		// "exit" টাইপ করলে বের হয়ে যাও
		if line == "exit" {
			fmt.Println("বাই! 👋")
			os.Exit(0)
		}

		// ── EXECUTE ──
		// Command execute করো
		if err := executor.Execute(line); err != nil {
			// os.Stderr মানে error output
			// Convention: error সবসময় stderr-এ লেখা হয়
			fmt.Fprintln(os.Stderr, "error:", err)
		}

		// ── LOOP ──
		// আবার শুরুতে চলে যাও (Go for loop automatically করে)
	}
}
```

---

### Step 2: `executor.go` — Fork-Exec এখানেই হয়

```go
// internal/executor/executor.go
package executor

import (
	"fmt"
	"os"
	"os/exec"
	"strings"
)

// Execute একটা command string নেয় এবং চালায়
func Execute(line string) error {
	// Command-কে parts-এ ভাগ করো
	// "ls -la /home"  →  ["ls", "-la", "/home"]
	// এখন simple split, Milestone 3-এ proper parser বানাব
	parts := strings.Fields(line)

	if len(parts) == 0 {
		return nil
	}

	// parts[0] = command নাম ("ls")
	// parts[1:] = বাকি সব argument (["-la", "/home"])
	cmdName := parts[0]
	args := parts[1:]

	return runCommand(cmdName, args)
}

// runCommand fork-exec করে command চালায়
func runCommand(name string, args []string) error {
	// exec.Command একটা Cmd struct তৈরি করে
	// এটা এখনো কিছু চালায়নি — শুধু "plan" তৈরি করেছে
	//
	// Go internally যা করে:
	//   1. /bin, /usr/bin ইত্যাদিতে "name" খোঁজে (PATH দিয়ে)
	//   2. পাওয়া গেলে full path রাখে (যেমন /bin/ls)
	cmd := exec.Command(name, args...)

	// Child process-এর stdin/stdout/stderr
	// আমাদের shell-এর same terminal connect করি
	//
	// এটা না করলে child process কিছু print করতে পারবে না!
	// কারণ by default child-এর stdout কোথাও যায় না।
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// এখন actually চালাও:
	// Go internally এখানে fork() করে child process বানায়
	// তারপর child-এ exec() করে আমাদের command চালায়
	// তারপর parent (shell) wait() করে child শেষ হওয়া পর্যন্ত
	err := cmd.Run()

	if err != nil {
		// *exec.ExitError মানে command চলেছে কিন্তু
		// non-zero exit code দিয়ে বের হয়েছে
		// (যেমন: ls একটা file না পেলে exit code 1 দেয়)
		if exitErr, ok := err.(*exec.ExitError); ok {
			return fmt.Errorf("%s: exit status %d", name, exitErr.ExitCode())
		}

		// exec.ErrNotFound মানে command-ই খুঁজে পায়নি
		if err == exec.ErrNotFound {
			return fmt.Errorf("%s: command not found", name)
		}

		return err
	}

	return nil
}
```

---

### Step 3: চালাও!

```bash
go run cmd/shell/main.go
```

তুমি দেখবে:
```
tiny-shell> ls
(ls-এর output আসবে)

tiny-shell> pwd
/home/তোমার-username/tiny-shell

tiny-shell> ls -la
(details সহ output)

tiny-shell> blahblah
error: blahblah: command not found

tiny-shell> exit
বাই! 👋
```

---

## 🔬 "What just happened?" — OS Level-এ কী ঘটলো

তুমি `ls` টাইপ করে Enter চাপার পর ঠিক কী হলো:

```
তুমি "ls" টাইপ করলে
         │
         ▼
scanner.Scan() keyboard থেকে পড়লো
         │
         ▼
strings.Fields() দিয়ে ["ls"] বানালো
         │
         ▼
exec.Command("ls") — /bin/ls খুঁজে পেলো
         │
         ▼
cmd.Run() call হলো
         │
    ┌────┴─────────────────────────────────┐
    │                                      │
    │  Go Runtime → Linux Kernel-কে বললো: │
    │  "আমার একটা copy বানাও" (fork)       │
    │                                      │
    └────┬─────────────────────────────────┘
         │
    ┌────┴──────────┐
    │               │
  Parent           Child
  (shell)          (shell-এর copy)
  wait করছে       │
                   │ exec("/bin/ls") syscall
                   │ নিজেকে ls দিয়ে replace করলো
                   │
                   ▼
                 ls চললো
                 output আসলো তোমার terminal-এ
                   │
                   │ exit(0)
                   ▼
  Parent ◄──── child শেষ হলো
  (shell)
  আবার prompt দেখালো
```

---

## 📋 Milestone 1 Checklist

- ✅ REPL loop কাজ করছে
- ✅ Command execute হচ্ছে
- ✅ Fork-exec model বুঝলাম
- ✅ Error handle হচ্ছে
- ✅ `exit` দিয়ে বের হওয়া যাচ্ছে

---

## 🧪 Experiment করো

কোড বোঝার সেরা উপায় হলো ভাঙা। এগুলো try করো:

```bash
# তোমার shell-এ এগুলো চালাও:
tiny-shell> echo hello world
tiny-shell> cat /etc/hostname
tiny-shell> whoami
tiny-shell> date
tiny-shell> sleep 2        # 2 সেকেন্ড wait করবে
tiny-shell> ls /nonexistent # error দেখো
```

এবং Go কোডে এটা add করো `runCommand`-এর শুরুতে — PID দেখতে:

```go
fmt.Printf("[shell PID: %d] '%s' চালাচ্ছি...\n", os.Getpid(), name)
```

Run করার পর আলাদা terminal-এ `ps aux | grep tiny-shell` চালাও — দেখবে process তৈরি হচ্ছে।
