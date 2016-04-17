% Обратные вызовы

```rust
macro_rules! call_with_larch {
    ($callback:ident) => { $callback!(larch) };
}

macro_rules! expand_to_larch {
    () => { larch };
}

macro_rules! recognise_tree {
    (larch) => { println!("#1, the Larch.") };
    (redwood) => { println!("#2, the Mighty Redwood.") };
    (fir) => { println!("#3, the Fir.") };
    (chestnut) => { println!("#4, the Horse Chestnut.") };
    (pine) => { println!("#5, the Scots Pine.") };
    ($($other:tt)*) => { println!("I don't know; some kind of birch maybe?") };
}

fn main() {
    recognise_tree!(expand_to_larch!());
    call_with_larch!(recognise_tree);
}
```

Соблюдая порядок, по которому этот макрос разворачивается, невозможно (по крайне
мере для Rust 1.2) передать ему информацию из развертывания *другого* макроса.
Это могло бы сделать модуляризацию макросов очень сложной.

Как вариант, можно использовать рекурсию и передавать функцию обратного вызова.
Вот как будет выглядеть трассировка примера в этом случае:

```ignore
recognise_tree! { expand_to_larch ! (  ) }
println! { "I don't know; some kind of birch maybe?" }
// ...

call_with_larch! { recognise_tree }
recognise_tree! { larch }
println! { "#1, the Larch." }
// ...
```

Используя повторяющиеся `tt`, можно также пересылать произвольные аргументы
функции обратного вызова.

```rust
macro_rules! callback {
    ($callback:ident($($args:tt)*)) => {
        $callback!($($args)*)
    };
}

fn main() {
    callback!(callback(println("Yes, this *was* unnecessary.")));
}
```

Вы можете вставлять дополнительные токены в аргументы в случае необходимости.
