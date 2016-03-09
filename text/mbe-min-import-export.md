% Импорт/Экспорт

Существует два способа выставить макрос наружу, в более широкую область
видимости.  Первый - использовать атрибут `#[macro_use]`. Его можно применять
*как* к модулям, так и к внешним контейнерам. Например:

```rust
#[macro_use]
mod macros {
    macro_rules! X { () => { Y!(); } }
    macro_rules! Y { () => {} }
}

X!();
#
# fn main() {}
```

Макрос можно экспортировать из текущего контейнера, используя атрибут
`#[macro_export]`.  Помните, что такой способ  *игнорирует* все области
видимости.

Если у нас есть следующее описание библиотечного пакета `macs`:

```ignore
mod macros {
    #[macro_export] macro_rules! X { () => { Y!(); } }
    #[macro_export] macro_rules! Y { () => {} }
}

// X! и Y! *не* определены здесь, а *экспортированы*,
// несмотря на то, что `macros` является приватным.
```

Следующий код работает, как и ожидается:

```ignore
X!(); // X определен
#[macro_use] extern crate macs;
X!();
# 
# fn main() {}
```

Помните, что вы можете использовать `#[macro_use]`, который ссылается на внешний
контейнер, *только* из корневого модуля.

Наконец, если макросы импортируются из внешнего контейнера, можно контролировать
*какой* именно макрос импортируется. Можете использовать эту возможность, чтобы
избежать раздутия пространства имен, или чтобы переопределить какой-либо
конкретный макрос, например, так:

```ignore
// Импортируем  *только* макрос `X!` .
#[macro_use(X)] extern crate macs;

// X!(); // X определен, а Y! не определен

macro_rules! Y { () => {} }

X!(); // X определен, и Y! определен

fn main() {}
```

При экспортировании макросов, часто полезно ссылаться на имена, не связанные с
макросами, в содержащем макросы контейнере. Из-за того, что контейнеры могут
переименовываться, есть специальная переменная замены: `$crate`.  Она будет
*всегда* разворачиваться в префикс абсолютного пути к содержащему макрос
контейнеру (*например*, `:: macs`).

Помните, что такой подход *не* работает для самих макросов, из-за того, что
макросы не используют стандартные преобразования имен каким бы то ни было
образом. Поэтому вы не можете использовать, что-то вроде  `$crate::Y!` для того,
чтобы сослаться на указанный макрос внутри вашего контейнера.  Следствием этой
особенности и особенности подхода к выборочному импорту через `#[macro_use]`,
является то, что на настоящей момент *нет способа* гарантировать, что любой ваш
макрос будет доступен при импорте в другом контейнере.

Рекомендуется *всегда* использовать абсолютные пути к именам, не связанным с
макросами, чтобы избежать конфликтов, *включая* конфликты с именами в стандартной
библиотеке.