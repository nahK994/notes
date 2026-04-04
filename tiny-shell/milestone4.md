# 🐚 Milestone 4: Pipes

## আগে বুঝি — Pipe আসলে কী?

Part 1-এ FD inherit section-এ একটা ছবি দেখিয়েছিলাম:

```
ls লেখে FD 1-এ             grep পড়ে FD 0 থেকে
    │                           ▲
    │                           │
    └──────── pipe ─────────────┘
```

এখন সেটা বাস্তবে implement করব। কিন্তু তার আগে pipe-এর ভেতরটা আরেকটু গভীরে দেখি।

---

## 🧠 Concept: Pipe আসলে কোথায় থাকে?

Pipe হলো Kernel-এর memory-তে একটা **buffer** — একটা ছোট্ট in-memory queue। Disk-এ যায় না, network-এ যায় না — শুধু RAM-এ থাকে।

```
Kernel Memory-তে Pipe Buffer:
┌─────────────────────────────────────────────┐
│           PIPE BUFFER (max 64KB)            │
│                                             │
│  write end →  [h][e][l][l][o][.][g][o]     │
│                                         │   │
│                         read end ←──────┘   │
│                                             │
│  • ls লেখে write end-এ                     │
│  • grep পড়ে read end থেকে                  │
│  • buffer ভরে গেলে ls WAITING-এ যায়        │
│  • buffer খালি হলে grep WAITING-এ যায়      │
└─────────────────────────────────────────────┘
```

Pipe তৈরি হলে দুটো FD পাওয়া যায়:
- **read end**: এখান থেকে পড়া যায়
- **write end**: এখানে লেখা যায়

---

## 🧠 Concept: FD বদলানো — fork() এর আগেই

Part 1-এ বলেছিলাম: **fork()-এর আগে FD বদলে দাও, exec()-এর পরেও সেই FD থাকবে।**

এটাই pipe-এর পুরো কৌশল। ধাপে ধাপে দেখি:

```
লক্ষ্য: ls | grep .go

Step 1: pipe() syscall করো
        ┌─────────────────────────────────┐
        │ Kernel একটা buffer বানালো      │
        │ FD 3 = read end                 │
        │ FD 4 = write end                │
        └─────────────────────────────────┘

Step 2: ls-এর জন্য fork() করো
        child পেলো:
        FD 0 → keyboard
        FD 1 → screen    ← এটা বদলাতে হবে
        FD 3 → pipe read
        FD 4 → pipe write

        FD 1 বদলে দাও → pipe write (FD 4)
        FD 3, FD 4 বন্ধ করো (আর দরকার নেই)

        exec("ls") করো
        এখন ls FD 1-এ লিখলে pipe-এ যাবে ✓

Step 3: grep-এর জন্য fork() করো
        child পেলো:
        FD 0 → keyboard  ← এটা বদলাতে হবে
        FD 1 → screen
        FD 3 → pipe read
        FD 4 → pipe write

        FD 0 বদলে দাও → pipe read (FD 3)
        FD 3, FD 4 বন্ধ করো (আর দরকার নেই)

        exec("grep") করো
        এখন grep FD 0 থেকে পড়লে pipe থেকে আসবে ✓
```

পুরো ছবি:

```
                        KERNEL
┌──────────────────────────────────────────────────────┐
│                  PIPE BUFFER                         │
│  ┌───────────────────────────────────┐               │
│  │  [main.go\nexecutor.go\n...]      │               │
│  └────────────────┬──────────────────┘               │
│      read(FD 3)   │         │  write(FD 4)           │
└───────────────────┼─────────┼────────────────────────┘
                    │         │
          ┌─────────┘         └──────────┐
          │                              │
          ▼                              ▼
┌──────────────────┐          ┌──────────────────┐
│  grep (child 2)  │          │  ls   (child 1)  │
│                  │          │                  │
│  FD 0 → pipe read│          │  FD 1 → pipe write
│  FD 1 → screen   │          │  FD 0 → keyboard │
│                  │          │                  │
│  pipe থেকে পড়ে  │◄─────────│  pipe-এ লেখে    │
│  screen-এ দেখায় │          │                  │
└──────────────────┘          └──────────────────┘

ls জানে না সে pipe-এ লিখছে — ভাবছে FD 1 = screen
grep জানে না সে pipe থেকে পড়ছে — ভাবছে FD 0 = keyboard
```

