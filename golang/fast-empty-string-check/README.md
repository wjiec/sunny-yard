字符串应该可以说是开发中最常用的类型了，而判断字符串是否为空可能是字符串操作中最常见的几种之一。那么问题来了，常见的判断字符串是否为空最高效、最快的方式是哪种呢？


### 常见的判断空字符串方式
最常见的判断空字符串的方式有两种：一是直接与空字符串进行对比，二是判断字符串的长度是否为零：
```go
package emptystring

func EqualsEmpty(s string) bool {
    return s == ""
}

func ZeroLength(s string) bool {
    return len(s) == 0
}
```

#### TL;DR
两种方法想用哪种用哪种，在编译器优化之后没有任何区别。


### 通过 Benchmark 来判断
Go 内置了强大的基准性能测试工具让我们可以很轻松地对某一段代码进行性能测试：
```go
var strings []string

func init() {
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < 1_000; i++ {
		if rand.Int()%2 == 0 {
			strings = append(strings, "")
		} else {
			data := make([]byte, 1024)
			rand.Read(data)
			strings = append(strings, hex.EncodeToString(data))
		}
	}
}

func BenchmarkEqualsEmpty(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for _, s := range strings {
			EqualsEmpty(s)
		}
	}
}

func BenchmarkZeroLength(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for _, s := range strings {
			ZeroLength(s)
		}
	}
}
```
通过命令 `go test -bench . -count 8` 对该文件进行基准性能测试，结果如下：
```
goos: darwin
goarch: amd64
pkg: go.jayson.wang/go-yes/emptystring
cpu: Intel(R) Core(TM) i7-4770HQ CPU @ 2.20GHz
BenchmarkEqualsEmpty-8           3386563               327.8 ns/op
BenchmarkEqualsEmpty-8           3664698               328.4 ns/op
BenchmarkEqualsEmpty-8           3671127               326.6 ns/op
BenchmarkEqualsEmpty-8           3681252               329.5 ns/op
BenchmarkEqualsEmpty-8           3650229               327.8 ns/op
BenchmarkEqualsEmpty-8           3630446               329.1 ns/op
BenchmarkEqualsEmpty-8           3581197               327.6 ns/op
BenchmarkEqualsEmpty-8           3666394               331.5 ns/op
BenchmarkZeroLength-8            3656193               327.0 ns/op
BenchmarkZeroLength-8            3646698               327.0 ns/op
BenchmarkZeroLength-8            3655800               326.7 ns/op
BenchmarkZeroLength-8            3664434               327.8 ns/op
BenchmarkZeroLength-8            3666055               328.6 ns/op
BenchmarkZeroLength-8            3645968               326.4 ns/op
BenchmarkZeroLength-8            3672844               326.7 ns/op
BenchmarkZeroLength-8            3669993               326.4 ns/op
PASS
```
可见两者在性能上不能说是一模一样，只能说是没有区别。


