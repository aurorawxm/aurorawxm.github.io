---
title: "Go for 循环踩坑记录"
date: 2023-05-09T05:47:00Z
draft: false
---

### 案例一：取地址符
在 Go 语言中，我们写 for 语句时有时会出现运行和猜想的结果不一致。例如以下第一个案例的代码：
```go
var all []*Item
for _, item := range items {
	all = append(all, &item)
}
```

这段代码有问题吗？变量 all 内的 item 变量，存储进去的是什么？ 是每次循环的 item 值吗？
实际上在 for 循环时，每次存入变量 all 的都是相同的 item，也就是最后一个循环的 item 值。
这是 Go 面试里经常出现的题目，结合 goroutine 更风骚，毕竟还会存在乱序输出等问题。
如果你想解决这个问题，就需要把程序改写成如下：
```go
var all []*Item
for _, item := range items {
	item := item
	all = append(all, &item)
}
```
要重新声明一个 item 变量把 for 循环的 item 变量给存储下来再追加进去。

### 案例二：闭包函数
接下来是第二个案例的代码：
```go
var prints []func()
for _, v := range []int{1, 2, 3} {
	prints = append(prints, func() { fmt.Println(v) })
}
for _, print := range prints {
	print()
}
```
这段程序的输出结果是什么？没有 & 取地址符，是输出 1，2，3 吗？
输出结果是 3，3，3。这又是为什么？
问题的重点之一，关注到闭包函数，实际上所有闭包都打印的是相同的 v。输出 3，是因为在 for 循环结束后，最后 v 的值被设置为了 3，仅此而已。
如果想要达到预期的效果，依然是使用万能的再赋值。改写后的代码如下：
```go
for _, v := range []int{1, 2, 3} {
	v := v
	prints = append(prints, func() { fmt.Println(v) })
}
```
增加 v := v 语句，程序输出结果为 1，2，3。 仔细翻翻你写过的 Go 工程，是不是都很熟悉？就这改造方法，赢了。

尤其是配合上 Goroutine 的写法，很多同学会更容易在此翻车。

### 修复思路
实际上 Go 核心团队在内部和社区已经讨论过许久，希望重新定义 for 循环的语法。要达到的目的是：使循环变量每次迭代而不是每次循环。
解决的办法是：在每个迭代变量 x 的每个循环体开头，加一个隐式的再赋值，也就是 x := x，就能够解决上述程序中所隐含的坑。和我们现在做的一样，只不过我们是自己手动加的，Go 团队做的是希望在编译器内隐式处理。

### 让用户自己决定
比较尴尬的是 Go 团队在 [Proposal: Go 2 transition](https://github.com/golang/proposal/blob/master/design/28221-go2-transitions.md#language-changes) 中禁止重新定义语言，所以 rsc 不能直接这么干。
因此将会由用户自己决定控制这个 “破坏”，方式将会是根据每个包的 go.mod 文件中的 go 行更改语义。
如果我们是在 Go1.30 对本文讨论的 for 循环改为迭代，那么在 go.mod 文件中的 go 版本声明是将是一个关键

### 总结
for 循环时的变量问题，一直是各大 Go 考官爱考的题目，另外也确实在实际编程 Go 代码时会遇到这类坑。
虽然 rsc 希望在 go.mod 文件上开创先河，利用 go 版本的声明，去修改语义（不允许添加和删除）。这无疑是给 Go1 兼容性保障开了一个后门。
如果实施，本次变更会导致 Go 的前后版本语义有所不同。还不如变成一个 go.mod 文件的一个语义开关。
这显然是一个很折腾的思考题。