---

## 🧠 Concept: Multiple Pipes

`ls | grep .go | wc -l` — তিনটা command, দুটো pipe:

```
N টা command → N-1 টা pipe লাগে

ls ──[pipe0]──► grep ──[pipe1]──► wc
│                │                 │
FD1→pipe0.w     FD0←pipe0.r       FD0←pipe1.r
                FD1→pipe1.w       FD1→screen

pipe0: ls এবং grep-এর মাঝে
pipe1: grep এবং wc-এর মাঝে
```

---

## 🧠 Concept: Deadlock — সবচেয়ে সাধারণ Pipe Bug

এটা এতটাই গুরুত্বপূর্ণ যে আলাদা করে বোঝাচ্ছি।

**Parent-এ write end বন্ধ না করলে কী হয়:**

```
পরিস্থিতি:
ls শেষ হয়ে গেছে → child 1 exit করলো
grep এখনো পড়ছে → pipe থেকে আরো data আসার অপেক্ষায়

grep কখন থামবে?
→ যখন pipe-এর write end সম্পূর্ণ বন্ধ হবে (EOF পাবে)

pipe-এর write end কার কাছে আছে?
→ child 1 (ls) এর কাছে ← exit করেছে, বন্ধ হয়ে গেছে
→ parent (shell) এর কাছে ← এখনো খোলা!

grep ভাবছে: "parent হয়তো আরো কিছু লিখবে"
grep অপেক্ষা করছে... করছে... করছে...

shell ভাবছে: "grep শেষ হলে এগোব"
shell অপেক্ষা করছে... করছে... করছে...

══► DEADLOCK — কেউ এগোতে পারছে না

┌─────────────────────────────────────────┐
│  shell → grep-এর জন্য অপেক্ষা          │
│  grep  → shell-এর write end বন্ধের     │
│          জন্য অপেক্ষা                  │
│                                         │
│  চিরকাল আটকে থাকবে!                   │
└─────────────────────────────────────────┘

Solution: fork() করার পরেই parent-এ
          write end বন্ধ করে দাও।
          তাহলে grep EOF পাবে এবং থামবে।
```

---

## 🏗️ Code

### নতুন file: `internal/executor/pipe.go`

