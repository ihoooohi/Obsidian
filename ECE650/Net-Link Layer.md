## Layer 2 -- Data Link

### what does Layer2 do?

Layer2 tries to make the **packet** of Layer3 to **frame** which can transport in physical link

### errors handling

Layer2 handle **bit-related errors**, not **packet drop(Layer4)**

## basic mechanism --- framing

Layer3 gives Layer2 a sequence of bits, Layer2 divides bit stream **frame by frame** and add **Header** + **Tailer**, then send frame over Layer1


![[Pasted image 20260126104457.png]]

`Frame = [Header | Packet | Trailer]`

### How does Layer2 divide bit stream into frames?

the answer is Bit Stuffing with Start & End flags，the flag is **6 consecutive 1's**, `111111`

if five 1's `11111` is in packet, it will insert 0 on the tail.

![[Pasted image 20260126111536.png]]

## Layer 2 Services – Service to Network Layer

- **Unacknowledged connectionless service**
	- no acknowledge(ack) , no connection
- **Acknowledged connectionless service**
	- have ack, no connection
-  **Acknowledged Connection-Oriented Service**
	- have ack, have connection (similar as TCP)
## flow control

Frames sent only when receiver gives permission
