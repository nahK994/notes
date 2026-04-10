# Milestone 3: Command Parsing

## আগে বুঝি — Parsing কেন দরকার?

এখন আমাদের shell-এ `strings.Fields()` দিয়ে input ভাগ করছি। এটা শুধু space দিয়ে split করে। কিন্তু real world-এ input অনেক জটিল হয়:

```
echo "hello world"        ← quotes-এর ভেতরে space আছে
echo 'it\'s fine'         ← escaped character
ls -la | grep ".go"       ← pipe আছে
cat file.txt > out.txt    ← redirection আছে
echo $HOME                ← variable আছে
```

`strings.Fields()` এগুলো handle করতে পারে না। তাই আমাদের নিজস্ব **Parser** বানাতে হবে।

---

## 🧠 Concept: Parsing কী?

Parsing মানে হলো একটা raw string-কে **structured data**-তে রূপান্তর করা।

Compiler যেভাবে source code বোঝে, আমাদের shell-ও সেভাবে input বুঝবে। দুটো ধাপে কাজ হয়:

```
Raw Input (String)
"ls -la | grep .go"
        │
        │  Step 1: LEXING / TOKENIZING
        │  (ছোট ছোট টুকরো করো, প্রতিটাকে label দাও)
        ▼
Tokens (labeled pieces)
┌──────┬──────┬──────┬──────┬────────┐
│ WORD │ WORD │ PIPE │ WORD │  WORD  │
│ "ls" │"-la" │ "|"  │"grep"│ ".go"  │
└──────┴──────┴──────┴──────┴────────┘
        │
        │  Step 2: PARSING
        │  (tokens দেখে মানে বোঝো, structure তৈরি করো)
        ▼
Command Structure
┌─────────────────────────────────────┐
│  Command 1    │    Command 2        │
│  Name: "ls"   │    Name: "grep"    │
│  Args: ["-la"]│    Args: [".go"]   │
│               │                    │
│  stdout ──────────────► stdin      │
│         (pipe connection)          │
└─────────────────────────────────────┘
```

এই দুই ধাপ আলাদা রাখলে code অনেক clean থাকে। আজকে আমরা **Lexer** আর **Parser** দুটোই বানাব।

---

## 🧠 Concept: Token কী?

Token হলো input-এর সবচেয়ে ছোট অর্থবহ টুকরো। প্রতিটা token-এর একটা **type** আছে:

```
Input: echo "hello world" | cat > out.txt

Token Analysis:
┌─────────────────┬───────────┬──────────────────────────────┐
│ Raw Text        │ Type      │ মানে                         │
├─────────────────┼───────────┼──────────────────────────────┤
│ echo            │ WORD      │ একটা সাধারণ word             │
│ "hello world"   │ WORD      │ quoted word (space সহ)       │
│ |               │ PIPE      │ pipe operator                │
│ cat             │ WORD      │ command নাম                  │
│ >               │ REDIRECT  │ output redirect              │
│ out.txt         │ WORD      │ redirect target              │
│ (শেষ)           │ EOF       │ input শেষ                    │
└─────────────────┴───────────┴──────────────────────────────┘
```

Lexer এই কাজটা করে — raw characters পড়ে token বানায়।

---

## 🧠 Concept: Lexer কীভাবে কাজ করে?

Lexer একটা character-by-character state machine। সে এক character পড়ে, দেখে কোন "অবস্থায়" (state) আছে, সেই অনুযায়ী সিদ্ধান্ত নেয়:

```
Input: echo "hello world"
       ↑
       এখানে আছি

States:
┌─────────────────────────────────────────────────────┐
│                                                     │
│  NORMAL ──(space দেখলে)──► token শেষ করো           │
│    │                                                │
│    │ ('"' দেখলে)                                    │
│    ▼                                                │
│  IN_DOUBLE_QUOTE ──('"' আবার দেখলে)──► NORMAL      │
│    │                                                │
│    │ (যেকোনো character)                             │
│    ▼                                                │
│  token-এ জুড়তে থাকো (space হলেও!)                  │
│                                                     │
└─────────────────────────────────────────────────────┘

Character by character:
'e' → NORMAL, token = "e"
'c' → NORMAL, token = "ec"
'h' → NORMAL, token = "ech"
'o' → NORMAL, token = "echo"
' ' → NORMAL + space → token "echo" DONE ✓, নতুন token শুরু
'"' → IN_DOUBLE_QUOTE state-এ যাও
'h' → IN_DOUBLE_QUOTE, token = "h"
'e' → IN_DOUBLE_QUOTE, token = "he"
'l' → IN_DOUBLE_QUOTE, token = "hel"
'l' → IN_DOUBLE_QUOTE, token = "hell"
'o' → IN_DOUBLE_QUOTE, token = "hello"
' ' → IN_DOUBLE_QUOTE, space IGNORE করো না! token = "hello "
'w' → IN_DOUBLE_QUOTE, token = "hello w"
...
'"' → NORMAL state-এ ফিরে যাও, token "hello world" DONE ✓
```

