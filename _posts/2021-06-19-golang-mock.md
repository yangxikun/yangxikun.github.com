---
layout: post
title: "Golang Mock 原理分析"
description: "Golang Mock 原理分析"
category: GoLang
tags: []
---
{% include JB/setup %}

在写单元测试时，通常需要对某些不容易构造或者不容易获取的对象进行 mock，那么在 Golang 中，我们可以 mock 哪些对象，又有哪些 mock 的方法呢，以及它们是如何实现的？本文将对几个 Golang 常见的开源库进行分析，了解其实现原理。

## gomock 的实现

https://github.com/golang/mock 用于 interface 接口的 mock，需要先通过命令行工具 mockgen 生成 interface 的 mock 类型，通常会把命令用 `//go:generate` 写在代码中，比如：

```go
package main

//go:generate mockgen -source=main.go -destination=foo_mock.go -package=main Foo

type Foo interface {
	Say(string, []string) string
}

func main() {
}
```

mockgen 会为 Foo 生成 MockFoo 类型的实现：

```go
// MockFoo is a mock of Foo interface.
type MockFoo struct {
   ctrl     *gomock.Controller
   recorder *MockFooMockRecorder
}

// MockFooMockRecorder is the mock recorder for MockFoo.
type MockFooMockRecorder struct {
   mock *MockFoo
}

// NewMockFoo creates a new mock instance.
func NewMockFoo(ctrl *gomock.Controller) *MockFoo {
   mock := &MockFoo{ctrl: ctrl}
   mock.recorder = &MockFooMockRecorder{mock}
   return mock
}
```

<!--more-->

MockFoo 的方法有：
* `EXPECT() *MockFooMockRecorder`：返回 MockFoo.recorder 用于对 MockFoo 的方法进行 mock
* `Say(string, []string) string`：实现 Foo 的接口，调用 MockFoo.ctrl.Call(MockFoo, "Say")，查找匹配的 gomock.Call 并执行，没找到会报错

MockFooMockRecorder 的方法有：
* `Say(arg0 string, arg1 []string) *gomock.Call`：调用 `gomock.Controller.RecordCallWithMethodType(MockFoo, "Say", reflect.TypeOf((*MockFoo)(nil).Say), arg0)` 创建 gomock.Call

gomock.Call 用于表示期望的 mock 调用，根据方法的接收者、方法名、参数进行匹配，部分字段说明：
* preReqs：依赖的 gomock.Call 必须执行至少 gomock.Call.minCalls 才能调用当前的 gomock.Call
* minCalls: 最少调用次数
* maxCalls：最多调用次数
* numCalls：已调用次数
* actions：每次调用，会顺序执行数组里的每个 action 函数，最后一个返回非 nil 值的结果作为这次调用的返回结果。`newCall(...) *Call` 时默认会插入一个返回零值的 action 函数。我们在使用时会调用 gomock.Call 的四个方法往 actions 追加：
    * DoAndReturn(f interface{})：执行函数 f，同时使用 f 的返回值
    * Do(f interface{})：执行函数 f，但返回 nil
    * Return(rets ...interface{})：设置返回值
    * SetArg(n int, value interface{})：修改第 n 个参数的值，参数类型必须为 ptr、interface、slice

```go
// Call represents an expected call to a mock.
type Call struct {
   t TestHelper // for triggering test failures on invalid call setup

   receiver   interface{}  // the receiver of the method call
   method     string       // the name of the method
   methodType reflect.Type // the type of the method
   args       []Matcher    // the args
   origin     string       // file and line number of call setup

   preReqs []*Call // prerequisite calls

   // Expectations
   minCalls, maxCalls int

   numCalls int // actual number made

   // actions are called when this Call is called. Each action gets the args and
   // can set the return values by returning a non-nil slice. Actions run in the
   // order they are created.
   actions []func([]interface{}) []interface{}
}
```

