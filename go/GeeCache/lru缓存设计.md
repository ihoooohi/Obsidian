## 一、设计原理

**哈希链表**（在go中是map + list）

lru缓存要实现fifo用链表比较合适，用数组的话动一次还要整体迁移，然后因为有一个要把访问过的内容优先级提高，单纯链表的查询效率太低，就加上哈希表map提高查询复杂度为o1。
（**用 Map 弥补链表“找人难”的缺点，用链表弥补数组“挪人难”的缺点**）
## 二、go中的list

go中的list是一个循环双向链表，来自`container/list` 标准库

由 **List** 和 **Element** 两个结构体构成，其结构分别如下

```go
type List struct {
	root *Element
	len int
}
```

```go
type Element struct {
	prev, next *Element
	list *List
	value any
}
```

## 三、Cache结构体设计

首先根据哈希链表的结构，cache一定会有一个map `cache map[string]*list.Element` 和一个list `ll *list.List`

然后要定义最大内存 `maxbytes int64` 和已使用的内存 `nbytes int64` 。之所以用int64是因为int在32位系统中是32位，64位系统中是64。如果只是int32的话最大cache只能约是2GB，其实已经很大了，但为了安全和保险用大点的int64

以及一个删除节点触发的回调函数 `OnEvicted`

总体如下：
```go
type Cache struct {
	maxbytes int64
	nbytes int64
	cache map[key]*list.Element
	ll list.List
	OnEvicted func(key string, val CacheValue) 
}
```
 
其中因为链表中的节点的值有两个，一个是key一个是真正的值，所以要再定义一个结构体entry（条目）

```go
type entry struct {
	key string
	val CacheValue
}
```

其中条目的值是它占的内存大小，这个用一个接口来实现，接口中的方法就是计算它的内存大小

```go
type CacheValue interface {
	Len() int
}
```

定义一个函数来新建一个Cache

```go
func (c *Cache) New(maxbytes int64, OnEvicted func(key stirng, val CacheValue)) *Cache {
	return &Cache{
		maxbytes: maxbytes,
		cache: make(map[string]*list.Element),
		ll: list.New(),
		OnEvicted: OnEvicted,
	}
}
```

## 四、增删改查方法的实现

### 1.增加&修改（Add）

为什么增加和修改是一个函数呢？因为传进一个键值对要先检查下这个key是不是已经存在，如果存在就是修改，不是就是新增。

**修改**需要改变：
1. 节点的值
2. 已使用的内存nbytes
3. 调整节点的位置

**增加**需要改变：
1. 队尾插入一个节点
2. 已使用的内存nbytes
3. 新增一个map的pair

函数最后需要改变：
1. 检查是否越界，若越界移除最老的节点（用for是因为，只移除一个不一定满足条件）

```go
func (c *Cache) Add(key string, val CacheValue) {
	if ele, ok := c.cache[key]; ok{
		kv := ele.value.(*entry)
		c.nbytes += int64(val.Len()) - int64(kv.val.Len())
		kv.val = val
		c.ll.MoveToFront(ele)
	} else {
		e := &entry{
			key: key,
			val: val,
		}
		ele := c.ll.PushFront(e)
		c.nbytes += int64(kv.val.Len())
		c.cache[key] = ele
	}
	for maxbytes != 0 && c.nbytes > c.maxbytes {
		c.RemoveOldest()
	}
}
```

### 2.删除（RemoveOldest）

lru的核心函数，淘汰最不常用的节点（队头）

淘汰需要改变：
1. 删除链表的队头节点
2. 删除map中的pair
3. 改变nbytes
4. 触发回调函数OnEvicted

```go
func (c *Cache) RemoveOldest() {
	if ele, ok := c.ll.Back(); ok {
		c.ll.Remove(ele)
		kv := ele.Value.(*entry)
		delete(c.cache, kv.key)
		c.nbytes -= int64(len(kv.key)) + int64(kv.val.Len())
		if OnEvicted != nil {
			OnEvicted(kv.key, kv.val)
		}
	}
}
```

### 3.查找（Get）

通过输入参数 `key string` 返回 所使用的内存 `val CacheValue` 

不要忘记要将访问的节点移到队尾（最新）

```go
func (c *Cache) Get(key string) (val CacheValue, ok bool) {
	if ele, ok := c.cache[key]; ok{
		c.ll.MoveToFront(ele)
		kv := ele.Value.(*entry)
		return kv.val, true
	}else{
		return 	
	}
}
```


## 五、⭐️存储的缓存值的类型设计

首先我先从上到下列出所有有关存储值的类型：
1. `maxbytes` 和 `nbytes` ：int64
2. struct `entry` 中的 `val`：CacheValue
3. interface `CacheValue` ：Len() int
4. `byteview` : []byte (type byte = uint8)

为什么 `maxbytes` 和 `nbytes` 是int64不是int？是因为int在32位系统中是int32（约2GB）不一定够用，int64就足够大了

为什么 struct `entry` 中的 `val` 的值是个接口不是具体的int或者byte而是CacheValue接口？因为这样可以保持可拓展性，不然像图片或者json格式的数据没法直接传

为什么CacheValue的Len()方法返回的是int不是跟nbytes一样的int64？因为标准里的len()返回的是int类型，所以要用int

最后注意存入值后更新nbytes是要用显式类型转换把int转成int64，如  `c.nbytes -= int64(len(kv.key)) + int64(kv.val.Len())`