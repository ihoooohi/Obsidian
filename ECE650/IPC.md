## Shared Memory
![[Pasted image 20260114104257.png]]
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

![[Pasted image 20260114104342.png]]