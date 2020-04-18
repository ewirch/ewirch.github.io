---
title: "Refactor For Maintainability"
categories: articles
date: 2020-04-10
modified: 2020-04-10
tags: [go,maintainability,concurrency]
image:
  feature:
  teaser:
  path:
  thumb: 
---

I stumbled across a [blog post series](https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/) and a [repository](https://github.com/eliben/raft) implementing the [Raft](https://raft.github.io/) consensus algorithm in [Go](https://golang.org/). [Eli](https://github.com/eliben), the author, did a great job there, but in my opinion the code suffers from some maintainability issues. I forked the repo to refactor the code for practice. This post documents the changes. In this post I will reference single commits from my [own fork](https://github.com/ewirch/raft), branch [`refactor-for-maintainability`](https://github.com/ewirch/raft/commits/refactor-for-maintainability).

# TL;DR

Handle errors where they appear. This makes tracing root cause of problems a lot easier.

Use `defer` to release resources.

Structure code in a way, so it documents intent.

Create data structures in a way, so it is hard (or impossible) to use them wrong.

# Code Overview
Eli split the repository in to three main parts, which go hand in hand with his blog posts. I picked part3 as it contains the complete implementation. Eli organized the code in three files:

- `raft.go` - the Raft implementation
- `server.go` - network related code
- `storage.go` - persistence layer abstraction

The methods in the project are grouped around two main types: `Server` and `ConsensusModule`. Both structs contain some effectively final fields (like `serverId` and `peerIds`) and mutable fields (like `Server.peerClients` and `ConsensusModule.currentTerm`). The code is highly concurrent. Two mutexes guard the shared mutable state: `Server.mu` and `ConsensusModule.mu`. Whenever the code has to access or mutate state, the mutex is acquired - and released when done.

# Bugs

I found some minor bugs in the code.

**[Commit "Fix unused parameter"](https://github.com/ewirch/raft/commits/refactor-for-maintainability)** `ConsensusModule.restoreFromStorage()` receives the parameter `storage Storage`, but is not using it. Instead, it directly accesses `ConsensusModule.storage`.
 
 ```go
func (cm *ConsensusModule) restoreFromStorage(storage Storage) {
    if termData, found := cm.storage.Get("currentTerm"); found {
        d := gob.NewDecoder(bytes.NewBuffer(termData))
        if err := d.Decode(&cm.currentTerm); err != nil {
            log.Fatal(err)
        }
    } else {
        log.Fatal("currentTerm not found in storage")
    }
    if votedData, found := cm.storage.Get("votedFor"); found {
    ...
```
The only invocation of `restoreFromStorage()`, passes `ConsensusModule.storage` anyway: 
```go
if cm.storage.HasData() {
    cm.restoreFromStorage(cm.storage)
}
```  
I assume this is a refactoring artifact, and remove the unused parameter.

**[Commit "Fix unhandled error"](https://github.com/ewirch/raft/commits/refactor-for-maintainability)** There are cases of not checking returned errors in production code and in test code. Most of them are kinda ok to ignore. For example, closing connections, when shutting down:
```go
func (s *Server) DisconnectAll() {
    s.mu.Lock()
    defer s.mu.Unlock()
    for id := range s.peerClients {
        if s.peerClients[id] != nil {
            s.peerClients[id].Close()
            s.peerClients[id] = nil
        }
    }
}
```
Return code of `s.peerClients[id].Close()` is ignored. Why is it ok to ignore? There is no meaningful way to handle this error at this stage. However, a pedantic program might log this error anyway.

However, ignoring *this* error return value is fatal:
```go
s.rpcServer = rpc.NewServer()
s.rpcProxy = &RPCProxy{cm: s.cm}
s.rpcServer.RegisterName("ConsensusModule", s.rpcProxy)
```
The return value of `RegisterName()` is not checked. If `RegisterName()` fails, the program will start up with a non-functioning rpc server. It will be not able to handle `ConsensusModule` RPC calls. You always want to handle those errors.
```go
s.rpcServer = rpc.NewServer()
s.rpcProxy = &RPCProxy{cm: s.cm}
err := s.rpcServer.RegisterName("ConsensusModule", s.rpcProxy)
if err != nil {
    log.Fatal(err)
}
```

The same applies to `Server.ConnectToPeer()`, which forwards the error value of `rpc.Dial()`:
```go
for i := 0; i < n; i++ {
    for j := 0; j < n; j++ {
        if i != j {
            ns[i].ConnectToPeer(j, ns[j].GetListenAddr())
        }
    }
    connected[i] = true
}
``` 
 
 The bug is in test code only. Many would argue it is not important to handle errors there. It's true, the test will fail in some way anyway, but not handling the error value can make it hard to find the root cause of the problem.

```go
for i := 0; i < n; i++ {
    for j := 0; j < n; j++ {
        if i != j {
            err := ns[i].ConnectToPeer(j, ns[j].GetListenAddr())
            if err != nil {
                t.Fatal(err)
            }
        }
    }
    connected[i] = true
}
```

# Use `defer` Keyword Consequently

Eli's code has to acquire and release a mutex in many places. Often he uses `defer` to safely release the mutex again, which is good practice, but he doesn't do so in all cases. Sometimes, when the mutex is not necessary for the complete function body, he releases it in the middle of the function body.

Simple case:
```go
go func() {
    // The CM is dormant until ready is signaled; then, it starts a countdown
    // for leader election.
    <-ready
    cm.mu.Lock()
    cm.electionResetEvent = time.Now()
    cm.mu.Unlock()
    cm.runElectionTimer()
}()
```

Complex case:
```go
func (cm *ConsensusModule) runElectionTimer() {
    ...    
    for {
        <-ticker.C

        cm.mu.Lock()
        if cm.state != Candidate && cm.state != Follower {
            cm.dlog("in election timer state=%s, bailing out", cm.state)
            cm.mu.Unlock()
            return
        }

        if termStarted != cm.currentTerm {
            cm.dlog("in election timer term changed from %d to %d, bailing out", termStarted, cm.currentTerm)
            cm.mu.Unlock()
            return
        }

        // Start an election if we haven't heard from a leader or haven't voted for
        // someone for the duration of the timeout.
        if elapsed := time.Since(cm.electionResetEvent); elapsed >= timeoutDuration {
            cm.startElection()
            cm.mu.Unlock()
            return
        }
        cm.mu.Unlock()
    }
}
```

Using `defer` is essential for maintainability. It allows easier reasoning about lock state. It makes clear, that the mutex is acquired after the `Lock()` until the function exits. Reading `defer`-less code requires tracing program flow and finding out which code lines run in which lock state. You can improve maintainability even further by acquiring the mutex only at the beginning of a function. This makes it clear, that the whole function code runs in locked state.

`defer` protects against human failure. When we think about maintainability, we have to think about future code changes. Maybe by the original author, who will be out of context by then, maybe by some dev not familiar with the code at all. A new exit point is easily added to a function, and maybe - lacking attention - without the mandatory `Unlock()`. Code using `defer` is immune to such bugs.

Code using `defer` is easier to review. `defer mu.Unlock()` is usually placed directly next to the mutex acquisition. This makes it easy to check for correctness, when reviewing the code. If there is no `defer mu.Unlock()` next to the `mu.Lock()`, it was most likely forgotten.

`defer` is `panic`/`recover` safe. Function execution can exit at any call to another method or function, if this method or function panics. A `Unlock()` without `defer` will not be executed in that case. `defer` [guarantees to run before the function returns](https://golang.org/ref/spec#Defer_statements):

> A "defer" statement invokes a function whose execution is deferred to the moment the surrounding function returns, either because the surrounding function executed a return statement, reached the end of its function body, or because the corresponding goroutine is panicking.

If a function panics, the call stack is unwound, executing any deferred function calls. If the panic is not recovered along the call chain, the program exits. Two problems can arise here if a mutex was acquired before panic and not released by `defer`. First, if a deferred function tries to acquire the mutex (maybe to do some thread safe clean up) it will dead lock. Second, if the program recovers from panic state, at some point, it will try to acquire the mutex again, it will dead lock.

Eli's production code does not contain any of these two problem scenarios right now (test code does), but from the perspective of maintainability, we have to think about future changes. It's quite possible, that in the future someone adds a deferred function, which performs complex clean up operations, acquiring the mutex along the way. Or a `recover` is added at some point. The tricky thing about `defer`/`panic`/`recover` is, that the panicking code line can be far away (in terms of call chain depth) from the recovering code line. The dev performing the change, may simply not think about code down the call chain acquiring mutex without defer-releasing them. But even *if* she thinks about it, it may be too hard to check the whole call hierarchy for such cases.

**[Commit "Use defer Unlock() consequently"](https://github.com/ewirch/raft/commits/refactor-for-maintainability)** A un-deferred `Unlock()` can be usually turned in a deferred `Unlock()` by extracting code lines into functions. The simple example from above turns to:
```go
    ...
    go func() {
        // The CM is dormant until ready is signaled; then, it starts a countdown
        // for leader election.
        <-ready
        cm.resetElectionTimer()
        cm.runElectionTimer()
    }()

    go cm.commitChanSender()
    return cm
}

func (cm *ConsensusModule) resetElectionTimer() {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    cm.electionResetEvent = time.Now()
}
```

More complex cases have control flow statements interwoven with mutex acquisition / release. Those can be refactored introducing returned states (complex case from above):
```go
func (cm *ConsensusModule) runElectionTimer() {
    ...    
    for {
        <-ticker.C

        stop := cm.electionTimerTick(termStarted, timeoutDuration)
        if stop {
            return
        }
    }
}

func (cm *ConsensusModule) electionTimerTick(termStarted int, timeoutDuration time.Duration) bool {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    if cm.state != Candidate && cm.state != Follower {
        cm.dlog("in election timer state=%s, bailing out", cm.state)
        return true
    }

    if termStarted != cm.currentTerm {
        cm.dlog("in election timer term changed from %d to %d, bailing out", termStarted, cm.currentTerm)
        return true
    }

    // Start an election if we haven't heard from a leader or haven't voted for
    // someone for the duration of the timeout.
    if elapsed := time.Since(cm.electionResetEvent); elapsed >= timeoutDuration {
        cm.startElection()
        return true
    }
    return false
}
```


# Concurrency

The biggest change I implement, is, how goroutines are synchronized. Eli choose to use a mutex to control concurrent access to shared mutable state.
```go
cm.mu.Lock()
termStarted := cm.currentTerm
cm.mu.Unlock()
```

## Problems

Looking at maintainability, this approach has some downsides. When looking at the code of a function, it is unclear whether the function runs in locked state or unlocked state. A dev, who has the task to modify the code, will have to check the call hierarchy and program flow to find this out. This slows down modifications and is error prone.

When calling functions, it is easy to mess up lock state and try to acquire a mutex twice. Assume a function is always called from unlocked state. It acquires the mutex on its own, but this is not clear from the function name alone. Now imagine a dev, new to the code base, naively uses the function while modifying the code. He wants to log `currentTerm`:
```go
func (cm *ConsensusModule) printDiagnostics() {
    cm.dlog("id=%v\n", cm.id)
    cm.dlog("state=%v\n", cm.state)
    cm.dlog("currentTerm=%v\n", cm.getTerm())
}
```
The modification looks fine at first sight, until you inspect the context of `printDiagnostics()` invocation:
```go
func (cm *ConsensusModule) runElectionTimer() {
    ...
    for {
        <-ticker.C

        cm.Lock()
        cm.printDiagnostics()
        ...
    }
}

func (cm *ConsensusModule) getTerm() int {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    term := cm.currentTerm
    return term
}
```
The dev created a dead lock, and it was not even hard. 

The last downside I see is, that it is unclear what exactly the mutex guards. This is the `ConsensusModule` declaration:
```go
type ConsensusModule struct {
    // mu protects concurrent access to a CM.
    mu sync.Mutex

    // id is the server ID of this CM.
    id int

    // peerIds lists the IDs of our peers in the cluster.
    peerIds []int

    // server is the server containing this CM. It's used to issue RPC calls
    // to peers.
    server *Server

    // storage is used to persist state.
    storage Storage

    // commitChan is the channel where this CM is going to report committed log
    // entries. It's passed in by the client during construction.
    commitChan chan<- CommitEntry

    // newCommitReadyChan is an internal notification channel used by goroutines
    // that commit new entries to the log to notify that these entries may be sent
    // on commitChan.
    newCommitReadyChan chan struct{}

    // triggerAEChan is an internal notification channel used to trigger
    // sending new AEs to followers when interesting changes occurred.
    triggerAEChan chan struct{}

    // Persistent Raft state on all servers
    currentTerm int
    votedFor    int
    log         []LogEntry

    // Volatile Raft state on all servers
    commitIndex        int
    lastApplied        int
    state              CMState
    electionResetEvent time.Time

    // Volatile Raft state on leaders
    nextIndex  map[int]int
    matchIndex map[int]int
}
```
Which fields require acquiring the mutex? A first guess would be: not thread safe types. Wrong, `id` and `peerIds` are effectively final. They are not modified after initialization, they don't need synchronization. `newCommitReadyChan` is thread safe, but it is used in locked context only. How about custom types like `Server` or `Storage`? Does access to these fields require synchronization?

## Improving

1. Group shared mutable state to one object. This makes it clear which fields need synchronization before being accessed, even for a dev new to the code base.
```go
type mutableConsensusModule struct {
    // storage is used to persist state.
    storage Storage

    // Persistent Raft state on all servers
    currentTerm int
    votedFor    int
    log         []LogEntry

    // Volatile Raft state on all servers
    commitIndex        int
    lastApplied        int
    state              CMState
    electionResetEvent time.Time

    // Volatile Raft state on leaders
    nextIndex  map[int]int
    matchIndex map[int]int
}
```

2. Acquire the mutex before calling [methods of the shared mutable state object](https://golang.org/ref/spec#Method_sets). Access the fields of the shared mutable state object **only** from methods of this object.
```go
func (m *mutableConsensusModule) getTerm() int {
    return m.currentTerm
}
```
This approach maps the state of the mutex acquisition to code structure. If you look at the code of a shared mutable state object method, then it is clear it is safe to access and mutate shared mutable state. If it is not a shared mutable state object method, then it is not safe. Code structured in this way is easy to read and modify (at least in terms of concurrency synchronization).

3. The shared mutable state object methods should not have access to the object referencing the mutex (`ConsensusModule`). These methods run while the mutex is acquired. A single call to `mu.Lock()` will deadlock. But if the methods have no reference to the mutex, it is impossible for them to create a deadlock. To implement this in Eli's Raft implementation a major refactoring is required, see next section.

4. Acquire the mutex in a dedicated method, and declare the release (using `defer`) immediately. Invoke the shared mutable state object method immediately. Do not put any other logic into these methods. Let's call these methods "delegate methods".

```go
func (cm *ConsensusModule) runElectionTimer() {
    timeoutDuration := cm.electionTimeout()
    termStarted := cm.getTerm()
    ...
}

func (cm *ConsensusModule) getTerm() int {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    return mutable.getTerm()
}

func (m *mutableConsensusModule) getTerm() int {
    return m.currentTerm
}
``` 
This looks cumbersome, but together with point 3 the benefit is a virtually zero chance of deadlocking. `ConsensusModule.getTerm()` has access to the mutex (possibility of acquiring the mutex twice), but the code is reduced to a minimum. It is trivial to review, that the method acquires the mutex only once, and releases it as well.

This is the essence of maintainability: the code structure itself enforces correctness.

## Reference Either `ConsensusModule` Or `mutableConsensusModule`
Point 3 from the previous section introduces a major refactoring I did. To improve maintainability I had to change the code in a way, so `mutableConsensusModule` methods won't have access to `ConsensusModule` (which stores the reference to the mutex). This was particularly tricky, since Eli's code creates goroutines accessing `ConsensusModule` from locked context. Example:
```go
func (cm *ConsensusModule) AppendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) error {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    ...

    if args.Term > cm.currentTerm {
        cm.dlog("... term out of date in AppendEntries")
        cm.becomeFollower(args.Term)
    }
    ...
}

func (cm *ConsensusModule) becomeFollower(term int) {
    cm.dlog("becomes Follower with term=%d; log=%v", term, cm.log)
    cm.state = Follower
    cm.currentTerm = term
    cm.votedFor = -1
    cm.electionResetEvent = time.Now()

    go cm.runElectionTimer()
}
```
`becomeFollower()` runs with an acquired mutex and starts `runElectionTimer()` in a new goroutine, which needs access to `ConsensusModule`. After changes for point 1 and 2 from previous section (group shared mutable state and modify these fields only from methods of this shared mutable state object), the code looks this way:
```go
func (m *mutableConsensusModule) becomeFollower(term int) {
    m.dlog("becomes Follower with term=%d; log=%v", term, m.log)
    m.state = Follower
    m.currentTerm = term
    m.votedFor = -1
    m.electionResetEvent = time.Now()

    // ??? go cm.runElectionTimer()
}
```
The method cannot run `runElectionTimer()`, since it has no access to `ConsensusModule` anymore. I solved this by decoupling declaration and start of goroutines. Instead of directly passing functions to `go` statement, they are wrapped in a function and pushed to a buffered channel. **[Commit "Use a top level loop to start new go routines"](https://github.com/ewirch/raft/commits/refactor-for-maintainability)**

```go
func (m *mutableConsensusModule) becomeFollower(term int) {
    m.dlog("becomes Follower with term=%d; log=%v", term, m.log)
    m.state = Follower
    m.currentTerm = term
    m.votedFor = -1
    m.electionResetEvent = time.Now()

    m.goRoutines <- func(cm *ConsensusModule) {
        cm.runElectionTimer()
    }
}
```
The channel is read by a long-lived (for the whole application lifetime) goroutine running in unlocked state and having a reference to `ConsensusModule`:
```go
func (cm *ConsensusModule) goRoutinesStarter() {
    for routine := range cm.goRoutines {
        go routine(cm)
    }
    cm.dlog("goRoutinesStarter done")
}
```
Side note: the change set in my repo looks a bit different, because I committed the changes in a different order. But the effect remains the same.

## Borrowed Ownership
I wanted to go one step further. I wanted to make it impossible to access the shared mutable state object without "locking" it. Using a mutex as synchronization primitive, it is always possible to access the shared mutable state object without acquiring the mutex. One of Go's [slogans](https://golang.org/doc/effective_go.html#sharing) to the rescue:

> Do not communicate by sharing memory; instead, share memory by communicating.

Background: When a goroutine owns an object and no other goroutine has access to it, then this is obviously thread safe. But this static ownership does not allow multiple goroutines to work on one object. To share access to an object, the object ownership is borrowed and returned after use. A synchronization primitive is used to implement the borrowing and returning. This is thread safe as well. The object is shared between goroutines, but only serially. Only one goroutine has access to the object at a time.

It is a different way to look at synchronization. Eli's approach uses this model implicitly. Acquiring the mutex `ConsensusModule.mu` borrows ownership of shared fields. Releasing the mutex returns ownership. To improve maintainability of this synchronization approach, I want to make it explicit.

I removed the reference to `mutableConensusmodule` from the `ConsensusModule` struct, and replaced it by a channel. A channel is a synchronization primitive by itself, so we don't need the mutex anymore. When the shared mutable state object is in the channel, it can be grabbed (borrowed) by any goroutine. When a goroutine tries to grab the object, while another has already borrowed it, reading from the channel will block. The shared mutable state object is returned by writing it to the channel again. The next goroutine can grab it.

This way it is not possible to access the shared mutable state object directly. A goroutine has to read it from the channel first. The very action of reading the object from the channel is the borrowing operation. No unintentional data races are possible. This is nice and elegant, isn't it?

This:
```go
func (cm *ConsensusModule) appendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) (commitIndexChanged bool) {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    return mutable.appendEntries(args, reply)
}
```
becomes:
```go
func (cm *ConsensusModule) appendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) (commitIndexChanged bool) {
    mutable := <-cm.mu
    defer func() { cm.mu <- mutable }()

    return mutable.appendEntries(args, reply)
}
```

**[Commit "Share memory by communicating"](https://github.com/ewirch/raft/commits/refactor-for-maintainability)**

# Outlook

There is one more thing I want to improve but didn't find enough energy to do it yet. The tests are flaky. Eli uses `time.Sleep()` in tests and expects some condition after a fixed duration. This is inherently fragile. The exact delay duration is unreliable. Documentation of `time.Sleep()` says:

> Sleep pauses the current goroutine for at least the duration d.

It guarantees a sleep duration of **at least** the specified amount, but it can sleep longer. The `TestCrashAfterSubmit` test seems to be affected the most by this problem. This test fails sometimes with Eli's version of the code. My refactorings seem to worsen this (the test fails a lot, but it also passes sometimes).

It is impossible to predict concurrent state using delays. But how can we sync test code to concurrently running production code without cluttering production code with test helper code?

The code already has many log statements. Eli uses the `ConsensusModule.dlog()`, which directly writes to console. I plan to introduce a `Logger` field to `ConsensusModule`, which will point to `stdout` for normal program execution and will be redirected to a buffered writer when running tests. This way some test code can read the log output and trigger state modifications based on logged messages.

Instead of:
```go
h.SubmitToServer(origLeaderId, 5)
sleepMs(1)
h.CrashPeer(origLeaderId)
```
something along the lines of:
```go
event := h.logStarsWith(origLeaderId, "AppendEntries sent to ")
event.whenHit(() -> h.CrashPeer(origLeaderId)))
h.SubmitToServer(origLeaderId, 5)
event.wait()
```
