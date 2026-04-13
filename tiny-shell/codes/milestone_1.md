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
