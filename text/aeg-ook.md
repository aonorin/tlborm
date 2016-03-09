% Ook!

Этот макрос представляет собой реализацию [эзотерического языка Ook!](http://www.dangermouse.net/esoteric/ook.html), которая изоморфна [эзотерическому языку Brainfuck](http://www.muppetlabs.com/~breadbox/bf/).

Модель выполнения языка очень проста: память представляет собой массив "ячеек" (обычно, как минимум, 8 бит) переменной длины (обычно, как минимум, 30.000 "ячеек"). В памяти есть указатель, который начинается с 0.  Наконец, есть стек выполнения (используемый для реализации циклов) и указатель на инструкцию (команду) в программе, хотя они относятся к среде выполнения, а не к выполняющейся программе.

Язык состоит всего из трех служебных слов: `Ook.`, `Ook?` и `Ook!`.  Они объединяются в пары, формируя 8 различных операций:

* `Ook. Ook?` - инкрементировать указатель.
* `Ook? Ook.` - декрементировать указатель.
* `Ook. Ook.` - инкрементировать значение в ячейке памяти по указателю.
* `Ook! Ook!` - декрементировать значение в ячейке памяти по указателю.
* `Ook! Ook.` - переписать значение из ячейки памяти по указателю в стандартный output.
* `Ook. Ook!` - переписать значение из стандартного input в ячейку памяти по указателю.
* `Ook! Ook?` - начать цикл.
* `Ook? Ook!` - перейти на начало цикла, если значение в ячейке памяти по указателю не равно 0; иначе, продолжить.

Ook! интересен тем, что он является тьюринг-полным, что означает, что среда, в которой вы можете реализовать его, должна быть *тоже* тьюринг-полной.

## Реализация

```ignore
#![recursion_limit = "158"]
```

Это, на самом деле, минимальный лимит рекурсии, для которого скомпилируется пример программы, приведенный в конце. Если вы хотите узнать, что может быть таким фантастически сложным, что сможет *оправдать* изменение лимита рекурсии в пять раз от дефолтного значения ... [угадайте](https://en.wikipedia.org/wiki/Hello_world_program).

```ignore
type CellType = u8;
const MEM_SIZE: usize = 30_000;
```

Это здесь, чтобы убедиться, что они видимы для расширения макроса.[^*]

[^*]: Эти поля *могли бы* быть определены внутри макроса, но тогда они бы явно передавались в каждом вызове (из-за гигиены макроса).  Честно говоря, к тому моменту как я понял, что мне *нужно* их определить, макрос был уже почти написан и ... ну, а *вам* бы хотелось все переделывать заново без особой на то важной причины?

```ignore
macro_rules! Ook {
```

Имя *по-хорошему* должно было быть `ook!` по правилам именования, но случай уж был слишком хорош, чтобы пройти мимо них.

Правила для этого макроса разбиты на секции с использованием паттерна [internal rules](../pat/README.html#internal-rules).

Первым будет правило `@start`, обрабатывающее настройки блока, в котором будет происходить остальное развертывание. Здесь нет ничего интересного: мы определяем некоторые переменные и вспомогательные функции, и затем выполняем основную часть разворота.

Несколько небольших замечаний:

* Мы разворачиваем в функцию, поэтому можем использовать `try!` для упрощения перехвата ошибок.
* Использование имен с подчеркиванием вначале нужно, чтобы компилятор не ругался на неиспользованные функции и переменные если, например, пользователь пишет Ook! для программы, у которой нет I/O.

```ignore
    (@start $($Ooks:tt)*) => {
        {
            fn ook() -> ::std::io::Result<Vec<CellType>> {
                use ::std::io;
                use ::std::io::prelude::*;
    
                fn _re() -> io::Error {
                    io::Error::new(
                        io::ErrorKind::Other,
                        String::from("ran out of input"))
                }
                
                fn _inc(a: &mut [u8], i: usize) {
                    let c = &mut a[i];
                    *c = c.wrapping_add(1);
                }
                
                fn _dec(a: &mut [u8], i: usize) {
                    let c = &mut a[i];
                    *c = c.wrapping_sub(1);
                }
    
                let _r = &mut io::stdin();
                let _w = &mut io::stdout();
        
                let mut _a: Vec<CellType> = Vec::with_capacity(MEM_SIZE);
                _a.extend(::std::iter::repeat(0).take(MEM_SIZE));
                let mut _i = 0;
                {
                    let _a = &mut *_a;
                    Ook!(@e (_a, _i, _inc, _dec, _r, _w, _re); ($($Ooks)*));
                }
                Ok(_a)
            }
            ook()
        }
    };
```

### Парсинг кода операции

Далее следуют правила "выполнения", которые используется для парсинга кода операции на входе.

Обычная форма этих правил - `(@e $syms; ($input))`. Как можно заметить из правила  `@start`, `$syms` - это коллекция символов, необходимая для того, чтобы реализовать программу: вход, выход, массив памяти, *и т.д.*.  Мы используем  [TT bundling](../pat/README.html#tt-bundling) для простой передачи этих символов дальше, промежуточным правилам.

Во-первых, это правило прерывает нашу рекурсию: если на входе у нас ничего нет - мы останавливаемся.

```ignore
    (@e $syms:tt; ()) => {};
```

Во-вторых, у нас есть одно правило для *почти* каждого кода операции. Для этого мы берем код операции, подставляем соответствующий код на Rust, затем рекурсивно перемещаемся в хвост входа: прямо по книге [TT muncher](../pat/README.html#incremental-tt-munchers).

```ignore
    // Increment pointer.
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook. Ook? $($tail:tt)*))
    => {
        $i = ($i + 1) % MEM_SIZE;
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // Decrement pointer.
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook? Ook. $($tail:tt)*))
    => {
        $i = if $i == 0 { MEM_SIZE } else { $i } - 1;
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // Increment pointee.
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook. Ook. $($tail:tt)*))
    => {
        $inc($a, $i);
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // Decrement pointee.
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook! Ook! $($tail:tt)*))
    => {
        $dec($a, $i);
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // Write to stdout.
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook! Ook. $($tail:tt)*))
    => {
        try!($w.write_all(&$a[$i .. $i+1]));
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // Read from stdin.
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook. Ook! $($tail:tt)*))
    => {
        try!(
            match $r.read(&mut $a[$i .. $i+1]) {
                Ok(0) => Err($re()),
                ok @ Ok(..) => ok,
                err @ Err(..) => err
            }
        );
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
```

Вот здесь вещи становятся более запутанными.  Этот код, `Ook! Ook?`, означает начало цикла. Циклы Ook! переводятся в следующий код на Rust:

> **Замечание**: это *не* рабочая часть кода.
>
> ```ignore
> while memory[ptr] != 0 {
>     // Содержимое цикла
> }
> ```

Конечно, мы не можем *на самом деле* выделить незаконченный цикл.  Это *можно* решить, используя  [pushdown](../pat/README.html#push-down-accumulation), что ведет к более фундаментальной проблеме: мы не можем *написать* `while memory[ptr] != {`, в принципе, *где бы то ни было*.  Это происходит потому, что в противном случае у нас появится лишняя, "висячая", скобка.

Для того, чтобы решить это, мы разделяем вход на две части: все *внутри* цикла, и все, что *после* него.  Правила `@x` берут на себя первое, `@s` - второе.

```ignore
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook! Ook? $($tail:tt)*))
    => {
        while $a[$i] != 0 {
            Ook!(@x ($a, $i, $inc, $dec, $r, $w, $re); (); (); ($($tail)*));
        }
        Ook!(@s ($a, $i, $inc, $dec, $r, $w, $re); (); ($($tail)*));
    };
```

### Извлечение цикла

Следующее - `@x`, или правило "извлечения". Оно отвечает за следующее - взять хвост со входа и извлечь содержимое из цикла.  Общая форма этого правила: `(@x $sym; $depth; $buf; $tail)`.

Назначение `$sym` такое же, как и выше.  `$tail` - это вход, который нужно распарсить, в то время как `$buf` - это [push-down accumulation buffer](../pat/README.html#push-down-accumulation), в который мы будем собирать коды операций, находящиеся внутри цикла. Но что на счет `$depth`?

Сложности добавляет то, что циклы могут быть  *вложенными*.  Итак, мы должны как-то отслеживать, сколько уровней вложенности у нас в данный момент.  Мы должны делать это очень аккуратно, чтобы не остановить парсинг слишком рано или слишком поздно, а остановить именно тогда *когда нужно*.[^когда нужно]

[^когда нужно]:
    Это известный факт[^факт], что сказка "Маша и три медведя" - это на самом деле аллегория на технику аккуратного лексического парсинга.

[^факт]: И под "фактом" я подразумеваю "бесстыдное вранье".

Из-за того, что мы не можем выполнять арифметические действия в макросах, а написание собственных правил явного совпадения с целым числом является неосуществимым (представьте очень много копи-паста таких правил для всех положительных целых чисел), мы вместо этого вернемся к самому древнему и самому почтенному методу счета в истории: счету на пальцах.

Но из-за того, что у макроса *нет* пальцев, мы используем [token abacus counter](../pat/README.html#abacus-counters) вместо них. Для специфики, мы будем использовать много`@`, где каждое `@` представляет собой один дополнительный уровень. Если мы объединим эти `@` в группу, мы сможем реализовать три операции, которые нам нужны:

* Инкремент: `($($depth:tt)*)` заменяем на `(@ $($depth)*)`.
* Декремент:  `(@ $($depth:tt)*)` заменяем на `($($depth)*)`.
* Сравнение с нулем: сравнение с `()`.

Во-первых, это правило, необходимое, чтобы найти совпадение с выражением `Ook? Ook!`, заканчивающее цикл, который мы парсим. В этом случае мы скармливаем накопленное содержание цикла правилам `@e`, определенным до этого.

Заметьте, что нам *не нужно* делать что-либо с оставшимся хвостом на входе (он будет обрабатываться правилами `@s`).

```ignore
    (@x $syms:tt; (); ($($buf:tt)*);
        (Ook? Ook! $($tail:tt)*))
    => {
        // Внешний цикл заканчивается. Обрабатываем значения из буфера.
        Ook!(@e $syms; ($($buf)*));
    };
```

Во-вторых, у нас есть правила для входа и выхода из вложенных циклов.  Это увеличивает счетчик и добавляет коды операций в буфер.

```ignore
    (@x $syms:tt; ($($depth:tt)*); ($($buf:tt)*);
        (Ook! Ook? $($tail:tt)*))
    => {
        // На один уровень глубже.
        Ook!(@x $syms; (@ $($depth)*); ($($buf)* Ook! Ook?); ($($tail)*));
    };
    
    (@x $syms:tt; (@ $($depth:tt)*); ($($buf:tt)*);
        (Ook? Ook! $($tail:tt)*))
    => {
        // На один уровень выше.
        Ook!(@x $syms; ($($depth)*); ($($buf)* Ook? Ook!); ($($tail)*));
    };
```

Наконец, у нас есть правило для "всего остального". Обратите внимание на метапеременные `$op0` и `$op1`: как подразумевается в Rust, наш токен Ook! - это всегда *два* токена в Rust: идентификатор `Ook` и другой токен. Таким образом, мы можем найти все нецикличные коды операций по совпадению с `!`, `?` и `.` вместо `tt`.

Здесь мы оставляем `$depth` нетронутым и просто добавляем коды операций в буфер.

```ignore
    (@x $syms:tt; $depth:tt; ($($buf:tt)*);
        (Ook $op0:tt Ook $op1:tt $($tail:tt)*))
    => {
        Ook!(@x $syms; $depth; ($($buf)* Ook $op0 Ook $op1); ($($tail)*));
    };
```

### Пропуск цикла

Это *в целом* то же самое, что и извлечение цикла, кроме того, что мы не заботимся о *содержании* цикла (и, в связи с этим, нам не нужен буфер накопления). Все, что нам нужно - это знать когда мы *прошли* мимо цикла. В этом случае мы продолжаем обрабатывать вход, используя правила `@e`.

В связи с вышесказанным, эти правила представлены без дальнейших объяснений.

```ignore
    // Конец цикла
    (@s $syms:tt; ();
        (Ook? Ook! $($tail:tt)*))
    => {
        Ook!(@e $syms; ($($tail)*));
    };

    // Вход во вложенный цикл.
    (@s $syms:tt; ($($depth:tt)*);
        (Ook! Ook? $($tail:tt)*))
    => {
        Ook!(@s $syms; (@ $($depth)*); ($($tail)*));
    };
    
    // Выход из вложенного цикла.
    (@s $syms:tt; (@ $($depth:tt)*);
        (Ook? Ook! $($tail:tt)*))
    => {
        Ook!(@s $syms; ($($depth)*); ($($tail)*));
    };

    // Не связанный с циклами код операции.
    (@s $syms:tt; ($($depth:tt)*);
        (Ook $op0:tt Ook $op1:tt $($tail:tt)*))
    => {
        Ook!(@s $syms; ($($depth)*); ($($tail)*));
    };
```

### Точка входа

Это единственное не-внутреннее правило.

Стоит отметить, что этот шаблон просто ищет совпадения по *всем* токенам, переданным ему, и поэтому он  *чрезвычайно опасен*.  Любая ошибка может привести к неправильному распознаванию всех правил выше, что может в свою очередь привести к бесконечной рекурсии.

Когда вы пишете, изменяете, или занимаетесь отладкой макроса как этот, будет уместно временно подставить в начало что-либо, навроде `@entry`.  Это позволит избежать случая бесконечной рекурсии, и, с большей вероятностью, приведет к ошибкам распознавания в правильных местах.

```ignore
    ($($Ooks:tt)*) => {
        Ook!(@start $($Ooks)*)
    };
}
```

### Использование

Вот, наконец, и наша тестовая программа.

```ignore
fn main() {
    let _ = Ook!(
        Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook! Ook?  Ook? Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook?  Ook! Ook!  Ook? Ook!  Ook? Ook.
        Ook! Ook.  Ook. Ook?  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook?  Ook? Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook?
        Ook! Ook!  Ook? Ook!  Ook? Ook.  Ook. Ook.
        Ook! Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook! Ook.  Ook! Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook.  Ook. Ook?  Ook. Ook?
        Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook?  Ook? Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook?
        Ook! Ook!  Ook? Ook!  Ook? Ook.  Ook! Ook.
        Ook. Ook?  Ook. Ook?  Ook. Ook?  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook?  Ook? Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook?  Ook! Ook!  Ook? Ook!  Ook? Ook.
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook.
        Ook? Ook.  Ook? Ook.  Ook? Ook.  Ook? Ook.
        Ook! Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook! Ook.  Ook! Ook!  Ook! Ook!  Ook! Ook!
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook.
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!
        Ook! Ook.  Ook. Ook?  Ook. Ook?  Ook. Ook.
        Ook! Ook.  Ook! Ook?  Ook! Ook!  Ook? Ook!
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook! Ook.
    );
}
```

На выходе после запуска (после продолжительной паузы, во время которой компилятор выполнит сотни рекурсивных разворотов макроса) получается следующее:

```text
Hello World!
```

Этим мы показали ужасающую правду, что  `macro_rules!` является тьюринг-полным!

### В дополнение

Представленное выше основано на макро-реализации изоморфного языка "Hodor!".  Manish Goregaokar затем [реализовал интерпретатор Brainfuck, используя макрос Hodor!](https://www.reddit.com/r/rust/comments/39wvrm/hodor_esolang_as_a_rust_macro/cs76rqk?context=10000).  Таким образом, это интерпретатор  Brainfuck, написанный на  Hodor!, который, в свою очередь, написан на  `macro_rules!`.

Легенда гласит, что после увеличения лимита рекурсии до *трех миллионов* и запуска на *четыре дня*, он наконец выполнился.

...переполнением стека и падением. И по настоящий день, эзоязык-как-макрос остается абсолютно *нежизнеспособным* методом разработки на Rust.