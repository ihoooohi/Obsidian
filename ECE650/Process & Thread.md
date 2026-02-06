## 进程和程序的关系

一个进程对应一段程序（process ---> a.out），一个程序可以有多个进程（浏览器是一个程序，不同页面是多个进程）

## 进程（process）的内存地址空间

![[images/Pasted image 20260105125219.png]]
stack存放局部变量，heap是动态申请的内存，static data是全局变量和静态变量，code是机器指令（二进制/十六进制）

⭐进程不是直接访问物理内存，而通过虚拟内存，每个进程有一个虚拟地址空间，通过页表映射成物理地址

进程中的stack其实只是个逻辑概念，实际上是线程的stack

## 线程（thread）的内存地址空间

![[images/Pasted image 20260105130351.png]]
线程代表一个执行的上下文：
- Program Counter （PC）
- Stack Pointer （SP）
- register