gomock.Controller 负责存储，匹配和调用 gomock.Call，字段说明：
* T：通常是 *testing.T
* mu：并发安全锁
* expectedCalls：存储创建出来的 gomock.Call
    * callSet.expected：待调用的 gomock.Call
        * 根据方法的接收者和方法名对 gomock.Call 进行索引，参数匹配由 gomock.Call.matches 完成
    * callSet.exhausted：有两种情况的 gomock.Call 会存储在这里
        * 调用次数 >= gomock.Call.maxCalls
        * 某个 gomock.Call 被匹配到了，并且其依赖的调用次数都 >= gomock.Call.minCalls，那么这些依赖都会被放到 callSet.exhausted
* finished：标记测试执行完了

```go
type Controller struct {
   // T should only be called within a generated mock. It is not intended to
   // be used in user code and may be changed in future versions. T is the
   // TestReporter passed in when creating the Controller via NewController.
   // If the TestReporter does not implement a TestHelper it will be wrapped
   // with a nopTestHelper.
   T             TestHelper
   mu            sync.Mutex
   expectedCalls *callSet
   finished      bool
}

// callSet represents a set of expected calls, indexed by receiver and method
// name.
type callSet struct {
	// Calls that are still expected.
	expected map[callSetKey][]*Call
	// Calls that have been exhausted.
	exhausted map[callSetKey][]*Call
}

// callSetKey is the key in the maps in callSet
type callSetKey struct {
	receiver interface{}
	fname    string
}
```

一个具体的调用例子：

```go
func TestMockFoo(t *testing.T) {
   ctrl := gomock.NewController(t)
   mockFoo := NewMockFoo(ctrl)
   mockFoo.EXPECT().
      Say("foo", []string{"a", "b"}). // 创建匹配 Say("foo", []string{"a", "b"}) 的 gomock.Call
      Do(func(arg string, arg2 []string) string { // 只执行，不会使用其返回值
         t.Logf("Do(%s, %v)", arg, arg2)
         return arg
      }).
      SetArg(1, []string{"c", "d"}). // 修改第 2 个参数
      DoAndReturn(func(arg string, arg2 []string) string { // 执行，并使用其返回值
         t.Logf("DoAndReturn(%s, %v)", arg, arg2)
         return arg
      }).
      Return("Return"). // 设置返回值为 "Return"
      AnyTimes()
   t.Log(mockFoo.Say("foo", []string{"a", "b"})) // 调用 Say，从 gomock.Controller.expectedCalls 中查找匹配的 gomock.Call，顺序调用 gomock.Call.actions
}
```

执行结果：

```
=== RUN   TestMockFoo
    main_test.go:15: Do(foo, [a b])
    main_test.go:20: DoAndReturn(foo, [c d])
    main_test.go:25: Return
--- PASS: TestMockFoo (0.00s)
PASS
```

## gostub 的实现

https://github.com/prashantv/gostub 使用 reflect 包实现，只能 mock 变量。比如需要 mock 一个方法时，需要这么写：

```go
bar := Bar{field: "field"}
say := bar.Say // 必须把方法复制给一个变量
stub = gostub.StubFunc(&say, "bar")
println(bar.Say("foo"), say("foo")) // fieldfoo bar
stub.Reset()
println(bar.Say("foo"), say("foo")) // fieldfoo fieldfoo
```

gostub.Stubs 字段说明：
* stubs：以 reflect.ValueOf(varToStub) 作为 KEY，Value 保存原始的值
* origEnv：存储原始的环境变量值，用于 mock 环境变量

```go
// Stubs represents a set of stubbed variables that can be reset.
type Stubs struct {
   // stubs is a map from the variable pointer (being stubbed) to the original value.
   stubs   map[reflect.Value]reflect.Value
   origEnv map[string]envVal
}
```