এই state machine টাই আমাদের Lexer।

---

## 🏗️ Code

### Step 1: Token Types — `internal/parser/token.go`

```go
// internal/parser/token.go
package parser

import "fmt"

// TokenType — token কী ধরনের
type TokenType int

const (
	// WORD: সাধারণ word — "ls", "-la", "hello world" (quoted)
	TOKEN_WORD TokenType = iota

	// PIPE: "|" — দুই command-এর মাঝে
	TOKEN_PIPE

	// Redirection operators
	TOKEN_REDIRECT_OUT    // ">"  — stdout কে file-এ পাঠাও
	TOKEN_REDIRECT_APPEND // ">>" — stdout কে file-এ append করো
	TOKEN_REDIRECT_IN     // "<"  — file থেকে stdin নাও

	// BACKGROUND: "&" — background-এ চালাও (Milestone 6-এ কাজে লাগবে)
	TOKEN_BACKGROUND

	// EOF: input শেষ
	TOKEN_EOF
)

// Token — একটা টুকরো
type Token struct {
	Type  TokenType
	Value string // TOKEN_WORD-এর জন্য actual text
}

// String — debug করার সময় সুন্দর দেখাবে
func (t Token) String() string {
	switch t.Type {
	case TOKEN_WORD:
		return fmt.Sprintf("WORD(%q)", t.Value)
	case TOKEN_PIPE:
		return "PIPE"
	case TOKEN_REDIRECT_OUT:
		return "REDIRECT_OUT"
	case TOKEN_REDIRECT_APPEND:
		return "REDIRECT_APPEND"
	case TOKEN_REDIRECT_IN:
		return "REDIRECT_IN"
	case TOKEN_BACKGROUND:
		return "BACKGROUND"
	case TOKEN_EOF:
		return "EOF"
	}
	return "UNKNOWN"
}
```

---

### Step 2: Lexer — `internal/parser/lexer.go`

