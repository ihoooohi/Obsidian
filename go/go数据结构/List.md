## 标准库
`import "container/list"`
## 实现原理
带哨兵节点（root）的循环双向链表
```go
type List struct {
	root Element //哨兵节点
	len int
}

type Element struct {
	prev, next *Element
	list *List //表示这个节点属于哪条链表
	Value any 
}
```