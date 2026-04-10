# অধ্যায় ৫: Process API

---

## 🔰 ভূমিকা

Operating System-এ একটি প্রোগ্রাম চালালে সেটা একটি **Process** হয়ে যায়। কিন্তু প্রশ্ন হলো — নতুন process কীভাবে তৈরি হয়? কীভাবে একটি process অন্য একটি process-কে নিয়ন্ত্রণ করে?

UNIX/Linux-এ এই কাজের জন্য তিনটি মূল **system call** আছে:

- `fork()` → নতুন process তৈরি
- `exec()` → নতুন প্রোগ্রাম চালানো
- `wait()` → child শেষ হওয়ার অপেক্ষা

এই তিনটি একসাথে বুঝলেই UNIX-এর process management পরিষ্কার হয়ে যাবে।

---

## ১. `fork()` — নতুন Process তৈরি

### মূল ধারণা

`fork()` হলো এমন একটি system call যেটা কল করলে **বর্তমান process-এর একটি হুবহু কপি তৈরি হয়।** মূল process-কে বলা হয় **parent**, আর নতুন তৈরি হওয়া process-কে বলা হয় **child**।

কপি মানে কী? Child process পাবে:
- parent-এর সব **code**
- সব **variable ও তাদের মান**
- সব **open file descriptor**
- **Program counter** (ঠিক কোথায় execution আছে)

কিন্তু child একটি **আলাদা process** — তার নিজস্ব **PID (Process ID)** আছে, এবং সে parent-এর থেকে স্বাধীনভাবে চলে।

### fork() এর return value

এটাই সবচেয়ে গুরুত্বপূর্ণ এবং একটু confusing অংশ:

| কে পাচ্ছে | return value |
|---|---|
| **Parent process** | Child-এর PID (একটি positive সংখ্যা) |
| **Child process** | `0` |
| **Error হলে** | `-1` |

অর্থাৎ `fork()` একবার কল হয়, কিন্তু **দুবার return করে** — একবার parent-এ, একবার child-এ। এটাই fork()-এর সবচেয়ে অদ্ভুত বৈশিষ্ট্য।

### উদাহরণ কোড (p1.c):

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    printf("hello (pid:%d)\n", (int) getpid());

    int rc = fork();

    if (rc < 0) {
        // fork ব্যর্থ হয়েছে
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // আমি child process
        printf("child (pid:%d)\n", (int) getpid());
    } else {
        // আমি parent process, rc = child এর PID
        printf("parent of %d (pid:%d)\n", rc, (int) getpid());
    }

    return 0;
}
```

### আউটপুট:
```
hello (pid:29146)
parent of 29147 (pid:29146)
child (pid:29147)
```

অথবা কখনো কখনো:
```
hello (pid:29146)
child (pid:29147)
parent of 29147 (pid:29146)
```

### কেন আউটপুটের ক্রম বদলায়?

`fork()` করার পর parent ও child **দুটোই ready** — কিন্তু কোনটা আগে চলবে সেটা **CPU scheduler** ঠিক করে। Scheduler যেকোনো একটিকে আগে চালাতে পারে। তাই আউটপুটের ক্রম **non-deterministic** — প্রতিবার একই নাও হতে পারে।

এই ধারণাটা concurrency বোঝার জন্য পরে অনেক কাজে আসবে।

---

## ২. `wait()` — Child শেষ হওয়ার অপেক্ষা

### মূল ধারণা

অনেক সময় parent process চায় যে child শেষ হওয়ার পরেই সে তার পরবর্তী কাজ করুক। এজন্য `wait()` ব্যবহার করা হয়।

`wait()` কল করলে **parent block হয়ে যায়** (মানে থেমে থাকে) যতক্ষণ না তার কোনো child process শেষ হয়। Child শেষ হলে `wait()` return করে এবং parent আবার চলতে শুরু করে।

### উদাহরণ কোড (p2.c):

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    printf("hello (pid:%d)\n", (int) getpid());

    int rc = fork();

    if (rc < 0) {
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // Child process
        printf("child (pid:%d)\n", (int) getpid());
    } else {
        // Parent process — child শেষ হওয়ার জন্য অপেক্ষা করছে
        int wc = wait(NULL);
        printf("parent of %d (wc:%d) (pid:%d)\n",
               rc, wc, (int) getpid());
    }

    return 0;
}
```

### আউটপুট (সবসময় একই ক্রমে):
```
hello (pid:29266)
child (pid:29267)
parent of 29267 (wc:29267) (pid:29266)
```

### বিস্তারিত ব্যাখ্যা:

- এখন parent সবসময় **child-এর পরে** প্রিন্ট করবে — কারণ `wait()` parent-কে আটকে রাখে।
- `wait()` return করে child-এর PID — সেটাই `wc` variable-এ আছে।
- এটা তখন দরকার হয় যখন parent-এর কাজ child-এর ফলাফলের উপর নির্ভর করে।