```go
// internal/parser/lexer.go
package parser

import "fmt"

// Lexer — raw string থেকে tokens বের করে
type Lexer struct {
	input []rune // input string (rune = unicode character)
	pos   int    // এখন কোন character-এ আছি
}

// NewLexer — নতুন lexer তৈরি করো
func NewLexer(input string) *Lexer {
	return &Lexer{
		input: []rune(input), // string কে rune slice-এ convert করো
		pos:   0,
	}
}

// current — এখন যে character-এ আছি সেটা দেখো
func (l *Lexer) current() rune {
	if l.pos >= len(l.input) {
		return 0 // 0 মানে EOF
	}
	return l.input[l.pos]
}

// advance — পরের character-এ যাও
func (l *Lexer) advance() {
	l.pos++
}

// skipWhitespace — space, tab skip করো
func (l *Lexer) skipWhitespace() {
	for l.current() == ' ' || l.current() == '\t' {
		l.advance()
	}
}

// Tokenize — পুরো input-কে tokens-এ ভাগ করো
func (l *Lexer) Tokenize() ([]Token, error) {
	var tokens []Token

	for {
		// আগে whitespace skip করো
		l.skipWhitespace()

		// Input শেষ?
		if l.current() == 0 {
			tokens = append(tokens, Token{Type: TOKEN_EOF})
			break
		}

		ch := l.current()

		// ── Special Characters ──────────────────────────
		switch ch {

		case '|':
			// Pipe operator
			tokens = append(tokens, Token{Type: TOKEN_PIPE, Value: "|"})
			l.advance()

		case '&':
			// Background operator
			tokens = append(tokens, Token{Type: TOKEN_BACKGROUND, Value: "&"})
			l.advance()

		case '>':
			// Output redirect বা append?
			l.advance()
			if l.current() == '>' {
				// ">>" append
				tokens = append(tokens, Token{Type: TOKEN_REDIRECT_APPEND, Value: ">>"})
				l.advance()
			} else {
				// ">" overwrite
				tokens = append(tokens, Token{Type: TOKEN_REDIRECT_OUT, Value: ">"})
			}

		case '<':
			// Input redirect
			tokens = append(tokens, Token{Type: TOKEN_REDIRECT_IN, Value: "<"})
			l.advance()

		// ── Quoted Strings ──────────────────────────────
		case '"':
			// Double quote: "hello world"
			word, err := l.readDoubleQuoted()
			if err != nil {
				return nil, err
			}
			tokens = append(tokens, Token{Type: TOKEN_WORD, Value: word})

		case '\'':
			// Single quote: 'hello world'
			word, err := l.readSingleQuoted()
			if err != nil {
				return nil, err
			}
			tokens = append(tokens, Token{Type: TOKEN_WORD, Value: word})

		// ── Regular Words ────────────────────────────────
		default:
			// সাধারণ word পড়ো
			word := l.readWord()
			tokens = append(tokens, Token{Type: TOKEN_WORD, Value: word})
		}
	}

	return tokens, nil
}

// readWord — space বা special character না পাওয়া পর্যন্ত পড়ো
func (l *Lexer) readWord() string {
	var result []rune

	for {
		ch := l.current()

		// এগুলো দেখলে word শেষ
		if ch == 0 || ch == ' ' || ch == '\t' ||
			ch == '|' || ch == '&' ||
			ch == '>' || ch == '<' {
			break
		}

		// Backslash escape: \n, \t, \" ইত্যাদি
		if ch == '\\' {
			l.advance()
			next := l.current()
			if next == 0 {
				break
			}
			result = append(result, next)
			l.advance()
			continue
		}

		result = append(result, ch)
		l.advance()
	}

	return string(result)
}

// readDoubleQuoted — double quote-এর ভেতরে পড়ো
// "hello world" → hello world
// escape sequences handle করো: \", \\, \n
func (l *Lexer) readDoubleQuoted() (string, error) {
	l.advance() // opening '"' skip করো

	var result []rune

	for {
		ch := l.current()

		if ch == 0 {
			// Quote বন্ধ হয়নি — error
			return "", fmt.Errorf("lexer: unterminated double quote")
		}

		if ch == '"' {
			// Closing quote পাওয়া গেছে
			l.advance()
			break
		}

		// Escape sequence inside double quote
		if ch == '\\' {
			l.advance()
			next := l.current()
			switch next {
			case '"':
				result = append(result, '"')
			case '\\':
				result = append(result, '\\')
			case 'n':
				result = append(result, '\n')
			case 't':
				result = append(result, '\t')
			default:
				// Unknown escape: backslash রাখো
				result = append(result, '\\')
				result = append(result, next)
			}
			l.advance()
			continue
		}

		result = append(result, ch)
		l.advance()
	}

	return string(result), nil
}

// readSingleQuoted — single quote-এর ভেতরে পড়ো
// 'hello world' → hello world
// Single quote-এ কোনো escape নেই — সব literal
func (l *Lexer) readSingleQuoted() (string, error) {
	l.advance() // opening "'" skip করো

	var result []rune

	for {
		ch := l.current()

		if ch == 0 {
			return "", fmt.Errorf("lexer: unterminated single quote")
		}

		if ch == '\'' {
			l.advance()
			break
		}

		// Single quote-এ সব character literal
		// কোনো escape নেই
		result = append(result, ch)
		l.advance()
	}

	return string(result), nil
}
```

---

### Step 3: Parser — `internal/parser/parser.go`

Lexer দিল tokens। এখন Parser সেই tokens দেখে মানে বোঝাবে।

