## sync.Map
```
type Map struct {
	mu Mutex // 加锁
	read atomic.Value // readOnly，只读
	dirty map[interface{}]*entry // 包含新写入的数据，misses计数到伐值则拷贝到read
	misses int // 读失败计数
}
```

## sync.Once
```
type Once struct {
    //done变量用来标识函数是否执行完毕
    done uint32
        //m用来保证函数执行期间，其他goroutine阻塞
    m    Mutex
}

func (o *Once) Do(f func()) {
    //1.判断函数是否执行过，done=0说明没执行过
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}
 
 
func (o *Once) doSlow(f func()) {
    //2.加锁
    o.m.Lock()
    //6.函数返回后，释放锁
    defer o.m.Unlock()
    //3.判断是否执行过
    if o.done == 0 {
        //5函数返回后，设置执行标识
        defer atomic.StoreUint32(&o.done, 1)
        //4具体执行函数
        f()
    }
}
```

## sync.Mutex
```
type Mutex struct {
    state int32
    sema  uint32
}
```
- Mutex.state
  - waiter_num：等待goroutine
  - starving：是否处于饥饿状态
  - woken：是否有goroutine正在加锁
  - locked：是否有goroutine持有该锁
- Mutex.sema
  - 信号量，用于唤醒阻塞的goroutine

#### 锁模式
- 正常模式
1. 当前的mutex只有一个goruntine来获取，那么没有竞争，直接返回。
2. 新的goruntine进来，如果当前mutex已经被获取了，则该goruntine进入一个先入先出的waiter队列，在mutex被释放后，waiter按照先进先出的方式获取锁。该goruntine会处于自旋状态(不挂起，继续占有cpu)。
3. 新的goruntine进来，mutex处于空闲状态，将参与竞争。新来的 goroutine 有先天的优势，它们正在 CPU 中运行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。

- 饥饿模式
1. 在饥饿模式下，Mutex 的拥有者将直接把锁交给队列最前面的 waiter。
2. 新来的 goroutine 不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会 spin（自旋），它会乖乖地加入到等待队列的尾部。

饥饿模式的触发条件：
1. 有一个goroutine获取锁的时间超过1ms

饥饿模式的取消条件：
1. 获取到锁的goroutine为阻塞队列的最后一个时，恢复正常模式
2. 获取到锁的goroutine等待时间小于1ms，恢复正常模式

**注意点**
- 不可重入，重复Mutex.Lock会panic
- 先调用Mutex.Unlock会panic
- 同一把锁，一个goroutine Mutex.Lock，另一个goroutine可以Mutex.Unlock

## sync.RWMutex
```
type RWMutex struct {
    w           Mutex  // 保证只会有一个写锁加锁成功
    writerSem   uint32 // 用于writer等待读完成排队的信号量
    readerSem   uint32 // 用于reader等待写完成排队的信号量
    readerCount int32  // 读操作goroutine数量
    readerWait  int32  // 阻塞写操作goroutine的读操作goroutine数量
}
```
- 加读锁
  - readerCount > 0，说明存在读锁，则加锁成功
  - readerCount < 0，说明存在写锁，则阻塞，等待readerSem信号量唤醒
- 释放读锁
  - readerCount - 1，若readerCount<0，说明有写锁等待
  - readerWait - 1，若readerWait == 0，说明最后一个解读锁了，则唤起写锁信号量（释放全部读锁后，唤醒写锁）
- 加写锁
  - m.lock保证读锁间互斥
  - readerCount - rwmutexMaxReaders ，阻塞后面的读锁再加锁
  - readerWait>0，说明存在读锁，则阻塞，等待写锁信号量唤醒
- 释放写锁
  - readerCount + rwmutexMaxReaders，readerCount复位
  - 读锁信号量唤醒所有读锁

**总结**
- 写锁通过递减rwmutexMaxReaders常量，使readerCount < 0，实现对读锁的抢占
- atomic.AddInt32操作是通过LOCK来进行CPU总线加锁的
- m.lock保证写锁之间的公平
- 先入先出（FIFO）的原则进行加锁，实现公平读写锁，解决线程饥饿问题