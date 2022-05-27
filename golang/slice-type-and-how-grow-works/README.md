从源码解析 Go 的切片类型以及扩容机制
------------------------------------------

首先我们来看看 Go 官方团队是如何定义切片的，在 [A Tour of Go][1] 中是这样写道：

> An array has a fixed size. A slice, on the other hand, is a dynamically-sized, flexible view into the elements of an array. In practice, slices are much more common than arrays.

简单的说切片就是一个建立在数组之上的动态伸缩、灵活的「视图」。实际项目中，切片会比数组更常用。以下是显式在数组上创建切片和创建切片的切片的方法（实际项目中更常用的应该是通过 `make` 内置函数来创建）：

```go
//
// main.go
//

func main() {
    arr := [...]int{1, 3, 5, 7, 9}

    s1 := arr[:3]
    fmt.Printf("s1 = %v, len = %d, cap = %d\n", s1, len(s1), cap(s1))
    // s1 = [1 3 5], len = 3, cap = 5

    s2 := arr[2:]
    fmt.Printf("s2 = %v, len = %d, cap = %d\n", s2, len(s2), cap(s2))
    // s2 = [5 7 9], len = 3, cap = 3

    s3 := s1[3:]
    fmt.Printf("s3 = %v, len = %d, cap = %d\n", s3, len(s3), cap(s3))
    // s3 = [], len = 0, cap = 2

    s4 := s1[2:3:3]
    fmt.Printf("s4 = %v, len = %d, cap = %d\n", s4, len(s4), cap(s4))
    // s4 = [5], len = 1, cap = 1

    s5 := []int{2, 4, 6, 8}
    fmt.Printf("s5 = %v, len = %d, cap = %d\n", s5, len(s5), cap(s5))
    // s5 = [2 4 6 8], len = 4, cap = 4
}
```

需要说明的是，基于切片的切片和直接创建的切片（`make` 方法）可以看作是底层隐含了一个匿名数组（新建的数组或是引用其他切片的底层数组），这与前面对切片的定义并不相违背。



### 约定

本文基于 `go1.16` 版本的源码进行分析，由于自 `go.17` 之后 Go 使用「基于寄存器的调用约定」替代了「基于堆栈的调用约定」。这个修改让编译后的二进制文件更小同时也带来了些微的性能提升，但是同时稍微增加了分析汇编代码的复杂程度。为了便于分析调用和展示运行时的一些特性，所以这里采用 `go1.16` 版本进行分析。

关于「基于寄存器的调用约定」可以参考以下内容进行了解：

* [Proposal: Register-based Go calling convention][2]
* [Discussion: switch to a register-based calling convention for Go functions][3]
* [src/cmd/compile/ABI-internal.md][4]



### 切片的底层数据结构

在 Go 语言中切片实际上是一个包含三个字段的结构体，该结构体的定义可以在 `runtime/slice.go` 中找到：

```go
//
// runtime/slice.go
//

type slice struct {
    array unsafe.Pointer    // 指向底层数组，是某一块内存的首地址
    len   int               // 切片的长度
    cap   int               // 切片的容量
}
```

当我们需要通过对底层数据结构做一些手脚来达到某些目的时，我们可以使用 `reflect` 包中导出的结构体定义：

```go
//
// reflect/value.go
//

// SliceHeader is the runtime representation of a slice.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

如果感觉文字看起来比较抽象，可以从下面的这张图中去理解（其中 `data` 指向数组的的起始位置）：

<img src="assets/slice-structure.png" alt="Structure of the Slice" style="zoom:50%;" />

#### 切片的技巧与陷阱

如果单纯看内存中的数据只是一串无意义的 0 或者 1 ，而赋予这些数据意义的则是我们的程序如何解释他们。由于切片的 `Data` 字段只是一个指向某一块内存的地址，而我们可以通过一些「危险」的方式赋予内存意义，从而实现一些 Go 语法上不允许的事情。**请注意：在使用这些技巧时，你应该清晰地明白你正在做的事情与相应的副作用**

##### 字符串与字节切片的零拷贝转换

Go 中的字符串其实就是一段固定长度的字节数组，而我们知道切片就是建立在数组之上的视图，所以我们可以这么做：

```go
func String(bs []byte) (s string) {
	hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
	hdr.Data = (*reflect.SliceHeader)(unsafe.Pointer(&bs)).Data
	hdr.Len = (*reflect.SliceHeader)(unsafe.Pointer(&bs)).Len
	return
}

func Bytes(s string) (bs []byte) {
	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	hdr.Data = (*reflect.StringHeader)(unsafe.Pointer(&s)).Data
	hdr.Len = (*reflect.StringHeader)(unsafe.Pointer(&s)).Len
	hdr.Cap = hdr.Len
	return
}
```

##### 一道面试陷阱题

首先我们知道 Go 都是以传值方式调用的（切片、通道的内部实现都是一个结构体），接着我们来看一个陷阱：

```go
func AppendSlice(s []int) {
	s = append(s, 1234)
	fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
	// s = [1234], len = 1, cap = 8
}

