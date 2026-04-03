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
4. [Sync Package](#4-sync-package)
   - [Mutex](#mutex)
   - [RWMutex](#rwmutex)
   - [WaitGroup](#waitgroup)
   - [Once](#once)
   - [Pool](#pool)
   - [Decision Cheat Sheet](#sync-decision-cheat-sheet)
5. [Context Package](#5-context-package)
   - [WithCancel](#withcancel)
   - [WithTimeout](#withtimeout)
   - [WithValue](#withvalue)
   - [Propagation](#propagation--how-context-flows)
6. [Generics](#6-generics)
   - [Type Constraints](#type-constraints)
   - [Custom Constraints](#custom-constraints)
   - [Generic Types](#generic-types)
   - [Type Inference](#type-inference)
   - [When to Use vs Not](#when-to-use-vs-not)
7. [Defer, Panic, Recover](#7-defer-panic-recover)
   - [Defer Patterns](#defer-patterns)
   - [Panic — When and When Not](#panic--when-and-when-not)
   - [Recover — Catching Panics](#recover--catching-panics)
   - [Bug vs Error](#bug-vs-error)
8. [Memory Model](#8-memory-model)

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

## 4. Sync Package

> Channels are for **passing data** between goroutines. Sync is for **protecting shared data**.
> Channels = communication. Sync = protection.

### Mutex

**Analogy: Bathroom with a lock.** Only one person can use it at a time. Lock the door, do your thing, unlock.

```go
var (
    balance int
    mu      sync.Mutex
)

func deposit(amount int) {
    mu.Lock()
    defer mu.Unlock()
    balance += amount
}
```

**Without Mutex** — race condition:
```
A reads balance: 100
B reads balance: 100    ← stale!
A writes: 150
B writes: 130           ← overwrites A's work!
```

**With Mutex** — safe:
```
A locks → reads 100 → writes 150 → unlocks
B locks → reads 150 → writes 180 → unlocks  ✓
```

> Always use `defer mu.Unlock()` to avoid forgetting to unlock.

### RWMutex

**Analogy: Library rules.** Many people can read a book at the same time. Only one person can write (edit), and nobody reads while writing.

```go
var (
    cache = make(map[string]string)
    rw    sync.RWMutex
)

func get(key string) string {
    rw.RLock()           // read lock — many readers allowed
    defer rw.RUnlock()
    return cache[key]
}

func set(key, value string) {
    rw.Lock()            // write lock — exclusive
    defer rw.Unlock()
    cache[key] = value
}
```

| Scenario | Use |
|---|---|
| Mostly writes | `sync.Mutex` (simpler, less overhead) |
| Many reads, rare writes | `sync.RWMutex` (readers don't block each other) |

### WaitGroup

**Analogy: Parent waiting for all kids to finish shopping.** "I'll wait at the exit. Each of you check in when you're done."

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)           // "one more kid going in"
    go func(id int) {
        defer wg.Done() // "I'm back!"
        fmt.Printf("Worker %d done\n", id)
    }(i)
}

wg.Wait()               // "I'll wait until all kids are back"
```

| Method | Meaning |
|---|---|
| `wg.Add(n)` | n more tasks starting |
| `wg.Done()` | one task finished (same as `Add(-1)`) |
| `wg.Wait()` | block until counter reaches 0 |

> **Always call `Add()` before launching the goroutine, not inside it.** Otherwise main might reach `Wait()` before `Add()` runs.

### Once

**Analogy: "Only open the restaurant once, no matter how many customers arrive at the same time."**

```go
var (
    db   *Database
    once sync.Once
)

func getDB() *Database {
    once.Do(func() {
        db = connectToDB()   // runs only ONCE
    })
    return db
}
```

Even if 100 goroutines call `getDB()` simultaneously — first one runs init, other 99 **block and wait** (they don't skip), all 100 get the same instance.

**Use cases:** Singleton initialization, one-time config loading, lazy setup.

### Pool

**Analogy: Office supply closet.** Instead of buying a new stapler every time, keep a closet. Take one, use it, put it back. If empty, buy a new one.

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)   // create new if closet is empty
    },
}

func processRequest() {
    buf := bufPool.Get().(*bytes.Buffer)  // grab from pool
    buf.Reset()                            // clean before use
    defer bufPool.Put(buf)                 // put back when done
    buf.WriteString("hello")
}
```

- `Get()` — take from pool (or create new if empty)
- `Put()` — return to pool for reuse
- Pool can be **garbage collected** anytime — don't rely on items staying
- **Use for:** High-frequency allocations (buffers, slices). Reduces GC pressure.
- **Don't use for:** Goroutine pools (use worker pool pattern with channels).

### Sync Decision Cheat Sheet

```
Need to protect a shared variable?
  Mostly writes           → sync.Mutex
  Many reads, few writes  → sync.RWMutex

Need to wait for goroutines to finish?
  → sync.WaitGroup

Need to run something exactly once?
  → sync.Once

Need to reuse expensive objects?
  → sync.Pool

Need goroutines to talk to each other?
  → Don't use sync — use channels
```

---

## 5. Context Package

> **Analogy: A walkie-talkie your boss gives you.** Boss can cancel the mission, set a timer, or attach a sticky note with info. Every sub-goroutine gets the same walkie-talkie. When boss cancels → everyone stops.

```
Boss (main) gives ctx to:
  ├── API Handler
  │     ├── DB Query         ← all stop when
  │     └── External API Call     ctx is cancelled
  Boss cancels ctx → ALL of them stop
```

### WithCancel

**"Boss presses a button → everyone stops."**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()   // ALWAYS defer this

go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():       // walkie-talkie buzzes
            return               // stop working
        default:
            doWork()
        }
    }
}(ctx)

cancel()   // press the button → goroutine stops
```

- `cancel()` closes `ctx.Done()` channel
- `ctx.Err()` returns `context.Canceled`

### WithTimeout

**"Bomb timer — you have 3 seconds, then auto-cancel."**

```go
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()

select {
case result := <-doSlowWork(ctx):
    fmt.Println(result)
case <-ctx.Done():
    fmt.Println("timed out:", ctx.Err())   // deadline exceeded
}
```

- `WithTimeout` = relative ("5s from now")
- `WithDeadline` = absolute ("at 3:00 PM exactly")

### WithValue

**"Sticky note on the walkie-talkie — pass-along metadata."**

```go
type contextKey string
const reqIDKey contextKey = "requestID"

ctx := context.WithValue(parentCtx, reqIDKey, "abc-123")
reqID := ctx.Value(reqIDKey).(string)   // downstream reads it
```

- Use **custom key type** (not bare strings) to avoid collisions
- Only for: request ID, auth token, trace ID
- Never for: DB connections, config, loggers

### Propagation — How Context Flows

```
Parent cancels → ALL children cancel
Child cancels  → only that child dies, parent is fine

         parent
        /      \
    child1    child2
      |
   grandchild

cancel parent → everyone dies
cancel child1 → child1 + grandchild die, parent + child2 fine
```

**Rules:**
1. Context is the **first parameter**: `func Foo(ctx context.Context, ...)`
2. Never store context in a struct — pass it through calls
3. Always `defer cancel()`
4. Every I/O function should accept ctx: `db.QueryContext(ctx, ...)`, `http.NewRequestWithContext(ctx, ...)`

**Without context in microservices:** user cancels request but DB query keeps running, goroutines pile up, connections exhaust → cascading failure. With context → everything stops cleanly.

---

## 6. Generics

> **Analogy: A vending machine blueprint.** Without generics = separate machine for Coke, Pepsi, Water. With generics = one machine, tell it what to dispense.

**The problem:**
```go
func MinInt(a, b int) int { ... }
func MinFloat(a, b float64) float64 { ... }
func MinString(a, b string) string { ... }
// Same logic, copy-pasted 3 times. Only the type changes.
```

**With generics — one function:**
```go
func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }
    return b
}
Min(3, 5)           // int
Min(3.14, 2.71)     // float64
Min("apple", "banana") // string
```

### Type Constraints

**Constraint = "what shape fits the slot."** Not every type supports `<`, so you tell Go what's allowed.

```go
[T any]                    // anything — no operations guaranteed
[T comparable]             // supports == and != (can be map key)
[T constraints.Ordered]    // supports < > <= >= (numbers, strings)
```

### Custom Constraints

```go
type Number interface {
    int | int32 | int64 | float32 | float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums { total += n }
    return total
}
```

**The `~` tilde — include derived types:**
```go
type Celsius float64

// ~float64 means "float64 OR any type defined as type X float64"
type Temperature interface { ~float64 }

// Now Celsius works too
func Avg[T Temperature](temps []T) T { ... }
```

### Generic Types

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() T {
    last := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return last
}

intStack := Stack[int]{}
strStack := Stack[string]{}
```

### Type Inference

Go can figure out the type from arguments — no need to specify explicitly:

```go
Min[int](3, 5)    // explicit
Min(3, 5)         // inferred — Go sees int, figures it out

// When inference FAILS (must be explicit):
s := Stack[int]{}      // no args to infer from
Min(3, 3.14)           // ambiguous — int or float64?
Min[float64](3, 3.14)  // explicit fixes it
```

### When to Use vs Not

```
✅ USE: Same logic, different types (Min, Max, Contains, Filter)
✅ USE: Generic data structures (Stack, Queue, Tree)
✅ USE: Utility/library code across many types

❌ DON'T: Only one type will ever be used
❌ DON'T: Different types need different behavior → use interfaces
❌ DON'T: Simple cases — []int is fine, don't overcomplicate
```

> **Rule of thumb:** About to copy-paste a function and only change the type? → Generics. Different types need different behavior? → Interfaces.

---

## 7. Defer, Panic, Recover

> **defer** = "clean up when I leave" (hotel checkout — always return the key)
> **panic** = "fire alarm" (building is on fire, not coffee machine broken)
> **recover** = "catch the alarm before evacuation" (only at boundaries)

### Defer Patterns

**Close what you open:**
```go
f, err := os.Open(path)
if err != nil { return err }
defer f.Close()   // guaranteed, even if panic or early return
```

**Unlock what you lock:**
```go
mu.Lock()
defer mu.Unlock()   // one miss without defer = deadlock
```

**DB transaction safety:**
```go
tx, _ := db.Begin()
defer tx.Rollback()   // no-op if already committed
// do work...
return tx.Commit()
```

**Defer runs in LIFO (stack) order:**
```go
defer fmt.Println("first")    // runs 3rd
defer fmt.Println("second")   // runs 2nd
defer fmt.Println("third")    // runs 1st
```

**Gotcha — arguments evaluated immediately:**
```go
start := time.Now()
defer fmt.Println(time.Since(start))   // ← captures 0s at defer line!

// Fix — use closure:
defer func() {
    fmt.Println(time.Since(start))     // ← evaluated when defer runs ✓
}()
```

**Timing helper pattern:**
```go
func Timer(name string) func() {
    start := time.Now()
    return func() { fmt.Printf("%s took %v\n", name, time.Since(start)) }
}

func processOrder() {
    defer Timer("processOrder")()   // prints: processOrder took 1.23s
}
```

### Panic — When and When Not

```
✅ Panic (bug — code is wrong):
  - Impossible switch default on internally-controlled values
  - Nil pointer from your own initialization
  - regexp.MustCompile, template.Must (hardcoded patterns)

❌ Don't panic (error — world isn't cooperating):
  - File not found, DB down, network timeout
  - Bad user input
  - External API failure
```

**What happens during panic:**
```
panic() → current function stops
       → deferred functions run (LIFO)
       → goes up to caller → their defers run
       → ... up the call stack
       → reaches main → program crashes with stack trace
```

### Recover — Catching Panics

**Only works inside a deferred function.** Three real-world uses:

**HTTP middleware — one bad request shouldn't crash the server:**
```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

**Convert panic to error — wrapping risky third-party code:**
```go
func safeProcess(data []byte) (result string, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("failed: %v", r)   // named return gets set
        }
    }()
    result = riskyLib(data)
    return result, nil
}
```

**Worker pool — one bad job shouldn't kill the worker:**
```go
for job := range jobs {
    func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("job %s panicked: %v", job.ID, r)
            }
        }()
        process(job)   // panics are caught, worker continues
    }()
}
```

### Bug vs Error

> **"Can this happen in production with perfect code?"**
> **Yes** → error → `return error`
> **No** → bug → `panic` (or fix the code)

| | Bug (panic) | Error (return err) |
|---|---|---|
| Whose fault? | Developer | External world |
| Can user cause it? | No | Yes |
| Fix | Fix the code | Handle gracefully |

**Same switch, different source:**
```
status from YOUR code  → default is unreachable → panic (bug)
status from USER input → default is expected    → return error
```

### Decision Cheat Sheet

```
Closing/unlocking/rolling back?     → defer
Impossible state in your own code?  → panic
Network/DB/file/user input error?   → return error
HTTP handler might crash?           → recover middleware
Third-party code might panic?       → defer/recover, convert to error
```

---

## 8. Memory Model

> **Core question:** "When goroutine A writes a value, can goroutine B see it?"
> **Answer:** Only if there's a **happens-before** relationship between them.

**Why it's not obvious:** Each CPU core has its own cache. Writes may stay in one core's cache and never reach the other. Compiler/CPU can also reorder instructions.

### Happens-Before Guarantees

| Mechanism | Guarantee |
|---|---|
| Channel send → receive | Receiver sees everything sender wrote before send |
| `mu.Unlock()` → `mu.Lock()` | Next locker sees everything previous unlocker wrote |
| `wg.Done()` → `wg.Wait()` returns | Waiter sees everything done before Done() |
| `once.Do(f)` completes → next `once.Do` returns | All callers see f's writes |
| `close(ch)` → receive of zero value | Receiver sees everything before close |
| `go` statement → goroutine starts | New goroutine sees everything before `go` |

### Race Condition = No Happens-Before

| Code | Safe? | Why |
|---|---|---|
| `go func(){ x=42 }()` then `<-ch; print(x)` | Yes | Channel is sync point |
| `go func(){ mu.Lock(); x=42; mu.Unlock() }()` | Yes | Mutex is sync point |
| `go func(){ x=42 }()` then `print(x)` | **NO** | No sync → data race |

### How to Fix Races

| Fix | When to use |
|---|---|
| Channel | Goroutines need to communicate |
| Mutex | Protecting shared state |
| `atomic` | Simple counters only |
| No sharing | Each goroutine owns its data |

**Detect:** `go test -race ./...` or `go run -race main.go`

> **Rule:** Two goroutines + same variable + at least one writes + no sync point = **data race = undefined behavior**.

---

## Practice Exercises

### Exercise 1: Ping-Pong (bidirectional channels)
Two goroutines pass a string back and forth 5 times using **two channels** (one per direction).

### Exercise 2: Pipeline (chained channels + close)
Goroutine A generates numbers → channel → Goroutine B doubles them → channel → main prints. Must `close()` channels when done.

### Exercise 3: Timeout (select + time.After)
Goroutine does slow work (3s sleep). Use `select` with `time.After(2s)` to timeout.

### Exercise 4: Worker Pool with WaitGroup
Launch 3 workers. Send 10 jobs through a channel. Each worker prints "Worker X processed job Y". Main waits for all workers using WaitGroup.

### Exercise 5: Context Cancellation
Make 3 goroutines do "slow work" concurrently. Return the first result, cancel the other two using `context.WithCancel`.

### Exercise 6: Generic Filter
Write a `Filter[T any]` function that takes a slice and a predicate `func(T) bool`, returns only matching elements.

---

*Last updated: 2026-04-03*