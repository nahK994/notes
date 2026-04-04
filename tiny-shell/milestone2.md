# Milestone 2: Built-in Commands

## আগে একটা প্রশ্ন — `cd` কেন আলাদা?

Shell-এ `ls` টাইপ করলে আমরা নতুন process বানাই, সেখানে `ls` চালাই। কিন্তু `cd` এভাবে কাজ করে না। কেন?

কারণটা বুঝতে হলে আগে **Working Directory** বুঝতে হবে।

---

## 🧠 Concept: Current Working Directory (CWD)

প্রতিটা process-এর নিজস্ব একটা "আমি এখন কোথায় আছি" আছে। এটাকে বলে **Current Working Directory**।

```
Process Table (Kernel রাখে)
┌─────────────────────────────────────────┐
│ PID  │ Name       │ CWD                 │
├──────┼────────────┼─────────────────────┤
│ 890  │ bash       │ /home/shomi         │
│ 1234 │ tiny-shell │ /home/shomi         │ ← shell-এর CWD
│ 1235 │ ls         │ /home/shomi         │ ← child inherit করে
└─────────────────────────────────────────┘
```

Child process সবসময় parent-এর CWD **copy** করে নেয় — inherit করে।

এখন সমস্যাটা দেখো:

```
tiny-shell (PID: 1234, CWD: /home/shomi)
       │
       │  "cd /tmp" command আসলো
       │
       ▼
  নতুন child process বানাই (PID: 1235)
  child-এর CWD: /home/shomi (copy করা)
       │
       │  child-এ chdir("/tmp") করি
       │  child-এর CWD বদলে /tmp হলো
       │
       ▼
  child exit করলো
       │
       ▼
tiny-shell (PID: 1234, CWD: /home/shomi) ← একই রয়ে গেলো!
```

Child-এর CWD বদলালে Parent-এর CWD বদলায় না। Process-গুলো আলাদা — একজনের পরিবর্তন অন্যজনকে ছোঁয় না।

তাই `cd` অবশ্যই **shell নিজেই** করতে হবে — কোনো child process ছাড়া। এই ধরনের commands যেগুলো shell নিজে handle করে, সেগুলোকে বলে **Built-in Commands**।

```
External Command (ls, grep, cat...)    Built-in Command (cd, exit, echo)
──────────────────────────────────     ─────────────────────────────────
Shell → fork() → child process         Shell নিজেই করে
      → exec() → ls চালে              কোনো child নেই
      → wait() → শেষ হয়              সরাসরি shell-এর state বদলায়
```

---

## 🧠 Concept: Environment Variables

`cd` বোঝার পর আরেকটা জিনিস বুঝতে হবে — **Environment Variables**।

প্রতিটা process-এর সাথে একটা key-value store থাকে:

```
Process (tiny-shell)
├── Code
├── Stack / Heap
├── File Descriptors
└── Environment Variables ◄── এটা নতুন
    ├── PATH=/usr/bin:/bin:/usr/local/bin
    ├── HOME=/home/shomi
    ├── USER=shomi
    ├── PWD=/home/shomi
    └── OLDPWD=/tmp
```

**PATH** সবচেয়ে গুরুত্বপূর্ণ। তুমি যখন `ls` টাইপ করো, shell কীভাবে বোঝে `/bin/ls` চালাতে হবে?

```
PATH = /usr/bin : /bin : /usr/local/bin
         │           │         │
         ▼           ▼         ▼
"ls" খোঁজো:
/usr/bin/ls  → নেই
/bin/ls      → পাওয়া গেছে! ✓ এটাই চালাও
```

Child process parent-এর environment **copy** করে পায়। তাই `ls` জানে PATH কী, HOME কোথায়।

---

## 🏗️ Code: Milestone 2

এখন আমরা `executor.go` আর `main.go` আপডেট করব এবং `builtins.go` তৈরি করব।

### Step 1: `builtins.go` — Built-in Commands