gostub.Stubs 用于 mock 的方法只有如下 3 个：
* Stub(varToStub interface{}, stubVal interface{})：修改 varToStub 的值为 stubVal
* StubFunc(funcVarToStub interface{}, stubVal ...interface{})：设置 funcVarToStub 的返回值为 stubVal...
* SetEnv(k, v string)：设置环境变量

StubFunc 的实现很简单，最后还是调用的 Stub：
* funcVarToStub：必须为指针函数类型
* FuncReturning：使用 reflect.MakeFunc 创建一个函数，返回 stubVal...

```go
// StubFunc replaces a function variable with a function that returns stubVal.
// funcVarToStub must be a pointer to a function variable. If the function
// returns multiple values, then multiple values should be passed to stubFunc.
// The values must match be assignable to the return values' types.
func (s *Stubs) StubFunc(funcVarToStub interface{}, stubVal ...interface{}) *Stubs {
   funcPtrType := reflect.TypeOf(funcVarToStub)
   if funcPtrType.Kind() != reflect.Ptr ||
      funcPtrType.Elem().Kind() != reflect.Func {
      panic("func variable to stub must be a pointer to a function")
   }
   funcType := funcPtrType.Elem()
   if funcType.NumOut() != len(stubVal) {
      panic(fmt.Sprintf("func type has %v return values, but only %v stub values provided",
         funcType.NumOut(), len(stubVal)))
   }

   return s.Stub(funcVarToStub, FuncReturning(funcPtrType.Elem(), stubVal...).Interface())
}
```

Stub 的实现也不复杂：

```go
// Stub replaces the value stored at varToStub with stubVal.
// varToStub must be a pointer to the variable. stubVal should have a type
// that is assignable to the variable.
func (s *Stubs) Stub(varToStub interface{}, stubVal interface{}) *Stubs {
   v := reflect.ValueOf(varToStub)
   stub := reflect.ValueOf(stubVal)

   // varToStub 必须为变量的指针
   if v.Type().Kind() != reflect.Ptr {
      panic("variable to stub is expected to be a pointer")
   }

   if _, ok := s.stubs[v]; !ok {
      // 存储 varToStub 原始值
      s.stubs[v] = reflect.ValueOf(v.Elem().Interface())
   }

   // 设置为新的值
   // *varToStub = stubVal
   v.Elem().Set(stub)
   return s
}
```

一个具体的调用例子：

```go
type Bar struct {
   field string
}

func (b Bar) Say(arg string) string {
   return b.field + arg
}

func fn(i int) int {
   return i+1
}

func main() {
   foo := "foo"
   stub := gostub.Stub(&foo, "stubFoo")
   println(foo) // stubFoo
   stub.Reset()
   println(foo) // foo

   slice := []int{1, 2}
   stub = gostub.Stub(&slice, []int{3})
   fmt.Println(slice) // [3]
   stub.Reset()
   fmt.Println(slice) // [1 2]

   ff := fn
   stub = gostub.Stub(&ff, func(i int) int{return 0})
   println(ff(1)) // 0
   stub.Reset()
   println(ff(1)) // 2

   stub = gostub.StubFunc(&ff, 1)
   println(ff(1)) // 1
   stub.Reset()
   println(ff(1)) // 2

   bar := Bar{field: "field"}
   say := bar.Say
   stub = gostub.StubFunc(&say, "bar")
   println(bar.Say("foo"), say("foo")) // fieldfoo bar
   stub.Reset()
   println(bar.Say("foo"), say("foo")) //fieldfoo fieldfoo

   os.Setenv("GOSTUB_VAR", "value")
   stub = gostub.New()
   stub.SetEnv("GOSTUB_VAR", "stub_value")
   println(os.Getenv("GOSTUB_VAR")) // stub_value
   stub.Reset()
   println(os.Getenv("GOSTUB_VAR")) // value
}
```

## gomonkey 的实现

https://github.com/agiledragon/gomonkey 对变量的 mock 实现原理跟 gostub 一样都是通过 reflect 包实现的。除了 mock 变量，gomonkey 还可以直接 mock 导出函数/方法、mock 代码所在包的非导出函数。

