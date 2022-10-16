### itoa
```
const (
    x = iota
    _
    y
    z = "zz"
    k
    p = iota
)

func main() {
    // 0 2 zz zz 5
    fmt.Println(x, y, z, k, p)
}
```

### defer
```
func test(x int) {
    defer fmt.Println("a")
    defer fmt.Println("b")

    defer func() {
        fmt.Println(100 / x) // div0 异常未被捕获，逐步往外传递，最后终止进程。
    }()

    defer fmt.Println("c")
}

func main() {
    test(0)
}

// c
// b
// a
// panic: runtime error: integer divide by zero
```
```
// i统一为2

// 输出2
func defer1(i int) int {
	t := i
	defer func() {
		t += i
	}()
	return t
}

// 输出2
func defer2(i int) (t int) {
	defer func() {
		t += i
	}()
	return t
}

// 输出3
func defer3(i int) (t int) {
	defer func() {
		t += i
	}()
	return 1
}

// 输出2
// 输出3
func defer4(i int) (t int) {
	defer func(i int) {
		fmt.Println(i)
		fmt.Println(t)
	}(i)
	i++
	return i
}
```

### slice
```
func main() {
    sli := [11]int{0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 0}
    // s1 := sli
    // s1 := sli[:]
    s1 := sli[2:10]
    fmt.Println(s1)
    fmt.Println(len(s1))
    fmt.Println(cap(s1))
    s1[1] = 123
    s1 = append(s1, 20)
    fmt.Println(s1)
    fmt.Println(len(s1))
    fmt.Println(cap(s1))
    fmt.Println(sli)
    fmt.Println(len(sli))
    fmt.Println(cap(sli))
}

// [4 6 8 10 12 14 16 18]
// 8
// 9
// [4 123 8 10 12 14 16 18 20]
// 9
// 9
// [0 2 4 123 8 10 12 14 16 18 20]
// 11
// 11
```
```
func main() {
    sli := [11]int{0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 0}
    // s1 := sli
    s1 := sli[:]
    // s1 := sli[2:10]
    fmt.Println(s1)
    fmt.Println(len(s1))
    fmt.Println(cap(s1))
    s1[1] = 123
    s1 = append(s1, 20)
    fmt.Println(s1)
    fmt.Println(len(s1))
    fmt.Println(cap(s1))
    fmt.Println(sli)
    fmt.Println(len(sli))
    fmt.Println(cap(sli))
}

// [0 2 4 6 8 10 12 14 16 18 0]
// 11
// 11
// [0 123 4 6 8 10 12 14 16 18 0 20]
// 12
// 22
// [0 123 4 6 8 10 12 14 16 18 0]
// 11
// 11
```
```
func main() {
    s := make([]int, 2)
    s = append(s, 1, 2, 3)
    fmt.Println(s[0])
    fmt.Println(s[1])
    fmt.Println(s[2])
    fmt.Println(s[3])
    fmt.Println(s[4])
    fmt.Println(len(s))
    fmt.Println(cap(s))
}

// 0
// 0
// 1
// 2
// 3
// 5
// 6
```