```go
// internal/builtins/builtins.go
package builtins

import (
	"fmt"
	"os"
	"strings"
)

// IsBuiltin চেক করে command টা built-in কিনা
func IsBuiltin(name string) bool {
	switch name {
	case "cd", "exit", "echo", "pwd", "env":
		return true
	}
	return false
}

// Execute built-in command চালায়
// return: (handled bool, err error)
// handled=true মানে আমরা এটা handle করেছি
func Execute(args []string) (bool, error) {
	if len(args) == 0 {
		return false, nil
	}

	switch args[0] {
	case "cd":
		return true, runCd(args[1:])
	case "exit":
		return true, runExit(args[1:])
	case "echo":
		return true, runEcho(args[1:])
	case "pwd":
		return true, runPwd()
	case "env":
		return true, runEnv()
	}

	return false, nil
}

// ── cd ──────────────────────────────────────────────────────
func runCd(args []string) error {
	var target string

	if len(args) == 0 {
		// "cd" একা মানে HOME-এ যাও
		// os.UserHomeDir() HOME env variable পড়ে
		home, err := os.UserHomeDir()
		if err != nil {
			return fmt.Errorf("cd: HOME not set")
		}
		target = home
	} else {
		target = args[0]
	}

	// চলে যাওয়ার আগে আগের directory মনে রাখো
	// এটা "cd -" এর জন্য দরকার হবে
	oldPwd, _ := os.Getwd()

	// os.Chdir() kernel-কে বলে এই process-এর CWD বদলাও
	// এটা syscall: chdir(path)
	// Shell নিজে call করছে — তাই shell-এর CWD বদলাবে
	if err := os.Chdir(target); err != nil {
		return fmt.Errorf("cd: %s: %v", target, err)
	}

	// নতুন PWD বের করো (symlink resolve হতে পারে)
	newPwd, err := os.Getwd()
	if err != nil {
		return err
	}

	// Environment update করো
	// এই দুটো variable অনেক program ব্যবহার করে
	os.Setenv("OLDPWD", oldPwd)
	os.Setenv("PWD", newPwd)

	return nil
}

// ── exit ─────────────────────────────────────────────────────
func runExit(args []string) error {
	exitCode := 0

	if len(args) > 0 {
		// "exit 1" — exit code সহ বের হও
		code := 0
		_, err := fmt.Sscanf(args[0], "%d", &code)
		if err != nil {
			return fmt.Errorf("exit: %s: numeric argument required", args[0])
		}
		exitCode = code
	}

	fmt.Println("বাই! 👋")
	// os.Exit() এই process সম্পূর্ণ বন্ধ করে দেয়
	// kernel syscall: exit_group(exitCode)
	os.Exit(exitCode)
	return nil // এখানে কখনো পৌঁছাবে না
}

// ── echo ─────────────────────────────────────────────────────
func runEcho(args []string) error {
	// args গুলো জোড়া লাগিয়ে print করো
	// "echo hello world" → "hello world"
	fmt.Println(strings.Join(args, " "))
	return nil
}

// ── pwd ──────────────────────────────────────────────────────
func runPwd() error {
	// os.Getwd() kernel-কে জিজ্ঞেস করে "আমার CWD কী?"
	// syscall: getcwd()
	cwd, err := os.Getwd()
	if err != nil {
		return fmt.Errorf("pwd: %v", err)
	}
	fmt.Println(cwd)
	return nil
}

// ── env ──────────────────────────────────────────────────────
func runEnv() error {
	// os.Environ() সব environment variable দেয়
	// format: "KEY=VALUE"
	for _, env := range os.Environ() {
		fmt.Println(env)
	}
	return nil
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
	"strings"

	"tiny-shell/internal/builtins"
)

func Execute(line string) error {
	parts := strings.Fields(line)
	if len(parts) == 0 {
		return nil
	}

	// ── Built-in আগে চেক করো ──
	// Built-in হলে shell নিজেই handle করবে
	// External হলে fork-exec করবে
	handled, err := builtins.Execute(parts)
	if handled {
		return err
	}

	// Built-in না হলে external command চালাও
	return runCommand(parts[0], parts[1:])
}

func runCommand(name string, args []string) error {
	cmd := exec.Command(name, args...)

	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		if exitErr, ok := err.(*exec.ExitError); ok {
			return fmt.Errorf("%s: exit status %d",
				name, exitErr.ExitCode())
		}
		if err == exec.ErrNotFound {
			return fmt.Errorf("%s: command not found", name)
		}
		return err
	}

	return nil
}
```

---

