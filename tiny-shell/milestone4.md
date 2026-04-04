# Milestone 4: Pipes

## আগে বুঝি — Pipe আসলে কী?

তুমি এটা হয়তো অনেকবার লিখেছ:

```bash
ls -la | grep .go | wc -l
```

কিন্তু এই `|` এর ভেতরে আসলে কী হচ্ছে? এটা কোনো magic না — এটা OS-এর একটা খুব সুন্দর feature।

---

## 🧠 Concept: File Descriptor আবার দেখি

Milestone 1-এ বলেছিলাম প্রতিটা process তিনটা FD নিয়ে জন্মায়:

```
Process
├── FD 0 ──► stdin  (পড়ে)
├── FD 1 ──► stdout (লেখে)
└── FD 2 ──► stderr (error লেখে)
```

FD আসলে শুধু একটা **নম্বর**। Kernel একটা table রাখে — এই নম্বর দিয়ে আসলে কোথায় যাচ্ছে সেটা বলে দেয়:

```
Kernel-এর FD Table (process 1234-এর জন্য):
┌─────┬──────────────────────────────────┐
│ FD  │ আসলে কোথায় point করছে           │
├─────┼──────────────────────────────────┤
│  0  │ /dev/pts/0 (terminal keyboard)   │
│  1  │ /dev/pts/0 (terminal screen)     │
│  2  │ /dev/pts/0 (terminal screen)     │
│  3  │ /home/shomi/file.txt (খোলা file) │
└─────┴──────────────────────────────────┘
```

Key insight: **FD একটা নম্বর মাত্র। এই নম্বর যা point করে সেটা বদলে দেওয়া যায়।**

এটাই Pipe-এর পুরো রহস্য।

---

## 🧠 Concept: Pipe আসলে কী?

Pipe হলো Kernel-এর মেমরিতে একটা **buffer** — একটা ছোট্ট in-memory queue।

```
Kernel Memory-তে Pipe Buffer:
┌─────────────────────────────────────┐
│  PIPE BUFFER (64KB পর্যন্ত)         	  │
│                                     │
│  ← read end        write end →      │
│                                     │
│  [h][e][l][l][o][ ][w][o][r][l][d]  │
│   ↑                              ↑  │
│ পড়া হবে এখান থেকে    লেখা হয় এখানে  │  |
└─────────────────────────────────────┘
```

Pipe তৈরি হলে দুটো FD পাওয়া যায়:
- **read end**: এখান থেকে পড়া যায়
- **write end**: এখানে লেখা যায়

---

## 🧠 Concept: Pipe দিয়ে দুটো Process কীভাবে কথা বলে?

`ls | grep .go` — এটা করতে হলে:

```
লক্ষ্য:
ls-এর stdout ──────────────► grep-এর stdin

বাস্তবে যা হয়:

Step 1: pipe() syscall করো
        দুটো FD তৈরি হলো:
        pipeRead  (FD 3)
        pipeWrite (FD 4)

Step 2: ls-এর জন্য child 1 তৈরি করো (fork)
        child 1-এর stdout (FD 1) বদলে দাও
        FD 1 → pipeWrite (FD 4)
        এখন ls যা লিখবে সব pipe-এ যাবে

Step 3: grep-এর জন্য child 2 তৈরি করো (fork)
        child 2-এর stdin (FD 0) বদলে দাও
        FD 0 → pipeRead (FD 3)
        এখন grep যা পড়বে সব pipe থেকে আসবে

Step 4: দুই child-কে চালাও
        ls লেখে → pipe buffer-এ জমা হয়
        grep পড়ে ← pipe buffer থেকে নেয়
```

ASCII-তে পুরো ছবি:

```
                    KERNEL
┌───────────────────────────────────────────────────┐
│                                                   │
│  PIPE BUFFER                                      │
│  ┌─────────────────────────────┐                  │
│  │  [data flowing through]     │                  │
│  └──────────┬──────────────────┘                  │
│    read(FD3)│           │write(FD4)               │
└─────────────┼───────────┼─────────────────────────┘
              │           │
     ┌────────┘           └────────┐
     │                             │
     ▼                             ▼
┌─────────────┐             ┌─────────────┐
│  grep       │             │  ls         │
│  (child 2)  │             │  (child 1)  │
│             │             │             │
│  FD 0 ◄───  │ ←━━━━━━━━━ 	│ ───► FD 1   │
│  (stdin)    │             │   (stdout)  │
│  pipe থেকে  │           	│  pipe-এ     │
│  পড়ছে       │             │  লিখছে      │
└─────────────┘             └─────────────┘
```

