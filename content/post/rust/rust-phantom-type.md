---
title: "Rust Phantom Type"
date: 2018-11-27T21:18:47+08:00
draft: false
tags: [ "Rust", "Type System"]
categories: [ "Rust", "Type System" ]
---
在具有高级类型系统的语言里面,有一种类型叫Phantom Type,Rust如此强大的类型系统当然也不再话下.

Phantom Type或叫幽灵类型,你或许想起了Java的PhantomReference,嗯...或许没有,但不管怎么样以Phantom为名的都具有一个共性:
它们都是对其所拥有的东东进行"行为标记"在特定的上下文环境执行一些更严格的逻辑或者约束.对于Java可以通过PhantomReference跟踪
GC或者最后进行一些清理的工作,对于Rust那就更多了后边会讲到.

由于Rust本身一些喜闻乐见的特性(LifeTime,Drop Checker)Phantom Type被赋予了更多的功能(自己挖的抗,含泪也要填完).在Rust中
Phantom Type对应的实现为PhantomData这是一个Zero-sized type.对于编译后的代码来说，完全没有任何的副作用.
这也是zero-cost abstractions的一种体现.
下边列举下它具体的使用场景:
#### Compile time type check
编译期的静态类型检测是Phantom Type的一种常见用法,在一些其他语言里面也叫类型标记法,这里它标记的是它所拥有的类型并对这个类型进行约束
这里列举一个常见的单位运算的例子.在支付系统中经常会遇到不同单位的价格进行计算的问题一不留神就会出错
```rust
use std::marker::PhantomData;
use std::ops::Add;

type USD = f64;
type CNY = f64;

#[derive(Debug)]
struct Money<Num>(Num);

impl<Num: Add<Output = Num>> Add for Money<Num> {
    type Output = Money<Num>;
    fn add(self, rhs: Money<Num>) -> Money<Num> {
        Length(self.0 + rhs.0)
    }
}

fn main() {
    let usd = Money(5.0 as USD);
    let cny = Money(5.0 as CNY);
    println!("{:?}", usd + cny);
}
```
上边的代码中有一个明显的bug,美元和人民币直接相加了,如果在系统中存在一个含有几十种货币的复杂业务场景中或者在大规模系统下这种
问题难以发现，在语法的直观上是没有任何的错误.

可不可以通过类型系统本身去做这一部分的约束呢?答案是肯定的也正是Phantom Type的使用场景

```rust
use std::marker::PhantomData;
use std::ops::Add;

mod currency {
    #[derive(Debug)]
    pub enum USD {}
    #[derive(Debug)]
    pub enum CNY {}
}

#[derive(Debug)]
struct Money<Num, P>(Num, PhantomData<P>);

impl<Num: Add<Output = Num>, P> Add for Money<Num, PhantomData<P>> {
    type Output = Money<Num, PhantomData<P>>;
    fn add(self, rhs: Money<Num, PhantomData<P>>) -> Money<Num, PhantomData<P>> {
        Money(self.0 + rhs.0, PhantomData)
    }
}

fn main() {
    let usd: Money<f64, PhantomData<currency::USD>> = Money(5.0, PhantomData);
    let cny: Money<f64, PhantomData<currency::CNY>> = Money(5.0, PhantomData);
    println!("{:?}", usd + cny); 
}
```
编译上面那段的代码，编译器如期的给出了错误：
```
error[E0308]: mismatched types
   --> main.rs:115:26
    |
115 |     println!("{:?}", usd + cny);    
    |                          ^ expected enum `currency::USD`, found enum `currency::CNY`
    |
    = note: expected type `Money<_, std::marker::PhantomData<currency::USD>>`
               found type `Money<_, std::marker::PhantomData<currency::CNY>>`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0308`.
```
意思是说期待的的是USD,但实质的是给定的是CNY.通过添加phantom type编译器就自动做了类型检测并给出了提示.这是怎么做到的呢?

当输入5.0$的时候同时输入了两个信息一个是数字5.0另一个是货币的种类$,分割两者来看单独看都不具有意义但结合两者(5.0$)$就赋予了5.0class信息改变了它的运算规则.而phantom type在此就起到了这个量纲作用。

#### Unused lifetime parameters
在一些unsafe的代码里(FFI,数据结构)时常会遇到在定义struct的时候type或者lifetime和逻辑是有关联的,但却不是struct字段的一部分比如:
```rust
struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
}
```
Iter的lifetime是 'a 也就是不能超过 'a 的生命周期,但实质在Iter内部并没有使用'a 其lifetime就变成了unbounded.
Rust中是禁止定义这种unbounded lifetime.这中情况可以用PhantomData来对"无用的"lifetime做出标记,当然这里的PhantomData
起到的作用远不如此,它不但把lifetime从unbounded变成了bounded,还提供了静态分析需要的信息(Iter对T的拥有关系)
```rust
use std::marker::PhantomData;

struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: PhantomData<&'a T>,
}
```
#### Unused type parameters
这里和上边的情况类似这次是FFI场景下,Rust也是禁止在struct中定义未使用的类型参数,这种情况用PhantomData进行包裹.
```rust
// #![allow(dead_code)]
// trait ResType { }
// struct ParamType;
// mod foreign_lib {
//      pub fn new(_: usize) -> *mut () { 42 as *mut () }
//      pub fn do_stuff(_: *mut (), _: usize) {}
//  }
// fn convert_params(_: ParamType) -> usize { 42 }

use std::marker::PhantomData;
use std::mem;

struct ExternalResource<R> {
   resource_handle: *mut (),
   resource_type: PhantomData<R>,
}

impl<R: ResType> ExternalResource<R> {
    fn new() -> ExternalResource<R> {
        let size_of_res = mem::size_of::<R>();
        ExternalResource {
            resource_handle: foreign_lib::new(size_of_res),
            resource_type: PhantomData,
        }
    }

    fn do_stuff(&self, param: ParamType) {
        let foreign_params = convert_params(param);
        foreign_lib::do_stuff(self.resource_handle, foreign_params);
    }
}
```
#### Ownership and the drop check
Rust中通过`Ownership` + `Lifetime`提供了一些规则来避免像闲空指针这类的内存安全问题但梦想很美好现实很残酷,
很多时候编译器本身其实分辨不出来到底哪个的`Lifetime`更长,比如:
```rust
let (x, y) = (vec![], vec![]);
```
这种情况下是无法确认x和y到底哪个活得更长.这有啥关系吗?关系大了,这动摇了`Ownership` + `Lifetime`的体系根基
如果处理不好就会出现闲空指针.
```rust
struct Inspector<'a>(&'a u8);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        // 1. 因为无法确认days和inspector的生命周期到底谁长,如果days先被drop这里就会构成闲空指针
        println!("I was only {} days from retirement!", self.0);
        // 2. 现在的编译器还没有智能到能看出drop的方法实现即使不使用也不能通过编译
        //println!("Just drop!");
    }
}

fn main() {
    let (inspector, days);
    days = Box::new(1);
    inspector = Inspector(&days);
}
```
针对上门面的问题Rust先后出台了[RFC 1238](https://github.com/rust-lang/rfcs/blob/master/text/1238-nonparametric-dropck.md)和
[RFC 1327](https://github.com/rust-lang/rfcs/blob/master/text/1327-dropck-param-eyepatch.md)的RFC.
简单概括就是

1. 对于一个实现了Drop的范型类型对于它的范型参数必须**严格**的超过它
2. 通过`#[may_dangle]`明确指明不会去使用借用的数据

That's All?当然不是,看个例子
```rust
struct Vec<T> {
    data: *const T, // *const for variance!
    len: usize,
    cap: usize,
}
```
`*const T`是Row poiner这时编译器并不会为其附加所有权关系,也就是说`Vec`和类型`T`之间没有半毛钱关系
更别提`lifetime`约束了(至少在drop checker的视角下是这样).当drop checker路过的时候也不去关心这里面到底是怎么样的
就有可能导致在drop函数中访问到已经被析构的数据导致悬空指针,当然可以通过PhantomData来解决这个问题.通过添加PhantomData
明确告诉编译器`Vec`拥有类型`T`,在drop`Vec`的时候顺带的一起drop掉`T`的实例：
```rust
use std::marker;

struct Vec<T> {
    data: *const T, // *const for covariance!
    len: usize,
    cap: usize,
    _marker: marker::PhantomData<T>,
}
```
当`struct`并没有实际拥有类型`T`的数据时需要使用`PhantomData<&'a T>` 或者 `PhantomData<* const T>`来明确指定所有权关系
#### 参考链接
1. [dropck](https://doc.rust-lang.org/nomicon/dropck.html)
2. [phantom-data](https://doc.rust-lang.org/nomicon/phantom-data.html#phantomdata)