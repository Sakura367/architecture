```
redis数据结构
① string：sds / long（maxSize：512M）
② list：ziplist（数据少）/ list（数据多）【3.2版本前】/ quicklist【3.2版本后】
③ hash：ziplist（数据少）/ dict（数据多）
④ set：intset（数据少）/ dict（数据多）
⑤ sortedset：ziplist（数据少）/ zset (dict + skiplist)（数据多）
```

## redis数据结构
### String
### List
### Hash
### Set
### Sorted Set

## 底层数据结构
### sds
#### 定义
```
// 3.2
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
- len：记录当前已使用的字节数（不包括'\0'），获取SDS长度的复杂度为O(1)
- alloc：记录当前字节数组总共分配的字节数量（不包括'\0'）
- flags：标记当前字节数组的属性，是sdshdr8还是sdshdr16等，flags值的定义可以看下面代码
- buf：字节数组，用于保存字符串，包括结尾空白字符'\0'
```
// flags值定义
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```
#### C字符串与SDS区别
| C字符串 | SDS |
|  :---:  | :---:  |
| 获取字符串长度复杂度为O(N) | 获取字符串长度复杂度为O(1) |
| API是不安全的，可能会造成缓冲区溢出 | API是安全的，不会造成缓冲区溢出 |
| 修改字符串长度必然会需要执行内存重分配 | 修改字符串长度N次最多会需要执行N次内存重分配 |
| 只能保存文本数据 | 可以保存文本或二进制数据 |

### list
#### 定义
```
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode;

typedef struct list {
    // 链表头节点
    listNode *head;
    // 链表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;
} list;
```
每个节点listNode可以通过prev和next指针分布指向前一个节点和后一个节点组成双端链表，同时每个链表还会有一个list结构为链表提供表头指针head、表尾指针tail、以及链表长度计数器len，还有三个用于实现多态链表的类型特定函数
- dup：用于复制链表节点所保存的值
- free：用于释放链表节点所保存的值
- match：用于对比链表节点所保存的值和另一个输入值是否相等
#### 特性
- 双端链表：带有指向前置节点和后置节点的指针，获取这两个节点的复杂度为O(1)
- 无环：表头节点的prev和表尾节点的next都指向NULL，对链表的访问以NULL结束
- 链表长度计数器：带有len属性，获取链表长度的复杂度为O(1)
- 多态：链表节点使用 void*指针保存节点值，可以保存不同类型的值

### dict
#### 定义
```
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    // 指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;

typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值，等于size-1
    unsigned long sizemask;
    // 哈希表已有节点的数量
    unsigned long used;
} dictht;

