# 新版`?`操作符

## 前置知识(并不全)

`FromResidual` 是 `Try` 的 supertrait, 即:
`trait Try : FromResidual`

## 正文

```rust
fn f() -> Target {
  let x: Source = todo!();
  let v: Output = x? // 这里
  todo!()
}
```

`Source` -- `?` -> `Target` or `Output`

由此 `?` 作用如下:

- 根据 `Source` 决定流程
  - `<Source as Try>::branch()` 返回的 `ControlFlow` 来控制
- 根据 流程 从 `Source` 提取值
  - `ControlFlow` 内包裹的值.
- 从 值 构建 `Target` 或 直接输出 `Output`
  - `<Target as FromResidual< <Source as Try>::Residual> >::from_residual()` 来构建, 后面细说.

先看下解糖`x?`([原文](https://rust-lang.github.io/rfcs/3058-try-trait-v2.html#desugaring-))

```rust
match Try::branch(x) {
    ControlFlow::Continue(v) => v,
    ControlFlow::Break(r) => return FromResidual::from_residual(r),
}
```

下面主要说 `Break` 分支, 也就是短路时的情况.

`Source` -- 通过 ->
`Source` 的 `Try::branch` -- 若得到 Break 分支, 包裹的值类型为 ->
`Source` 的 `Try::Residual` -- 通过 ->
`Target` 对 `Source::Residual`(即上面的类型) 的 `FromResidual::from_residual` -- 得到 ->
`Target`

首先发现 `Source` 需要实现 `Try`, 而 `Target` 只需要实现 `FromResidual`.
也就是, 如果需要支持 `x?` 操作, 那么`x`的类型需要实现 `Try`,
而一个函数里面包含 `?` 操作, 那么函数的返回类型只需要实现 `FromResidual`, 不需要实现`Try`.

然后, 想把 `Source` -- 转换为 -> `Target`, 重点是:

- 在 `Target` 上实现 `FromResidual<T>`
- 其中 `T` 要涵盖 `Source` 的 `Residual` 类型

可以通过不同的`Source`来进行不同的流程控制,
`Source::Residual` 不代表失败, 只代表短路时的结果.
("短路"指提前返回, 没用"返回"是为了与整个函数的返回值做区分)
而短路结果的意义, 则通过`Target`来明确, 这通过
`Target::<Source::Residual>::from_residual()`来实现,
其通过把短路结果转换为函数的返回结果, 来给短路结果赋予意义.

例如类似短路 or 的逻辑中, 当遇到正确值的时候, 就会立刻返回,
`Source::Residual` 代表的实际上是正确值.

更具体一点, 在`nom`等组合式解析器中, `alt()`函数的内部可以用`?`立刻返回, 类似:

```rust
enum Target {
  Ok(...)
  Error(...)
  Fail(...)
}
struct AltSource(Target);

fn alt(...) -> Target {
  let err = error_parser(...).to_alt_source()?; // AltSource 会在 Error 时继续.
  let ok = ok_parser(...).to_alt_source()?; // 在 正确 时返回.
  let fail = fail_parser(...).to_alt_source()?; // 当然严重错误也会返回.
}

// 对比一般流程
struct NormalSource(Target);
fn xxx_parser(...) -> Target {
  let ok = ok_parser(...).to_normal_source()?; // NormalSource 会在 正确 时继续.
  let err = error_parser(...).to_normal_source()?; // 在 Error 时返回.
  let fail = fail_parser(...).to_normal_source()?; // Fail也返回.
}
```

上面的 Target 只需要实现 `FromResidual`,
而 `NormalSource` 和 `AltSource` 则需要实现 `Try` (和 `FromResidual`).
