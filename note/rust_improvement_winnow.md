# 语法解析器 winnow 可以改进的地方

## repeat

原版指定返回的集合类型(用 Vec 还是 VecDeque 等), 相当麻烦.
尤其在组合起来的时候. 如下:

```rust
let a: (_, _, _, _, _, Vec<(Option<()>, _)>) = (
    self.meta(),
    self.head(),
    self.multi_empty_line(),
    self.column_config(),
    self.multi_empty_line(),
    repeat(.., (opt(repeat(.., self.default_())), self.data())),
)
    .parse_next(t)?;
```

理想状态是可以这样写:

```rust
let a = (
    self.meta(),
    self.head(),
    self.multi_empty_line(),
    self.column_config(),
    self.multi_empty_line(),
    repeat::<Vec<_>>(.., (opt(repeat::<()>(.., self.default_())), self.data())),
)
    .parse_next(t)?;
```

变通方法有两种:

可以加个类似 iter 的 collect , 在 collect 的泛型上指定类型.

或者加个通用的 identity 专门用来声明类型.
parser 是函数, 而 identity 只用来约束 parser 的输出类型.