gomonkey 提供了如下 mock 方法：

* ApplyGlobalVar(target, double interface{})：使用 reflect 包，将 target 的值修改为 double
* ApplyFuncVar(target, double interface{})：检查 target 是否为指针类型，与 double 函数声明是否相同，最后调用 ApplyGlobalVar
* ApplyFunc(target, double interface{})：修改 target 的机器指令，跳转到 double 执行
* ApplyMethod(target reflect.Type, methodName string, double interface{})：修改 method 的机器指令，跳转到 double 执行
* ApplyFuncSeq(target interface{}, outputs []OutputCell)：修改 target 的机器指令，跳转到 gomonkey 生成的一个函数执行，每次调用会顺序从 outputs 取出一个值返回
* ApplyMethodSeq(target reflect.Type, methodName string, outputs []OutputCell)：修改 target 的机器指令，跳转到 gomonkey 生成的一个方法执行，每次调用会顺序从 outputs 取出一个值返回
* ApplyFuncVarSeq(target interface{}, outputs []OutputCell)：gomonkey 生成一个函数顺序返回 outputs 中的值，调用 ApplyGlobalVar

#### Apply*Seq 的实现

getDoubleFunc 会通过 reflect 包创建一个函数，每次调用会顺序返回 outputs 中的值。

```go
func getDoubleFunc(funcType reflect.Type, outputs []OutputCell) reflect.Value {
    // 判断返回值个数是否正确
   if funcType.NumOut() != len(outputs[0].Values) {
      panic(fmt.Sprintf("func type has %v return values, but only %v values provided as double",
         funcType.NumOut(), len(outputs[0].Values)))
   }

    // 构造返回结果集
   slice := make([]Params, 0)
   for _, output := range outputs {
      t := 0
      if output.Times <= 1 { // 每个值至少被返回一次
         t = 1
      } else {
         t = output.Times
      }
      for j := 0; j < t; j++ { // 根据返回次数将值追加到结果集中
         slice = append(slice, output.Values)
      }
   }

   i := 0 // 调用次数统计
   len := len(slice)
   return reflect.MakeFunc(funcType, func(_ []reflect.Value) []reflect.Value {
      if i < len {
         i++
         return GetResultValues(funcType, slice[i-1]...)
      }
      panic("double seq is less than call seq")
   })
}
```

#### 指令替换的实现

通过将函数开头的机器指令替换为无条件JMP指令，跳转到 mock 函数执行。要实现这个功能，需要分三步走：

1、获取函数的内存地址

以如下代码为例子：

```go
//go:noinline
func bar() string {
   return "bar"
}

func main() {
   fmt.Println(bar) // 0x10a2e20
   println(unsafe.Pointer(reflect.ValueOf(bar).Pointer())) // 0x10a2e20
}
```

执行命令 `go build -o main . && go tool objdump -s 'bar' main` 查看 bar 函数的内存地址为：0x10a2e20，与程序的输出一致，也就是我们可以使用 reflect 包获取到函数在内存中的地址。

```
TEXT main.bar(SB) /Users/roketyyang/Work/mock/gomonkey/f/f.go
  f.go:11               0x10a2e20               488d05f2450200          LEAQ go.string.*+217(SB), AX    
  f.go:11               0x10a2e27               4889442408              MOVQ AX, 0x8(SP)                
  f.go:11               0x10a2e2c               48c744241003000000      MOVQ $0x3, 0x10(SP)             
  f.go:11               0x10a2e35               c3                      RET 
```

在 gomonkey 中替换指令的实现为：