### 追加一轮判断不为空的测试
为了严谨，防止编译器在其中做了一些奇奇怪怪的事情导致问题出现，我们追加一轮判断非空的方法并配置这些方法不允许内联：
```go
package emptystring

//go:noinline
func EqualsEmpty(s string) bool {
	return s == ""
}

//go:noinline
func ZeroLength(s string) bool {
	return len(s) == 0
}

//go:noinline
func NotEqualsEmpty(s string) bool {
	return s != ""
}

//go:noinline
func NotZeroLength(s string) bool {
	return len(s) != 0
}
```
并增加对应的基准性能测试用例：
```go
package emptystring

import (
	"encoding/hex"
	"math/rand"
	"testing"
	"time"
)

var strings []string

func init() {
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < 1_000; i++ {
		if rand.Int()%2 == 0 {
			strings = append(strings, "")
		} else {
			data := make([]byte, 1024)
			rand.Read(data)
			strings = append(strings, hex.EncodeToString(data))
		}
	}
}

func BenchmarkEqualsEmpty(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for _, s := range strings {
			EqualsEmpty(s)
		}
	}
}

func BenchmarkZeroLength(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for _, s := range strings {
			ZeroLength(s)
		}
	}
}

func BenchmarkNotEqualsEmpty(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for _, s := range strings {
			NotEqualsEmpty(s)
		}
	}
}

func BenchmarkNotZeroLength(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for _, s := range strings {
			NotZeroLength(s)
		}
	}
}

```
再次运行命令 `go test -bench . -count 8` 得到结果如下：
```
goos: darwin
goarch: amd64
pkg: go.jayson.wang/go-yes/emptystring
cpu: Intel(R) Core(TM) i7-4770HQ CPU @ 2.20GHz
BenchmarkEqualsEmpty-8            552691              2138 ns/op
BenchmarkEqualsEmpty-8            546831              2135 ns/op
BenchmarkEqualsEmpty-8            553880              2134 ns/op
BenchmarkEqualsEmpty-8            558354              2136 ns/op
BenchmarkEqualsEmpty-8            557022              2135 ns/op
BenchmarkEqualsEmpty-8            557986              2135 ns/op
BenchmarkEqualsEmpty-8            552715              2143 ns/op
BenchmarkEqualsEmpty-8            555046              2136 ns/op
BenchmarkZeroLength-8             560852              2154 ns/op
BenchmarkZeroLength-8             554137              2146 ns/op
BenchmarkZeroLength-8             562004              2145 ns/op
BenchmarkZeroLength-8             557760              2140 ns/op
BenchmarkZeroLength-8             554934              2134 ns/op
BenchmarkZeroLength-8             537969              2158 ns/op
BenchmarkZeroLength-8             556741              2135 ns/op
BenchmarkZeroLength-8             555208              2137 ns/op
BenchmarkNotEqualsEmpty-8         556560              2137 ns/op
BenchmarkNotEqualsEmpty-8         556093              2133 ns/op
BenchmarkNotEqualsEmpty-8         554662              2140 ns/op
BenchmarkNotEqualsEmpty-8         554407              2144 ns/op
BenchmarkNotEqualsEmpty-8         554498              2144 ns/op
BenchmarkNotEqualsEmpty-8         516790              2166 ns/op
BenchmarkNotEqualsEmpty-8         556368              2139 ns/op
BenchmarkNotEqualsEmpty-8         561820              2140 ns/op
BenchmarkNotZeroLength-8          549516              2140 ns/op
BenchmarkNotZeroLength-8          550549              2138 ns/op
BenchmarkNotZeroLength-8          562420              2139 ns/op
BenchmarkNotZeroLength-8          554427              2140 ns/op
BenchmarkNotZeroLength-8          557434              2135 ns/op
BenchmarkNotZeroLength-8          555290              2137 ns/op
BenchmarkNotZeroLength-8          558577              2133 ns/op
BenchmarkNotZeroLength-8          556191              2140 ns/op
PASS
```
确实没啥区别。


### 从汇编代码进行分析