func main() {
	s := make([]int, 0, 8)

	AppendSlice(s)
	fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
	// s = [], len = 0, cap = 8
}
```

这里的切片 `s` 的容量足够容纳 `append` 之后的元素，但为什么在 `main` 函数中打印的切片是空的呢？可以自己思考下后再查看答案：

答案：由于切片在运行时世纪上是一个结构体，同时 Go 是通过传值方式进行调用的。所以实际上我们可以换个方式再看这个代码：

```go
type Slice struct {
	Data uintptr
	Len  int
	Cap  int
}

func AppendSlice(s Slice) {
	if s.Len+1 > s.Cap {
		// grow slice ...
	}

	*(*int)(unsafe.Pointer(s.Data + uintptr(s.Len)*8)) = 1024
	s.Len += 1
}

func main() {
	s := Slice{Data: 0x12345, Len: 0, Cap: 8}

	AppendSlice(s)
	fmt.Printf("s = %+v, len = %d, cap = %d\n", s, s.Len, s.Cap)
}
```

相信看到这里你已经懂了我想表达的内容。最后，我们 `append` 的内容实际已经写到内存里，由于 `main` 函数中的切片 `s.len` 还是 0 ，导致我们无法看到这个元素，我们可以通过以下方法重新发现：

```go
func main() {
	s := make([]int, 0, 8)

	AppendSlice(s)
	fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
	// s = [], len = 0, cap = 8

	(*reflect.SliceHeader)(unsafe.Pointer(&s)).Len = 1
	fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
	// s = [1234], len = 1, cap = 8
}
```



### 切片的扩容

在日常开发中，经常会通过 `append` 内置方法将一个或多个值添加到切片的末尾来达到扩容的目的。由于切片是基于一个数组的「视图」，而数组的大小是不可变的，所以在 `append` 过程中如果数组的长度不足以添加更多的值，就需要对底层数组进行扩容。可以通过如下代码从外部看一下扩容是怎么样的：

```go
//
// main.go
//

func main() {
    var s []int
    fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
    // s = [], len = 0, cap = 0

    s = append(s, 1)
    fmt.Printf("append(1) => %v, len = %d, cap = %d\n", s, len(s), cap(s))
    // append(1) => [1], len = 1, cap = 1

    s = append(s, 2)
    fmt.Printf("append(2) => %v, len = %d, cap = %d\n", s, len(s), cap(s))
    // append(2) => [1 2], len = 2, cap = 2

    s = append(s, 3)
    fmt.Printf("append(3) => %v, len = %d, cap = %d\n", s, len(s), cap(s))
    // append(3) => [1 2 3], len = 3, cap = 4

    s = append(s, 4, 5)
    fmt.Printf("append(4, 5) => %v, len = %d, cap = %d\n", s, len(s), cap(s))
    // append(4, 5) => [1 2 3 4 5], len = 5, cap = 8

    s = append(s, 6, 7, 8, 9)
    fmt.Printf("append(6, 7, 8, 9) => %v, len = %d, cap = %d\n\n", s, len(s), cap(s))
    // append(6, 7, 8, 9) => [1 2 3 4 5 6 7 8 9], len = 9, cap = 16

    s1 := []int{1, 2, 3}
    fmt.Printf("s1 = %v, len = %d, cap = %d\n", s1, len(s1), cap(s1))
    // s1 = [1 2 3], len = 3, cap = 3

    s1 = append(s1, 4)
    fmt.Printf("append(4) => %v, len = %d, cap = %d\n", s1, len(s1), cap(s1))
    // append(4) => [1 2 3 4], len = 4, cap = 6

    s1 = append(s1, 5, 6, 7)
    fmt.Printf("append(5, 6, 7) => %v, len = %d, cap = %d\n\n", s1, len(s1), cap(s1))
    // append(5, 6, 7) => [1 2 3 4 5 6 7], len = 7, cap = 12

    s2 := []int{0}
    fmt.Printf("s2 => len = %d, cap = %d\n", len(s2), cap(s2))
    // s2 => len = 1, cap = 1

    for i := 0; i < 13; i++ {
        for j, n := 0, 1<<i; j < n; j++ {
            s2 = append(s2, j)
        }
        fmt.Printf("append(<%d>...) => len = %d, cap = %d\n", 1<<i, len(s2), cap(s2))
        // append(<1>...) => len = 2, cap = 2
        // append(<2>...) => len = 4, cap = 4
        // append(<4>...) => len = 8, cap = 8
        // append(<8>...) => len = 16, cap = 16
        // append(<16>...) => len = 32, cap = 32
        // append(<32>...) => len = 64, cap = 64
        // append(<64>...) => len = 128, cap = 128
        // append(<128>...) => len = 256, cap = 256
        // append(<256>...) => len = 512, cap = 512
        // append(<512>...) => len = 1024, cap = 1024
        // append(<1024>...) => len = 2048, cap = 2304
        // append(<2048>...) => len = 4096, cap = 4096
        // append(<4096>...) => len = 8192, cap = 9216
    }
}
```

观察输出结果，切片增长的趋势大致上是以翻倍的形式进行，接下来我们来验证一下。



### 从反汇编开始

内置方法 `append` 的函数签名为 `func append(slice []Type, elems ...Type) []Type`，该方法并没有具体的方法体实现，而是在编译器执行中间代码生成时将 `append` 方法替换成真实的运行时函数。我们来看示例代码：

```go
//
// main.go
//

