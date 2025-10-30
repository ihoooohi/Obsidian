- **初始化dummy node：**
    - ```python
	      dummy = ListNode(-1)
      ```
- **使用情况**
    - **删除节点（安全删除头节点）**
		```python
		dummy = ListNode(0)
		dummy.next = head
		current = dummy
		while current.next:
		    if current.next.val == target:
	        current.next = current.next.next
		    else:
		        current = current.next
			return dummy.next
		```
		==这里，dummy 节点的存在让你能安全地操作 current.next，不用担心删除的是头节点==
	- **创建链表**
		- ```python
		  # 3. 根据排序后的数组，重新构建一个有序链表 
		  # 使用虚拟头节点技巧 
		  dummy = ListNode(-1) 
		  tail = dummy 
		  # tail 指针用来构建新链表 
		  for val in values: 
			  # 创建新节点 
			  new_node = ListNode(val) 
			  # 连接到新链表的末尾 
			  tail.next = new_node 
			  # 更新 tail 指针 
			  tail = tail.next 
		  # 返回新链表的真正头节点 
		  return dummy.next
		  
		  ```