```go
func (this *Patches) ApplyCore(target, double reflect.Value) *Patches {
   this.check(target, double) // 类型检查
   if _, ok := this.originals[target]; ok {
      panic("patch has been existed")
   }

   this.valueHolders[double] = double // 因为 mock 函数通常是一个闭包，也就是个局部作用域的对象，为了防止 mock 函数被 GC 回收掉，需要增加引用
   // 替换 target 的机器指令，返回的 origin 是 target 会被覆盖的机器指令
   original := replace(*(*uintptr)(getPointer(target)), uintptr(getPointer(double)))
   // 保存 target 被覆盖的机器指令，用于恢复 target
   this.originals[target] = original
   return this
}
```

其中 `*(*uintptr)(getPointer(target))` 为 target 的函数地址，等同于 `target.Pointer()`，getPointer 返回的是指向 target 函数的指针，其实现如下：

```go
type funcValue struct {
	_ uintptr
	p unsafe.Pointer
}

func getPointer(v reflect.Value) unsafe.Pointer {
	return (*funcValue)(unsafe.Pointer(&v)).p
}
```

`reflect.Value` 的结构如下，getPointer 相当于直接拿到了未导出的属性 `reflect.Value.ptr`，这是指向 target 函数的指针，所以要拿到 target 的函数地址，还得进行一次解引用。通过 `target.Pointer()` 可以直接拿到 target 的函数地址是因为 `reflect.Value.Pointer()` 在返回的时候就对 `reflect.Value.ptr`做了一次解引用。

```go
type Value struct {
   typ *rtype
   ptr unsafe.Pointer
   flag
}
```

以如下代码为例子，看下函数变量的值是怎么存储的：

```go
package main

//go:noinline
func foo() string {
   return "foo"
}

func main() {
   funcVar := foo
   println(funcVar())
   funcVar2 := foo
   println(funcVar2())
}
```

查看汇编代码：`go tool compile -S ff.go`，可以看到两个函数变量的调用都是通过把符号 `"".foo·f(SB)` 所指向的内存值放到 AX 寄存器，然后执行 CALL 指令。而 `"".foo·f(SB)` 使用到的内存大小是 8 个字节，并且值为 `"".foo+0`，即函数 foo 的地址，而 `reflect.Value.ptr` 实际上是符号 `"".foo·f(SB)` 的地址。

```go
"".main STEXT size=205 args=0x0 locals=0x28 funcid=0x0
		        ......
        0x0021 00033 (ff.go:10) MOVQ    "".foo·f(SB), AX ; funcVar()
        0x0028 00040 (ff.go:10) LEAQ    "".foo·f(SB), DX
        0x002f 00047 (ff.go:10) PCDATA  $1, $0
        0x002f 00047 (ff.go:10) CALL    AX
		        ......
        0x006f 00111 (ff.go:12) MOVQ    "".foo·f(SB), AX ; funcVar2()
        0x0076 00118 (ff.go:12) LEAQ    "".foo·f(SB), DX
        0x007d 00125 (ff.go:12) CALL    AX
		        ......
"".foo·f SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 "".foo+0
```

2、生成跳转指令

gomonkey 替换指令的代码：

```go
// target 目标函数地址
// double mock 函数的指针
func replace(target, double uintptr) []byte {
	code := buildJmpDirective(double) // 生成跳转到 mock 函数的机器指令
	bytes := entryAddress(target, len(code))
	original := make([]byte, len(bytes))
	copy(original, bytes)
	modifyBinary(target, code)
	return original
}

func buildJmpDirective(double uintptr) []byte {
    d0 := byte(double)
    d1 := byte(double >> 8)
    d2 := byte(double >> 16)
    d3 := byte(double >> 24)
    d4 := byte(double >> 32)
    d5 := byte(double >> 40)
    d6 := byte(double >> 48)
    d7 := byte(double >> 56)

    // 返回跳转的机器指令
    return []byte{
        0x48, 0xBA, d0, d1, d2, d3, d4, d5, d6, d7, // MOV rdx, double 将 mock 函数的指针值放到 rdx 中
        0xFF, 0x22,     // JMP [rdx] 因为rdx 中存储的是 mock 函数的指针，所以需要使用[]，从内存中获得 mock 函数的地址，然后跳转
    }
}
```