func main() {
    var s []int
    s = append(s, 1234)
    s = append(s, 5678)
}

// go tool compile -N -l -S main.go > main.S
// 
// -N disable optimizations : 禁止优化
// -l disable inlining : 禁止内联
// -S print assembly listing : 输出汇编
```

可以使用 `go tool compile` 命令导出以上代码的汇编内容，如下所示：

```assembly
"".main STEXT size=270 args=0x0 locals=0x68 funcid=0x0
        // func main()
    0x0000 00000 (main.go:3)    TEXT    "".main(SB), ABIInternal, $104-0

    // 检查是否需要扩展栈空间
    0x0000 00000 (main.go:3)    MOVQ    (TLS), CX
    0x0009 00009 (main.go:3)    CMPQ    SP, 16(CX)
    0x000d 00013 (main.go:3)    JLS     260

    // 为 main 函数开辟栈空间
    0x0013 00019 (main.go:3)    SUBQ    $104, SP
    0x0017 00023 (main.go:3)    MOVQ    BP, 96(SP)
    0x001c 00028 (main.go:3)    LEAQ    96(SP), BP

    // 初始化切片 s, 分别为切片的三个字段赋零值
    0x0021 00033 (main.go:4)    MOVQ    $0, "".s+72(SP)       // s.Data = null
    0x002a 00042 (main.go:4)    XORPS   X0, X0                // 这里使用 128 位的 XMM 寄存器一次性初始化两个字段
    0x002d 00045 (main.go:4)    MOVUPS  X0, "".s+80(SP)       // s.Len = s.Cap = 0
    0x0032 00050 (main.go:5)    JMP     52

    // 第一次插入并进行切片扩容
    // func growslice(et *_type, old slice, cap int) slice
    0x0034 00052 (main.go:5)    LEAQ    type.int(SB), AX        // 获取切片元素类型的指针
    0x003b 00059 (main.go:5)    MOVQ    AX, (SP)                // 第一个参数 et 压栈
    0x003f 00063 (main.go:5)    XORPS   X0, X0                  // 使用 128 位的 XMM 寄存器减少使用的指令数量
    0x0042 00066 (main.go:5)    MOVUPS  X0, 8(SP)               // 第二个参数 old 的 .Data 和 .Len 字段初始化
    0x0047 00071 (main.go:5)    MOVQ    $0, 24(SP)              // 第二个参数 old 的 .Cap 字段初始化
    0x0050 00080 (main.go:5)    MOVQ    $1, 32(SP)              // 第三个参数 cap 压栈, 值为 1
    0x0059 00089 (main.go:5)    CALL    runtime.growslice(SB)   // 调用 runtime.growslice 方法进行切片扩容
    0x005e 00094 (main.go:5)    MOVQ    40(SP), AX              // 返回值 r.Data
    0x0063 00099 (main.go:5)    MOVQ    56(SP), CX              // 返回值 r.Cap
    0x0068 00104 (main.go:5)    MOVQ    48(SP), DX              // 返回值 r.Len = 0
    0x006d 00109 (main.go:5)    LEAQ    1(DX), BX               // 返回值 r.Len = 0 的值加 1 赋值给 BX
    0x0071 00113 (main.go:5)    JMP     115
    0x0073 00115 (main.go:5)    MOVQ    $1234, (AX)             // 将需要 append 的元素保存到 .Data 所指向的位置
    0x007a 00122 (main.go:5)    MOVQ    AX, "".s+72(SP)         // s.Data = r.Data
    0x007f 00127 (main.go:5)    MOVQ    BX, "".s+80(SP)         // s.Len = r.Len
    0x0084 00132 (main.go:5)    MOVQ    CX, "".s+88(SP)         // s.Cap = r.Cap

    // 检查是否需要切片扩容, 再执行插入操作
    0x0089 00137 (main.go:6)    LEAQ    2(DX), SI               // 返回值 r.Len = 0 的值加 2 赋值给 SI
    0x008d 00141 (main.go:6)    CMPQ    CX, SI                  // 对比 r.Cap 和 Len 的值 (r.Cap - Len)
    0x0090 00144 (main.go:6)    JCC     148                     // >= unsigned
    0x0092 00146 (main.go:6)    JMP     190                     // 如果 r.Cap < Len, 则跳转到 190 进行切片扩容
    0x0094 00148 (main.go:6)    JMP     150                     // 如果 r.Cap >= Len, 则直接将元素添加到末尾
    0x0096 00150 (main.go:6)    LEAQ    (AX)(DX*8), DX          // 保存元素的地址 DX = (r.Data + r.Len(0) * 8)
    0x009a 00154 (main.go:6)    LEAQ    8(DX), DX               // r.Len = 0 且已经有一个元素, 所以这里需要 +8
    0x009e 00158 (main.go:6)    MOVQ    $5678, (DX)             // 写入数据
    0x00a5 00165 (main.go:6)    MOVQ    AX, "".s+72(SP)         // s.Data = r.Data
    0x00aa 00170 (main.go:6)    MOVQ    SI, "".s+80(SP)         // s.Len = r.Len
    0x00af 00175 (main.go:6)    MOVQ    CX, "".s+88(SP)         // s.Cap = r.Cap

    // 清理 main 函数的栈空间并返回
    0x00b4 00180 (main.go:7)    MOVQ    96(SP), BP
    0x00b9 00185 (main.go:7)    ADDQ    $104, SP
    0x00bd 00189 (main.go:7)    RET

    // 第二次插入空间不够, 需要再次扩容
    // func growslice(et *_type, old slice, cap int) slice
    0x00be 00190 (main.go:5)    MOVQ    DX, ""..autotmp_1+64(SP)    // 备份 DX = r.Len = 0 到一个临时变量上
    0x00c3 00195 (main.go:6)    LEAQ    type.int(SB), DX            // 获取切片元素类型的指针
    0x00ca 00202 (main.go:6)    MOVQ    DX, (SP)                    // 第一个参数 et 压栈
    0x00ce 00206 (main.go:6)    MOVQ    AX, 8(SP)                   // 第二个参数 old.Data 压栈
    0x00d3 00211 (main.go:6)    MOVQ    BX, 16(SP)                  // 第二个参数 old.Len = 1 压栈
    0x00d8 00216 (main.go:6)    MOVQ    CX, 24(SP)                  // 第二个参数 old.Cap 压栈
    0x00dd 00221 (main.go:6)    MOVQ    SI, 32(SP)                  // 第三个参数 cap = 2 压栈
    0x00e2 00226 (main.go:6)    CALL    runtime.growslice(SB)       // 执行切片扩容, 扩容之后只有 Cap, Data 会变
    0x00e7 00231 (main.go:6)    MOVQ    40(SP), AX                  // 覆盖切片 s.Data = r.Data
    0x00ec 00236 (main.go:6)    MOVQ    48(SP), CX                  // 覆盖切片 s.Len = r.Len
    0x00f1 00241 (main.go:6)    MOVQ    56(SP), DX                  // 覆盖切片 s.Cap = r.Cap
    0x00f6 00246 (main.go:6)    LEAQ    1(CX), SI                   // 将 r.Len + 1 保证后续步骤的寄存器能对上
    0x00fa 00250 (main.go:6)    MOVQ    DX, CX                      // 保证后续步骤的寄存器能相对应
    0x00fd 00253 (main.go:6)    MOVQ    ""..autotmp_1+64(SP), DX    // 恢复 DX 的值
    0x0102 00258 (main.go:6)    JMP     150

    // 执行栈扩展
    0x0104 00260 (main.go:6)    NOP
    0x0104 00260 (main.go:3)    CALL    runtime.morestack_noctxt(SB)
    0x0109 00265 (main.go:3)    JMP     0
