# 10 cgo

「cgo 不是银弹」，cgo 是连接 Go 与 C （乃至其他任何语言）之间的桥梁。
cgo 性能远不及原生 Go 程序的性能，执行一个 cgo 调用的代价很大。
下图展示了 cgo, go, c 之间的性能差异（网络 I/O 场景）：

![](../images/cgo-go-c.png)

**图1: cgo/Go/C/net 包 在网络 I/O 场景下的性能对比，图取自 [changkun/cgo-benchmarks](https://github.com/changkun/cgo-benchmarks)**

本文则具体研究 cgo 在运行时中的实现方式。

## 入口

先来编写一个最简单的 cgo 程序：

```go
package main

/*
#include "stdio.h"
void print() {
	printf("hellow, cgo");
}
*/
import "C"

func main() {
	C.print()
}
```

我们先观察一下汇编的结果：

```asm
TEXT main._Cfunc_print(SB) _cgo_gotypes.go
  _cgo_gotypes.go:40	0x40503c0		65488b0c2530000000	MOVQ GS:0x30, CX				
  (...)
  _cgo_gotypes.go:41	0x40503e7		488b053aca0600		MOVQ main._cgo_fd63072f180f_Cfunc_print(SB), AX	
  _cgo_gotypes.go:41	0x40503ee		48890424		MOVQ AX, 0(SP)					
  _cgo_gotypes.go:41	0x40503f2		488b442418		MOVQ 0x18(SP), AX				
  _cgo_gotypes.go:41	0x40503f7		4889442408		MOVQ AX, 0x8(SP)				
  _cgo_gotypes.go:41	0x40503fc		e8ff3bfbff		CALL runtime.cgocall(SB)			
  (...)

TEXT main.main(SB) /Users/changkun/dev/go-under-the-hood/demo/10-cgo/main.go
  main.go:11		0x4050420		65488b0c2530000000	MOVQ GS:0x30, CX			
  (...)
  main.go:12		0x405043b		e880ffffff		CALL main._Cfunc_print(SB)		
  (...)	
```

说明 Go 代码在进入 C 代码前，最终以用编译器配合的形式，进入了运行时的 `runtime.cgocall`。

再来看一下整个编译过程中的临时文件，临时文件中的入口文件为 `main.cgo1.go`：

```go
// Code generated by cmd/cgo; DO NOT EDIT.

//line main.go:1:1
package main

/*
#include "stdio.h"
void print() {
	printf("hellow, cgo");
}
*/
import _ "unsafe"

func main() {
	(_Cfunc_print)()
}
```

可以看到 Go 编译器会将我们原有的 cgo 调用替换为：`_Cfunc_print`。
我们可以在 `_cgo_gotypes.go` 中看到这个函数的定义：

```go
//go:cgo_unsafe_args
func _Cfunc_print() (r1 _Ctype_void) {
    // 调用 _cgo_runtime_cgocall 传递 C 函数的入口地址以及相关参数
	_cgo_runtime_cgocall(_cgo_222b4724d882_Cfunc_print, uintptr(unsafe.Pointer(&r1)))
	if _Cgo_always_false {
	}
	return
}
```

而 `_cgo_runtime_cgocall` 的定义：

```go
//go:linkname _cgo_runtime_cgocall runtime.cgocall
func _cgo_runtime_cgocall(unsafe.Pointer, uintptr) int32
```

可以看到编译器通过编译标志 `go:linkname` 将这个调用链接为了 `runtime.cgocall`。
因此，从 Go 进入 C 空间的 cgo 调用，以 Go 程序为主体（运行时依然存在），通过编译器的配合，
当需要调用 C 代码时，会向运行时传递 C 函数的入口地址及所需传递的参数。

那么剩下的工作就是去分析 `runtime.cgocall` 这个调用如何与 Go 运行时进行交互了。

## `cgocall`

### 原理概述

#### Go 调用 C

为从 Go 调用 C 函数 f，cgo 生成的代码调用 `runtime.cgocall(_cgo_Cfunc_f, frame)`,
其中 `_cgo_Cfunc_f` 为由 cgo 编写的并由 gcc 编译的函数。

`runtime.cgocall` 会调用 `entersyscall`，从而不会阻塞其他 goroutine 或垃圾回收器
而后调用 `runtime.asmcgocall(_cgo_Cfunc_f, frame)`。

`runtime.asmcgocall` 会切换到 m->g0 栈(操作系统分配的栈，因此能安全的在运行 gcc 编译的代码) 
并调用 `_cgo_Cfunc_f(frame)`。
`_cgo_Cfunc_f` 获取了帧结构中的参数，调用了实际的 C 函数 f，在帧中记录其结果，
并返回到 `runtime.asmcgocall`。
在重新获得控制权后，`runtime.asmcgocall` 会切换回原来的 g (`m->curg`) 的执行栈
并返回 `runtime.cgocall`。
在重新获得控制权后，`runtime.cgocall` 会调用 `exitsyscall`，并阻塞，直到该 m 运行能够在不与
`$GOMAXPROCS` 限制冲突的情况下运行 Go 代码。

```
Go --> runtime.cgocall --> runtime.entersyscall --> runtime.asmcgocall --> _cgo_Cfunc_f
                                                                                 |
                                                                                 |
Go <-- runtime.exitsyscall <-- runtime.cgocall <-- runtime.asmcgocall <----------+
```

#### C 调用 Go

上面的描述跳过了当 gcc 编译的函数 f 调用回 Go 的情况。如果此类情况发生，则下面描述了 f 执行期间的调用过程。

为了 gcc 编译的 C 代码调用 Go 函数 `p.GoF` 成为可能，cgo 编写了以 `GoF` 命名的 gcc 编译的函数
（不是 `p.GoF`，因为 gcc 没有包的概念）。然后 gcc 编译的 C 函数 f 调用 `GoF`。
GoF 调用了 `crosscall2(_cgoexp_GoF, frame, framesize)`，而
Crosscall2（gcc 编译的汇编文件）为一个具有两个参数的从 gcc 函数调用 ABI 到 6c 函数调用 API 的适配器。
gcc 通过调用它来调用 6c 函数。这种情况下，它会调用 `_cgoexp_GoF(frame, framesize)`，
仍然会在 m->g0 栈上运行，且不受 `$GOMAXPROCS` 的限制。因此该代码不能直接调用任意的 Go 代码，
并且必须非常小心的分配内存以及小心的使用 m->g0 栈。

`_cgoexp_GoF` 调用了 `runtime.cgocallback(p.GoF, frame, framesize, ctxt)`。
（使用 `_cgoexp_GoF` 而不是编写 `crosscall3` 直接进行此调用的原因是 `_cgoexp_GoF`
是用 6c 而不是 gcc 编译的，可以引用像 runs.cgocallback 和 p.GoF 这样的带点的名称。）
`runtime.cgocallback` 从 `m->g0` 的堆切换到原始 g（`m->curg`）的栈，
并在在栈上调用 `runtime.cgocallbackg(p.GoF，frame，framesize)`。
作为栈切换的一部分，`runtime.cgocallback` 将当前 SP 保存为 `m->g0->sched.sp`，
因此在执行回调期间任何使用 `m->g0` 的栈都将在现有栈帧之下完成。
在覆盖 `m->g0->sched.sp` 之前，它会在 `m->g0` 栈上将旧值压栈，以便以后可以恢复。

`runtime.cgocallbackg` 现在在一个真正的 goroutine 栈上运行（不是 `m->g0` 栈）。
首先它调用 `runtime.exitsyscall`，它将阻塞到不与 `$GOMAXPROCS` 限制冲突的情况下运行此 goroutine。
一旦 `exitsyscall` 返回，就可以安全地执行调用内存分配器或调用 Go 的 `p.GoF` 回调函数等操作。

`runtime.cgocallbackg` 首先推迟一个函数来 unwind `m->g0.sched.sp`，这样如果 `p.GoF` 发生 panic
`m->g0.sched.sp` 将恢复到其旧值：`m->g0` 栈和 `m->curg` 栈将在 unwind 步骤中展开。
接下来它调用 `p.GoF`。最后它弹出但不执行 defer 函数，而是调用 `runtime.entersyscall`，
并返回到 `runtime.cgocallback`。
在重新获得控制权后，`runtime.cgocallback` 切换回 `m->g0` 栈（指针仍然为 `m->g0.sched.sp`），
从栈中恢复原来的 `m->g0.sched.sp` 的值，并返回到 `_cgoexp_GoF`。
`_cgoexp_GoF` 直接返回 `crosscall2`，从而为 gcc 恢复调用方寄存器，并返回到 `GoF`，从而返回到 `f` 中。

```
f --> GoF --> crosscall2 --> _cgoexp_GoF --> runtime.cgocallbackg --> runtime.cgocallback --> runtime.exitsyscall --> p.GoF
                                                                                                                        |
                                                                                                                        |
f <-- GoF <-- crosscall2 <-- _cgoexp_GoF <-- runtime.cgocallback <-- runtime.entersyscall <-----------------------------+
```

## 实际代码

下面我们来仔细讨论这些涉及的调用。

TODO:

## 进一步阅读的参考文献

1. [Command cgo](https://golang.org/cmd/cgo/)
2. [LINUX SYSTEM CALL TABLE FOR X86 64](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)

## 许可

[Go under the hood](https://github.com/changkun/go-under-the-hood) | CC-BY-NC-ND 4.0 & MIT &copy; [changkun](https://changkun.de)