### Step 3: `main.go` আপডেট — Smarter Prompt

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
	scanner := bufio.NewScanner(os.Stdin)

	for {
		// ── Dynamic Prompt ──
		// Real shell-এর মতো current directory দেখাও
		cwd, err := os.Getwd()
		if err != nil {
			cwd = "?"
		}

		// HOME কে "~" দিয়ে replace করো (real shell-এর মতো)
		// "/home/shomi/projects" → "~/projects"
		home, _ := os.UserHomeDir()
		if home != "" && strings.HasPrefix(cwd, home) {
			cwd = "~" + cwd[len(home):]
		}

		fmt.Printf("tiny-shell %s> ", cwd)

		if !scanner.Scan() {
			fmt.Println("\nবাই! 👋")
			break
		}

		line := strings.TrimSpace(scanner.Text())
		if line == "" {
			continue
		}

		if err := executor.Execute(line); err != nil {
			fmt.Fprintln(os.Stderr, "error:", err)
		}
	}
}
```

---

## চালাও এবং দেখো

```bash
go run cmd/shell/main.go
```

```
tiny-shell ~> pwd
/home/shomi

tiny-shell ~> cd /tmp
tiny-shell /tmp> pwd
/tmp

tiny-shell /tmp> cd
tiny-shell ~> pwd
/home/shomi

tiny-shell ~> echo hello world
hello world

tiny-shell ~> echo $HOME
$HOME

tiny-shell ~> env | grep PATH
PATH=/usr/bin:/bin:/usr/local/bin:...
```

---

## 🔬 "What just happened?" — `cd /tmp` করলে OS-এ কী হলো

```
তুমি "cd /tmp" টাইপ করলে
         │
         ▼
executor.Execute() → builtins.Execute() চেক করলো
         │
         │  "cd" built-in? → হ্যাঁ
         │
         ▼
runCd(["/tmp"]) call হলো
         │
         │  os.Getwd() → syscall: getcwd()
         │  Kernel বললো: "এখন /home/shomi"
         │  oldPwd = "/home/shomi"
         │
         ▼
os.Chdir("/tmp")
         │
         │  syscall: chdir("/tmp")
         │  Kernel tiny-shell process-এর
         │  CWD বদলে /tmp করে দিলো
         │
         ▼
os.Setenv("OLDPWD", "/home/shomi")
os.Setenv("PWD", "/tmp")
         │
         │  Environment table আপডেট হলো
         │
         ▼
main.go তে আবার prompt বানানোর সময়
os.Getwd() → "/tmp"
         │
         ▼
"tiny-shell /tmp> " দেখালো ✓

── কোনো child process তৈরি হয়নি ──
── shell নিজের CWD নিজে বদলেছে ──
```

---

## 🧪 Experiment করো

**১. cd - (আগের directory)** নিজে implement করো:

```go
// runCd-এ এটা add করো, os.Chdir-এর আগে:
if len(args) > 0 && args[0] == "-" {
    oldPwd := os.Getenv("OLDPWD")
    if oldPwd == "" {
        return fmt.Errorf("cd: OLDPWD not set")
    }
    // args[0] বদলে দাও
    args[0] = oldPwd
    fmt.Println(oldPwd) // bash এটা print করে
}
```

**২. Environment variable expand করো:**

Shell-এ `echo $HOME` করলে এখন `$HOME` literal আসে। Real shell এটাকে value দিয়ে replace করে। Milestone 3-এ parser বানানোর সময় এটা করব।

**৩. নিজে verify করো:**

```bash
# tiny-shell চালু করো, তারপর:
tiny-shell ~> cd /tmp
tiny-shell /tmp> cat /proc/$$/status | grep -i pwd
# এটা কাজ করবে না কারণ $$ expand হয় না এখনো
# কিন্তু আলাদা terminal থেকে:
# cat /proc/[tiny-shell PID]/cwd দেখো
ls -la /proc/$(pgrep tiny-shell)/cwd
```

দেখবে `/tmp` দেখাচ্ছে — kernel সত্যিই CWD বদলে দিয়েছে।

---

## 📋 Milestone 2 Checklist

- ✅ Built-in vs External command পার্থক্য বুঝলাম
- ✅ `cd` implement হলো — CWD কেন shell নিজে বদলায় বুঝলাম
- ✅ `echo`, `pwd`, `env` কাজ করছে
- ✅ Environment variables বুঝলাম
- ✅ Dynamic prompt — current directory দেখাচ্ছে

---

Milestone 2 চললে বলো — **Milestone 3: Command Parsing** শুরু করব। সেখানে শিখব কীভাবে `echo "hello world"` বা `ls -la | grep txt` এই ধরনের complex input-কে ভাঙা যায়। Tokenizer আর Parser বানাব — compiler-এর মতো চিন্তা করতে হবে। 🚀