```

以上汇编代码中有一些比较有意思的点，我们来说道说道：

* 使用 XMM 寄存器对应的 `XORPS` 和 `MOVUPS` 指令一次性初始化 16 字节的数据，而如果使用 `MOVQ` 一次只能清空 8 个字节，这意味着需要两倍的指令才能达到相同的目的。
* 使用 `LEAQ` 来计算而不是 `ADDQ` 或 `MULQ`。原因是 `LEAQ` 的指令非常短，还能做简单的算式运算。并且 `LEAQ` 指令不占用 `ALU`，对并行的支持比较友好。

回到正题，通过汇编代码可以观察到在运行时执行扩容操作的是 `runtime.growslice` 方法（位于 `runtime/slice.go` 文件内），接下来我们就来详细扒拉扒拉它是如何实现切片扩容的。

关于切片扩容的相关提案：

* [proposal: Go 2: allow cap(make([]T, m, n)) > n][14]



### 深入扩容的实现

首先我们来看看 `runtime.growslice` 的方法签名以及注释内容：

```go
//
// runtime.slice.go
//

// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
// The new slice's length is set to the old slice's length,
// NOT to the new requested capacity.
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
func growslice(et *_type, old slice, cap int) slice {}
```

从方法签名以及注释可以得知：该方法会根据**切片的元素类型**、**当前切片**以及**新切片所需的最小容量**（即新切片的长度）进行扩容，返回一个**至少具有指定容量的新切片**，并将旧切片中的数据拷贝过去。

签名中 `slice` 就是上面我们提到的切片结构体，另外一个 `_type` 类型则比较陌生，先来看看这个类型是怎么定义的：

```go
//
// runtime/type.go
//

