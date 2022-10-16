https://segmentfault.com/a/1190000039294140
## 数据结构
```
type Context interface {
    Deadline() (deadline time.Time, ok bool)    // 获取当前context的截止时间
    Done() <-chan struct{}  // 识别channel是否被关闭
    Err() error             // 获取context被关闭的原因
    Value(key interface{}) interface{}  // 获取当前context中所存储的value
}
```
## context的继承
WithCancel：创建一个可以取消的Context
WithDeadline：创建一个到截止日期就取消的Context
WithTimeout：创建一个超时自动取消的Context
WithValue：在Context中设置键值对
### cancelCtx
```
type cancelCtx struct {
    Context
    mu          sync.Mutex      // 并发安全，加互斥锁进行操作
    done        struct{}        // context取消会关闭
    children    map[canceler]struct{}   // 包含context对应的子集，关闭通知所有的子集context
    err         error           // 报错信息
}
```
### timerCtx
```
type timerCtx struct {
    cancelCtx
    timer       *time.Timer
    deadline    time.Time    
}  
```
### valueCtx
```
type valueCtx struct {
    Context
    key, val    interface{}
}
```

## 场景
- 上下文信息传递 （request-scoped），比如处理 http 请求、在请求处理链路上传递信息
- 控制子 goroutine 的运行
- 超时控制的方法调用
- 可以取消的方法调用

**time.Timer与time.Ticker**
https://blog.csdn.net/qq_34417408/article/details/109444880