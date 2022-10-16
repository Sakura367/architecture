Go 语言中的 select语句用于监控channel中的数据流动并选择一组case语句执行相应的代码。它看起来类似于switch语句，但是select语句中所有case中的表达式都必须是channel的发送或接收操作。一个典型的select使用示例如下：
```
select {
case <-ch1:
	fmt.Println("ch1")
case ch2 <- 1:
	fmt.Println("ch2")
}
```

#### 特性
select 不存在任何的 case：永久阻塞当前 goroutine
select 只存在一个 case：阻塞的发送/接收
select 存在多个 case：随机选择一个满足条件的case执行
select 存在 default，其他case都不满足时：执行default语句中的代码

#### 实现优先级
```
func worker2(ch1, ch2 <-chan int, stopCh chan struct{}) {
	for {
		select {
		case <-stopCh:
			return
		case job1 := <-ch1:
			fmt.Println(job1)
		case job2 := <-ch2:
		priority:
			for {
				select {
				case job1 := <-ch1:
					fmt.Println(job1)
				default:
					break priority
				}
			}
			fmt.Println(job2)
		}
	}
}
```
不仅使用了嵌套的select，还组合使用了for循环和LABEL来实现题目的要求。上面的代码在外层select选中执行job2 := <-ch2时，进入到内层select循环继续尝试执行job1 := <-ch1,当ch1就绪时就会一直执行，否则跳出内层select。

## 原理
golang实现select的时候，实际上为每一个case语句定义了一个数据结构，select语句块执行的时候，实际上可以类比成对一个case数组处理的代码块（或者函数），然后程序流程转到选中的case块。
```
type scase struct {
	c           *hchan         // chan
	kind        uint16
	elem        unsafe.Pointer // data element
}
```
- scase.c表示当前case语句操作的chan指针，这也表明一个case只能监听一个chan。
- scase.kind表示当前的chan是可读还是可写channel或者是default。三种类型分别由常量定义：
	- caseRecv：case语句中尝试读取scase.c中的数据；
	- caseSend：case语句中尝试向scase.c中写入数据；
	- caseDefault： default语句
- scase.elem表示缓冲区地址，跟据scase.kind不同，有不同的用途：
	- scase.kind == caseRecv ： scase.elem表示读出channel的数据存放地址；
	- scase.kind == caseSend ： scase.elem表示将要写入channel的数据存放地址；

### 执行流程
1. 锁定scase语句中所有的channel
2. 按照随机顺序检测scase中的channel是否ready
2.1 如果case可读，则读取channel中数据，解锁所有的channel，然后返回(case index)
2.2 如果case可写，则将数据写入channel，解锁所有的channel，然后返回(case index)
2.3 所有case都未ready，则解锁所有的channel，然后返回（default index）
3. 所有case都未ready，且没有default语句
3.1 将当前协程加入到所有channel的等待队列
3.2 当将协程转入阻塞，等待被唤醒
4. 唤醒后返回channel对应的case index
4.1 如果是读操作，解锁所有的channel，然后返回(case index)
4.2 如果是写操作，解锁所有的channel，然后返回(case index)