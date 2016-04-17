% Отладка

`rustc` предоставляет несколько инструментов для отладки макросов. Один из самых
полезных -  `trace_macros!`, который представляет собой директиву компилятору,
заставляющую его делать дамп каждого вызова макроса до развертывания. Например,
имеем следующее:

```rust
# // Важно: убедитесь, что вы используете nightly версию компилятора.
#![feature(trace_macros)]

macro_rules! each_tt {
    () => {};
    ($_tt:tt $($rest:tt)*) => {each_tt!($($rest)*);};
}

each_tt!(foo bar baz quux);
trace_macros!(true);
each_tt!(spim wak plee whum);
trace_macros!(false);
each_tt!(trom qlip winp xod);
# 
# fn main() {}
```

Вывод:

```text
each_tt! { spim wak plee whum }
each_tt! { wak plee whum }
each_tt! { plee whum }
each_tt! { whum }
each_tt! {  }
```

Это *особенно* неоценимо при отладке макросов с большой глубиной рекурсии. Вы
можете также включить эту директиву из командной строки, добавив к команде 
`-Z trace-macros`.

Во вторых, есть `log_syntax!`, который заставляет компилятор выводить все
токены, которые подаются на вход. Например, это заставит компилятор петь песню:

```rust
# // Важно: убедитесь, что вы используете nightly версию компилятора.
#![feature(log_syntax)]

macro_rules! sing {
    () => {};
    ($tt:tt $($rest:tt)*) => {log_syntax!($tt); sing!($($rest)*);};
}

sing! {
    ^ < @ < . @ *
    '\x08' '{' '"' _ # ' '
    - @ '$' && / _ %
    ! ( '\t' @ | = >
    ; '\x08' '\'' + '$' ? '\x7f'
    , # '"' ~ | ) '\x07'
}
# 
# fn main() {}
```

Эту команду можно использовать, чтобы сделать отладку более направленной, чем
`trace_macros!`.

Иногда представляет проблему то, во что макрос *разворачивается*. Для отладки
можно использовать аргумент компилятора `--pretty`. Ниже пример:

```ignore
// Короткая инициализация  `String`.
macro_rules! S {
    ($e:expr) => {String::from($e)};
}

fn main() {
    let world = S!("World");
    println!("Hello, {}!", world);
}
```

скомпилировано со следующей командой :

```shell
rustc -Z unstable-options --pretty expanded hello.rs
```

выдает следующий вывод (изменен ради форматирования):

```ignore
#![feature(no_std, prelude_import)]
#![no_std]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std as std;
// Shorthand for initialising a `String`.
fn main() {
    let world = String::from("World");
    ::std::io::_print(::std::fmt::Arguments::new_v1(
        {
            static __STATIC_FMTSTR: &'static [&'static str]
                = &["Hello, ", "!\n"];
            __STATIC_FMTSTR
        },
        &match (&world,) {
             (__arg0,) => [
                ::std::fmt::ArgumentV1::new(__arg0, ::std::fmt::Display::fmt)
            ],
        }
    ));
}
```

Другие опции `--pretty` можно посмотреть так - `rustc -Z unstable-options --help
-v`; полный список опций не приводится из-за того, что он относится к
нестабильным возможностям и может поменяться в любое время.