#### 未开启优化情况下（-N）
如果想知道两种方式到底有没有高下之分，那么可以指令的数量上来判断，通过以下命令将代码转成汇编格式再对汇编指令进行分析：
```shell
go tool compile -N -l -S emptystring.go > emptystring.S
```
打开生成的 emptystring.S 文件，如下所示（代码经过删减，去掉了 GC 辅助信息等）：
```go
"".EqualsEmpty STEXT nosplit size=52 args=0x10 locals=0x10 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:4)	TEXT	"".EqualsEmpty(SB), NOSPLIT|ABIInternal, $16-16
	0x0000 00000 (emptystring.go:4)	SUBQ	$16, SP
	0x0004 00004 (emptystring.go:4)	MOVQ	BP, 8(SP)
	0x0009 00009 (emptystring.go:4)	LEAQ	8(SP), BP // 准备栈空间

	0x000e 00014 (emptystring.go:4)	MOVQ	AX, "".s+24(SP) // s.data 赋值
	0x0013 00019 (emptystring.go:4)	MOVQ	BX, "".s+32(SP) // s.len 赋值
	0x0018 00024 (emptystring.go:4)	MOVB	$0, "".~r0+7(SP) // 返回值初始化
	0x001d 00029 (emptystring.go:5)	CMPQ	"".s+32(SP), $0 // 判断 s.len 是否为 0
	0x0023 00035 (emptystring.go:5)	SETEQ	AL // 如果上一个判断的结果为 0 则置位 AL
	0x0026 00038 (emptystring.go:5)	MOVB	AL, "".~r0+7(SP) // 返回结果

	0x002a 00042 (emptystring.go:5)	MOVQ	8(SP), BP // 清理栈空间
	0x002f 00047 (emptystring.go:5)	ADDQ	$16, SP
	0x0033 00051 (emptystring.go:5)	RET


"".ZeroLength STEXT nosplit size=59 args=0x10 locals=0x18 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:9)	TEXT	"".ZeroLength(SB), NOSPLIT|ABIInternal, $24-16
	0x0000 00000 (emptystring.go:9)	SUBQ	$24, SP
	0x0004 00004 (emptystring.go:9)	MOVQ	BP, 16(SP)
	0x0009 00009 (emptystring.go:9)	LEAQ	16(SP), BP // 准备栈空间

	0x000e 00014 (emptystring.go:9)	MOVQ	AX, "".s+32(SP) // s.data 赋值
	0x0013 00019 (emptystring.go:9)	MOVQ	BX, "".s+40(SP) // s.len 赋值
	0x0018 00024 (emptystring.go:9)	MOVB	$0, "".~r0+7(SP) // 返回值初始化
	0x001d 00029 (emptystring.go:10)	MOVQ	"".s+40(SP), CX // 获取字符串的长度 s.len
	0x0022 00034 (emptystring.go:10)	MOVQ	CX, ""..autotmp_2+8(SP) // 额外的栈上变量赋值
	0x0027 00039 (emptystring.go:10)	TESTQ	CX, CX // 判断 CX 即字符串的长度是否为 0
	0x002a 00042 (emptystring.go:10)	SETEQ	AL // 如果上一个判断的结果为 0 则置位 AL
	0x002d 00045 (emptystring.go:10)	MOVB	AL, "".~r0+7(SP) // 返回结果

	0x0031 00049 (emptystring.go:10)	MOVQ	16(SP), BP // 清理栈空间
	0x0036 00054 (emptystring.go:10)	ADDQ	$24, SP
	0x003a 00058 (emptystring.go:10)	RET


"".NotEqualsEmpty STEXT nosplit size=52 args=0x10 locals=0x10 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:14)	TEXT	"".NotEqualsEmpty(SB), NOSPLIT|ABIInternal, $16-16
	0x0000 00000 (emptystring.go:14)	SUBQ	$16, SP
	0x0004 00004 (emptystring.go:14)	MOVQ	BP, 8(SP)
	0x0009 00009 (emptystring.go:14)	LEAQ	8(SP), BP // 准备栈空间

	0x000e 00014 (emptystring.go:14)	MOVQ	AX, "".s+24(SP) // s.data 赋值
	0x0013 00019 (emptystring.go:14)	MOVQ	BX, "".s+32(SP) // s.len 赋值
	0x0018 00024 (emptystring.go:14)	MOVB	$0, "".~r0+7(SP) // 返回值初始化
	0x001d 00029 (emptystring.go:15)	CMPQ	"".s+32(SP), $0 // 判断 s.len 是否为 0
	0x0023 00035 (emptystring.go:15)	SETNE	AL // 如果上一个判断的结果不为 0 则置位 AL
	0x0026 00038 (emptystring.go:15)	MOVB	AL, "".~r0+7(SP) // 返回结果

	0x002a 00042 (emptystring.go:15)	MOVQ	8(SP), BP // 清理栈空间
	0x002f 00047 (emptystring.go:15)	ADDQ	$16, SP
	0x0033 00051 (emptystring.go:15)	RET


"".NotZeroLength STEXT nosplit size=59 args=0x10 locals=0x18 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:19)	TEXT	"".NotZeroLength(SB), NOSPLIT|ABIInternal, $24-16
	0x0000 00000 (emptystring.go:19)	SUBQ	$24, SP
	0x0004 00004 (emptystring.go:19)	MOVQ	BP, 16(SP)
	0x0009 00009 (emptystring.go:19)	LEAQ	16(SP), BP // 准备栈空间

	0x000e 00014 (emptystring.go:19)	MOVQ	AX, "".s+32(SP) // s.data 赋值
	0x0013 00019 (emptystring.go:19)	MOVQ	BX, "".s+40(SP) // s.len 赋值
	0x0018 00024 (emptystring.go:19)	MOVB	$0, "".~r0+7(SP) // 返回值初始化
	0x001d 00029 (emptystring.go:20)	MOVQ	"".s+40(SP), CX // 获取字符串的长度 s.len
	0x0022 00034 (emptystring.go:20)	MOVQ	CX, ""..autotmp_2+8(SP) // 额外的栈上变量赋值
	0x0027 00039 (emptystring.go:20)	TESTQ	CX, CX // 判断 CX 即字符串的长度是否为 0
	0x002a 00042 (emptystring.go:20)	SETNE	AL // 如果上一个判断的结果不为 0 则置位 AL
	0x002d 00045 (emptystring.go:20)	MOVB	AL, "".~r0+7(SP) // 返回结果

	0x0031 00049 (emptystring.go:20)	MOVQ	16(SP), BP // 清理栈空间
	0x0036 00054 (emptystring.go:20)	ADDQ	$24, SP
	0x003a 00058 (emptystring.go:20)	RET
```
通过对两种方式的汇编代码分析，可以发现以下几点区别：

