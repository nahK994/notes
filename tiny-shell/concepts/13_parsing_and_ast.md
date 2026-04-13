# Parsing এবং AST: যখন Shell একজন Compiler হয়ে ওঠে

তুমি terminal-এ লিখলে:

```bash
ls -la | grep ".go" > output.txt
```

Shell এই line টা দেখে কী করে? সে কি শুধু space দিয়ে ভাগ করে নেয়? `["ls", "-la", "|", "grep", ".go", ">", "output.txt"]` — এই list করে রাখে?

না। এটা করলে shell কখনো জানতে পারত না `|` একটা operator, `".go"` একটা quoted string, আর `>` মানে output কোথাও পাঠাও। সবকিছু এক করে দেখলে মানে হারিয়ে যায়।

Shell আসলে যা করে সেটা compiler-এর কাজের সাথে হুবহু মিলে যায়। সে input-কে **বোঝে** — শুধু ভাগ করে না।

এই বোঝার প্রক্রিয়াটাই **Parsing**।

---

## দুটো ধাপ, দুটো ভিন্ন কাজ

Parsing আসলে এক ধাপের কাজ না। দুটো আলাদা স্তরে হয়:

```
Raw Input (String)
"ls -la | grep ".go" > output.txt"
            │
            │  ধাপ ১: LEXING
            │  (অক্ষর পড়ো, অর্থপূর্ণ টুকরো বানাও)
            ▼
         Tokens
┌──────┬──────┬──────┬────────┬──────┬──────────┐
│ WORD │ WORD │ PIPE │  WORD  │  >   │   WORD   │
│ "ls" │"-la" │ "|"  │ ".go"  │      │"output"  │
└──────┴──────┴──────┴────────┴──────┴──────────┘
            │
            │  ধাপ ২: PARSING
            │  (tokens দেখে সম্পর্ক বোঝো, structure বানাও)
            ▼
         AST (Abstract Syntax Tree)
```

প্রথম ধাপে raw text-কে **token** বানানো হয় — ছোট ছোট অর্থপূর্ণ টুকরো। দ্বিতীয় ধাপে সেই token গুলো দেখে **গঠন** বোঝা হয় — কে কার সাথে সম্পর্কিত, কোনটা command, কোনটা argument, কোনটা operator।

এই দুটো ধাপ আলাদা রাখার কারণ আছে। একসাথে করতে গেলে code জটিল হয়ে যায়, bug ধরা কঠিন হয়। আলাদা রাখলে প্রতিটা ধাপ নিজের কাজটুকু করে — বাকিটা পরের ধাপের দায়িত্ব।

---

## Lexer: অক্ষর থেকে Token

Lexer একটা **state machine**। সে একটা একটা করে character পড়ে, আর মনে মনে ঠিক করে — "এই character দিয়ে কী হচ্ছে?"

ভাবো একজন মানুষ একটা বাক্য পড়ছে। সে শব্দের মধ্যে letter দেখলে পড়তে থাকে। Space দেখলে বোঝে শব্দ শেষ। উদ্ধৃতিচিহ্ন দেখলে বোঝে ভেতরের সব একটাই unit — space হলেও।

Lexer ঠিক এভাবেই ভাবে:

```
Input: echo "hello world" | cat

e → NORMAL state, token = "e"
c → NORMAL state, token = "ec"
h → NORMAL state, token = "ech"
o → NORMAL state, token = "echo"
' '→ space! token "echo" শেষ ✓ → WORD("echo")

'"'→ DOUBLE_QUOTE state চালু
h → DOUBLE_QUOTE, token = "h"
e → DOUBLE_QUOTE, token = "he"
l → DOUBLE_QUOTE, token = "hel"
l → DOUBLE_QUOTE, token = "hell"
o → DOUBLE_QUOTE, token = "hello"
' '→ DOUBLE_QUOTE-এ space ignore হয় না!
    token = "hello "
w → DOUBLE_QUOTE, token = "hello w"
o → DOUBLE_QUOTE, token = "hello wo"
r → DOUBLE_QUOTE, token = "hello wor"
l → DOUBLE_QUOTE, token = "hello worl"
d → DOUBLE_QUOTE, token = "hello world"
'"'→ DOUBLE_QUOTE শেষ! ✓ → WORD("hello world")

' '→ space, skip
'|'→ → PIPE token ✓

' '→ space, skip
c → NORMAL, token = "c"
a → NORMAL, token = "ca"
t → NORMAL, token = "cat"
END → token "cat" শেষ ✓ → WORD("cat")

চূড়ান্ত tokens:
[WORD("echo"), WORD("hello world"), PIPE, WORD("cat"), EOF]
```