// Needs to be in sync with ../cmd/link/internal/ld/decodesym.go:/^func.commonsize,
// ../cmd/compile/internal/reflectdata/reflect.go:/^func.dcommontype and
// ../reflect/type.go:/^type.rtype.
// ../internal/reflectlite/type.go:/^type.rtype.
type _type struct {
	size       uintptr // 类型所占内存的大小
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32  // 类型的哈希值，在接口断言和接口查询中使用
	tflag      tflag   // 类型的特征标记
	align      uint8   // _type 作为整体保存时的对齐字节数
	fieldAlign uint8   // 当前结构字段的对齐字节数
	kind       uint8   // 基础类型的枚举值，与 reflect.Kind 的值相同，决定了如何解析该类型
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte    // GC 的相关信息
	str       nameOff  // 类型的名称字符串在二进制文件中的偏移值。由链接器负责填充
	ptrToThis typeOff  // 类型元信息（即当前结构体）的指针在编译后二进制文件中的偏移值。由链接器负责填充
}
```

从上面的汇编代码中，可以看到传入的符号是 `type.int(SB)` ，这个符号会在链接阶段进行符号重定位时进行填充和替换。其中我们只需要用到 `_type` 类型中的 `size` 字段（用于计算新切片占用内存的大小）。

#### 第一部分：计算新切片的容量

回到代码中，首先来看前半部分的实现。这部分代码主要是为了计算得到新切片的容量，方便之后申请内存使用。直接看代码：

```go
//
// runtime/slice.go 
//
// -- 为了便于分析和展示, 代码经过删减 --
//
// et: 切片的元素类型
// old: 当前切片
// cap: 新切片所需的最小容量
func growslice(et *_type, old slice, cap int) slice {
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.cap < 1024 {
            newcap = doublecap
        } else {
            // Check 0 < newcap to detect overflow
            // and prevent an infinite loop.
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            // Set newcap to the requested cap when
            // the newcap calculation overflowed.
            if newcap <= 0 {
                newcap = cap
            }
        }
    }
    // ...
}
```

根据以上代码可以整理出如下扩容策略：

* 如果**新切片所需的最小容量**大于**当前切片容量**的两倍，那么就直接用**新切片所需的最小容量**
* 如果**新切片所需的最小容量**小于等于**当前切片的容量**的两倍
  * 如果**当前切片的容量**小于 1024 ，则直接把**当前切片的容量**翻倍作为**新切片的容量**
  * 如果**当前切片的容量**大于等于 1024 ，则每次递增**切片容量**的 1/4 倍，直到大于**新切片所需的最小容量**为止。

总结以上扩容策略，我们可以用如下伪代码表示：

```pseudocode
if NewCap > CurrCap * 2
    return NewCap

if CurrCap < 1024
    return CurrCap * 2

while CurrCap < NewCap
    CurrCap += CurrCap / 4
```

#### 第二部分：进行内存分配

当得到**新切片的容量**之后，最简单的方式就是直接向系统申请 `ElementTypeSize * NewSliceCap` 的内存。但这么做不可避免的会产生很多的内存碎片，同时对高性能不友好（在堆内存不足时，需要通过 `brk` 系统调用扩展堆）。

##### Go 的内存管理

那我们应该如何实现高性能的内存管理的同时减少内存碎片的产生？我们需要实现以下功能：

* 内存池技术：一次从操作系统中申请大块内存，避免用户态到内核态的切换
* 垃圾回收：动态、自动的垃圾回收机制让内存可以重复使用
* 内存分配算法：可以实现高效的内存分配同时避免竞争和碎片化

Go 运行时的内存分配算法基于 [`TCMalloc`][5] 实现。该算法的核心思想就是将内存划分为多个不同的级别进行管理从而降低锁的粒度。具体的内存管理如何实现以及为何这么实现可以参考以下文章学习：

* [A visual guide to Go Memory Allocator from scratch (Golang)][7]
* [Go: Memory Management and Allocation][8]
* [Memory Management in Golang][9]
* [Golang's memory allocation][10]
* [Github: TCMalloc design][6]
* [TCMalloc : Thread-Caching Malloc][11]
* [浅析Go内存管理架构][12]
* [图解Golang内存分配和垃圾回收][13]

**简单的说 Go 将小于 32K 字节的内存分为从 8B - 32K 等大约70种不同的规格（span）。如果申请的内存小于 32K 则直接向上取到某一个规格（可能会造成一定的浪费）。如果申请的内存大于 32K 则直接从 Go 持有的堆中按页（一个页面为 8K 字节）向上取整进行申请。**

##### 在切片扩容中的应用

接下来回到正题，看看内存管理是如何在切片扩容中应用的：

```go
//
// runtime/slice.go 
//

