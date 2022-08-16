### Go实现
```
type LFUNode struct {
	key  string
	val  string
	freq int
	prev *LFUNode
	next *LFUNode
}

func InitLFUNode(k string, v string, freq int) *LFUNode {
	node := new(LFUNode)
	node.key = k
	node.val = v
	node.freq = freq
	return node
}

type LFUList struct {
	head *LFUNode
	tail *LFUNode
	size int
}

func InitLFUList() *LFUList {
	list := new(LFUList)
	list.head = new(LFUNode)
	list.tail = new(LFUNode)
	list.head.next = list.tail
	list.tail.prev = list.head
	list.size = 0
	return list
}

func (list *LFUList) addLast(node *LFUNode) {
	node.prev = list.tail.prev
	node.next = list.tail
	list.tail.prev.next = node
	list.tail.prev = node
	list.size++
}

func (list *LFUList) remove(node *LFUNode) {
	node.prev.next = node.next
	node.next.prev = node.prev
	list.size--
}

func (list *LFUList) removeFirst() *LFUNode {
	if list.head.next == list.tail {
		return nil
	}
	node := list.head.next
	list.remove(node)
	return node
}

func (list *LFUList) Size() int {
	return list.size
}

type LFUCache struct {
	keyMap      map[string]*LFUNode
	freqListMap map[int]*LFUList
	capacity    int
	minFreq     int
}

func InitLFUCache(capacity int) *LFUCache {
	lfu := new(LFUCache)
	lfu.keyMap = make(map[string]*LFUNode)
	lfu.freqListMap = make(map[int]*LFUList)
	lfu.capacity = capacity
	return lfu
}

func (lfu *LFUCache) addNode(k string, v string, freq int) {
	node := InitLFUNode(k, v, freq)
	lfu.keyMap[k] = node
	if _, ok := lfu.freqListMap[freq]; !ok {
		lfu.freqListMap[freq] = InitLFUList()
	}
	lfu.freqListMap[freq].addLast(node)
}

func (lfu *LFUCache) increaseFreq(k string) {
	node, _ := lfu.keyMap[k]
	list, _ := lfu.freqListMap[node.freq]
	list.remove(node)
	if list.Size() == 0 {
		delete(lfu.freqListMap, node.freq)
		if node.freq == lfu.minFreq {
			lfu.minFreq += 1
		}
	}
	lfu.addNode(node.key, node.val, node.freq+1)
}

func (lfu *LFUCache) removeMinFreq() {
	list, _ := lfu.freqListMap[lfu.minFreq]
	node := list.removeFirst()
	delete(lfu.keyMap, node.key)
	if list.Size() == 0 {
		delete(lfu.freqListMap, lfu.minFreq)
	}
}

func (lfu *LFUCache) Get(k string) *LFUNode {
	if _, ok := lfu.keyMap[k]; !ok {
		return nil
	}
	lfu.increaseFreq(k)
	return lfu.keyMap[k]
}

func (lfu *LFUCache) Put(k string, v string) {
	if _, ok := lfu.keyMap[k]; ok {
		lfu.keyMap[k].val = v
		lfu.increaseFreq(k)
		return
	}
	if lfu.capacity == len(lfu.keyMap) {
		lfu.removeMinFreq()
	}
	lfu.addNode(k, v, 1)
	lfu.minFreq = 1
}

func Test_LFU(t *testing.T) {
	lfuCache := InitLFUCache(4)
	lfuCache.Put("001", "用户１信息")
	lfuCache.Put("002", "用户１信息")
	lfuCache.Put("003", "用户１信息")
	lfuCache.Put("004", "用户１信息")
	lfuCache.Put("005", "用户１信息")
	lfuCache.Put("002", "醉卧沙场")
	lfuCache.Put("004", "君莫笑")
	lfuCache.Put("006", "用户6信息更新")
	fmt.Println(lfuCache.Get("003"))
	fmt.Println(lfuCache.Get("005"))
	fmt.Println(lfuCache.Get("006"))
	lfuCache.Put("008", "用户8信息更新")
	fmt.Println("success")
}
```