লক্ষ্য করো — `"hello world"` একটাই token হয়েছে। Quote না থাকলে দুটো হতো। Lexer-এর state machine-এই এই পার্থক্যটা ধরা পড়ে।

### Token-এর প্রকারভেদ

```
┌─────────────────────┬──────────────────────────────┐
│ Token Type          │ উদাহরণ                       │
├─────────────────────┼──────────────────────────────┤
│ WORD                │ ls, -la, "hello world", file  │
│ PIPE                │ |                             │
│ REDIRECT_OUT        │ >                             │
│ REDIRECT_APPEND     │ >>                            │
│ REDIRECT_IN         │ <                             │
│ BACKGROUND          │ &                             │
│ EOF                 │ (input শেষ)                   │
└─────────────────────┴──────────────────────────────┘
```

Lexer শুধু token চেনে — মানে বোঝে না। `>` দেখলে সে বলে "এটা REDIRECT_OUT"। কিন্তু এর পরে কী আসবে, এটা কোন command-এর অংশ — এসব Lexer-এর কাজ না। সেটা Parser-এর কাজ।

---

## Parser: Token থেকে গঠন

Lexer tokens দিয়েছে। এখন Parser সেই tokens দেখে **মানে** বোঝে।

কিন্তু শুধু মানে বুঝলেই হবে না — সেই মানেটাকে একটা **structure**-এ রাখতে হবে যেটা পরে execute করা যাবে। এই structure-এর নাম **AST — Abstract Syntax Tree**।

"Tree" কেন? কারণ এই structure একটা গাছের মতো — উপরে root, নিচে branches, শেষে leaves।

```
Input: ls -la | grep ".go" > output.txt

AST:
              PIPELINE
             /        \
        COMMAND       COMMAND
        /     \       /  |   \
      "ls"   "-la"  "grep" ".go"  REDIRECT
                               /        \
                             ">"      "output.txt"
```

গাছের প্রতিটা node একটা অর্থপূর্ণ unit। Root থেকে নিচে নামলে ছোট ছোট অংশে পৌঁছানো যায়। Executor এই গাছ হেঁটে হেঁটে কাজ করে।

---

## AST: একটু জটিল উদাহরণ

Simple pipeline বোঝা গেল। কিন্তু real shell-এ input অনেক জটিল হতে পারে:

```bash
cat file.txt | grep "error" | sort | uniq -c > report.txt
```

এর AST:

```
                    PIPELINE
                 /     |     |    \
            CMD       CMD   CMD    CMD
            │          │     │      │  \
           "cat"     "grep" "sort" "uniq" REDIRECT
            │          │            │    /        \
         "file.txt" "error"        "-c" ">"   "report.txt"
```

Executor এই গাছ দেখে বুঝবে:
- চারটা command আছে
- তিনটা pipe লাগবে (N command → N-1 pipe)
- শেষ command-এর output `report.txt`-এ যাবে

---

## কেন AST? Simple List কেন না?

এই প্রশ্নটা স্বাভাবিক। কেন এত জটিল structure? Simple list রাখলেই তো হতো।

কারণ হলো — real shell-এ commands nested হতে পারে:

```bash
(ls | grep .go) | wc -l
```

