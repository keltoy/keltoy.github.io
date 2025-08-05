# Rust


<!--more-->

## 指令工具

### cargo 工具

| 指令   | 说明                    | 举例                                |
| ------ | ----------------------- | ----------------------------------- |
| fmt    | 格式化项目              | ```cargo fmt```                     |
| clippy | 检查代码并给出建议      | ```cargo clippy```                  |
| search | 查找包 ，对描述也会搜索 | ```cargo search bracket-terminal``` |

## 代码宏

| 代码                                          | 说明                 |
| --------------------------------------------- | -------------------- |
| ```#![warn(clippy::all, clippy::pedantic)]``` | 暴露更多问题，更严格 |
| ```#[derive(Debug)]``` |                      |

## format
| 字符       | 说明  |
| ---------- | ----- |
| ```{:?}``` | debug |

## 语法糖

### if let 

```rust
match my_option {
    Some(value) => {
        // 使用 value
        do_something(value);
    }
    _ => {}
}
```
就可写成
```rust
if let Some(value) = my_option {
    // 使用 value
    do_something(value);
}
```

### unwrap

```rust
match my_result {
    Ok(value) => value,
    Err(error) => {
        // 处理错误
        handle_error(error);
    }
}
```

```rust
let value = my_option.unwrap();
```

如果返回 result 

```rust
let value = my_result?;
```




---

> 作者: toxi  
> URL: https://example.com/rust_1/  

