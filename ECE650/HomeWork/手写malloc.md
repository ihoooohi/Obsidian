怎么手写malloc呢，找到符合条件的空间给变量用

什么是对齐

就是malloc 返回的指针，**必须能安全存放任何类型**
```c
void *p = malloc(100);

(int*)p;        // OK
(double*)p;     // OK
(struct S*)p;   // OK

```