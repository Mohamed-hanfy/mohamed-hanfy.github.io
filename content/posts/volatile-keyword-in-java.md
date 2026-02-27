---
title: "volatile keyword in Java with Real World Example!"
date: 2026-02-27
draft: false
---

Last week, while working on my Graduation Project [UPPLY](https://github.com/upply-org/upply-backend), my task was to export data into an Excel file. I started building it using an HTTP polling communication pattern and executing the work in virtual threads. This raised an interesting question: how can I ensure that all threads write data to a shared memory location correctly? Before diving in, I want to make sure you have a basic understanding of the Java Memory Model (JMM).

## Java Memory Model (JMM)

The Java Memory Model defines how threads interact through memory — essentially, the rules governing when writes by one thread become visible to other threads.

### The Core Problem

Modern CPUs and compilers reorder instructions and cache values in registers and local caches for performance. Without clear rules, one thread might write a value that another thread never "sees" because it is reading a stale cached copy.

### Main Memory vs. Working Memory

Conceptually, each thread has its own working memory (think: CPU cache/registers), and there is a shared main memory. Threads don't read from or write to main memory directly — they work on local copies and periodically sync.

```
┌─────────────────────┐      ┌─────────────────────┐
│   Virtual Thread    │      │   Request Thread    │
│  ┌───────────────┐  │      │  ┌───────────────┐  │
│  │ Working Mem   │  │      │  │ Working Mem   │  │
│  │ status=COMPLETED │      │  │ status=processing│
│  └───────┬───────┘  │      │  └───────┬───────┘  │
└──────────┼──────────┘      └──────────┼──────────┘
           │    NO guaranteed sync ✗    │
           └──────────┬─────────────────┘
              ┌───────┴────────┐
              │  Main Memory   │
              │  status = ?    │
              └────────────────┘
```

Without synchronization, the request thread may never see what the virtual thread wrote.

## volatile

### What volatile Actually Does
Marking a field volatile makes two guarantees:

*Visibility*  Every read of a volatile field goes directly to main memory, and every write is flushed to main memory immediately. No thread ever reads a stale cached copy.

*Ordering* (memory fence). A volatile write acts as a memory fence. All writes that happened before the volatile write are flushed along with it. Any thread that subsequently reads that volatile field is guaranteed to see not just the new value, but everything that was written before it.

This second point is subtle but critical, and we'll see exactly why it matters in the real-world example below.

The trade-off is performance. Bypassing CPU caches and preventing certain compiler optimizations has a cost, so volatile should be used deliberately — not sprinkled everywhere as a safety blanket.


```java
boolean running = true; // not volatile

// Thread 1
while (running) { ... }

// Thread 2
running = false; // Thread 1 may NEVER see this!
```

Changing this to `volatile boolean running = true` solves the problem. The write in Thread 2 is immediately flushed to main memory, and Thread 1's read always fetches the latest value. but the trade off is it prevents CPU optimizations.

## Real World Example — Excel Export with Virtual Threads and HTTP Polling

Here's the actual scenario I ran into. The task was to export application data to an Excel file. Because the generation can take a while, I used an HTTP polling pattern:

1. The client sends a **start** request and gets back a `taskId`.
2. A **virtual thread** runs the export in the background and writes the result to a shared `ExportTask` object.
3. The client **polls** a status endpoint until the task completes, then fetches the file.

### The ExportTask Class (Broken Version)

```java
public class ExportTask {
    private Status status;        // ← plain field, NOT volatile
    private byte[] data;          // ← plain field, NOT volatile
    private String errorMessage;  // ← plain field, NOT volatile
    private final Instant createdAt;
    private final Instant expireAt; 

    public enum Status { PENDING, COMPLETED, FAILED }
}
```

These tasks are stored in a `ConcurrentHashMap<String, ExportTask>` and accessed across three different threads:

- A **virtual thread** calls `processExport()`, which writes `data`, `status`, and `errorMessage`.
- **HTTP request threads** read `status` and `data` in `getExportTaskStatus()` and `getExportedFileData()`.
- A **scheduled thread** reads `expireAt` in `cleanExpiredTasks()` to evict old tasks.

### Why ConcurrentHashMap Doesn't Save You

A very common misconception: storing objects *in* a `ConcurrentHashMap` does **not** make those objects themselves thread-safe. `ConcurrentHashMap` only guarantees safe concurrent access to the **map structure** — the put, get, and remove operations. Once you hold a reference to an `ExportTask`, the fields inside it are completely unguarded.

```
ConcurrentHashMap.put(id, task)  →  happens-before  →  ConcurrentHashMap.get(id)
                     ↑
          That's all you get.
          The fields INSIDE task? Zero guarantees.
```

### The Race Condition

```java
private void processExport(ExportTask task, List applications) {
    try {
        byte[] data = applicationExcelExportService.generateExcel(applications);
        task.setData(data);              // plain write — stays in CPU cache
        task.setStatus(COMPLETED);   
    } catch (Exception e) {
        task.setStatus(FAILED);
        task.setErrorMessage(e.getMessage());
    }
}
```

Two things can go wrong here:

**Stale read.** The virtual thread's writes sit in its CPU core's cache and never reach main memory before the request thread polls. The request thread spins forever on `status == PENDING` even though the virtual thread has already finished.

**Reordering.** The JIT is allowed to reorder the writes to `data` and `status`. It could write `status = COMPLETED` before the `data` array is fully written. A request thread could then see `COMPLETED`, go to fetch the data, and read a null or incomplete byte array.


### The Fix — `volatile` Fields

```java
public class ExportTask {
    private volatile Status status;         
    private volatile byte[] data;           
    private volatile String errorMessage;   
    private final Instant createdAt;       //  final is already safe
    private final Instant expireAt;       //  final is already safe
}
```

A `volatile` write does two things:

1. **Flushes** the written value to main memory immediately.
2. **Acts as a memory fence** — all writes that happened *before* this volatile write are also flushed and will be visible to any thread that subsequently reads this volatile variable.

So the write order still matters: write `data` first, then write `status`. Because the `volatile` write to `status` fences everything before it, any thread that reads `status = COMPLETED` is **guaranteed by the JMM** to also see the fully written `data`. The JIT is no longer allowed to reorder across this boundary.

```
Virtual Thread                        Request Thread
──────────────                        ──────────────
task.data = bytes    (plain write)
task.status = COMPLETED  ← volatile   ← memory fence
                                      volatile read: status = COMPLETED
                                      plain read:    data = [bytes] ✓ always visible
```

---

## Full source:

[ExportTask.java](https://github.com/upply-org/upply-backend/blob/main/src/main/java/com/upply/job/ExportTask.java)

[JobService.java](https://github.com/upply-org/upply-backend/blob/926b3172e6d97f63bbb6076bbd00f4571edda40c/src/main/java/com/upply/job/JobService.java#L296)