- 使用空字符串字面量进行判断的方法栈桢大小为 16 字节，而通过长度进行判断的方法栈桢是 24 字节
   - 会将字符串的长度赋值给一个生成的临时变量 `autotmp_2`
- 通过长度进行判断的方法会额外多几个指令

#### 在开启优化之后
在开启代码优化之后，我们通过 `go tool compile -l -S emptystring.go > emptystring.S` 得到以下结果：
```go
"".EqualsEmpty STEXT nosplit size=12 args=0x10 locals=0x0 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:4)	TEXT	"".EqualsEmpty(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (emptystring.go:4)	MOVQ	AX, "".s+8(FP)
	0x0005 00005 (emptystring.go:5)	TESTQ	BX, BX
	0x0008 00008 (emptystring.go:5)	SETEQ	AL
	0x000b 00011 (emptystring.go:5)	RET


"".ZeroLength STEXT nosplit size=12 args=0x10 locals=0x0 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:9)	TEXT	"".ZeroLength(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (emptystring.go:9)	MOVQ	AX, "".s+8(FP)
	0x0005 00005 (emptystring.go:10)	TESTQ	BX, BX
	0x0008 00008 (emptystring.go:10)	SETEQ	AL
	0x000b 00011 (emptystring.go:10)	RET


"".NotEqualsEmpty STEXT nosplit size=12 args=0x10 locals=0x0 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:14)	TEXT	"".NotEqualsEmpty(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (emptystring.go:14)	MOVQ	AX, "".s+8(FP)
	0x0005 00005 (emptystring.go:15)	TESTQ	BX, BX
	0x0008 00008 (emptystring.go:15)	SETNE	AL
	0x000b 00011 (emptystring.go:15)	RET


"".NotZeroLength STEXT nosplit size=12 args=0x10 locals=0x0 funcid=0x0 align=0x0
	0x0000 00000 (emptystring.go:19)	TEXT	"".NotZeroLength(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (emptystring.go:19)	MOVQ	AX, "".s+8(FP)
	0x0005 00005 (emptystring.go:20)	TESTQ	BX, BX
	0x0008 00008 (emptystring.go:20)	SETNE	AL
	0x000b 00011 (emptystring.go:20)	RET
```
只能说，想用哪种用哪种，两者没有任何区别。