// -- 为了便于分析和展示, 代码经过删减 --
func growslice(et *_type, old slice, cap int) slice {
    // ...
    var overflow bool
    // lenmem    -> 当前切片元素所占用内存的大小
    // newlenmem -> 在 append 之后切片元素所占用的内存大小
    // capmem    -> 实际申请到的内存大小
    var lenmem, newlenmem, capmem uintptr
    // Specialize for common values of et.size.
    // For 1 we don't need any division/multiplication.
    // For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
    // For powers of 2, use a variable shift.
    switch {
    case et.size == 1:
        lenmem = uintptr(old.len)
        newlenmem = uintptr(cap)
        capmem = roundupsize(uintptr(newcap))
        overflow = uintptr(newcap) > maxAlloc
        newcap = int(capmem)
    case et.size == sys.PtrSize:
        lenmem = uintptr(old.len) * sys.PtrSize
        newlenmem = uintptr(cap) * sys.PtrSize
        capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
        overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
        newcap = int(capmem / sys.PtrSize)
    case isPowerOfTwo(et.size):
        var shift uintptr
        if sys.PtrSize == 8 {
            // Mask shift for better code generation.
            shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
        } else {
            shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
        }
        lenmem = uintptr(old.len) << shift
        newlenmem = uintptr(cap) << shift
        capmem = roundupsize(uintptr(newcap) << shift)
        overflow = uintptr(newcap) > (maxAlloc >> shift)
        newcap = int(capmem >> shift)
    default:
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
        capmem = roundupsize(capmem)
        newcap = int(capmem / et.size)
    }
}
```

其实只要看 `default` 分支中的逻辑即可，其他分支只是根据元素类型的大小来做针对性的优化（通过移位等运算来少执行几个指令）。在这段代码中，决定最终申请多少内存的是 `roundupsize` 方法：

```go
//
// runtime/msize.go
//

// Malloc small size classes.
//
// See malloc.go for overview.
// See also mksizeclasses.go for how we decide what size classes to use.

// Returns size of the memory block that mallocgc will allocate if you ask for the size.
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize { // _MaxSmallSize = 32768
		if size <= smallSizeMax-8 { // smallSizeMax = 1024
            // smallSizeDiv = 8
			return uintptr(class_to_size[size_to_class8[divRoundUp(size, smallSizeDiv)]])
		} else {
            // smallSizeMax = 1024, largeSizeDiv = 128
			return uintptr(class_to_size[size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]])
		}
	}
	if size+_PageSize < size { // _PageSize = 8192
		return size
	}
	return alignUp(size, _PageSize)
}


//
// runtime/stubs.go
//

// divRoundUp returns ceil(n / a).
func divRoundUp(n, a uintptr) uintptr {
	// a is generally a power of two. This will get inlined and
	// the compiler will optimize the division.
	return (n + a - 1) / a
}

// alignUp rounds n up to a multiple of a. a must be a power of 2.
func alignUp(n, a uintptr) uintptr {
	return (n + a - 1) &^ (a - 1)
}
```

该方法传入 `size` 参数计算自 `NewSliceCap * ElementType.Size` 。如果传入的 `size` 小于 32K 则会从以下规格中选择一个「刚好能满足要求的规格」返回：

```go
// 0 表示大对象(large object)
var class_to_size = [_NumSizeClasses]uint16{ // _NumSizeClasses = 68
        0,     8,    16,    24,    32,    48,    64,    80,    96,   112, 
      128,   144,   160,   176,   192,   208,   224,   240,   256,   288, 
      320,   352,   384,   416,   448,   480,   512,   576,   640,   704, 
      768,   896,  1024,  1152,  1280,  1408,  1536,  1792,  2048,  2304, 
     2688,  3072,  3200,  3456,  4096,  4864,  5376,  6144,  6528,  6784, 
     6912,  8192,  9472,  9728, 10240, 10880, 12288, 13568, 14336, 16384, 
    18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768, 
}
```

如果申请的内存大于 32K 则会直接从堆中申请「刚好能满足的要求」的页（页大小为 8K）。相关内容可以参考以下源码：

* `runtime/malloc.go`：内存管理相关的实现
* `runtime/mksizeclasses.go`：生生以上规格的逻辑

#### 第三部分：拷贝对象和返回

由于实际申请得到的内存容量可能会大于需求的容量，所以在确定申请内存的规格之后需要再次确定所申请到的内存规格到底可以容纳多少元素。如下代码所示：

```go
//
// runtime/slice.go 
//

