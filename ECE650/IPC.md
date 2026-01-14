## Shared Memory
![[Pasted image 20260114104257.png|300]]
mmap
```
Process A        Process B
   |                 |
mmap(file)      mmap(file)
   |                 |
same physical memory
```


mmap
```
open file
   ↓
kernel creates mapping
   ↓
virtual address ↔ file content
   ↓
page fault loads data
   ↓
multiple processes map same file
   ↓
shared physical memory (IPC)
```

## Message Passing

![[Pasted image 20260114104342.png|300]]

```
Shared Memory
  → race condition
  → mutex / semaphore

Message Passing
  → blocking read
  → multiple producers
  → select / poll
```