```go
// internal/parser/parser.go
package parser

import "fmt"

// Redirection — একটা I/O redirect
// যেমন: > out.txt, < in.txt, >> log.txt
type Redirection struct {
	Type TokenType // TOKEN_REDIRECT_OUT, IN, APPEND
	File string    // কোন file-এ
}

// Command — একটা single command
// যেমন: ls -la, grep ".go", echo "hello"
type Command struct {
	Name         string        // command নাম: "ls"
	Args         []string      // arguments: ["-la", "/tmp"]
	Redirections []Redirection // I/O redirections
	Background   bool          // background-এ চালাতে হবে?
}

// Pipeline — pipe দিয়ে জোড়া commands
// ls -la | grep .go | wc -l
// এটা তিনটা Command-এর একটা list
type Pipeline struct {
	Commands   []*Command
	Background bool // পুরো pipeline background-এ?
}

// Parser — tokens থেকে structure বানায়
type Parser struct {
	tokens []Token
	pos    int
}

// NewParser — নতুন parser তৈরি করো
func NewParser(tokens []Token) *Parser {
	return &Parser{tokens: tokens, pos: 0}
}

// current — এখন যে token-এ আছি
func (p *Parser) current() Token {
	if p.pos >= len(p.tokens) {
		return Token{Type: TOKEN_EOF}
	}
	return p.tokens[p.pos]
}

// advance — পরের token-এ যাও
func (p *Parser) advance() Token {
	t := p.current()
	p.pos++
	return t
}

// Parse — সব tokens পড়ে একটা Pipeline বানাও
func (p *Parser) Parse() (*Pipeline, error) {
	pipeline := &Pipeline{}

	for {
		// একটা command পড়ো
		cmd, err := p.parseCommand()
		if err != nil {
			return nil, err
		}

		// Command খালি? (শুধু pipe বা EOF)
		if cmd == nil || cmd.Name == "" {
			break
		}

		pipeline.Commands = append(pipeline.Commands, cmd)

		// এরপর কী আছে?
		switch p.current().Type {

		case TOKEN_PIPE:
			// "|" দেখলে আরেকটা command আসবে
			p.advance() // "|" consume করো
			// loop চলতে থাকবে, পরের command পড়বে

		case TOKEN_BACKGROUND:
			// "&" দেখলে background
			p.advance()
			pipeline.Background = true
			return pipeline, nil

		case TOKEN_EOF:
			// শেষ
			return pipeline, nil

		default:
			return nil, fmt.Errorf("parser: unexpected token: %v",
				p.current())
		}
	}

	return pipeline, nil
}

// parseCommand — একটা command parse করো
// pipe বা EOF পেলে থামো
func (p *Parser) parseCommand() (*Command, error) {
	cmd := &Command{}

	for {
		tok := p.current()

		switch tok.Type {

		case TOKEN_WORD:
			p.advance()
			if cmd.Name == "" {
				// প্রথম word = command নাম
				cmd.Name = tok.Value
			} else {
				// বাকিগুলো = arguments
				cmd.Args = append(cmd.Args, tok.Value)
			}

		case TOKEN_REDIRECT_OUT, TOKEN_REDIRECT_APPEND, TOKEN_REDIRECT_IN:
			// Redirection: "> file", "< file", ">> file"
			redirectType := tok.Type
			p.advance() // operator consume করো

			// পরের token = file নাম
			if p.current().Type != TOKEN_WORD {
				return nil, fmt.Errorf(
					"parser: expected filename after redirect operator")
			}
			fileName := p.advance().Value

			cmd.Redirections = append(cmd.Redirections, Redirection{
				Type: redirectType,
				File: fileName,
			})

		case TOKEN_PIPE, TOKEN_BACKGROUND, TOKEN_EOF:
			// এই command শেষ — caller handle করবে
			return cmd, nil

		default:
			return nil, fmt.Errorf("parser: unexpected token: %v", tok)
		}
	}
}

// Parse — convenience function: string থেকে সরাসরি Pipeline
func Parse(input string) (*Pipeline, error) {
	// Step 1: Lex
	lexer := NewLexer(input)
	tokens, err := lexer.Tokenize()
	if err != nil {
		return nil, fmt.Errorf("lex error: %w", err)
	}

	// Step 2: Parse
	parser := NewParser(tokens)
	pipeline, err := parser.Parse()
	if err != nil {
		return nil, fmt.Errorf("parse error: %w", err)
	}

	return pipeline, nil
}
```

---