// -- 为了便于分析和展示, 代码经过删减 --
func growslice(et *_type, old slice, cap int) slice {
    // ...
    capmem = roundupsize(capmem)
    newcap = int(capmem / et.size)
    // ...
}
```

在扩容过程中有关内存大小的几个变量这里做个统一的解释：

* `lenmem`：在 `append` **之前**切片中所有元素所占用的内存大小（`OldSlice.len * ElementTypeSize `）
* `newlenmem`：在 `append` **之后**切片中所有元素所占用内存大小（`(OldSlice.len + AppendSize) * ElementTypeSize`）
* `size`：在 `append` **之后**新切片的容量（第一部分计算得到）所需要的内存大小（`NewSliceCap * ElementTypeSize`）
* `capmem`：经过内存规格匹配之后实际申请到的内存大小（第二部分计算得到）

接下来就需要将数据从旧切片拷贝到新切片中，直接看代码：

```go
//
// runtime/slice.go 
//

// -- 为了便于分析和展示, 代码经过删减 --
func growslice(et *_type, old slice, cap int) slice {
    // ...
    // p 指向申请到的内存的首地址
    var p unsafe.Pointer
	if et.ptrdata == 0 { // 如果元素类型中不包含指针
        // 申请 capmem 个字节
		p = mallocgc(capmem, nil, false)
        // 由于可能会申请到大于需求容量的内存，所以需要将目前用不到的内存清零
        // 在需求容量之内的内存会由之后的程序负责填充和赋值
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
        // 申请指定大小的内存并将所有内存清零
		p = mallocgc(capmem, et, true)
        // 为内存添加写屏障
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
		}
	}
    // 将旧切片中的的数据拷贝过来，从 old.array 拷贝 lenmem 个字节到 p
	memmove(p, old.array, lenmem)

    // 返回新切片
	return slice{p, old.len, newcap}
}


//
// runtime/stubs.go
// 

// memmove copies n bytes from "from" to "to".
func memmove(to, from unsafe.Pointer, n uintptr)
```

至此，所有扩容相关的工作就结束了，接下来会由程序的其他部分将数据填充到刚刚扩容的切片中。这里我们使用如下代码加上一张图来总结以上的所有内容，试试你能不能将以上各个部分的逻辑在图中对应起来。代码如下：

```go
type Triangle [3]byte

func main() {
	s := []Triangle{{1, 1, 1}, {2, 2, 2}}

	s = append(s, Triangle{3, 3, 3}, Triangle{4, 4, 4})
	fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
	// s = [[1 1 1] [2 2 2] [3 3 3] [4 4 4]], len = 4, cap = 5
}
```

根据以上代码，我们可以画出如下的结构图（其中长方体表示一个 `Triangle` 类型，占用 3 个字节的空间）：

<img src="assets/memory-structure.png" alt="Structure of the memory" style="zoom:50%;" />



### 验证切片扩容

纸上得来终觉浅，绝知此事要躬行。上面所分析的内容毕竟只是理论和猜测，我们需要通过实践来验证所总结的规律靠不靠谱。

#### 小于 32K 的切片扩容

首先来试试在小于 32K 的切片上进行扩容，有如下代码：

```go
func main() {
	var s []int

	s = append(s, 1, 2, 3, 4, 5)
	fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
    // s = [1 2 3 4 5], len = 5, cap = 6

	s = append(s, 6, 7, 8, 9)
	fmt.Printf("s = %v, len = %d, cap = %d\n", s, len(s), cap(s))
    // s = [1 2 3 4 5 6 7 8 9], len = 9, cap = 12
}
```

我们按照上面所总结的规律来一步步验证，有如下过程：

```pseudocode
var s []int // s.data = null, s.len = 0, s.cap = 0

s = append(s, 1, 2, 3, 4, 5)
// 旧切片的容量不足，需要进行扩容
// 执行扩容方法：growslice(et = type.int, old = {null, 0, 0}, cap = 5)
//    1. 由于 cap > 2 * old.cap ，所以新切片的容量至少需要为 5 (NewSliceCap)
//    2. 执行内存规格匹配，需要的内存容量为 5 * 8(int) = 40 字节，查表可得实际使用的内存规格为 48 字节
//    3. 所以实际的切片容量为 48(capmem) / 8(et.size) = 6
// 验证正确

