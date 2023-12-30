---
title: "Go by Example"
date: 2023-12-10T13:36:32+08:00
draft: false
---

# 重归基础
通过例子来学习Go  
英文版本：https://gobyexample.com/  
中文版本：https://gobyexample-cn.github.io/

# 笔记

## 迭代和递归
重新感受一下斐波那契数列的递归和迭代算法性能差距
```go
// feb.go
func FebRecursive(n int) int {
	if n == 0 {
		return 0
	}
	if n == 1 {
		return 1
	}
	return FebRecursive(n-1) + FebRecursive(n-2)
}

func FebIterative(n int) int {
	if n == 0 {
		return 0
	}

	a, b := 0, 1
	for i := 1; i < n; i++ {
		b = a + b
		a = b - a
	}
	return b
}

// feb_test.go
func Benchmark_FebIterative(b *testing.B) {
	for i := 0; i < b.N; i++ {
		FebIterative(30)
	}
}

func Benchmark_FebRecursive(b *testing.B) {
	for i := 0; i < b.N; i++ {
		FebRecursive(30)
	}
}


```

## 字符串和rune
调用 len 得到字节长度
for range 是按照rune来遍历



> 下面内容引用自 https://liyucang-git.github.io/2019/06/17/%E5%BD%BB%E5%BA%95%E5%BC%84%E6%87%82Unicode%E7%BC%96%E7%A0%81/

它从 0 开始，为每个符号指定一个编号，这叫做”码点”（code point）。比如，码点 0 的符号就是 null（表示所有二进制位都是 0）。

U+0000 = null
上式中，U+表示紧跟在后面的十六进制数是 Unicode 的码点。

这么多符号，Unicode 不是一次性定义的，而是分区定义。每个区可以存放 65536 个（2^16）字符，称为一个平面（plane）。目前，一共有 17 个平面，也就是说，整个 Unicode 字符集的大小现在是 2^21。

最前面的 65536 个字符位，称为基本平面（缩写 BMP），它的码点范围是从 0 一直到 2^16-1，写成 16 进制就是从 U+0000 到 U+FFFF。所有最常见的字符都放在这个平面，这是 Unicode 最先定义和公布的一个平面。

剩下的字符都放在辅助平面（缩写 SMP），码点范围从 U+010000 一直到 U+10FFFF。

Go 源码文件默认采用 Unicode 字符集，Unicode 码点和内存中字节序列的变换实现使用了 UTF-8

## 范型

https://segmentfault.com/a/1190000041634906

## 池

思路：协程池和多路I/O复用

实现参考: https://github.com/bytedance/gopkg/tree/develop/util/gopool

高并发场景下，使用协程池来减少协程创建的开销，减少内存分配的频率

模仿(照抄)上面的gopool实现了一个协程池：https://github.com/binc4t/wpool

主要组件：
task pool  
worker pool

1. 从task pool取出task，赋值一个执行func, 放入队列，这里用单向链表实现队列
2. 查看task队列的task数量，超过阈值则从worker pool中取得一个worker
3. worker main 循环，不断获取task并执行，如果没有task，则回收这个worker，放入worker pool