```go
// internal/executor/pipe.go
package executor

import (
	"fmt"
	"os"
	"os/exec"

	"tiny-shell/internal/parser"
)

func executePipeline(pipeline *parser.Pipeline) error {
	cmds := pipeline.Commands
	n := len(cmds)

	// ── প্রতিটা command-এর জন্য exec.Cmd তৈরি করো ──────
	processes := make([]*exec.Cmd, n)
	for i, cmd := range cmds {
		processes[i] = exec.Command(cmd.Name, cmd.Args...)
	}

	// ── N-1 টা pipe তৈরি করো ────────────────────────────
	//
	// cmd0 | cmd1 | cmd2
	//      pipe0       pipe1
	//
	// pipes[i] = cmd[i] এবং cmd[i+1]-এর মাঝের pipe
	//
	// os.Pipe() দুটো *os.File দেয়:
	//   r = read end
	//   w = write end

	type pipePair struct {
		r *os.File
		w *os.File
	}

	pipes := make([]pipePair, n-1)
	for i := range pipes {
		r, w, err := os.Pipe()
		if err != nil {
			return fmt.Errorf("pipe: %w", err)
		}
		pipes[i] = pipePair{r, w}
	}

	// ── প্রতিটা command-এর stdin/stdout connect করো ──────
	//
	// index    stdin              stdout
	// ──────   ──────────────     ──────────────────
	// 0        terminal           pipes[0].w
	// 1        pipes[0].r         pipes[1].w
	// 2        pipes[1].r         pipes[2].w
	// n-1      pipes[n-2].r       terminal
	//
	// সহজ নিয়ম:
	//   প্রথম command → stdin = terminal
	//   শেষ command  → stdout = terminal
	//   বাকি সব      → stdin/stdout = pipe

	for i, proc := range processes {

		// stdin
		if i == 0 {
			proc.Stdin = os.Stdin       // প্রথম: keyboard
		} else {
			proc.Stdin = pipes[i-1].r  // বাকি: আগের pipe থেকে
		}

		// stdout
		if i == n-1 {
			proc.Stdout = os.Stdout    // শেষ: screen
		} else {
			proc.Stdout = pipes[i].w   // বাকি: পরের pipe-এ
		}

		// stderr সবার জন্য terminal
		proc.Stderr = os.Stderr
	}

	// ── সব command Start করো ─────────────────────────────
	//
	// Run() নয় — Run() = Start() + Wait()
	// আমরা সব Start() করব, তারপর সব Wait() করব
	//
	// কেন?
	// একটা Start() করে Wait() করলে সে block হবে
	// কারণ pipe full হলে ls লিখতে পারবে না
	// কিন্তু grep পড়ছে না (এখনো Start হয়নি)
	// → deadlock!
	//
	// তাই সব আগে Start(), তারপর সব Wait()

	for i, proc := range processes {
		if err := proc.Start(); err != nil {
			if err == exec.ErrNotFound {
				return fmt.Errorf("%s: command not found",
					cmds[i].Name)
			}
			return fmt.Errorf("start %s: %w", cmds[i].Name, err)
		}
	}

	// ── Parent-এ write end বন্ধ করো ──────────────────────
	//
	// এটাই deadlock এড়ানোর key!
	//
	// fork() হওয়ার পরে pipe-এর write end দুই জায়গায়:
	//   child (ls)    → Start() হলে সে exit করলে বন্ধ হবে
	//   parent (shell)→ এখনো খোলা! ← এটা এখনই বন্ধ করো
	//
	// parent-এর write end বন্ধ না করলে:
	//   ls শেষ হলেও grep EOF পাবে না
	//   grep চিরকাল অপেক্ষা করবে
	//   shell grep-এর জন্য অপেক্ষা করবে
	//   → deadlock!

	for _, p := range pipes {
		// write end বন্ধ করো
		// read end এখনো খোলা — child (grep) পড়ছে
		p.w.Close()
	}

	// ── সব command শেষ হওয়ার জন্য Wait করো ──────────────
	//
	// Wait() করার সময় যা হয়:
	//   ls শেষ হলো → pipe-এর write end (child-এর copy) বন্ধ
	//   parent-এর copy আগেই বন্ধ করেছি
	//   → pipe-এর write end সম্পূর্ণ বন্ধ
	//   → grep EOF পেলো → grep শেষ হলো
	//   → Wait(grep) return করলো

	var lastErr error
	for _, proc := range processes {
		if err := proc.Wait(); err != nil {
			lastErr = err
		}
	}

	// ── Read end গুলোও বন্ধ করো ──────────────────────────
	// সব শেষ, এখন read end-ও বন্ধ করা যায়
	for _, p := range pipes {
		p.r.Close()
	}

	return lastErr
}
```

---

### `executor.go` আপডেট

```go
// internal/executor/executor.go
package executor

import (
	"fmt"
	"os"
	"os/exec"

	"tiny-shell/internal/builtins"
	"tiny-shell/internal/parser"
)

func Execute(line string) error {
	// Parse করো
	pipeline, err := parser.Parse(line)
	if err != nil {
		return err
	}

	if len(pipeline.Commands) == 0 {
		return nil
	}

	// Single command — pipe নেই
	if len(pipeline.Commands) == 1 {
		return executeSingle(pipeline.Commands[0])
	}

	// Multiple commands — pipe আছে
	return executePipeline(pipeline)
}

func executeSingle(cmd *parser.Command) error {
	if cmd.Name == "" {
		return nil
	}

	// Built-in চেক করো আগে
	allArgs := append([]string{cmd.Name}, cmd.Args...)
	handled, err := builtins.Execute(allArgs)
	if handled {
		return err
	}

	return runCommand(cmd)
}

func runCommand(cmd *parser.Command) error {
	c := exec.Command(cmd.Name, cmd.Args...)
	c.Stdin  = os.Stdin
	c.Stdout = os.Stdout
	c.Stderr = os.Stderr

	if err := c.Run(); err != nil {
		if exitErr, ok := err.(*exec.ExitError); ok {
			return fmt.Errorf("%s: exit status %d",
				cmd.Name, exitErr.ExitCode())
		}
		if err == exec.ErrNotFound {
			return fmt.Errorf("%s: command not found", cmd.Name)
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
tiny-shell ~> ls | grep .go
executor.go
pipe.go

tiny-shell ~> echo "hello world" | grep hello
hello world

tiny-shell ~> cat /etc/passwd | grep root | wc -l
2

tiny-shell ~> ls -la | sort | head -5
(sorted output)
```

