前序遍历
```python
	def preorderTraversal(self, root: Optional[TreeNode]) -> List[int]:
		result = []
		def dfs(root: Optional[TreeNode]):
			if root is None:
				return
			result.append(root.val)
			dfs(root.left)
			dfs(root.right)
		dfs(root)
		return result
```
二叉树的最大深度
```python（分解问题）
def maxDepth(self, root):
	if not root:
		return 0
	leftmax = self.maxDepth(root.left)
	rightmax = self.maxDepth(root.right)
	return 1 + max(leftmax, rightmax)
```