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