---

## 🔬 "What just happened?" — `ls | grep .go` করলে OS-এ কী হলো

```
"ls | grep .go" টাইপ করলে:
        │
        ▼
Parse: Pipeline{
  Command[0]: {Name:"ls",   Args:[]}
  Command[1]: {Name:"grep", Args:[".go"]}
}
        │
        ▼
executePipeline() call হলো
        │
        │ os.Pipe() syscall
        │ Kernel buffer বানালো:
        │ pipes[0] = {r:FD3, w:FD4}
        │
        ▼
processes[0] (ls) setup:
  Stdin  = terminal
  Stdout = pipes[0].w (FD4) ← pipe-এ লিখবে
  Stderr = terminal

processes[1] (grep) setup:
  Stdin  = pipes[0].r (FD3) ← pipe থেকে পড়বে
  Stdout = terminal
  Stderr = terminal
        │
        ▼
processes[0].Start() → fork() + exec("ls")
processes[1].Start() → fork() + exec("grep", ".go")

দুটো process এখন একসাথে চলছে:
        │
        ▼
pipes[0].w.Close() ← parent-এর write end বন্ধ!
        │
        ▼
ls চলছে:                    grep চলছে:
main.go → pipe buffer       pipe buffer → "main.go" → .go আছে? হ্যাঁ → screen
pipe.go → pipe buffer       pipe buffer → "pipe.go" → .go আছে? হ্যাঁ → screen
        │
ls exit(0)
pipe write end সম্পূর্ণ বন্ধ
(child copy + parent copy দুটোই বন্ধ)
        │
grep EOF পেলো → grep exit(0)
        │
Wait(ls)   ✓
Wait(grep) ✓
        │
        ▼
tiny-shell ~> (prompt ফিরে এলো)
```

---

## 🧪 Experiment করো

**১. Deadlock নিজে দেখো:**

`pipe.go`-তে এই line comment করো:
```go
// p.w.Close()  ← এটা comment করো
```

এখন `ls | grep .go` চালাও — shell আটকে যাবে। Ctrl+C দিয়ে বের হও। তারপর comment সরিয়ে দাও।

**২. Long pipeline:**
```bash
tiny-shell ~> cat /etc/passwd | cut -d: -f1 | sort | uniq | wc -l
```

**৩. Pipe buffer size দেখো:**

আলাদা একটা Go file-এ (tiny-shell-এর বাইরে) এটা চালাও:

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    _, w, _ := os.Pipe()
    written := 0
    buf := make([]byte, 4096)
    for i := range buf { buf[i] = 'x' }

    for {
        n, err := w.Write(buf)
        written += n
        if err != nil {
            fmt.Printf("pipe full: %d bytes\n", written)
            break
        }
    }
}
```

Linux-এ দেখবে **65536 bytes (64KB)** — এটাই pipe buffer-এর default size।

---

## 📋 Milestone 4 Checklist

- ✅ Pipe কী এবং kernel-এ কোথায় থাকে বুঝলাম
- ✅ FD বদলে দুই process connect করা বুঝলাম
- ✅ Deadlock কেন হয় এবং কীভাবে এড়ানো যায় বুঝলাম
- ✅ Start() vs Run() পার্থক্য বুঝলাম
- ✅ Multiple pipe (3+ command) কাজ করছে

---

Milestone 4 চললে বলো — **Milestone 5: I/O Redirection** শুরু করব। সেখানে `>`, `>>`, `<` implement করব — FD নিজে হাতে বদলাব, `dup2` syscall সরাসরি ব্যবহার করব। 🚀