s = append(s, 6, 7, 8, 9)
// 旧切片容量为 6 ，需求容量为 9 ，需要进行扩容
// 执行扩容方法：growslice(et = type.int, old = {<addr>, 5, 6}, cap = 9)
//    1. 由于 cap < 2 * old.cap = 12 且 old.cap < 1024 ，所以新切片的容量至少需要为 2 * old.cap = 12
//    2. 执行内存规格匹配，需要的内存容量为 12 * 8(int) = 96 字节，查表可得实际使用的内存规格为 96 字节
//    3. 所以实际的切片容量为 96(capmem) / 8(et.size) = 12
// 验证正确
```

#### 大于 32K 的切片扩容

我们通过长度为 128 的字节数组（占用 1024 个字节）来逐步验证大于 32K 的切片扩容：

```go
type Array [128]int // 128 * 8 = 1024 Bytes

func main() {
	var s []Array

	s = append(s, Array{}, Array{}, Array{}, Array{}, Array{}, Array{}, Array{}) // 7K
	fmt.Printf("s, len = %d, cap = %d\n", len(s), cap(s))
    // s, len = 7, cap = 8

	s = append(s, make([]Array, 26)...) // 33K
	fmt.Printf("s, len = %d, cap = %d\n", len(s), cap(s))
    // s, len = 33, cap = 40
}
```

同样，按照总结的规律进行推算，有如下过程：

```pseudocode
var s []Array // s.data = null, s.len = 0, s.cap = 0

s = append(s, Array{}, Array{}, Array{}, Array{}, Array{}, Array{}, Array{})
// 旧切片的容量不足，需要进行扩容
// 执行扩容方法：growslice(et = type.Array, old = {null, 0, 0}, cap = 7)
//    1. 由于 cap > 2 * old.cap ，所以新切片的容量至少需要为 7 (NewSliceCap)
//    2. 执行内存规格匹配，需要的内存容量为 7 * 1024(Array) = 7168 字节，查表可得实际使用的内存规格为 8192 字节
//    3. 所以实际的切片容量为 8192(capmem) / 1024(et.size) = 8
// 验证正确

s = append(s, make([]Array, 26)...)
// 旧切片容量为 8 ，需求容量为 33 ，需要进行扩容
// 执行扩容方法：growslice(et = type.Array, old = {<addr>, 7, 8}, cap = 33)
//    1. 由于 cap > 2 * old.cap = 16 ，所以新切片的容量至少需要为 cap = 33
//    2. 执行内存规格匹配，需要的内存容量为 33 * 1024(int) = 33792 字节，由于大于 32768 ，按页向上取整得到 40960 字节
//    3. 所以实际的切片容量为 40960(capmem) / 1024(et.size) = 40
// 验证正确
```

到这里本文的所有内容就结束了，希望你能有所收获。如有错漏望不吝赐教！



### 其他参考资料

以下是一些关于 Go 汇编的参考资料

* [What are the conditional jump instructions for Go's assembler?][15]
* [Intel x86 JUMP quick reference][16]
* [A Manual for the Plan 9 assembler][17]



[1]: https://go.dev/tour/moretypes/7 "Slices"

[2]: https://go.googlesource.com/proposal/+/refs/changes/78/248178/1/design/40724-register-calling.md "Proposal"
[3]: https://github.com/golang/go/issues/40724 "Discussion"
[4]: https://github.com/golang/go/blob/master/src/cmd/compile/abi-internal.md#function-call-argument-and-result-passing    "ABI-internal"

[5]: https://github.com/google/tcmalloc "TCMalloc"
[6]: https://github.com/google/tcmalloc/blob/master/docs/design.md "TCMalloc Design"
[7]: https://medium.com/@ankur_anand/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed "A visual guide to Go Memory Allocator from scratch (Golang)"
[8]: https://medium.com/a-journey-with-go/go-memory-management-and-allocation-a7396d430f44 "Go: Memory Management and Allocation"
[9]: https://www.timqi.com/2020/06/11/golang-memory-management/ "Memory Management in Golang"
[10]: https://www.sobyte.net/post/2022-04/golang-memory-allocation/ "Golang's memory allocation"
[11]: http://goog-perftools.sourceforge.net/doc/tcmalloc.html "TCMalloc : Thread-Caching Malloc"
[12]: https://mp.weixin.qq.com/s/Jxw5y4BVHSDKUl9A3citLw "浅析Go内存管理架构"
[13]: https://mp.weixin.qq.com/s/iAy9ReQhnmCYUFvwYroGPA "图解Golang内存分配和垃圾回收"
[14]: https://github.com/golang/go/issues/24204	"proposal: Go 2: allow cap(make([]T, m, n)) > n"
[15]: https://stackoverflow.com/questions/30146642/what-are-the-conditional-jump-instructions-for-gos-assembler "What are the conditional jump instructions for Go's assembler?"
[16]: http://www.unixwiz.net/techtips/x86-jumps.html "Intel x86 JUMP quick reference"
[17]: https://9p.io/sys/doc/asm.html "A Manual for the Plan 9 assembler"
