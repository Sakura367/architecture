### 数据结构
map的底层结构是hmap（即hashmap的缩写），核心元素是一个由若干个桶（bucket，结构为bmap）组成的数组，每个bucket可以存放若干元素（通常是8个），key通过哈希算法被归入不同的bucket中。当超过8个元素需要存入某个bucket时，hmap会使用extra中的overflow来拓展该bucket。

hmap的结构体如下：
```
type hmap struct {
    count     int                  // 元素个数
    flags     uint8
    B         uint8                // 扩容常量相关字段B是buckets数组的长度的对数 2^B
    noverflow uint16               // 溢出的bucket个数
    hash0     uint32               // hash seed
    buckets    unsafe.Pointer      // buckets 数组指针
    oldbuckets unsafe.Pointer      // 结构扩容的时候用于赋值的buckets数组
    nevacuate  uintptr             // 搬迁进度
    extra *mapextra                // 用于扩容的指针
}
```
bucket（bmap）的结构如下
```
type bmap struct {
    tophash [bucketCnt]uint8
}
```
- tophash用于记录8个key哈希值的高8位，这样在寻找对应key的时候可以更快，不必每次都对key做全等判断。
- bucket并非只有一个tophash，而是后面紧跟8组kv对和一个overflow的指针，这样才能使overflow成为一个链表的结构。但是这两个结构体并不是显示定义的，而是直接通过指针运算进行访问的。
- kv的存储形式为key0key1key2key3…key7val1val2val3…val7，这样做的好处是：在key和value的长度不同的时候，节省padding空间。如上面的例子，在map[int64]int8中，4个相邻的int8可以存储在同一个内存单元中。如果使用kv交错存储的话，每个int8都会被padding占用单独的内存单元（为了提高寻址速度）。

### 增加元素
1. 对k进行hash，
2. hash值的低8位和bucket数组长度取余，定位到bucket数组的下标
3. k的hash值高8位和bucket的tophash对比，判断k是否已经存在
4. 将kv存储到该bucket中，若bucket满了，新建一个新的bucket，并用overflow指向新的bucket。

### 删除元素
1. 如果删除的元素是值类型，如int，float，bool，string以及数组和struct，map的内存不会自动释放
2. 如果删除的元素是引用类型，如指针，slice，map，chan等，map的内存会自动释放，但释放的内存是子元素应用类型的内存占用
3. 将map设置为nil后，内存被回收

### 扩容
随着元素的增加，在一个bucket链中寻找特定的key会变得效率低下，所以在插入的元素个数/bucket个数达到某个阈值（当前设置为6.5，实验得来的值）时，map会进行扩容，代码中详见 hashGrow函数。首先创建bucket数组，长度为原长度的两倍，然后替换原有的bucket，原有的bucket被移动到oldbucket指针下。

扩容完成后，每个hash对应两个bucket（一个新的一个旧的）。oldbucket不会立即被转移到新的bucket下，而是当访问到该bucket时，会调用growWork方法进行迁移，growWork方法会将oldbucket下的元素rehash到新的bucket中。随着访问的进行，所有oldbucket会被逐渐移动到bucket中。

但是这里有个问题：如果需要进行扩容的时候，上一次扩容后的迁移还没结束，怎么办？在代码中我们可以看到很多again标记，会不断进行迁移，知道迁移完成后才会进行下一次扩容。

但这个迁移并没有在扩容之后一次性完成，而是逐步完成的，每一次insert或remove时迁移1到2个pair，即增量扩容。增量扩容的原因主要是缩短map容器的响应时间。若hashmap很大扩容时很容易导致系统停顿无响应。增量扩容本质上就是将总的扩容时间分摊到了每一次hash操作上。由于这个工作是逐渐完成的，导致数据一部分在old table中一部分在new table中。old的bucket不会删除，只是加上一个已删除的标记。只有当所有的bucket都从old table里迁移后才会将其释放掉。

**for range map不是有序的**
1. map 在扩容后，会发生 key 的搬迁
2. 每次从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历