typedef struct dictType {
    // 计算哈希值的行数
    uint64_t (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

typedef struct dict {
    // 和类型相关的处理函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash 索引，当rehash不再进行时，值为-1
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    // 迭代器数量
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
- dict的ht属性是两个元素的数组，包含两个dictht哈希表，一般字典只使用ht[0]哈希表，ht[1]哈希表会在对ht[0]哈希表进行rehash（重哈希）的时候使用，即当哈希表的键值对数量超过负载数量过多的时候，会将键值对迁移到ht[1]上
- rehashidx也是跟rehash相关的，rehash的操作不是瞬间完成的，rehashidx记录着rehash的进度，如果目前没有在进行rehash，它的值为-1

### inset
#### 定义
```
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```
- contents数组：整数集合的每个元素在数组中按值的大小从小到大排序，且不包含重复项
- length记录整数集合的元素数量，即contents数组长度
- encoding决定contents数组的真正类型，如INTSET_ENC_INT16、INTSET_ENC_INT32、INTSET_ENC_INT64
#### 元素编码升级
当想要添加一个新元素到整数集合中时，并且新元素的类型比整数集合现有的所有元素的类型都要长，整数集合需要先进行升级（upgrade），才能将新元素添加到整数集合里面。每次想整数集合中添加新元素都有可能会引起升级，每次升级都需要对底层数组已有的所有元素进行类型转换

升级添加新元素：
- 根据新元素类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
- 把数组现有的元素都转换成新元素的类型，并将转换后的元素放到正确的位置，且要保持数组的有序性
- 添加新元素到底层数组

整数集合的升级策略可以提升整数集合的灵活性，并尽可能的节约内存
另外，整数集合不支持降级，一旦升级，编码就会一直保持升级后的状态

### skiplist
#### 定义
```
typedef struct zskiplistNode {
    // 成员对象 （robj *obj;）
    sds ele;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        // 跨度实际上是用来计算元素排名(rank)的，在查找某个节点的过程中，将沿途访过的所有层的跨度累积起来，得到的结果就是目标节点在跳跃表中的排位
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```
- zskiplistNode结构
    - level数组（层）：每次创建一个新的跳表节点都会根据幂次定律计算出level数组的大小，也就是次层的高度，每一层带有两个属性-前进指针和跨度，前进指针用于访问表尾方向的其他指针；跨度用于记录当前节点与前进指针所指节点的距离（指向的为NULL，阔度为0）
    - backward（后退指针）：指向当前节点的前一个节点
    - score（分值）：用来排序，如果分值相同看成员变量在字典序大小排序
    - obj或ele：成员对象是一个指针，指向一个字符串对象，里面保存着一个sds；在跳表中各个节点的成员对象必须唯一，分值可以相同
- zskiplist结构
    - header、tail表头节点和表尾节点
    - length表中节点的数量
    - level表中层数最大的节点的层数

### ziplist
压缩列表（ziplist）是为了节约内存而设计的，是由一系列特殊编码的连续内存块组成的顺序性（sequential）数据结构，一个压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者一个整数值

压缩列表是列表（List）和散列（Hash）的底层实现之一，一个列表只包含少量列表项，并且每个列表项是小整数值或比较短的字符串，会使用压缩列表作为底层实现（在3.2版本之后是使用quicklist实现）

**redis没有提供一个结构体来保存压缩列表的信息，而是提供了一组宏来定位每个成员的地址**

#### 定义
##### ziplist构成
一个ziplist可以包含多个节点（entry），每个节点可以保存一个字节数组或者一个整数值
- zlbytes：记录整个压缩列表占用的内存字节数，在压缩列表内存重分配，或者计算zlend的位置时使用
- zltail：记录压缩列表表尾节点距离压缩列表的起始地址有多少字节，通过该偏移量，可以不用遍历整个压缩列表就可以确定表尾节点的地址
- zllen：记录压缩列表包含的节点数量，但该属性值小于UINT16_MAX（65535）时，该值就是压缩列表的节点数量，否则需要遍历整个压缩列表才能计算出真实的节点数量
- entryX：压缩列表的节点
- zlend：特殊值0xFF（十进制255），用于标记压缩列表的末端

##### ziplist节点构成
每个ziplist节点可以保存一个字节数字或者一个整数值
- previous_entry_ength：记录压缩列表前一个字节的长度
- encoding：节点的encoding保存的是节点的content的内容类型
- content：content区域用于保存节点的内容，节点内容类型和长度由encoding决定

### quicklist
quicklist结构是在redis 3.2版本中新加的数据结构，由ziplist组成的双向链表，用在列表的底层实现。

#### ziplist问题
- ziplist结构本身就是一个连续的内存块，由表头、若干个entry节点和压缩列表尾部标识符zlend组成，通过一系列编码规则，提高内存的利用率，使用于存储整数和短字符串。
- ziplist结构的缺点是：每次插入或删除一个元素时，都需要进行频繁的调用realloc()函数进行内存的扩展或减小，然后进行数据”搬移”，甚至可能引发连锁更新，造成严重效率的损失。

#### quicklist与ziplist
quicklist是由ziplist组成的双向链表，链表中的每一个节点都以压缩列表ziplist的结构保存着数据，而ziplist有多个entry节点，保存着数据。相当与一个quicklist节点保存的是一片数据，而不再是一个数据。

- quicklist宏观上是一个双向链表，因此，它具有一个双向链表的有点，进行插入或删除操作时非常方便，虽然复杂度为O(n)，但是不需要内存的复制，提高了效率，而且访问两端元素复杂度为O(1)。
- quicklist微观上是一片片entry节点，每一片entry节点内存连续且顺序存储，可以通过二分查找以 log2(n)log2(n) 的复杂度进行定位。

#### 定义
```
typedef struct quicklistNode {
    struct quicklistNode *prev;     //前驱节点指针
    struct quicklistNode *next;     //后继节点指针
 
    //不设置压缩数据参数recompress时指向一个ziplist结构
    //设置压缩数据参数recompress指向quicklistLZF结构
    unsigned char *zl;
 
    //压缩列表ziplist的总长度
    unsigned int sz;                  /* ziplist size in bytes */
 
    //ziplist中包的节点数，占16 bits长度
    unsigned int count : 16;          /* count of items in ziplist */
 
    //表示是否采用了LZF压缩算法压缩quicklist节点，1表示压缩过，2表示没压缩，占2 bits长度
    unsigned int encoding : 2;        /* RAW==1 or LZF==2 */
 
    //表示一个quicklistNode节点是否采用ziplist结构保存数据，2表示压缩了，1表示没压缩，默认是2，占2bits长度
    unsigned int container : 2;       /* NONE==1 or ZIPLIST==2 */
 
    //标记quicklist节点的ziplist之前是否被解压缩过，占1bit长度
    //如果recompress为1，则等待被再次压缩
    unsigned int recompress : 1; /* was this node previous compressed? */
 
    //测试时使用
    unsigned int attempted_compress : 1; /* node can't compress; too small */
 
    //额外扩展位，占10bits长度
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

typedef struct quicklist {
    //指向头部(最左边)quicklist节点的指针
    quicklistNode *head;
 
    //指向尾部(最右边)quicklist节点的指针
    quicklistNode *tail;
 
    //ziplist中的entry节点计数器
    unsigned long count;        /* total count of all entries in all ziplists */
 
    //quicklist的quicklistNode节点计数器
    unsigned int len;           /* number of quicklistNodes */
 
    //保存ziplist的大小，配置文件设定，占16bits
    int fill : 16;              /* fill factor for individual nodes */
 
    //保存压缩程度值，配置文件设定，占16bits，0表示不压缩
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```