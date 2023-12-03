---
title: "Go有自动装箱吗？"
date: 2023-12-03T06:51:00+08:00
draft: false
toc: false
images:
tags:
  - Go
  - asm
---

# 前言
Go是一种强类型语言，在大部分情况下只有相同的变量才可以互相赋值。但同时Go也存在类似Java中Object定位的"万能引用"--interface{}。任何基本类型都可以被转换为该类型，也可以由该类型再次转换回对应的基本类型。相类似的，Java中也存在着基本类型到包装类型的转换如int到Integer的转换。那么Go中是否也存在像java一样的自动装箱呢？
# interface\{\}
作为Go语言动态性的基础，interface{}本身的实现并不复杂。
```Go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}

type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```
可以看出，interface{}真正的内存表示eface其实只包含两部分：  
1. _type,指向真实类型的指针
2. data,指向真实数据的指针  

了解完底层结构后，自动装箱的问题的答案便也呼之欲出：Go确实是有自动装箱机制的，可以将基本类型转换为interface{}。那另一个问题也接踵而来，装箱又是通过什么手段实现呢。
# Go Assembly
为了获得这个问题的答案，我们需要直接观察编译出的汇编语言。
```Go
package main

func main() {
	k := 123
	var p interface{} = k
	print(p)
}
```
使用命令`go build -gcflags=all="-N -l"`编译上述代码，之后使用gdb进行断点调试
```gdb
(gdb) disas
Dump of assembler code for function main.main:
=> 0x0000000000460ee0 <+0>:     cmp    0x10(%r14),%rsp
   0x0000000000460ee4 <+4>:     jbe    0x460f43 <main.main+99> \\ 检查是否需要扩展栈
   0x0000000000460ee6 <+6>:     sub    $0x38,%rsp \\ 栈顶指针rsp减少48byte
   0x0000000000460eea <+10>:    mov    %rbp,0x30(%rsp) \\ 存储父caller的栈底指针rbp
   0x0000000000460eef <+15>:    lea    0x30(%rsp),%rbp \\ 计算新的栈底指针存入rbp
   0x0000000000460ef4 <+20>:    movq   $0x7b,0x10(%rsp) \\ 给k赋值123
   0x0000000000460efd <+29>:    movq   $0x7b,0x18(%rsp) \\ 给interface{}赋值int等价于重新声明了一个int 同样赋值
   0x0000000000460f06 <+38>:    lea    0x4553(%rip),%rax        # 0x465460 \\ int类型*_type地址
   0x0000000000460f0d <+45>:    mov    %rax,0x20(%rsp) \\ 填入栈中eface的第一个成员
   0x0000000000460f12 <+50>:    lea    0x18(%rsp),%rax \\ 指向底层数据的指针
   0x0000000000460f17 <+55>:    mov    %rax,0x28(%rsp) \\ 填入栈中eface的第二个成员
   0x0000000000460f1c <+60>:    nopl   0x0(%rax)
   0x0000000000460f20 <+64>:    call   0x434820 <runtime.printlock> \\ 函数调用
   0x0000000000460f25 <+69>:    mov    0x20(%rsp),%rax
   0x0000000000460f2a <+74>:    mov    0x28(%rsp),%rbx
   0x0000000000460f2f <+79>:    call   0x435260 <runtime.printeface>
   0x0000000000460f34 <+84>:    call   0x4348a0 <runtime.printunlock>
   0x0000000000460f39 <+89>:    mov    0x30(%rsp),%rbp
   0x0000000000460f3e <+94>:    add    $0x38,%rsp
   0x0000000000460f42 <+98>:    ret    
   0x0000000000460f43 <+99>:    call   0x45db60 <runtime.morestack_noctxt>
   0x0000000000460f48 <+104>:   jmp    0x460ee0 <main.main>
End of assembler dump.
```
通过对汇编的观察，我们可以看出0x465460即为内存中int对应的*_type的地址 通过gdb我们可以获取到如下的内容
```gdb
(gdb) x /56b 0x465460
0x465460:       0x08    0x00    0x00    0x00    0x00    0x00    0x00    0x00
                size: 8byte
0x465468:       0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
                doesn't contain ptr value
0x465470:       0x92    0x57    0x73    0xcb    0x0f    0x08    0x08    0x02
                hash:0xcd735792                 tflag   align fieldAlign kind=int
0x465478:       0x68    0x48    0x47    0x00    0x00    0x00    0x00    0x00
                equalFunc: 0x00474868
0x465480:       0x30    0x69    0x47    0x00    0x00    0x00    0x00    0x00
0x465488:       0x4e    0x01    0x00    0x00    0xe0    0x29    0x00    0x00
```
可以看出 这确实是int对应的_type
所以确实 Go确实会对转成interface{}的基本类型进行装箱

