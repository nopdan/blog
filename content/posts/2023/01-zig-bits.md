---
title: "Zig Bits 0x1: 从函数返回切片"
date: 2023-03-24T03:48:00+08:00
# categories: ["zig"]
# tags: ["zig"]
series: ["zig"]
# draft: true
---

> 原文：<https://blog.orhun.dev/zig-bits-01/>

我决定开始一个新的博客系列，名为 "Zig Bits"，我会分享关于 [Zig 程序设计语言](https://ziglang.org/)。这个系列将特别为初学者，因为我也是一个初学者。

<!-- more -->

<center>

<img src="/zig/ziggy_fly.svg" style="width: 25%"/>

</center>

由于这是本系列的第一部分，让我先介绍一些关于 Zig 的信息和我的背景。

`Zig` 是一种命令式的、通用的、静态类型的、编译的系统编程语言和工具链，用于维护健壮的、最佳的和可重用的软件。

- _健壮:_ 即使在内存不足等边缘情况下，行为也是正确的。
- _最佳:_ 以程序的最佳行为和执行方式编写程序。
- _可重用:_ 相同的代码可以在具有不同约束的许多环境中工作。
- _可维护:_ 将意图准确地传达给编译器和其他程序员。这种语言的代码阅读开销很低，并且能够适应不断变化的需求和环境。

它支持编译时泛型、反射和求值、交叉编译和手动内存管理。Zig 的一个主要目标是改进 [C 语言](<https://en.wikipedia.org/wiki/C_(programming_language)>)，同时也从 [Rust](https://www.rust-lang.org/) 和其他语言中获得灵感。

有很多资源可以学习 Zig，主要是：

- [Zig Documentation](https://ziglang.org/documentation/master/)
- [Zig `std` Reference](https://ziglang.org/documentation/master/std/#A;std)
- [Ziglearn.org](https://ziglearn.org/)
- [Zig Crash Course](https://ikrima.dev/dev-notes/zig/zig-crash-course/)
- [Zig Strings in 5 minutes](https://www.huy.rocks/everyday/01-04-2022-zig-strings-in-5-minutes)

我经常使用 `Rust` 开发，我决定学习 Zig 是因为这种语言的低级引发了我的兴趣。我一直喜欢写 C，我肯定会考虑的学习一些更健壮的东西。我已经在业余时间编写了我的第一个 Zig 项目（来自其他开源项目），我真的很喜欢它！

好了，是时候开始重头戏了。

### 在 Zig 中返回切片

<img src="/zig/ziggy_fix.svg" style="width: 25%"/>

下面是演示的代码片段：

```zig
// Zig version: 0.10.1

// 引入标准库
const std = @import("std");

/// 返回一个切片
fn zigBits() []u8 {
    // 创建一个字符数组
    var message = [_]u8{ 'z', 'i', 'g', 'b', 'i', 't', 's' };

    // 以字符串形式打印
    std.log.debug("{s}", .{message});

    // 我们需要使用取地址运算符 `&` 强制转为切片类型 '[]u8'.
    return &message;
}

/// 程序入口
pub fn main() void {
    // Get the message.
    const message = zigBits();

    // Print the message.
    std.log.debug("{s}", .{message});
}
```

**Q**: Cool! 所以我们只是从函数中返回一个切片并打印它的值？

对! 我们预计会看到两次 `zigbits` . 一个来自函数内部，一个来自 `main`.

让我们运行它:

```sh
$ zig build run

debug: zigbits
debug: �,�$
```

嗯... 怎么回事？这不符合预期。再运行一次？

```sh
$ zig build run

debug: zigbits
debug:
       �;�
```

什么?! 用 `u8` 数字打印看看？

```diff
- std.log.debug("{s}", .{message});
+ std.log.debug("{d}", .{message});
```

```sh
$ zig build run

debug: { 122, 105, 103, 98, 105, 116, 115 }
debug: { 80, 129, 179, 51, 255, 127, 0 }
```

它们是不一样的两个数组！这里发生了什么？

### 解释

<img src="/zig/ziggy_think.svg" style="width: 25%"/>

注意这一行：

```zig
return &message;
```

这里我们实际上是返回了一个 **栈-分配** 数组的一个切片。

```zig
try std.testing.expect(@TypeOf(&message) == *[7]u8);
```

由于这个数组是被分配在 _栈_ 上的， 所以当它被释放时，也就是当我们从函数返回时，它可能会被破坏。文档的 [生存期和所有权](https://ziglang.org/documentation/master/#Lifetime-and-Ownership) 部分对此进行了说明:

> Zig 程序员有责任确保当指针指向的内存不再可用时，指针不会被访问。请注意，切片是指针的一种形式，因为它引用其他内存。

这就是当我们从函数返回后试图打印数组内容时会出现随机乱码的原因。

此外，[`ziglang/zig`](https://github.com/ziglang/zig) 仓库中有一个关于此的问题，它指出这种情况应该导致编译错误：[https://github.com/ziglang/zig/issues/5725](https://github.com/ziglang/zig/issues/5725)

### 解决

<img src="/zig/ziggy_cool.svg" style="width: 20%"/>

我们可以通过以下几种方式解决这个问题：

- 将切片作为参数传递给函数
- 使数组成为全局变量
- 分配切片 (返回已分配切片的拷贝) [最常见的]

让我们看看如何与每种解决方案配合使用。

#### 将切片作为参数传递

写入的字节数对 main 来说是未知的，因此如果没有 len，最后的调试日志将打印整个缓冲区，而不仅仅是实际的消息部分。

```zig
// Zig version: 0.10.1

// Import the standard library.
const std = @import("std");

/// Takes a slice as a parameter and fills it with a message.
fn zigBits(slice: []u8) usize {
    // Create an array literal.
    var message = [_]u8{ 'z', 'i', 'g', 'b', 'i', 't', 's' };

    // Print the array as string.
    std.log.debug("{s}", .{message});

    // Update the slice.
    std.mem.copy(u8, slice, &message);
}

/// Entrypoint of the program.
pub fn main() void {
    // Define the message buffer.
    var message: [9]u8 = undefined;

    // Get the message and save the length.
    const len = zigBits(&message);

    // Print the message.
    std.log.debug("{s}", .{message[0..len]});
    std.log.debug("{s}", .{message});
}
```

如你所见，我们已经将函数的返回值改为 `void` 并使其接受切片参数。代替 `return`, 我们使用 [`std.mem.cpy`](https://ziglang.org/documentation/master/std/#A;std:mem.copy) 方法更新切片。

(This 方法类似于在 Rust 中向函数传递可变引用 `&mut` )

让我们运行它:

```sh
$ zig build run

debug: zigbits
debug: zigbits
debug: zigbits�,�$
```

Yay!

#### 使用全局数组

```zig
// Zig version: 0.10.1

// Import the standard library.
const std = @import("std");

// Create a global array literal.
var message = [_]u8{ 'z', 'i', 'g', 'b', 'i', 't', 's' };

/// Returns a slice.
fn zigBits() []u8 {
    // Print the array as string.
    std.log.debug("{s}", .{message});

    // We need to use address-of operator (&) to coerce to slice type '[]u8'.
    return &message;
}

/// Entrypoint of the program.
pub fn main() void {
    // Get the message.
    const msg = zigBits();

    // Print the message.
    std.log.debug("{s}", .{msg});
}
```

我们已经将字符数组声明为全局变量，因此可以从内部作用域对其进行修改和访问。

```sh
$ zig build run

debug: zigbits
debug: zigbits
```

### 分配切片

```zig
// Zig version: 0.10.1

// Import the standard library.
const std = @import("std");

/// Returns a slice.
fn zigBits() ![]u8 {
    // Create an array literal.
    var message = [_]u8{ 'z', 'i', 'g', 'b', 'i', 't', 's' };

    // Print the array as string.
    std.log.debug("{s}", .{message});

    // Allocate the slice on the heap and return.
    var message_copy = try std.heap.page_allocator.dupe(u8, &message);
    return message_copy;
}

/// Entrypoint of the program.
pub fn main() !void {
    // Get the message.
    const message = try zigBits();

    // Print the message.
    std.log.debug("{s}", .{message});
}
```

下一行我们在堆中复制切片：

```zig
var message_copy = try std.heap.page_allocator.dupe(u8, &message);
```

这使得切片在函数外部可用，因为它现在被分配到堆中：

```sh
$ zig build run

debug: zigbits
debug: zigbits
```

处理这种情况的更常见和惯用的方法是将 [`std.mem.Allocator`](https://ziglang.org/documentation/master/std/#A;std:mem.Allocator) 作为分配内存的函数。这样，我们就可以让调用者决定使用哪个分配器。

```zig
// Zig version: 0.10.1

// Import the standard library.
const std = @import("std");

/// Returns a slice.
fn zigBits(allocator: std.mem.Allocator) ![]u8 {
    // Create an array literal.
    var message = [_]u8{ 'z', 'i', 'g', 'b', 'i', 't', 's' };

    // Print the array as string.
    std.log.debug("{s}", .{message});

    // Allocate the slice on the heap and return.
    var message_copy = try allocator.dupe(u8, &message);
    return message_copy;
}

/// Entrypoint of the program.
pub fn main() !void {
    // Use an allocator.
    // https://ziglang.org/documentation/master/#Choosing-an-Allocator
    const allocator = std.heap.page_allocator;
    // Get the message.
    const message = try zigBits(allocator);

    // Print the message.
    std.log.debug("{s}", .{message});
}
```

#### 额外的

让我们改进我们的程序，返回指定长度的切片，而不是堆栈分配的数组。

```zig
// Zig version: 0.10.1

// Import the standard library.
const std = @import("std");

/// Returns a slice with the length of `len`.
fn zigBits(len: usize) ![]u8 {
    // Create an array literal.
    var message = [_]u8{ 'z', 'i', 'g', 'b', 'i', 't', 's' };

    // Print the array as string.
    std.log.debug("{s}", .{message});

    // A slice is a pointer and a length. The difference between an array and
    // a slice is that the array's length is part of the type and known at
    // compile-time, whereas the slice's length is known at runtime.
    try std.testing.expect(@TypeOf(message[0..len]) == []u8);

    // We're using `len` parameter to slice with a runtime-known value.
    // If `len` was declared as `comptime len`, then this value would be '*[N]u8'
    return message[0..len];
}

/// Entrypoint of the program.
pub fn main() !void {
    // Get the message.
    const message = try zigBits(7);

    // Print the message.
    std.log.debug("{s}", .{message});
}
```

这行不通，和第一个例子中的悬空指针问题相同。在 Zig 中，`&message` 和 `message[0..]` 将返回相同的切片。

[@zenith391](https://www.reddit.com/user/zenith391/) 很好地解释了为什么这个例子在我的测试中可能有效，但它仍然是一个未定义的行为。

在第一个示例中，问题在于它在栈上分配，然后释放。现在 _假设_ 调用 `std.log.debug` 需要 8 个字节的栈空间。但是调用 `std.log.debug` 会导致分配更多的栈空间，于是它很容易重用和覆盖用于 "zigbits" 的字节。

它不起作用的原因是当返回类型为 `![]u8` 时，像 `![]u8` 这样的错误联合类型 `![]u8` 占用的堆栈字节数比 `[]u8` 更多，这意味着我们的 "zigbits" 字节在栈中被分配得更远。
结果当 `std.log.debug` 消耗其（假设的）8 字节的堆栈，它不能足够的覆盖 "zigbits".

你可以用 [https://godbolt.org/z/cPWjajYxb](https://godbolt.org/z/cPWjajYxb) 检查这个行为。它显示用 `![]u8` 时，"zigbits" 被放置在 40 字节的堆栈中，而用 `[]u8` 时只有 16 字节。

当然，如果你分配了更多的堆栈字节（通过创建变量或调用更多的函数），它最终会覆盖 "zigbits"。这也意味着这个额外片段遭受同样的问题。

省流: 由于 `![]u8` 更大, 则必须使用更多的栈来覆盖 "zigbits"

### 结论

<img src="/zig/ziggy_pizza.svg" style="width: 25%"/>

我对 Zig 还是个新手，我真的很喜欢学习这些概念。如果我错过了什么或者有更简单的方法，请通过下面的评论告诉我。

欢迎任何反馈！

P.S. [Here](https://bryce.fisher-fleig.org/strategies-for-returning-references-in-rust/) 附言在 [这里](https://bryce.fisher-fleig.org/strategies-for-returning-references-in-rust/) 你可以找到这篇文章的 Rust 版本。
