% Видимость

Поиск совпадений и замена видимости может быть достаточна хитра в Rust, из-за
отсутствия типа `vis` в сопоставлении.

## Поиск совпадений и игнорирование

В зависимости от контекста это можно сделать повторениями:

```rust
macro_rules! struct_name {
    ($(pub)* struct $name:ident $($rest:tt)*) => { stringify!($name) };
}
# 
# fn main() {
#     assert_eq!(struct_name!(pub struct Jim;), "Jim");
# }
```

Примеру подойдут элементы `struct`, которые и приватны и публичны. Или `pub pub`
(очень публичны), или даже `pub pub pub pub` (очень очень публичны). Лучшей
защитой от этого является надежда. Надежда на то, что люди не будут использовать
его так глупо.

## Поиск совпадений и замена

Из-за того, что нельзя связать повторение с переменной, нет и возможности
захватить `$(pub)*` так, чтоб его можно было заменить. В результате нужно
несколько правил.

```rust
macro_rules! newtype_new {
    (struct $name:ident($t:ty);) => { newtype_new! { () struct $name($t); } };
    (pub struct $name:ident($t:ty);) => { newtype_new! { (pub) struct $name($t); } };
    
    (($($vis:tt)*) struct $name:ident($t:ty);) => {
        as_item! {
            impl $name {
                $($vis)* fn new(value: $t) -> Self {
                    $name(value)
                }
            }
        }
    };
}

macro_rules! as_item { ($i:item) => {$i} }
# 
# #[derive(Debug, Eq, PartialEq)]
# struct Dummy(i32);
# 
# newtype_new! { struct Dummy(i32); }
# 
# fn main() {
#     assert_eq!(Dummy::new(42), Dummy(42));
# }
```

> **Смотри также**: [Приведение к AST].

В этом случае мы ищем совпадения произвольной последовательности токенов внутри
группы с `()` или `(pub)`, а затем подставляем содержимое группы на выход. Из-за
того, что парсер не ожидает развертывания повторяющихся `tt` в этом месте,
нам придется использовать [ast-coercion], чтобы развертывание правильно
разбиралось.

[ast-coercion]: blk-ast-coercion.html
