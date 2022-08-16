```
channel OR mutex

关注数据流动，考虑使用channel解决
数据不流动，保护数据，使用mutex
```

### 数据结构
```
type hchan struct {
	qcount   uint   // channel 里的元素计数
	dataqsiz uint   // 可以缓冲的数量，如 ch := make(chan int, 10)。 此处的 10 即 dataqsiz
	elemsize uint16 // 要发送或接收的数据类型大小
	buf      unsafe.Pointer // 当 channel 设置了缓冲数量时，该 buf 指向一个存储缓冲数据的区域，该区域是一个循环队列的数据结构
	closed   uint32 // 关闭状态
	sendx    uint  // 当 channel 设置了缓冲数量时，数据区域即循环队列此时已发送数据的索引位置
	recvx    uint  // 当 channel 设置了缓冲数量时，数据区域即循环队列此时已接收数据的索引位置
	recvq    waitq // 等待接收的 goroutine 队列
	sendq    waitq // 等待发送的 goroutine 队列

	lock mutex
}
```

### 特性
1. 读写值 nil 管道会永久阻塞
2. 关闭的管道读数据仍然可以读数据
3. 往关闭的管道写数据会 panic
4. 关闭为 nil 的管道 panic
5. 关闭已经关闭的管道 panic

### 发送数据（写channel）
1. 如果等待接收队列 recvq 不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从 recvq 取出 G,并把数据写入，最后把该 G 唤醒，结束发送过程
2. 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程
3. 如果缓冲区中没有空余位置，将待发送数据写入 G，将当前 G 加入 sendq，进入睡眠，等待被读 goroutine 唤醒

### 接收数据（读channel）
1. 如果等待发送队列 sendq 不为空，且没有缓冲区，直接从 sendq 中取出 G，把 G 中数据读出，最后把 G 唤醒，结束读取过程
2. 如果等待发送队列 sendq 不为空，且缓冲区已满，从缓冲区中首部读出数据，把 G 中数据写入缓冲区尾部，把 G 唤醒，结束读取过程
3. 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程
4. 将当前 goroutine 加入 recvq，进入睡眠，等待被写 goroutine 唤醒