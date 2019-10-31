---
layout: post
title: "golang slice底层实现"
description: "golang slice底层实现"
category: GoLang
tags: []
---
{% include JB/setup %}

本文内容基于go1.13.1源码。

通过`make`给局部变量分配空间时，如果空间较少，则会直接在栈上分配 slice 的 header 和 数组，否则会调用`runtime.makeslice`。

slice 结构：

```go
type SliceHeader struct {
	Data uintptr // 指向连续内存数组
	Len  int
	Cap  int
}
```

<!--more-->

#### 初始化

```go
// et 是元素的类型
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	// 判断 len 和 cap 参数是否合法
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

    // 在堆上分配一段连续的内存
	return mallocgc(mem, et, true)
}
```

#### 元素的赋值/访问

汇编实现，直接计算出元素的内存地址。

#### 追加操作

如果 cap 足够，则会直接计算出元素的内存地址并赋值，然后修改 len。

如果 cap 不足，则会调用`runtime.growslice`进行扩容：

* 先按 doublecap = 2 * oldSlice.cap 扩容一倍
    *如果大于等于 newSlice.cap，则
        * 当 oldSlice.len < 1024 时，按 doublecap 扩容
        * 当 oldSlice.len > 1024 时，每次扩容1/4倍的 oldSlice.cap，直到满足 newSlice.cap
            * 如果出现溢出，则按 newSlice.cap 扩容
    * 如果小于 newSlice.cap，则按 newSlice.cap 扩容
* 如果确定slice大小应该预先申请好，因为扩容的时候是需要复制整个数组内存的

```go
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

    // 先按旧的slice容量翻倍，如果还不满足实际需要的容量，则按照实际需要的容量扩容
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
	    // 判断旧数组的长度是否小于1024，如果是的话就按旧数组的容量扩容一倍
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// 判断上面newcap的加法是否溢出了
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	// 根据元素类型的大小，选择对应的计算逻辑，节省计算量
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
	    // 2的倍数，可以通过位移计算
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

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	// 分配内存，这里还有一些跟 GC 相关的逻辑
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	// 将旧数组的数据复制到新新数组中
	memmove(p, old.array, lenmem)

    // 返回新的slice
	return slice{p, old.len, newcap}
}
```

#### 有趣的关于 slice 的题目

```go
package main

import "fmt"

func main() {
	a := make([]int, 0, 3)
	b := append(a, 1)
	_ = append(a, 2)
	fmt.Println(b[0])

	c := make([]int, 3, 3)
	fmt.Println(cap(c[1:]))
	fmt.Println(cap(c[:1]))
}
```

上面这段代码的输出是：

```text
2 // 因为 a 的长度是 0，所以每次 append 修改的是底层数组的第一个元素
2 // 进行切片操作，新的 slice.cap = oldSlice.cap - 切片开始索引
3
```