---

## 🧠 Concept: dup2 — FD বদলে দেওয়ার syscall

FD বদলানো হয় `dup2()` syscall দিয়ে।

`dup2(oldFD, newFD)` মানে: "newFD কে oldFD-এর মতো বানিয়ে দাও"

```
dup2(pipeWrite, 1) মানে:

আগে:                        পরে:
FD 1 → terminal screen      FD 1 → pipe write end
FD 4 → pipe write end       FD 4 → pipe write end

এখন ls FD 1-এ লিখলে
terminal-এ না গিয়ে
pipe-এ যাবে!
```

---

## 🧠 Concept: Multiple Pipes

`ls | grep .go | wc -l` — তিনটা command, দুটো pipe:

```
ls ──[pipe1]──► grep ──[pipe2]──► wc
│                │                 │
stdout→pipe1    stdin←pipe1       stdin←pipe2
                stdout→pipe2
```

প্রতিটা command জোড়ার মাঝে একটা করে pipe লাগে। N টা command হলে N-1 টা pipe।

---

## 🏗️ Code

### Step 1: `internal/executor/pipe.go` — নতুন file

```go
// internal/executor/pipe.go
package executor

import (
	"fmt"
	"os"
	"os/exec"

	"tiny-shell/internal/parser"
)

// executePipeline — pipe দিয়ে জোড়া commands চালাও
func executePipeline(pipeline *parser.Pipeline) error {
	cmds := pipeline.Commands
	n := len(cmds)

	// প্রতিটা command-এর জন্য একটা *exec.Cmd তৈরি করো
	processes := make([]*exec.Cmd, n)
	for i, cmd := range cmds {
		processes[i] = exec.Command(cmd.Name, cmd.Args...)
	}

	// ── Pipe তৈরি করো ──────────────────────────────────────
	//
	// N টা command-এ N-1 টা pipe লাগে:
	// cmd0 | cmd1 | cmd2
	//      pipe0       pipe1
	//
	// pipes[i] = cmd[i] এবং cmd[i+1] এর মাঝের pipe
	//
	// os.Pipe() করলে দুটো *os.File পাই:
	//   r = read end  (এখান থেকে পড়া যাবে)
	//   w = write end (এখানে লেখা যাবে)

	type pipePair struct {
		r *os.File // read end
		w *os.File // write end
	}

	pipes := make([]pipePair, n-1)
	for i := range pipes {
		r, w, err := os.Pipe()
		if err != nil {
			return fmt.Errorf("pipe: %w", err)
		}
		pipes[i] = pipePair{r, w}
	}

	// ── প্রতিটা command-এর stdin/stdout connect করো ────────
	//
	// cmd[0]:  stdin = terminal,     stdout = pipe[0].w
	// cmd[1]:  stdin = pipe[0].r,    stdout = pipe[1].w
	// cmd[2]:  stdin = pipe[1].r,    stdout = terminal
	//
	// সহজ মনে রাখার উপায়:
	//   প্রথম command-এর আগে কোনো pipe নেই → stdin = terminal
	//   শেষ command-এর পরে কোনো pipe নেই  → stdout = terminal
	//   মাঝের সবার stdin ও stdout pipe-এ

	for i, proc := range processes {

		// ── stdin সেট করো ──
		if i == 0 {
			// প্রথম command: keyboard থেকে পড়ো
			proc.Stdin = os.Stdin
		} else {
			// বাকিরা: আগের command-এর pipe থেকে পড়ো
			proc.Stdin = pipes[i-1].r
		}

		// ── stdout সেট করো ──
		if i == n-1 {
			// শেষ command: terminal-এ লেখো
			proc.Stdout = os.Stdout
		} else {
			// বাকিরা: পরের command-এর pipe-এ লেখো
			proc.Stdout = pipes[i].w
		}

		// stderr সবার জন্য terminal-এ যাক
		proc.Stderr = os.Stderr
	}

	// ── সব command একসাথে Start করো ───────────────────────
	//
	// Run() নয়! Run() = Start() + Wait()
	// আমরা সব Start() করব, তারপর সব Wait() করব
	// কারণ: সব process একসাথে চলতে হবে
	//        একটা শেষ হওয়ার জন্য অপেক্ষা করলে
	//        pipe block হয়ে যাবে (deadlock!)

	for i, proc := range processes {
		if err := proc.Start(); err != nil {
			// এই command start করতে পারিনি
			if err == exec.ErrNotFound {
				return fmt.Errorf("%s: command not found",
					cmds[i].Name)
			}
			return fmt.Errorf("start %s: %w", cmds[i].Name, err)
		}
	}

	// ── Write end গুলো বন্ধ করো (parent-এ) ────────────────
	//
	// এটা খুব গুরুত্বপূর্ণ! কেন?
	//
	// grep পড়তে থাকে যতক্ষণ pipe-এর write end খোলা আছে।
	// ls শেষ হয়ে গেলে child-এর write end বন্ধ হয়।
	// কিন্তু parent (shell)-এর কাছেও write end-এর copy আছে!
	// তাই grep কখনো EOF পাবে না — চিরকাল অপেক্ষা করবে।
	//
	// তাই parent-এ write end বন্ধ করে দিতে হবে।
	//
	// ┌─────────────────────────────────────────────────────┐
	// │  pipe.w এর copies:                                  │
	// │  - child 1 (ls) এর কাছে ← Start() এর পরে বন্ধ হবে │
	// │  - parent (shell) এর কাছে ← আমরা এখনই বন্ধ করি    │
	// └─────────────────────────────────────────────────────┘

	for _, p := range pipes {
		p.w.Close() // parent-এর write end বন্ধ করো
	}

	// ── সব command শেষ হওয়ার জন্য অপেক্ষা করো ─────────────
	var lastErr error
	for _, proc := range processes {
		if err := proc.Wait(); err != nil {
			// শেষ command-এর error টা রাখো
			lastErr = err
		}
	}

	// ── Read end গুলোও বন্ধ করো ────────────────────────────
	for _, p := range pipes {
		p.r.Close()
	}

	return lastErr
}
```

