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

## Error Control

### Hamming Distance

the number of bit differences between two binary strings

11100
11111

the hamming distance is 2

### codeword and code

a frame is now n = m + r bits
- m = data bits
- r = redundant bits

the n-bit chunk is called **codeword**

**code** is a set of codewords

### Error Detection & Error Correction

assume hamming distance is d
error detection: detect (d-1) single-bit error
error correction: correct (d-1)/2 single-bit error

reliable channel(e.g., fiber): error detection
unreliable channel(e.g., wireless): error correction

because if unreliable channel use error detection, it will often resend, which can take too much time

### Error Correction Example: Hamming Codes
### Error Detection Example: CRC
> The sender treats the data as one large number and divides it by a predetermined number (the generator).

> The sender then sends both the original data and the remainder（余数）.

> When you receive it, you divide the received data by the same divisor again.

> If the remainder is **0**, the data is assumed to be intact（完好无损的）;

> if the remainder is **not 0**, it means the data was corrupted（乱码的） during transmission.

## Data Link Layer Performance

throughput(吞吐量)

![[Pasted image 20260128111600.png]]