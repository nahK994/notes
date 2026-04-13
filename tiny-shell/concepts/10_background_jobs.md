## Background Jobs: Shell-এর হাতে একাধিক কাজ

আগের chapter-এ দেখেছো — `&` দিলে process background-এ যায়, Shell `wait()` করে না।

কিন্তু এখানে একটা সমস্যা আছে।

Shell যদি কখনো `wait()` না করে, তাহলে background process শেষ হলে zombie হয়ে পড়ে থাকবে। আর Shell যদি সব সময় `wait()` করে বসে থাকে, তাহলে background-এর কোনো মানেই নেই — terminal আবার আটকে যাবে।

তাহলে Shell আসলে কী করে?

**সে অপেক্ষা করে, কিন্তু block হয় না।**

পার্থক্যটা সূক্ষ্ম কিন্তু গুরুত্বপূর্ণ। সাধারণ `wait()` মানে — *"child শেষ না হওয়া পর্যন্ত আমি কিছু করবো না।"* কিন্তু background job-এর জন্য Shell বলে — *"child শেষ হয়েছে কিনা একটু দেখো, না হলে থাক, আমি অন্য কাজ করি।"*

এই কাজটা করে `waitpid()` — `wait()` এর একটু চালাক ভাই।

---

### waitpid(): অপেক্ষার চালাক উপায়

```go
// সাধারণ wait() — block করে
// child শেষ না হওয়া পর্যন্ত এখানেই আটকে থাকবে
syscall.Wait4(-1, &status, 0, nil)

// waitpid() WNOHANG দিয়ে — block করে না
// child শেষ হয়ে থাকলে তার info দাও
// না হলে সাথে সাথে ফিরে এসো
syscall.Wait4(-1, &status, syscall.WNOHANG, nil)
```

`WNOHANG` মানে — **"No Hang"** — আটকে থেকো না।

```
WNOHANG ছাড়া:              WNOHANG দিয়ে:

waitpid() call করলাম       waitpid() call করলাম
    │                           │
    │ child চলছে...             │ child চলছে?
    │ আটকে আছি                 │ না → সাথে সাথে ফিরলাম
    │ child চলছে...             │ হ্যাঁ → তার info নিলাম
    │ child শেষ হলো             │
    │ ফিরলাম                   অন্য কাজ করতে পারছি
```

Shell প্রতিটা নতুন command চালানোর আগে একবার `WNOHANG` দিয়ে check করে — কোনো background job শেষ হয়েছে কিনা। শেষ হলে জানায়, zombie সাফ করে, তারপর নতুন command চালায়।

---

### Job Table: Shell যেভাবে হিসাব রাখে

Shell প্রতিটা background job-এর জন্য একটা record রাখে। এই পুরো তালিকাটাকে বলে **Job Table**।

```
Job Table:
┌───────┬───────┬────────────┬──────────────────────┐
│  Job  │  PID  │   Status   │       Command        │
├───────┼───────┼────────────┼──────────────────────┤
│  [1]  │ 1234  │  Running   │  sleep 100           │
│  [2]  │ 1235  │  Running   │  wget http://...     │
│  [3]  │ 1236  │  Stopped   │  vim file.txt        │
└───────┴───────┴────────────┴──────────────────────┘
```

প্রতিটা entry-তে চারটা তথ্য:

**Job Number** — Shell-এর নিজস্ব নম্বর। `fg %2`, `kill %3` — এই `%` দিয়ে job number refer করা হয়।

**PID** — OS-এর কাছে process-এর আসল পরিচয়। `kill 1234` করলে এই নম্বর লাগে।

**Status** — এই মুহূর্তে process কী করছে। Running, Stopped, Done — তিনটা অবস্থা।

**Command** — কোন command থেকে এই job তৈরি হয়েছে। `jobs` চালালে তুমি যা দেখো।

---

### Job-এর তিনটা অবস্থা

```
          fork() + exec()
               │
               ▼
          ┌─────────┐
          │ Running │ ◄──── bg %N অথবা SIGCONT
          └────┬────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
  ┌─────────┐      ┌──────┐
  │ Stopped │      │ Done │
  └────┬────┘      └──────┘
  Ctrl+Z বা            │
  SIGTSTP         zombie সাফ,
                  job table
                  থেকে মুছলো
```

**Running** — চলছে। Background-এ কাজ করছে, terminal তোমার কাছে।

**Stopped** — pause হয়ে আছে। Ctrl+Z চাপলে বা SIGTSTP signal পেলে এই অবস্থায় আসে। CPU খাচ্ছে না, memory-তে আছে। `fg` বা `bg` দিলে আবার চালু হবে।

**Done** — কাজ শেষ। Shell `waitpid()` দিয়ে exit code নিয়ে নেবে, job table থেকে মুছে দেবে।

