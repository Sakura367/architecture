```
type _defer struct {
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	openDefer bool
	sp        uintptr  // stack point 栈指针
	pc        uintptr  // process count 程序计数器
	fn        *funcval // can be nil for open-coded defers
	_panic    *_panic  // panic that is running defer
	link      *_defer  // 指向下一个 defer，因为这是一个单链表
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr       
	framepc uintptr
}
```