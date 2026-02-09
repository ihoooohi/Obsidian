
## Part 1 — Implementation Description
### 2.2 Version 1: Lock-Based Thread-Safe Malloc/Free
#### 2.2.1 Concurrency Model
The lock-based implementation uses a **single global mutex** to protect the entire allocator. This means:

- At most **one thread** can execute inside `ts_malloc_lock()` or `ts_free_lock()` at any time.
- All allocator operations become effectively serialized.

Concurrency is therefore **not allowed inside the allocator**, but is allowed outside the allocator in the rest of the program.

#### 2.2.2 Critical Sections
The following operations were identified as critical sections because they modify shared allocator state:

- Traversing and updating the global free list (best-fit search, insertion, removal)
- Splitting blocks
- Coalescing blocks and updating physical neighbor links
- Calling `sbrk()` to extend the heap
- Updating global pointers such as the free list head and physical list tail

Without synchronization, multiple threads could remove the same block, corrupt list pointers, or cause overlapping heap extensions.

#### 2.2.3 Synchronization Strategy
A global mutex `malloc_lock` is acquired at the start of each allocator call and released at the end:

```mermaid
flowchart TB
  subgraph P[Same Process]
    H[(Heap via sbrk)]
    FL[Global Free List]
    PL[Global Phys List]
    L[Global Lock]
  end

  T1[Thread 1] --> L
  T2[Thread 2] --> L

  L --> FL
  L --> PL
  L --> H

```

---

### 2.3 Version 2: Non-Locking Thread-Safe Malloc/Free

#### 2.3.1 Concurrency Model
The non-locking implementation is designed to allow concurrency across threads. The key idea is to avoid shared allocator metadata whenever possible.

- Each thread maintains its own free list using **Thread-Local Storage (TLS)**.
- Most malloc/free operations are performed on thread-private data, so they can run concurrently without locks.

The only shared operation is `sbrk()`, because the heap break is global and `sbrk()` is not thread-safe.

#### 2.3.2 Where Concurrency Is Allowed
This implementation allows concurrency in:

- best-fit search in a thread-local free list
- removing blocks from a thread-local free list
- inserting blocks into a thread-local free list
- splitting blocks in thread-local context

Multiple threads can perform these operations at the same time because they operate on independent free lists.

#### 2.3.3 Critical Section
The only critical section in the non-locking version is:

- calling `sbrk()` to extend the heap

If multiple threads call `sbrk()` simultaneously, they may receive overlapping memory regions. Therefore, `sbrk()` must be protected.

#### 2.3.4 Synchronization Strategy
A mutex `sbrk_lock` is used only around `sbrk()` calls:

```mermaid
flowchart TB
  subgraph P[Same Process]
    H[(Heap via sbrk)]
    SL[Sbrk Lock only]
  end

  subgraph T1[Thread 1]
    FL1[Free List TLS]
  end

  subgraph T2[Thread 2]
    FL2[Free List TLS]
  end

  T1 --> FL1
  T2 --> FL2

  FL1 --> SL
  FL2 --> SL

  SL --> H

```
---

## 3. Part 2 — Experimental Results

### 3.1 Test Description
The provided test program `thread_safe_measurement.c` reports:

1. **Execution Time**
2. **Data Segment Size** (the total size of the heap after running the test)

The test includes randomized behavior, so results may vary slightly between runs.

---

### 3.2 Results

| Version | Execution Time (s) | Data Segment Size (bytes) |
|--------:|--------------------:|--------------------------:|
| Non-locking | 0.095157 | 44,297,536 |
| Locking | 0.189614 | 43,910,832 |

---

### 3.3 Discussion of Tradeoffs

#### 3.3.1 Performance (Execution Time)
The non-locking version was significantly faster:

- Non-locking: **0.095157 s**
- Locking: **0.189614 s**

The non-locking version is about **2× faster** in this experiment.

This is expected because the locking version serializes all allocator calls using a global mutex. With multiple threads frequently allocating and freeing memory, the lock becomes a bottleneck and forces threads to wait.

In contrast, the non-locking version allows multiple threads to allocate and free concurrently in most cases because each thread operates on its own free list.

---

#### 3.3.2 Allocation Efficiency (Data Segment Size)
The locking version produced a slightly smaller final heap size:

- Locking: **43,910,832 bytes**
- Non-locking: **44,297,536 bytes**

This indicates that the locking version was slightly more memory efficient in this run.

A likely explanation is that in the locking version, all threads share the same global free list. This means freed blocks are immediately available for reuse by any thread, improving reuse and reducing the need for heap expansion.

In the non-locking version, free blocks are stored in thread-local free lists. A thread may request new memory via `sbrk()` even while another thread has free blocks available, simply because those free blocks are stored in a different thread’s local list. This can lead to a larger data segment and increased fragmentation.

---

#### 3.3.3 Summary of Tradeoffs
Overall, the experiment demonstrates a classic tradeoff:

- **Locking version**
  - Pros: slightly better allocation efficiency, simple correctness reasoning
  - Cons: slower due to lock contention and serialized execution

- **Non-locking version**
  - Pros: much faster due to increased concurrency and reduced contention
  - Cons: slightly worse memory efficiency due to per-thread free lists and reduced cross-thread reuse

---

## 4. Conclusion

Both implementations satisfy the requirement of thread-safe `malloc/free` using the best-fit policy.

The lock-based version ensures correctness through a single global mutex, which makes the allocator simple and robust but limits concurrency.

The non-locking version achieves higher concurrency by using thread-local free lists and only synchronizing the `sbrk()` operation. This design significantly improves performance under multi-threaded workloads, at the cost of slightly reduced memory efficiency.

The experimental results support these conclusions: the non-locking version is faster, while the locking version produces a slightly smaller final data segment.