---

### Job Control-এর command গুলো

```bash
$ sleep 100 &          # background-এ পাঠাও
[1] 1234               # job number আর PID দেখালো

$ sleep 200 &
[2] 1235

$ jobs                 # সব background job দেখো
[1]-  Running    sleep 100 &
[2]+  Running    sleep 200 &

$ fg %1                # job 1 কে foreground-এ আনো
sleep 100              # terminal আটকে গেলো

[Ctrl+Z]               # আবার pause করো
^Z
[1]+ Stopped    sleep 100

$ bg %1                # আবার background-এ পাঠাও
[1]+ sleep 100 &

$ kill %2              # job 2 কে মারো
[2]- Terminated  sleep 200
```

`jobs` output-এ `+` আর `-` চিহ্ন দেখবে:

```
[1]-  Running    sleep 100 &
[2]+  Running    sleep 200 &
```

**`+`** মানে — most recent job। `fg` বা `bg` argument ছাড়া দিলে এই job-টাই নেবে।

**`-`** মানে — তার আগেরটা।

---

### Shell-এর ভেতরে Job Control কীভাবে কাজ করে

তোমার tiny-shell-এ Job Control implement করতে হলে তিনটা জিনিস লাগবে।

**প্রথমত, Job Table বানাও:**

```go
type JobStatus int

const (
    Running JobStatus = iota
    Stopped
    Done
)

type Job struct {
    ID      int        // job number: 1, 2, 3...
    PID     int        // process ID
    Status  JobStatus  // Running, Stopped, Done
    Command string     // "sleep 100"
}

type JobTable struct {
    jobs   []Job
    nextID int
}
```

**দ্বিতীয়ত, background process চালু করো:**

```go
func (jt *JobTable) AddJob(pid int, cmd string) Job {
    job := Job{
        ID:      jt.nextID,
        PID:     pid,
        Status:  Running,
        Command: cmd,
    }
    jt.jobs = append(jt.jobs, job)
    jt.nextID++
    return job
}

// & দিয়ে command চালালে:
// wait() করো না, job table-এ add করো
if isBackground {
    job := jobTable.AddJob(childPID, command)
    fmt.Printf("[%d] %d\n", job.ID, job.PID)
}
```

**তৃতীয়ত, নিয়মিত check করো কোনো job শেষ হয়েছে কিনা:**

```go
func (jt *JobTable) CheckDone() {
    for i, job := range jt.jobs {
        if job.Status != Running {
            continue
        }

        // WNOHANG — block করো না, শুধু check করো
        var status syscall.WaitStatus
        pid, _ := syscall.Wait4(job.PID, &status, syscall.WNOHANG, nil)

        if pid == job.PID {
            // এই job শেষ হয়েছে
            jt.jobs[i].Status = Done
            fmt.Printf("\n[%d]+ Done\t%s\n", job.ID, job.Command)
        }
    }

    // Done job গুলো table থেকে সরাও
    jt.cleanDone()
}
```

এই `CheckDone()` প্রতিটা নতুন prompt দেখানোর আগে call করো। তাহলে background job শেষ হলে তুমি জানতে পারবে।

---

### একটা সম্পূর্ণ Session দেখি

```bash
$ sleep 5 &
[1] 1234

$ sleep 3 &
[2] 1235

$ jobs
[1]-  Running    sleep 5 &
[2]+  Running    sleep 3 &

$ echo "অন্য কাজ করছি"
অন্য কাজ করছি

[2]+ Done    sleep 3      ← prompt আসার আগে Shell check করলো

$ jobs
[1]+  Running    sleep 5 &   ← শুধু [1] বাকি

$ fg %1
sleep 5                      ← foreground-এ এলো, terminal আটকালো

[Ctrl+Z]
^Z
[1]+ Stopped    sleep 5

$ bg %1
[1]+ sleep 5 &

$ kill %1
[1]+ Terminated    sleep 5

$ jobs
                             ← খালি, কোনো job নেই
```

---

### মনে রাখার মতো একটা কথা

Job Control আসলে Shell-এর একটা bookkeeping সমস্যা।

OS শুধু process জানে — PID দিয়ে। Job number, `+`, `-` চিহ্ন, `jobs` command — এগুলো Shell নিজে তৈরি করেছে, OS-এর কোনো ধারণা নেই।

তুমি `kill %2` লিখলে Shell তার job table দেখে, job 2-এর PID বের করে, তারপর সেই PID-এ signal পাঠায়। OS শুধু signal-টা দেখে — `%2` মানে কী সে জানে না।

Shell হলো সেই মধ্যস্থতাকারী যে OS-এর plain process system-এর উপরে বসে একটা সুন্দর job management layer তৈরি করে।