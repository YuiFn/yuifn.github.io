---
title: "Hello"
date: 2018-11-20T13:36:58+08:00
---

### 开篇Higher-Order-Function -- Map
许久不写了,这次重新捡起.作为开篇没有太多内容要说,以[Higher-Order-Function](https://clojurebridge.org/community-docs/docs/clojure/higher-order-function/)中的**map**函数来做简单的说明.

map是一个接受一个函数并把这个函数作为一个functor应用于每一个元素的函数.如下图
![map](/imgs/map.png)
下边看看各种语言的一个大概实现,代码的意思是对一个[1,5]的列表,对其每个元素➕1

#### Scala
```scala
val list = 1 to 5;
list.map(_ + 1).toList();
```

#### Ocaml
```ocaml
let plus = ( + );;
(* 这里是为了对此,可以使用List.range 1 6;; *)
let list = List.init 5 ~f:(plus 1);
list 
    |> List.map ~f: plus
    |> List.map ~f: (fun fn -> fn 1);;
```

#### Rust
```rust
fn main(){
    // 这里指定下类型
    let list: Vec<_> = (1..5).map(|x| x + 1).collect();
}
```

#### Elixir
```
1..5
    |> Enum.map(&(&1 + 1))
```

#### Dart
```dart
main(List<String> args) {
  final list = [1, 2, 3, 4, 5];
  list.map((x) => x + 1);
}
```