এখানে `ls | grep .go` একটা unit, তারপর সেটার output `wc`-এ যাচ্ছে। Simple list-এ এটা বোঝানো যায় না। AST-এ:

```
        PIPELINE
       /         \
  SUBSHELL       CMD
     │            │
  PIPELINE       "wc"
  /      \        │
CMD      CMD     "-l"
 │        │
"ls"    "grep"
          │
         ".go"
```

Subshell একটা node — ভেতরে আরেকটা pipeline। Executor এই node দেখলে বুঝবে আগে ভেতরের pipeline চালাতে হবে, তারপর বাইরেরটা।

Flat list দিয়ে এই hierarchy বোঝানো সম্ভব না। Tree দিয়ে সম্ভব।

---

## Execution: গাছ হাঁটা

AST তৈরি হলে Executor সেই গাছ **recursive**ভাবে হাঁটে।

```
execute(node):
    node যদি PIPELINE হয়:
        প্রতিটা child-এর মাঝে pipe বানাও
        প্রতিটা child execute করো

    node যদি COMMAND হয়:
        redirect গুলো setup করো
        fork() করো
        exec() করো

    node যদি SUBSHELL হয়:
        fork() করো
        child-এ ভেতরের pipeline execute করো
```

Recursion এখানে স্বাভাবিক — কারণ structure নিজেই recursive। গাছের যেকোনো node-এ যেকোনো ধরনের subtree থাকতে পারে।

---

## Milestone 3 এবং Milestone 9-এর পার্থক্য

Milestone 3-এ আমরা simple parser বানিয়েছিলাম — flat pipeline, basic tokens। কাজ চলে যাচ্ছিল।

Milestone 9-এ AST-based parser বানাব। পার্থক্যটা এখানে:

```
Milestone 3 Parser:              Milestone 9 Parser:
──────────────────               ────────────────────
Pipeline struct                  AST Node (recursive)
  └── []Command                    └── যেকোনো depth

শুধু flat pipeline               Nested commands সম্ভব
চলে                              Subshell সম্ভব
                                 Complex chaining সম্ভব

ls | grep | wc ✓                 (ls | grep) | wc ✓
(ls | grep) | wc ✗               cmd1 && cmd2 ✓
cmd1 && cmd2 ✗                   cmd1 || cmd2 ✓
```

Simple parser দিয়ে শুরু করেছিলাম কারণ concept বোঝাটা আগে দরকার ছিল। এখন concept জানা আছে, তাই proper AST বানানো সম্ভব।

---

## পুরো flow একসাথে

```
তুমি লিখলে: ls | grep ".go" > out.txt
                    │
                    ▼
              ┌──────────┐
              │  LEXER   │
              └────┬─────┘
                   │ tokens:
                   │ [WORD("ls"), PIPE,
                   │  WORD("grep"), WORD(".go"),
                   │  REDIRECT_OUT, WORD("out.txt"),
                   │  EOF]
                   ▼
              ┌──────────┐
              │  PARSER  │
              └────┬─────┘
                   │ AST:
                   │ PIPELINE
                   │   ├── CMD("ls", [])
                   │   └── CMD("grep", [".go"])
                   │         └── REDIRECT(>, "out.txt")
                   ▼
              ┌──────────┐
              │ EXECUTOR │
              └────┬─────┘
                   │
                   │ pipe() বানাও
                   │ fork() + exec("ls")
                   │ fork() + exec("grep", ".go")
                   │   stdout → out.txt
                   │ wait() সবার জন্য
                   ▼
              output.txt-এ ফলাফল
```

---

Compiler লেখা মানে শুধু code translate করা না। সে আসলে **ভাষা বোঝে** — grammar জানে, structure চেনে, মানে বের করে।

Shell-এর parser ঠিক এটাই করে। তুমি যখন `ls | grep ".go" > out.txt` লিখছ, shell সেই line-টা পড়ছে না — **বুঝছে**।

Milestone 9-এ এই বোঝার engine নিজে হাতে বানাব। 🚀