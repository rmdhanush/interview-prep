# Go Concepts — Learning Notes

> Personal reference built topic-by-topic during interview prep.
> Each section uses real-world analogies for retention + code examples for practice.

---

## Table of Contents

1. [Goroutines & Channels](#1-goroutines--channels)
   - [What is a Goroutine?](#what-is-a-goroutine)
   - [What is a Channel?](#what-is-a-channel)
   - [The 3 Blocking Rules](#the-3-blocking-rules)
   - [Buffered vs Unbuffered Channels](#buffered-vs-unbuffered-channels)
   - [5 Common Patterns](#5-common-patterns)
2. [GMP Model & Goroutine Lifecycle](#2-gmp-model--goroutine-lifecycle)
   - [The Company Analogy](#the-company-analogy)
   - [5 Scheduling Rules](#5-scheduling-rules)
   - [Goroutine States](#goroutine-states)
   - [Syscall Handoff](#syscall-handoff)
   - [Stack Growth](#stack-growth)
   - [Goroutine Death & Leaks](#goroutine-death--leaks)
3. [Concurrency vs Parallelism](#3-concurrency-vs-parallelism)

---

## 1. Goroutines & Channels

### What is a Goroutine?

**Analogy:** Hiring a person and walking away. You say `go doWork()` — that person starts working independently. You don't wait for them.

```go
go cook()     // hire a chef, they start working
go serve()    // hire a waiter, they start working
// main continues immediately — doesn't wait
```

- Goroutines are **extremely cheap** (~2KB stack, ~0.3 microseconds to create)
- Creating 100,000 goroutines is fine. Creating 100,000 OS threads would kill your machine.

### What is a Channel?

**Analogy: Restaurant kitchen window.** The chef and waiter never touch each other's stuff. They communicate through the kitchen window (a channel).

```go
window := make(chan Order)   // the kitchen window
```

- Chef **sends** food through the window: `window <- food`
- Waiter **receives** food from the window: `food := <-window`

> **A channel is a hole in the wall. One side puts stuff in, the other side takes stuff out.**

### The 3 Blocking Rules

Think of the kitchen window as **tiny** — it can only hold one plate at a time (unbuffered channel).

| Situation | What happens |
|---|---|
| Chef puts food on window, **no waiter waiting** | Chef **stands frozen** until waiter comes |
| Waiter reaches into window, **no food there** | Waiter **stands frozen** until chef sends food |
| Both show up at the same time | **Instant handoff**, both continue |

```go
ch := make(chan string)

go func() {
    ch <- "pizza"     // stuck here until someone takes it
    fmt.Println("chef: delivered!")
}()

food := <-ch          // takes "pizza", NOW both sides unblock
fmt.Println("waiter: got", food)
```

> **Sends block until someone receives. Receives block until someone sends. This is how goroutines synchronize WITHOUT locks.**

### Buffered vs Unbuffered Channels

```go
// Unbuffered — chef waits for waiter every time
ch := make(chan Order)

// Buffered — counter fits 3 plates, chef can keep cooking
ch := make(chan Order, 3)
```

> **Buffered channel = "I can keep working until the buffer is full."**

### 5 Common Patterns

#### Pattern 1: Fire and Forget (background task)
```go
go sendEmail(user)   // don't care when it finishes
```

#### Pattern 2: Wait for Result (future/promise)
```go
result := make(chan int)
go func() {
    result <- heavyCalculation()
}()
// ... do other stuff ...
answer := <-result   // now I need the answer, I'll wait
```

#### Pattern 3: Fan-out (one sender, many workers)
```go
jobs := make(chan Job)

// hire 5 workers
for i := 0; i < 5; i++ {
    go worker(jobs)
}

// send work — any free worker picks it up
for _, j := range allJobs {
    jobs <- j
}
```

#### Pattern 4: Fan-in (many senders, one receiver)
```go
results := make(chan Result)

go fetchFromDB(results)
go fetchFromAPI(results)
go fetchFromCache(results)

// collect all results from one channel
for i := 0; i < 3; i++ {
    r := <-results
    fmt.Println(r)
}
```

#### Pattern 5: Signal done (quit/cancel)
```go
done := make(chan struct{})

go func() {
    // do long work...
    close(done)   // broadcast "I'm done" to EVERYONE listening
}()

<-done   // blocks until closed
```

### Key Rules

- **One channel = one direction.** For bidirectional communication (ping-pong), use **two channels**.
- **The sender closes the channel, never the receiver.** The person filling the pipe knows when they're done.
- **`range` over a channel** exits when the channel is closed. Without `close()` → deadlock.
- **`main()` doesn't wait for goroutines.** You must use channels, `sync.WaitGroup`, or `context` to wait.

### Decision Flowchart

```
Do I need something to run independently?
  YES → goroutine

Does it need to send data back?
  YES → channel

Can the sender keep going without waiting?
  YES → buffered channel
  NO  → unbuffered channel (default)

Do multiple goroutines need to stop together?
  YES → close(channel) or context.WithCancel()

Am I waiting for multiple channels?
  YES → select { }
```

---

## 2. GMP Model & Goroutine Lifecycle

### The Company Analogy

Think of a **small company**:

```
G (Goroutine)  = Task written on a sticky note
P (Processor)  = A desk with a to-do tray
M (Machine)    = An employee sitting at the desk
```

```
   Desk P0              Desk P1              Desk P2              Desk P3
  ┌────────┐          ┌────────┐           ┌────────┐          ┌────────┐
  │ Employee│          │ Employee│          │ Employee│          │  Empty │
  │  M0     │          │  M1     │          │  M2     │          │(no one)│
  └────┬───┘          └────┬───┘           └────┬───┘          └────────┘
       │                    │                    │
  To-do tray:          To-do tray:          To-do tray:         To-do tray:
  [G1, G2, G3]        [G4, G5]             [G6]                 []

                    Lobby (global queue): [G7, G8, G9]
```

### 5 Scheduling Rules

**Rule 1: One desk, one employee, one task at a time.**
An employee (M) sits at a desk (P) and works on one sticky note (G) at a time.

**Rule 2: `go func()` = write a new sticky note, drop it in a tray.**
```go
go doWork()   // new sticky note → into current P's tray
```

**Rule 3: When a task blocks, the employee picks the next one.**
```go
ch <- data    // G1 is waiting → set aside, pick up G2 from tray
```

**Rule 4: Empty tray? Steal from a neighbor.**
P2's tray is empty. P0 has [G2, G3]. P2 steals half → takes G3. This is **work stealing**.

**Rule 5: Employee goes to sleep if there's truly nothing to do.**
If trays are empty AND global queue is empty → M detaches from P and parks (sleeps).

### Goroutine States

A sticky note's life:

```
    go func()
        │
        ▼
   ┌─────────┐     picked by P    ┌─────────┐
   │ Runnable │ ──────────────────►│ Running  │
   │(in tray) │                    │(employee │
   └─────────┘◄───────────────────┤ doing it)│
                   preempted /     └────┬─────┘
                   yield                │
                                        │ ch <-, I/O,
                                        │ sleep, syscall
                                        ▼
                                   ┌─────────┐
                                   │ Waiting  │
                                   │(set aside)│
                                   └────┬─────┘
                                        │ unblocked
                                        │ (data arrived,
                                        │  I/O done)
                                        ▼
                                   Back to Runnable
                                   (back in tray)

         func returns → Dead (sticky note thrown away)
```

**When does a goroutine get swapped out (preempted)?**
- Channel send/receive (blocks)
- `time.Sleep`
- System call (file I/O, network)
- `runtime.Gosched()` (voluntary yield)
- Function call (preemption checkpoint)
- Since Go 1.14: **async preemption** — long-running loops get interrupted even without function calls

### Syscall Handoff

This is the tricky part interviewers love:

```
Before syscall:
  Desk P0: Employee M0 working on G1

During syscall (e.g., file read):
  M0 takes G1 and LEAVES the desk to go wait at the OS
  Desk P0 is now EMPTY → runtime puts a NEW employee M3 at P0
  M3 continues working on P0's tray [G2, G3]

After syscall:
  M0 returns with G1 → puts G1 back in a tray
  M0 either finds an empty desk or goes to sleep
```

> **The desk (P) never sits idle.** If an employee is stuck waiting at the OS, a replacement comes in immediately.

### Stack Growth

```
OS Thread:     Fixed 1-8MB stack → stack overflow if exceeded
Goroutine:     Starts at 2KB → grows dynamically up to 1GB
```

**How it works:**

1. Goroutine starts with **2KB** stack
2. Function call needs more space → runtime detects stack limit
3. Runtime allocates a **new stack, 2x the size**
4. **Copies** everything from old stack to new, updates all pointers, frees old stack
5. Goroutine continues on bigger stack — it never noticed

- Stack check happens at every function call (compiler inserts a check)
- Growth is **doubling**: 2KB → 4KB → 8KB → 16KB...
- Stacks can also **shrink** during garbage collection if underused

### Goroutine Death & Leaks

A goroutine dies when:
- Its function **returns**
- `runtime.Goexit()` is called (runs deferred functions first)
- The **program exits** (main returns → all goroutines killed, no cleanup)

```go
// Common mistake: goroutine leak
go func() {
    msg := <-ch   // if nobody ever sends on ch, this goroutine lives FOREVER
}()
```

> **There is no way to kill a goroutine from outside.** You must design it to exit itself using `context.WithCancel`, a `done` channel, or closing the input channel.

### Quick Reference Table

| Analogy | GMP | Key Behavior |
|---|---|---|
| Sticky note | G (goroutine) | Cheap, millions ok |
| Desk | P (processor) | Limited to GOMAXPROCS |
| Employee | M (OS thread) | Expensive, created sparingly |
| To-do tray | Local run queue | Fast, no locking needed |
| Lobby board | Global run queue | Shared, needs locking |
| Stealing notes from neighbor | Work stealing | Keeps all desks busy |
| Employee leaves for errand | Syscall handoff | Desk stays productive |

---

## 3. Concurrency vs Parallelism

### One-liner

- **Concurrency** = **dealing** with multiple things at once (structure)
- **Parallelism** = **doing** multiple things at once (execution)

### Analogy

- **Concurrency:** One cook switching between chopping vegetables and stirring soup
- **Parallelism:** Two cooks — one chops, one stirs — both working at the same instant

### In Go

```go
runtime.GOMAXPROCS(1)   // concurrency only — goroutines take turns on 1 core
runtime.GOMAXPROCS(4)   // parallelism possible — goroutines run simultaneously on 4 cores
```

| Concept | Go mechanism |
|---|---|
| Concurrency | `go` keyword launches goroutines |
| Parallelism | `GOMAXPROCS(n)` controls how many cores run goroutines |
| Communication | Channels (`chan`) to safely pass data between goroutines |

> Go makes concurrency easy with goroutines. Whether those goroutines run in parallel depends on how many cores you give it. By default, `GOMAXPROCS` = number of CPU cores, so goroutines often run in parallel automatically.

---

## Practice Exercises

### Exercise 1: Ping-Pong (bidirectional channels)
Two goroutines pass a string back and forth 5 times using **two channels** (one per direction).

### Exercise 2: Pipeline (chained channels + close)
Goroutine A generates numbers → channel → Goroutine B doubles them → channel → main prints. Must `close()` channels when done.

### Exercise 3: Timeout (select + time.After)
Goroutine does slow work (3s sleep). Use `select` with `time.After(2s)` to timeout.

---

*Last updated: 2026-03-29*