### Step 4: Executor আপডেট করো

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
	// Step 1: Parse করো
	pipeline, err := parser.Parse(line)
	if err != nil {
		return err
	}

	// Empty pipeline?
	if len(pipeline.Commands) == 0 {
		return nil
	}

	// Single command? (pipe নেই)
	if len(pipeline.Commands) == 1 {
		return executeSingle(pipeline.Commands[0])
	}

	// Multiple commands — pipe আছে (Milestone 4-এ পুরোটা করব)
	// এখনকে জন্য শুধু প্রথমটা চালাই
	fmt.Println("[pipes Milestone 4-এ আসবে]")
	return executeSingle(pipeline.Commands[0])
}

// executeSingle — একটা command চালাও
func executeSingle(cmd *parser.Command) error {
	if cmd.Name == "" {
		return nil
	}

	// Built-in চেক
	allArgs := append([]string{cmd.Name}, cmd.Args...)
	handled, err := builtins.Execute(allArgs)
	if handled {
		return err
	}

	// External command
	return runCommand(cmd)
}

// runCommand — fork-exec করো
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
tiny-shell ~> echo "hello world"
hello world

tiny-shell ~> echo 'single quoted'
single quoted

tiny-shell ~> echo hello     world
hello world

tiny-shell ~> ls -la
(output আসবে)

tiny-shell ~> echo "tab:\there"
tab:	here
```

---

## 🔬 "What just happened?" — Parsing flow দেখো

`echo "hello world"` টাইপ করলে:

```
Input: echo "hello world"
          │
          ▼ LEXER
          
char by char পড়ো:
'e','c','h','o' → WORD token: "echo"
' '             → space, token শেষ
'"'             → double quote mode চালু
'h','e','l','l','o',' ','w','o','r','l','d' → সব জুড়তে থাকো
'"'             → double quote mode বন্ধ → WORD token: "hello world"
'\0'            → EOF token

Tokens:
[WORD("echo"), WORD("hello world"), EOF]
          │
          ▼ PARSER

Token 1: WORD("echo") → cmd.Name = "echo"
Token 2: WORD("hello world") → cmd.Args = ["hello world"]
Token 3: EOF → command শেষ

Pipeline:
└── Command
    ├── Name: "echo"
    └── Args: ["hello world"]
          │
          ▼ EXECUTOR

builtins.Execute(["echo", "hello world"])
→ strings.Join(["hello world"], " ")
→ fmt.Println("hello world")
→ screen-এ দেখায়: hello world  ✓
```

---

## 🧪 Debug Mode যোগ করো

Parser ঠিকমতো কাজ করছে কিনা দেখতে একটা debug mode যোগ করো `main.go`-তে:

```go
// main.go-তে executor.Execute-এর আগে এটা যোগ করো:
if strings.HasPrefix(line, "debug ") {
    input := line[6:]
    pipeline, err := parser.Parse(input)
    if err != nil {
        fmt.Println("Parse error:", err)
        continue
    }
    fmt.Printf("Pipeline (bg=%v):\n", pipeline.Background)
    for i, cmd := range pipeline.Commands {
        fmt.Printf("  Command[%d]:\n", i)
        fmt.Printf("    Name: %q\n", cmd.Name)
        fmt.Printf("    Args: %v\n", cmd.Args)
        fmt.Printf("    Redirections: %v\n", cmd.Redirections)
    }
    continue
}
```

import-এ `"tiny-shell/internal/parser"` যোগ করো। এখন:

```
tiny-shell ~> debug echo "hello world" | grep hello
Pipeline (bg=false):
  Command[0]:
    Name: "echo"
    Args: ["hello world"]
    Redirections: []
  Command[1]:
    Name: "grep"
    Args: ["hello"]
    Redirections: []
```

Parser সব বুঝতে পারছে কিনা এখানেই দেখা যাবে।

---

## 📋 Milestone 3 Checklist

- ✅ Lexer বুঝলাম — state machine দিয়ে character পড়া
- ✅ Token কী এবং কেন দরকার বুঝলাম
- ✅ Parser বুঝলাম — tokens থেকে structure তৈরি
- ✅ Quoted strings handle হচ্ছে
- ✅ Pipe, redirection tokens চেনা যাচ্ছে
- ✅ Pipeline struct তৈরি হচ্ছে

---

Milestone 3 চললে বলো — **Milestone 4: Pipes** শুরু করব। এটা সবচেয়ে মজার part। File descriptor নিজে হাতে তৈরি করব, দুটো process-এর মাঝে data পাঠাব। OS-এর pipe() syscall সরাসরি ব্যবহার করব। 🚀