---
title: Rust中的生命周期杂谈
date: 2024-02-25 16:07:25
tags: 
    - Rust
    - 编程语言
idx: rust
categories: Rust
excerpt: Rust中保证代码安全性的特性——生命周期
---

# 引入：“悬垂”带来的内存管理问题
“悬垂”是一种在内存管理中常见的问题，在支持指针的语言中会出现的是悬垂指针，而在Rust中出现的则是悬垂引用问题。

悬垂引用（Dangling References）可以进行通过悬垂指针（Dangling Pointer）进行类比。悬垂指针是在释放内存时保留了指向它的指针时产生的，此时指向的内存可能已经被分配给其它持有者，最终可能导致错误的结果。悬垂引用则是Rust中引用带来的问题，是引用指向了一个不再有效的内存位置，它又与Rust的生命周期模型和所有权模型相关。

在Rust中，悬垂引用是不被允许存在的。在试图创建一个一个悬垂引用时，Rust会通过编译时错误来避免这个问题。一个尝试创建悬垂引用的代码如下[\[1\]](https://rustwiki.org/zh-CN/book/ch04-02-references-and-borrowing.html#%E6%82%AC%E5%9E%82%E5%BC%95%E7%94%A8dangling-references)：
```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {    // 返回字符串的引用
    let s = String::from("hello");   // s是新字符串

    &s // 返回字符串s的引用
} // s离开作用域被丢弃，内存释放
  // 此时返回的引用指向一个无效的String
```
实际上这里应当返回一个`String`，此时函数体内的`s`被直接返回，该字符串的所有权被移动给外面调用该函数并赋值的变量。

如果运行上面的代码，会得到以下的错误信息[\[1\]](https://rustwiki.org/zh-CN/book/ch04-02-references-and-borrowing.html#%E6%82%AC%E5%9E%82%E5%BC%95%E7%94%A8dangling-references)：
```shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0106]: missing lifetime specifier
 --> src/main.rs:5:16
  |
5 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime
  |
5 | fn dangle() -> &'static String {
  |                ^^^^^^^^

For more information about this error, try `rustc --explain E0106`.
error: could not compile `ownership` due to previous error
```
该错误信息提醒我们应当对返回的引用使用`'static`的生命周期注释，引出了生命周期的话题，而关于该注释将在后文进行介绍，此处暂且不谈。

由此可以看出，Rust在编译阶段就可以避免这种“悬垂”的问题。这是因为在编译器里面有一个借用检查器，它会在编译时就比较各借用的作用域，确保所有借用都是有效的。

# 保证安全的工具：借用检查器
借用这一概念基于Rust独特的所有权机制，它可以使Rust在不需要垃圾回收器的情况下保证内存的安全性。借用检查器基于两条基本原则来工作：
- **借用规则**：任一时刻，一个资源要么被可变借用独占，要么由多个不可变借用共享；
- **生命周期规则**：引用必须总是有效的，确保引用不能比它引用的数据获得更久；

其中，借用规则与读写锁的原理类似，还可以通过这一套规则来防止数据竞争，保证**并发任务**的安全性。

# 生命周期：保证引用的有效性
在Rust中，每一个引用都有它的生命周期，即引用保持有效的作用域。在大多数时候是可以自动推断的，但也有引用以不同方式关联的情况，此时就需要使用泛型生命周期参数来注明关系，以保证运行时使用的引用是有效的。也就是说，生命周期最大的一个目的就是——**避免产生悬垂引用**。
## 显式标注与隐式的自动推断（省略标注）
一个比较重要的问题就是：何时可以省略生命周期标注让编译器自行推断，而何时又必须使用显式标注？

关于这个问题，将在接下来的几个场景中进行讨论，但在此之前可以给出的是Rust中给出的**生命周期省略规则**（lifetime elision rules）[\[1\]](https://rustwiki.org/zh-CN/book/ch10-03-lifetime-syntax.html#%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E7%9C%81%E7%95%A5lifetime-elision)。

### 生命周期省略规则
1. 每一个是引用的参数都有它自己的生命周期参数；
    - 该条规则适用于输入生命周期，即函数或方法的参数的生命周期；
    - 举例：`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`，`x`和`y`是两个对`i32`类型的引用，按照规则它们拥有自己的生命周期，此处用`'a`和`'b`进行标注；
2. 如果只有一个输入生命周期参数，那么它被赋予所有输出生命周期参数；
    - 该条规则适用于输出生命周期，即返回值的生命周期；
    - 举例：`fn foo<'a>(x: &'a i32) -> &'a i32`，此处只有一个输入引用参数，只对应一个生命周期，则该生命周期也被赋予给返回值中的引用；
3. 如果方法有多个输入生命周期参数并且其中一个参数是`&self`或`&mut self`，说明是个对象的方法，则所有输出生命周期参数被赋予`self`的生命周期；
    - 该条规则适用于输出生命周期，且用于方法；

借用[\[1\]](https://rustwiki.org/zh-CN/book/ch10-03-lifetime-syntax.html#%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E7%9C%81%E7%95%A5lifetime-elision)中的多参数情况下对三条规则的分析示例来解读这三条规则：
- 给出一个函数签名，此时签名中的引用没有关联任何生命周期：
```rust
fn longest(x: &str, y: &str) -> &str {}
```
- 应用第一条规则——每个引用参数有其自己的生命周期，这里称之为`'a`和`'b`，改写函数签名：
```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str{}
```
- 应用第二条规则，由于函数存在多个生命周期，不适用于该种情况；
- 应用第三条规则，同样不适用，因为没有self；

由此出现了一个问题，该函数并不能通过编译器的通过。因为在应用了三条规则之后，仍然无法确定返回值类型的生命周期。如何确定返回值的生命周期呢？

### 函数中的泛型生命周期
以上一节中讨论的`longest`函数为例，它的作用是返回两个字符串中更长的那个字符串，其实现代码如下[\[1\]](https://rustwiki.org/zh-CN/book/ch10-03-lifetime-syntax.html#%E5%87%BD%E6%95%B0%E4%B8%AD%E7%9A%84%E6%B3%9B%E5%9E%8B%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)，其中生命周期标注已按上节的分析补全：
```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
该函数的内部实现包含一个选择分支结构，执行时只希望比较它们的长度而不转移所有权，所以传入了两个字符串的引用，而结果只有在运行时才能知晓，所以Rust无法得知返回的引用指向`x`还是`y`；借用检查器也不能确定`x`和`y`的生命周期如何与返回值生命周期关联。

实际上，Rust对带生命周期的签名函数有相应的限制：
- 任何引用都必须拥有标注好的生命周期；
- 任何被返回的引用都必须有和某个输入量相同的生命周期或是静态类型（`static`）；

也就是说可以对`longest`函数进行以下的修改：
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str{
    ...
}
```
这表明对于生命周期`'a`，函数会获取两个与`'a`存在一样长的字符串slice，函数也会返回一个与`'a`一样长的字符串slice。其实际含义是`longest`返回的引用的生命周期与传入该函数的生命周期的较小者一致，而函数不关心`x`和`y`具体存在多久，只知道有某个可以被`'a`替代的作用域会满足该签名。被`'a`替代的具体生命周期是`x`与`y`的作用域相重叠的部分，或者说是它们之中生命周期**较小**的那一个。

可以将上面的内容总结为下面几点：
- 无法确定生命周期的情况下，需要使用显式标注的方式来标注引用的生命周期。
- 函数的返回值的输出生命周期要么与输入生命周期有关联（相同），要么应当是静态类型。
- 若返回值总返回其中一个参数，其他参数也可以不标注生命周期，因为它们之间没有关联。

### 方法中的泛型生命周期
方法的生命周期标注与函数类似，但是由生命周期省略规则的第三条，对于方法而言，是属于某一对象的，其参数里面包括`self`，也就是说`self`的生命周期会赋给所有的输出生命周期参数。因此，标注可以省略。

### 结构体中的泛型生命周期
当一个结构体的变量中存在引用时，就需要对结构体中的每一个引用添加生命周期标注，如下的示例代码[\[2\]](https://rustwiki.org/zh-CN/rust-by-example/scope/lifetime/struct.html)：
```rust
struct NamedBorrowed<'a> {
    x: &'a i32,
    y: &'a i32,
}
```
在这里，生命周期标注`'a`的意思是该结构体的实例不能比这两个字段的引用存在的更久。

## 特殊的生命周期：`'static`
`'static`代表了静态生命周期，它可以存活于整个程序期间。特别地，所有的字符串字面量都拥有`'static`生命周期。使变量拥有`'static`生命周期有两种方式，它们把数据保存在可执行文件的只读内存取：
- 使用`static`声明常量；
- 产生一个拥有`&'static str`类型的`string`字面量；

另外，静态生命周期可以被强制转换成一个更短的生命周期[\[2\]](https://rustwiki.org/zh-CN/rust-by-example/scope/lifetime/static_lifetime.html)：
```rust
static NUM: i32 = 18;

fn coerce_static<'a>(_: &'a i32) -> &'a i32 {
    &NUM
}
```
上面这个函数返回对常量`NUM`的引用，但是其返回值的生命周期被设定为`'a`，它一定比`'static`更短，这种强制转换是合法的。

## 用于约束
由于生命周期也是泛型，所以也可以使用约束。

- `T: 'a`：在`T`中的**所有**引用都必须比生命周期`'a`长；
- `T: Trait + 'a`：`T`类型必须实现`Trait` trait，并且在`T`中的所有引用都必须比`'a`活得更长。

可以看出，`:`符号的含义在此处为位于前面的对象比位于符号后面的对象要活得更长。由此，也可以对生命周期进行约束：
- `'a: 'b`：`'a`比`'b`存活的时间更长，只要`&'b ()`有效，引用`&'a ()`就有效。

# 参考资料
\[1\] Rust程序语言设计. https://rustwiki.org/zh-CN/book/ch10-03-lifetime-syntax.html
\[2\] Rust By Example. https://rustwiki.org/zh-CN/rust-by-example/scope/lifetime.html
\[3\] Rust参考手册. https://rustwiki.org/zh-CN/reference/