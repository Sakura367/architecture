### Go实现
```
type LRUNode struct {
	key  string
	val  string
	prev *LRUNode
	next *LRUNode
}

func InitLRUNode(k string, v string) *LRUNode {
	node := new(LRUNode)
	node.key = k
	node.val = v
	return node
}

type LRUList struct {
	size int
	head *LRUNode
	tail *LRUNode
}

func InitLRUList() *LRUList {
	list := new(LRUList)
	list.head = new(LRUNode)
	list.tail = new(LRUNode)
	list.head.next = list.tail
	list.tail.prev = list.head
	list.size = 0
	return list
}

func (list *LRUList) addLast(node *LRUNode) {
	node.prev = list.tail.prev
	node.next = list.tail
	list.tail.prev.next = node
	list.tail.prev = node
	list.size++
}

func (list *LRUList) remove(node *LRUNode) {
	node.prev.next = node.next
	node.next.prev = node.prev
	list.size--
}

func (list *LRUList) removeFirst() *LRUNode {
	if list.head.next == list.tail {
		return nil
	}
	node := list.head.next
	list.remove(node)
	return node
}

func (list *LRUList) Size() int {
	return list.size
}

type LRUCache struct {
	data     *LRUList
	hash     map[string]*LRUNode
	capacity int
}

func InitLRUCache(c int) *LRUCache {
	lru := new(LRUCache)
	list := InitLRUList()
	lru.data = list
	lru.hash = make(map[string]*LRUNode)
	lru.capacity = c
	return lru
}

func (lru *LRUCache) refresh(k string) {
	node, _ := lru.hash[k]
	lru.data.remove(node)
	lru.data.addLast(node)
}

func (lru *LRUCache) add(k string, v string) {
	node := InitLRUNode(k, v)
	lru.data.addLast(node)
	lru.hash[k] = node
}

func (lru *LRUCache) remove(k string) {
	node, ok := lru.hash[k]
	if !ok {
		return
	}
	lru.data.remove(node)
	delete(lru.hash, k)
}

func (lru *LRUCache) removeFirst() {
	node := lru.data.removeFirst()
	delete(lru.hash, node.key)
}

func (lru *LRUCache) Get(k string) *LRUNode {
	if _, ok := lru.hash[k]; !ok {
		return nil
	}
	lru.refresh(k)
	return lru.hash[k]
}

func (lru *LRUCache) Put(k string, v string) {
	if _, ok := lru.hash[k]; ok {
		lru.remove(k)
		lru.add(k, v)
		return
	}
	if lru.capacity == lru.data.Size() {
		lru.removeFirst()
	}
	lru.add(k, v)
}

func Test_LRU(t *testing.T) {
	lruCache := InitLRUCache(4)
	lruCache.Put("001", "用户１信息")
	lruCache.Put("002", "用户１信息")
	lruCache.Put("003", "用户１信息")
	lruCache.Put("004", "用户１信息")
	lruCache.Put("005", "用户１信息")
	lruCache.Put("002", "醉卧沙场")
	lruCache.Put("004", "君莫笑")
	lruCache.Put("006", "用户6信息更新")
	fmt.Println(lruCache.Get("003"))
	lruCache.Put("007", "用户7信息更新")
	fmt.Println(lruCache.Get("002"))
	fmt.Println(lruCache.Get("004"))
	lruCache.Put("008", "用户8信息更新")
	fmt.Println("success")
}
```

### C++实现
```
struct CacheNode {
    int key;
    int val;
    CacheNode *pre, *next;
    CacheNode(int k, int v) : key(k), val(v), pre(nullptr), next(nullptr) {}
}

class LRUCache {
    int size;
    CacheNode *head, *tail;
    map<int, CacheNode*> mp;
    
    public LRUCache(int cap) : size(cap), head(nullptr), tail(nullptr) {}
    
    public void remove(CacheNode* node) {
        if (node->pre != nullptr) {
            node->pre->next = node->next;
        } else {
            head = node->next;
        }
        if (node->next != nullptr) {
            node->next->pre = node->pre;
        } else {
            tail = node->pre;
        }
        size--;
    }
    
    public void setHead(CacheNode* node) {
        node->next = head;
        node->pre = nullptr;
        if (head != nullptr) {
            head->pre = node;
        }
        head = node;
        if (tail == nullptr) {
            tail = head;
        }
        size++;
    }
    
    public void set(int key, int val) {
        map<int, CacheNode*>::iterator it = mp.find(key);
        if (it != mp.end()) {
            CacheNode* node = it->second;
            node->val = val;
            remove(node);
            setHead(node);
        } else {
            CacheNode* node = new CacheNode(key, val);
            if (mp.size() >= size) {
                map<int, CacheNode*>::iterator it = mp.find(tail->key);
                remove(tail);
                mp.erase(it);
            }
            setHead(node);
            mp[key] = node;
        }
    }
    
    public int get(int key) {
        map<int, CacheNode*>::iterator it = mp.find(key);
        if (it != mp.end()) {
            CacheNode* node = it->second;
            remove(node);
            setHead(node);
            return node->val;
        } else {
            return -1;
        }
    }
}
```