buildJmpDirective 的实现是间接近转移，这里其实也可以用直接转移 `JMP rdx`，rdx 中直接放 mock 函数的地址，这样就不需要 getPointer 了。当然也可以通过计算 target 函数地址和 mock 函数地址之间的距离，使用偏移量进行转移。

3、修改函数开头的指令

在 `replace(target, double uintptr) []byte` 中，首先通过 `bytes := entryAddress(target, len(code))` 拿到 target 函数开头 12 字节数据，放在 `[]byte` 变量中。再执行 `copy(original, bytes)` 把这 12 字节数据保存下来，便于之后恢复用。最后执行 `modifyBinary(target, code)` 修改指令：

```go
func modifyBinary(target uintptr, bytes []byte) {
    function := entryAddress(target, len(bytes))
    
    // 默认情况下，代码段的内存页是不可写的，需要调用 Mprotect 修改 target 所在页的权限为可写
    page := entryAddress(pageStart(target), syscall.Getpagesize())
    err := syscall.Mprotect(page, syscall.PROT_READ|syscall.PROT_WRITE|syscall.PROT_EXEC)
    if err != nil {
        panic(err)
    }
    // 替换 target 函数开头的指令为跳转指令
    copy(function, bytes)
    // 恢复为读和执行权限
    err = syscall.Mprotect(page, syscall.PROT_READ|syscall.PROT_EXEC)
    if err != nil {
        panic(err)
    }
}
```

#### super monkey 如何解决 mock 非导出函数/方法

gomonkey 依赖 reflect 包获取目标函数的地址，但如果我们在写测试代码的时候，依赖包中的非导出的函数/方法就没办法 mock 到了。上面分析了 gomonkey 的实现，可以知道最关键的是拿到目标函数的地址，如果有不通过 reflect 包就能拿到目标函数的地址的方法，那么问题就解决了。

https://github.com/cch123/supermonkey 的解决办法就是通过符号表获取到函数的内存地址，supermonkey 读取符号表：

```go
func init() {
   content, _ := nm.Parse(os.Args[0])

   lines := strings.Split(content, "\n")
   for _, line := range lines {
      line := strings.TrimSpace(line)
      arr := strings.Split(line, " ")
      if len(arr) < 3 {
         continue
      }

      funcSymbol, addr := arr[2], arr[0]
      addrUint, _ := strconv.ParseUint(addr, 16, 64) // addrUint 为函数地址
      symbolTable[funcSymbol] = uintptr(addrUint) // funcSymbol 函数符号，比如 supermonkey/pkg.Foo.say
   }
}
```

使用例子，PatchByFullSymbolName 需要传入函数符号，可以通过 `go tool nm -type supermonkey | grep 'Foo'` 命令获取：

```go
func main() {
   f := pkg.Foo{}
   patchGuard := sm.PatchByFullSymbolName("supermonkey/pkg.Foo.say", func() string {
      return "mock say"
   })
   println(f.Say())
   patchGuard.Unpatch()
}
```

supermonkey 只提供了 mock 函数的方法，通常是搭配 gomonkey 使用。

## 总结

在 Golang 中 mock 代码的作用域可见的接口、变量、函数、方法，以及不可见的非导出函数、方法都可以进行 mock 。接口的 mock 建议使用 gomock，其他对象的 mock 可以使用 gomonkey + supermonkey 的组合。

参考文章：

* [Monkey Patching in Go](https://bou.ke/blog/monkey-patching-in-go/)
* [golang实现运行时替换函数体及其原理](https://berryjam.github.io/2018/12/golang%E6%9B%BF%E6%8D%A2%E8%BF%90%E8%A1%8C%E6%97%B6%E5%87%BD%E6%95%B0%E4%BD%93%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86/)