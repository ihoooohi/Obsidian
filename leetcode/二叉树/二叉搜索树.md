1. 二叉搜索树（BST）的中序遍历结果是升序的(**从小到大**)
2. 要降序（**从大到小**）排列，则遍历顺序：右子树-根节点-左子树
3. 常见框架
	```python
	def BST(root: TreeNode, target: int) -> None:
    if root.val == target:
        # 找到目标，做点什么
    if root.val < target:
        BST(root.right, target)
    if root.val > target:
	        BST(root.left, target)
	```