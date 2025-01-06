# Parser 的通用模式

可以当 if 来用

```rust
alt((
  (peek(条件), 编译).map(|o|o.1),
  (peek(条件), 编译).map(|o|o.1),
  (peek(条件), 编译).map(|o|o.1),
))
```

下面的当做 `while !条件 { 编译 }` 来用.
(`repeat_till`是 `winnow` 库的一个 combinator)

```rust
repeat_till(
  编译,
  peek(条件)
)
```
