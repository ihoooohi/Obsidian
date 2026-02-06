- 递归终止条件
	- ```python
	  if not head or not head.next:
	  ```
  - 同时处理 **空链表** 和 **单节点链表**, 避免多余递归，提高可读性与健壮性。
- 快慢指针，走两步的fast
	- ```python
	  while fast and fast.next:
	  ```
	  不写while fast.next的原因：==若fast变成none，下一次循环判断时会触发none.next报错，所以要添加while fast的检查==

