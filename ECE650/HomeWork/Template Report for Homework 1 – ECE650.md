## Part I: Requirements and Development Overview

### 1. Development Goals and Main Requirements

The primary objective of this assignment is to implement a **custom memory allocator**, providing user-level implementations of the `malloc` and `free` functions. The core requirements are summarized as follows:

- **Memory Management**  
    Use the `sbrk()` system call to request heap memory from the operating system.
- **Allocation Policies**  
    Implement and compare two free-block selection strategies:
    1. **First Fit (FF)**: Select the first free block that is large enough to satisfy the request.
    2. **Best Fit (BF)**: Select the free block that is large enough and leaves the smallest remaining unused space.
- **Memory Reclamation**  
    Implement `free()` such that released blocks are returned to the free list and **coalesced** with physically adjacent free blocks to reduce external fragmentation.
    
- **Performance Evaluation**  
    Compare FF and BF in terms of:
    - Allocation time (runtime performance)
    - Memory fragmentation and space utilization

---

### 2. Development Environment and Tools

- **Operating System**: Ubuntu 22.04.5 LTS  
    (GNU/Linux 6.6.87.2-microsoft-standard-WSL2 x86_64)
- **Programming Language**: C
- **Compiler**: GCC
- **Build System**: Make

---

## Part II: Design, Implementation, and Testing

### 1. Data Structure Design

![[images/e3298a32ea04ba6a2ce934bd99f48992.jpg]]

To manage heap memory efficiently, a **block header structure** (`block_t`) is placed at the beginning of each memory block to store metadata.

Each block header contains the following fields:

- **Status and Size**
    - `size_t size`: Size of the usable payload (excluding metadata).
    - `bool isfree`: Indicates whether the block is free or allocated.

- **Free List Pointers (Logical Structure)**
    - `block_t *next_free`
    - `block_t *prev_free`  
        These pointers maintain a **doubly linked free list** that connects only free blocks, allowing fast traversal without scanning allocated blocks.

- **Physical Neighbor Pointers (Physical Structure)**
    - `block_t *next_phys`
    - `block_t *prev_phys`  
        These pointers link physically adjacent blocks in memory and are used to perform **immediate coalescing** when a block is freed.

---

### 2. Implementation Logic

#### Core Mechanisms

- **Heap Expansion**  
    When no suitable free block is found, the allocator uses `sbrk()` to extend the program break and obtain additional heap memory.
    
- **Alignment**  
    All blocks are aligned to **8-byte boundaries**. This is necessary because the largest fields (`size_t` and pointers) occupy 8 bytes.  
    Proper alignment avoids undefined behavior on certain architectures (e.g., ARM) and improves memory access performance.
    ```c
    typedef struct BlockHeader block_t;
	struct BlockHeader {
	    bool isfree;
	    size_t size;
	    block_t *next_free;
	    block_t *prev_free;
	    block_t *next_phys;
	    block_t *prev_phys;
	};
    ```
![[images/b9763f88dc951bdb2ca2261ca08adfe9.jpg]]

- **Block Splitting**  
    When a free block is significantly larger than the requested size, it is split into:
    - One allocated block
    - One smaller free block
    
    Splitting occurs only if the remaining space is at least  
    `sizeof(block_t) + MIN_SPLIT`.  
    Here, `MIN_SPLIT = 8`, which corresponds to the minimum aligned payload size.
    
- **Block Coalescing**  
    When `free()` is called, the allocator checks both the previous and next physical neighbors:
    - If either neighbor is free, it is removed from the free list.
    - All adjacent free blocks are merged into a single larger block.
    - The merged block is then reinserted into the free list.
    This **immediate coalescing** strategy significantly reduces external fragmentation.

---
### 3. Allocation Policy Comparison

| Feature              | First Fit (FF)                                                            | Best Fit (BF)                                                                  |
| -------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Algorithm**        | Traverses the free list and selects the _first_ block with size ≥ request | Traverses the _entire_ free list to find the block with minimal leftover space |
| **Allocation Speed** | **Fast** – usually terminates early                                       | **Slower** – requires full list traversal                                      |
| **Free List Policy** | LIFO insertion (newly freed blocks added to head)                         | Same LIFO insertion                                                            |
| **Cache Locality**   | High – tends to reuse recently freed blocks                               | No benefit, since full scan is required                                        |

---
### 4. Testing Methodology

The provided test suite was used for both functional verification and performance evaluation.

The following tests were executed:

- `alloc_policy_tests`
- `general_tests`
---
## Part III: Performance Results and Analysis

### 1. Experimental Results

**First Fit (FF)**  
![[images/Pasted image 20260121024416.png]]

**Best Fit (BF)**  
_![[images/Pasted image 20260121024510.png]]

---

### 2. Performance Analysis

#### Equal-Size Allocations

- **Observation**: FF and BF exhibit very similar performance.
- **Explanation**:  
    Since all requests are of identical size, Best Fit provides no additional benefit over First Fit. Its full traversal overhead only increases execution time slightly.
    
---

#### Small-Range Random Allocations

- **Runtime Performance**:  
    First Fit significantly outperforms Best Fit.  
    Best Fit suffers from an O(N) search cost on every allocation, while First Fit often finds a suitable block early in the free list.
- **Fragmentation**:  
    Best Fit typically achieves slightly better memory utilization. By filling smaller gaps first, it preserves larger blocks for future allocations.
    

---

#### Large-Range Random Allocations

- **Runtime Performance**:  
    First Fit continues to show a clear speed advantage.
    
- **Fragmentation Behavior**:  
    This scenario highlights the difference between the two strategies.  
    First Fit may split very large blocks to satisfy small requests, creating many small leftover fragments.  
    Best Fit instead attempts to find the smallest suitable block, preserving large blocks for future large allocations.
    
- **Result**:  
    Although Best Fit generally performs better in theory, in this particular test the fragmentation difference between the two strategies was relatively small.
    

---

### 3. Conclusions and Design Guidelines

- **First Fit**
    - Best suited for systems where **throughput and speed** are critical.
    - Provides near O(1) average allocation time.
    - Simple and efficient in general-purpose environments.

- **Best Fit**
    - Suitable for **memory-constrained systems** .
    - Achieves better space utilization at the cost of higher CPU overhead.
        

**Overall Conclusion**  
In typical general-purpose computing environments without extreme memory constraints, **First Fit** is often the better engineering choice. It offers a strong balance between performance, simplicity, and acceptable memory utilization.