### `waitpid()` — আরও নিয়ন্ত্রণ

`wait()` যেকোনো child শেষ হলে return করে। কিন্তু **নির্দিষ্ট একটি child**-এর জন্য অপেক্ষা করতে চাইলে `waitpid()` ব্যবহার করতে হয়।

```c
int wc = waitpid(rc, &status, 0);
// rc = যে child-এর জন্য অপেক্ষা করব তার PID
// &status = child কীভাবে শেষ হলো সেই তথ্য
// 0 = কোনো বিশেষ option নেই
```

`waitpid()` আরও বেশি options দেয়, যেমন:
- child কি normally exit করেছে?
- কোনো signal-এ মারা গেছে?
- exit code কত ছিল?

---

## ৩. `exec()` — সম্পূর্ণ নতুন প্রোগ্রাম চালানো

### মূল ধারণা

`fork()` করলে child, parent-এর কপি হয় — তারা একই কোড চালায়। কিন্তু যদি child-কে **সম্পূর্ণ আলাদা একটি প্রোগ্রাম** চালাতে হয়? তখন `exec()` ব্যবহার করা হয়।

`exec()` কল করলে:
1. বর্তমান process-এর **পুরনো code, data সব মুছে যায়**
2. নতুন প্রোগ্রাম **memory-তে লোড হয়**
3. সেই নতুন প্রোগ্রাম **শুরু থেকে চলতে শুরু করে**

গুরুত্বপূর্ণ: `exec()` **নতুন process তৈরি করে না** — বর্তমান process-কেই বদলে দেয়।

### exec()-এর বিভিন্ন variant

`exec()` আসলে একটি পরিবার — বিভিন্ন কাজের জন্য বিভিন্ন version আছে:

| Function | বৈশিষ্ট্য |
|---|---|
| `execl()` | arguments আলাদা আলাদাভাবে দেওয়া হয় |
| `execlp()` | PATH থেকে প্রোগ্রাম খোঁজে |
| `execle()` | environment variable সহ |
| `execv()` | arguments array হিসেবে দেওয়া হয় |
| `execvp()` | array + PATH থেকে খোঁজে ← **সবচেয়ে বেশি ব্যবহৃত** |
| `execve()` | array + environment ← **সবচেয়ে low-level** |

### উদাহরণ কোড (p3.c):

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main() {
    printf("hello (pid:%d)\n", (int) getpid());

    int rc = fork();

    if (rc < 0) {
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // Child process
        printf("child (pid:%d)\n", (int) getpid());

        char *myargs[3];
        myargs[0] = strdup("wc");    // চালাতে চাই "wc" প্রোগ্রাম
        myargs[1] = strdup("p3.c");  // argument: কোন ফাইলের word count
        myargs[2] = NULL;            // array-এর শেষ চিহ্ন

        execvp(myargs[0], myargs);   // wc চালাও

        // এই লাইন কখনো চলবে না (exec সফল হলে)
        printf("this shouldn't print\n");

    } else {
        // Parent process
        int wc = wait(NULL);
        printf("parent of %d (wc:%d) (pid:%d)\n",
               rc, wc, (int) getpid());
    }

    return 0;
}
```

### আউটপুট:
```
hello (pid:29383)
child (pid:29384)
  29  107 1030 p3.c
parent of 29384 (wc:29384) (pid:29383)
```

### বিস্তারিত ব্যাখ্যা ধাপে ধাপে:

**ধাপ ১:** Program শুরু হয়, "hello" প্রিন্ট হয়।

**ধাপ ২:** `fork()` হয় → parent ও child তৈরি।

**ধাপ ৩:** Child-এ `execvp("wc", myargs)` কল হয়।

**ধাপ ৪:** Child-এর পুরনো code মুছে যায়, `wc` প্রোগ্রাম লোড হয়।

**ধাপ ৫:** `wc` চলে এবং `p3.c` ফাইলের word count প্রিন্ট করে (29 lines, 107 words, 1030 bytes)।

**ধাপ ৬:** Child শেষ হয়। Parent `wait()` থেকে বের হয়ে প্রিন্ট করে।

**কেন `"this shouldn't print"` কখনো প্রিন্ট হয় না?**
কারণ `execvp()` সফল হলে child-এর পুরনো code-ই আর থাকে না — নতুন প্রোগ্রাম (wc) সেই জায়গা নিয়ে নেয়। তাই পরের লাইনে যাওয়ার সুযোগ নেই।

---

## ৪. কেন fork() ও exec() আলাদা? — Shell-এর গল্প

### এটা কি অদ্ভুত ডিজাইন না?

প্রথমে মনে হতে পারে — নতুন প্রোগ্রাম চালানোর জন্য আলাদা `fork()` + `exec()` কেন? একটাই system call দিলেই তো হতো।

কিন্তু এই আলাদা ডিজাইনটাই UNIX-এর **সবচেয়ে চতুর সিদ্ধান্তগুলোর একটি।**

### Shell কীভাবে কাজ করে?

তুমি terminal-এ যখন কিছু টাইপ করো, shell এভাবে কাজ করে:

```
১. তুমি command টাইপ করলে
        ↓