---

### Step 2: `executor.go` আপডেট

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

	// Single command?
	if len(pipeline.Commands) == 1 {
		return executeSingle(pipeline.Commands[0])
	}

	// Multiple commands — pipe আছে
	// Built-in commands pipe-এ কাজ করে না এখনো
	// (Milestone 8-এ handle করব)
	return executePipeline(pipeline)
}

func executeSingle(cmd *parser.Command) error {
	if cmd.Name == "" {
		return nil
	}

	allArgs := append([]string{cmd.Name}, cmd.Args...)
	handled, err := builtins.Execute(allArgs)
	if handled {
		return err
	}

	return runCommand(cmd)
}

func runCommand(cmd *parser.Command) error {
	c := exec.Command(cmd.Name, cmd.Args...)
	c.Stdin = os.Stdin
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

## চালাও এবং দেখো

```bash
go run cmd/shell/main.go
```

```
tiny-shell ~> ls | grep .go
main.go

tiny-shell ~> echo "hello world" | grep hello
hello world

tiny-shell ~> cat /etc/passwd | grep root | wc -l
2

tiny-shell ~> ls -la | sort | head -5
(sorted output)

tiny-shell ~> echo "one\ntwo\nthree" | wc -l
3
```

---

## 🔬 "What just happened?" — `ls | grep .go` করলে OS-এ কী হলো

```
tiny-shell ~> ls | grep .go
        │
        ▼
Parse করো:
Pipeline:
  Command[0]: {Name:"ls",   Args:[]}
  Command[1]: {Name:"grep", Args:[".go"]}
        │
        ▼
executePipeline() call হলো
        │
        │ os.Pipe() syscall
        │ Kernel একটা pipe buffer তৈরি করলো
        │
        ▼
pipes[0] = {r: FD3, w: FD4}

        │
        │ processes[0] (ls) setup:
        │   Stdin  = terminal (FD0)
        │   Stdout = pipes[0].w (FD4) ← pipe-এ লিখবে
        │
        │ processes[1] (grep) setup:
        │   Stdin  = pipes[0].r (FD3) ← pipe থেকে পড়বে
        │   Stdout = terminal (FD1)
        │
        ▼
processes[0].Start() → fork() + exec("ls")
processes[1].Start() → fork() + exec("grep", ".go")

এখন দুটো process একসাথে চলছে:

┌──────────────────────────────────────────────┐
│                                              │
│  ls (child 1)          grep (child 2)        │
│  ─────────────         ───────────────       │
│  main.go       ──►    pipe buffer    ──►     │
│  executor.go   ──►    [main.go\n     ──►     │
│  parser.go     ──►     executor.go\n ──►     │
│                        parser.go\n]          │
│                                     ──►  screen:
│                                         main.go
│  exit(0)               grep ".go" filter    │
│                        exit(0)              │
└──────────────────────────────────────────────┘
        │
        ▼
pipes[0].w.Close() → parent-এর write end বন্ধ
        │
        ▼
Wait(ls)   → শেষ হয়েছে ✓
Wait(grep) → শেষ হয়েছে ✓
        │
        ▼
pipes[0].r.Close()
        │
        ▼
tiny-shell ~> (prompt ফিরে এলো)
```

---

## 🧠 Deadlock — সবচেয়ে সাধারণ Pipe Bug

এটা এতটাই গুরুত্বপূর্ণ যে আলাদা করে বোঝাচ্ছি।

Parent-এ write end বন্ধ না করলে কী হয়:

```
ls শেষ হয়ে গেছে (child 1 exit)
grep এখনো পড়ছে (pipe থেকে)
grep ভাবছে: "আরো data আসবে, অপেক্ষা করি"

কেন ভাবছে?
কারণ pipe-এর write end এখনো খোলা আছে!
parent (shell) এর কাছে write end-এর copy আছে।

grep → pipe → "কেউ কি আরো লিখবে?" → হ্যাঁ! (parent এর কাছে আছে)
grep → চিরকাল অপেক্ষা করতে থাকে
shell → grep শেষ হওয়ার জন্য অপেক্ষা করছে
shell → চিরকাল অপেক্ষা করতে থাকে

══► DEADLOCK! কেউ এগোতে পারছে না।

Solution:
parent-এ write end বন্ধ করো:
pipes[i].w.Close()
এখন grep EOF পাবে এবং শেষ হবে।
```

---

## 🧪 Experiment করো

**১. Long pipeline চালাও:**
```bash
tiny-shell ~> cat /etc/passwd | cut -d: -f1 | sort | uniq | wc -l
```

**২. Pipe কতটুকু buffer রাখে দেখো:**

```go
// এটা একটা আলাদা test file-এ চালাও
// tiny-shell-এর বাইরে
package main

import (
    "fmt"
    "os"
)

func main() {
    r, w, _ := os.Pipe()
    
    // Pipe-এর capacity দেখো
    written := 0
    buf := make([]byte, 4096)
    for i := range buf {
        buf[i] = 'x'
    }
    
    // Non-blocking লেখার চেষ্টা
    // (এটা block করবে যখন pipe ভরে যাবে)
    for {
        n, err := w.Write(buf)
        written += n
        if err != nil {
            fmt.Printf("Pipe full at: %d bytes\n", written)
            break
        }
    }
    r.Close()
    w.Close()
}
```

Linux-এ pipe buffer সাধারণত **65536 bytes (64KB)**।

**৩. Debug mode দিয়ে pipeline দেখো:**
```
tiny-shell ~> debug ls -la | grep .go | wc -l
Pipeline (bg=false):
  Command[0]: Name:"ls"   Args:["-la"]
  Command[1]: Name:"grep" Args:[".go"]
  Command[2]: Name:"wc"   Args:["-l"]
```

---

## 📋 Milestone 4 Checklist

- ✅ Pipe কী এবং কীভাবে কাজ করে বুঝলাম
- ✅ FD redirect করা বুঝলাম
- ✅ fork + exec + pipe connect করা বুঝলাম
- ✅ Deadlock কেন হয় এবং কীভাবে এড়ানো যায় বুঝলাম
- ✅ Multiple pipe (3+ command) কাজ করছে
- ✅ `ls | grep | wc` style pipeline চলছে

---

Milestone 4 চললে বলো — **Milestone 5: I/O Redirection** শুরু করব। সেখানে `>`, `>>`, `<` implement করব। `dup2` syscall সরাসরি ব্যবহার করব — FD নিজে হাতে বদলে দেব। 🚀