২. Shell fork() করে → child তৈরি হয়
        ↓
৩. Child-এ exec() দিয়ে সেই command চালানো হয়
        ↓
৪. Parent (shell) wait() করে
        ↓
৫. Command শেষ হলে shell আবার prompt দেখায়
```

### fork() ও exec()-এর মাঝখানে কী হয়?

এখানেই আসল সৌন্দর্য। `fork()` করার পর কিন্তু `exec()` করার **আগে** — এই মাঝখানের সময়ে shell অনেক কিছু করতে পারে:

**উদাহরণ ১ — Output Redirect:**
```bash
wc p3.c > output.txt
```
এখানে `>` মানে `wc`-এর output screen-এ না দেখিয়ে `output.txt` ফাইলে লেখো।

Shell এটা করে এভাবে:
1. `fork()` করে child তৈরি করে
2. Child-এ `stdout` বন্ধ করে `output.txt` খোলে
3. তারপর `exec()` দিয়ে `wc` চালায়
4. `wc` জানেই না যে তার output ফাইলে যাচ্ছে!

**উদাহরণ ২ — Pipe:**
```bash
ls | grep ".c"
```
এখানে `ls`-এর output, `grep`-এর input হয়ে যায়। এটাও `fork()`+`exec()` এর মাঝখানে pipe সেটআপ করে করা হয়।

যদি fork ও exec একসাথে থাকত, এই মাঝখানের কাজ করার সুযোগ থাকত না।

---

## ৫. অন্যান্য গুরুত্বপূর্ণ System Call

### `kill()` — Process-কে Signal পাঠানো

```c
kill(pid, SIGINT);   // Ctrl+C এর মতো — process বন্ধ করো
kill(pid, SIGKILL);  // জোর করে বন্ধ করো
kill(pid, SIGSTOP);  // pause করো
kill(pid, SIGCONT);  // আবার চালু করো
```

Terminal-এ `Ctrl+C` চাপলে আসলে shell `SIGINT` signal পাঠায়।

### `signal()` — Signal Handle করা

Process চাইলে signal এলে কী করবে সেটা নিজে নির্ধারণ করতে পারে:

```c
signal(SIGINT, my_handler);  // SIGINT এলে my_handler function চালাও
```

এভাবে একটি প্রোগ্রাম Ctrl+C চাপলেও না থেমে নিজের কাজ করতে পারে।

---

## ৬. Zombie ও Orphan Process

### Zombie Process
Child শেষ হয়ে গেছে কিন্তু parent এখনো `wait()` কল করেনি — এই অবস্থায় child-এর একটু তথ্য memory-তে থেকে যায়। একে বলে **Zombie Process**।

এটা সমস্যা কারণ এই process সম্পূর্ণ মরেনি — সিস্টেমের কিছু resource দখল করে আছে।

### Orphan Process
Parent শেষ হয়ে গেছে কিন্তু child এখনো চলছে — এই child-কে বলে **Orphan Process**।

Linux-এ orphan process-গুলো স্বয়ংক্রিয়ভাবে **`init` process (PID 1)**-এর অধীনে চলে যায়, যে তাদের দেখাশোনা করে।

---

## ৭. সম্পূর্ণ চিত্র — একনজরে

```
Program শুরু (Parent)
        |
      fork()
      /    \
Parent      Child
  |           |
wait()      exec()
  |           |
  |      নতুন প্রোগ্রাম চলে
  |           |
  |      Child শেষ হয়
  |           |
wait() return ←
  |
Parent চলতে থাকে
```

---

## ৮. মূল শিক্ষা (Summary)

| বিষয় | মূল কথা |
|---|---|
| `fork()` | Parent-এর কপি করে child তৈরি করে। Parent পায় child-এর PID, child পায় 0। |
| `wait()` | Parent থামে যতক্ষণ child শেষ না হয়। Output-এর ক্রম নিশ্চিত হয়। |
| `exec()` | Child-এ নতুন প্রোগ্রাম লোড করে। পুরনো code মুছে যায়। |
| Shell | fork+exec ব্যবহার করে command চালায়, মাঝখানে redirect/pipe সেটআপ করে। |
| Zombie | Child মরেছে কিন্তু parent wait() করেনি। |
| Orphan | Parent মরেছে কিন্তু child চলছে, init তাকে দত্তক নেয়। |
