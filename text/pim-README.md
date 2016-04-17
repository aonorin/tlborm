% Макросы, Практическое введение

В этой главе мы расскажем о системе макросов Rust, называемой "макрос-по-
примеру". Рассказывать будем, рассматривая простой, но очень практичный макрос.
*Не* будем пытаться объяснить всю сложность системы; наша цель - сделать так,
чтобы вы чувствовали себя комфортно с макросами и понимали, как и для чего они
пишутся.

Есть также [глава по макросам в Rust Book](http://doc.rust-lang.org/book/macros.html),
в которой объяснения даются на более высоком уровне и [методическое введение](mbe-README.html) 
- глава в этой книге, которая объясняет систему макросов подробно.

## Немного контекста

> **Внимание**: не паникуйте! Дальше пойдет разговор о математике. Вы можете
спокойно пропустить этот раздел, если хотите добраться до самого мяса этой
статьи.

Возможно, вы уже знакомы с термином "рекуррентое соотношение". На всякий случай,
напомним, рекуррентное соотношение - это последовательность, в которой каждое
следующее значение определяется в терминах одного или нескольких *предыдущих*, с
одним или несколькими начальными значениями. Например, [последовательность Фибоначчи](https://en.wikipedia.org/wiki/Fibonacci_number) можно описать такой
связью:

<!-- Начало математики: $F_n = 0, 1, \ldots, F_n-1 + F_n - 2$ -->
<style type="text/css">
    .katex {
        font: 400 1.21em/1.2 KaTeX_Main;
        white-space: nowrap;
        font-family: "Cambria Math", "Cambria", serif;
    }

    .katex .vlist > span > {
        display: inline-block;
    }

    .mathit {
        font-style: italic;
    }

    .katex .reset-textstyle.scriptstyle {
        font-size: 0.7em;
    }

    .katex .reset-textstyle.textstyle {
        font-size: 1em;
    }

    .katex .textstyle > .mord + .mrel {
        margin-left: 0.27778em;
    }

    .katex .textstyle > .mrel + .minner, .katex .textstyle > .mrel + .mop, .katex .textstyle > .mrel + .mopen, .katex .textstyle > .mrel + .mord {
        margin-left: 0.27778em;
    }

    .katex .textstyle > .mclose + .minner, .katex .textstyle > .minner + .mop, .katex .textstyle > .minner + .mord, .katex .textstyle > .mpunct + .mclose, .katex .textstyle > .mpunct + .minner, .katex .textstyle > .mpunct + .mop, .katex .textstyle > .mpunct + .mopen, .katex .textstyle > .mpunct + .mord, .katex .textstyle > .mpunct + .mpunct, .katex .textstyle > .mpunct + .mrel {
        margin-left: 0.16667em;
    }

    .katex .textstyle > .mord + .mbin {
        margin-left: 0.22222em;
    }

    .katex .textstyle > .mbin + .minner, .katex .textstyle > .mbin + .mop, .katex .textstyle > .mbin + .mopen, .katex .textstyle > .mbin + .mord {
        margin-left: 0.22222em;
    }
</style>

<div class="katex" style="font-size: 100%; text-align: center;">
    <span class="katex"><span class="katex-inner"><span style="height: 0.68333em;" class="strut"></span><span style="height: 0.891661em; vertical-align: -0.208331em;" class="strut bottom"></span><span class="base textstyle uncramped"><span class="reset-textstyle displaystyle textstyle uncramped"><span class="mord displaystyle textstyle uncramped"><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord mathit">n</span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span><span class="mrel">=</span><span class="mord">0</span><span class="mpunct">,</span><span class="mord">1</span><span class="mpunct">,</span><span class="mpunct">…</span><span class="mpunct">,</span><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord scriptstyle cramped"><span class="mord mathit">n</span><span class="mbin">−</span><span class="mord">1</span></span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span><span class="mbin">+</span><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord scriptstyle cramped"><span class="mord mathit">n</span><span class="mbin">-</span><span class="mord">2</span></span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span></span></span></span></span></span>
</div>
<!-- Конец Математики -->

Итак, первые два числа в последовательности - 0 и 1, а третье -
<em>F<sub>0</sub></em> + <em>F<sub>1</sub></em> = 0 + 1 = 1, четвертое -
<em>F<sub>1</sub></em> + <em>F<sub>2</sub></em> = 1 + 1 = 2, и так далее.

Описание функции `fibonacci` получилось довольно хитрым, *потому что* такая
последовательность может продолжаться бесконечно, но ведь вам же не нужно
возвращать список всех элементов. Все, что вы *хотите* - это вернуть что-то,
лениво считающее определенное количество элементов.

В Rust это достигается путём создания `Iterator`. Это не особо *трудно*, хотя и
требует довольно много рутинных действий: нужно определить свой тип, понять,
какое состояние хранить в нем, и затем реализовать типаж `Iterator` для него.

В то же время рекуррентное соотношение настолько просто, что почти от всех
конкретных деталей можно абстрагироваться и создать маленький генератор кода на
базе макроса.

Итак, поняв, что мы хотим, начнем.

## Создание

Обычно, если я работаю над новым макросом, первое, что я решаю - это то, как
будет выглядеть вызов макроса. В данном конкретном случае при первом приближении
получится следующее:

```ignore
let fib = recurrence![a[n] = 0, 1, ..., a[n-1] + a[n-2]];

for e in fib.take(10) { println!("{}", e) }
```

После этого можем перейти к определению макроса, даже при том, что мы не
уверены, во что он должен разворачиваться. Это полезно, потому что если вам
непонятно, как разбирать входящий синтаксис, то, *возможно*, вам придется
переделать его.

```rust
macro_rules! recurrence {
    ( a[n] = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}
# fn main() {}
```

Подразумевая, что вы не знакомы с синтаксисом, позвольте мне объяснить это
определение. Здесь представлено определение макроса, созданное с помощью системы
`macro_rules!`, с названием `recurrence!`. У этого макроса одно правило разбора.
Это правило говорит, что вход данного макроса должен совпадать с:

- последовательностью литеральных токенов `a` `[` `n` `]` `=`,
- повторяющейся один или больше раз(`+`) последовательностью (`$( ... )`), 
использующей `,` в качестве разделителя, содержащей:
    - правильные *выражения*, захваченные в переменную `inits` (`$inits:expr`)
- последовательностью литеральных токенов `,` `...` `,`,
- правильного *выражения*, захваченного в переменную `recur` (`$recur:expr`).

Наконец, правило говорит, *если* вход совпадает с образцом, то вызов макроса
нужно заменить на последовательность токенов `/* ... */`.

Стоит отметить, что `inits`, как понятно из названия, на самом деле содержит
*все* выражения, которые совпадают на этой позиции, а не только первое или
последнее. Более того, они захватываются *как последовательность*, а не
склеиваются все вместе в одно. Также помните, что вы можете изменить количество
повторений на "ноль и больше" раз, используя `*` вместо `+`. Поддержки "нуля
или одного" или еще более конкретного числа повторений тут нет.

В качестве упражнения давайте возьмем прелагаемый вход и пропустим его через
правило, чтобы посмотреть, как оно будет обрабатываться. Колонка "позиция",
показывающая, какая часть паттерна должна совпасть следующей, отмечается "⌂".
Помните, что в некоторых случаях может быть больше одного возможного
"следующего" элемента, с которым найдется совпадение. "Вход" будет содержать все
токены, которые еще *не* были обработаны. `inits` и `recur` будут содержать
содержимое их выражений.

<style type="text/css">
    /* Customisations. */

    .small-code code {
        font-size: 60%;
    }

    table pre.rust {
        margin: 0;
        border: 0;
    }

    table.parse-table code {
        white-space: pre-wrap;
        background-color: transparent;
        border: none;
    }

    table.parse-table tbody > tr > td:nth-child(1) > code:nth-of-type(2) {
        color: red;
        margin-top: -0.7em;
        margin-bottom: -0.6em;
    }

    table.parse-table tbody > tr > td:nth-child(1) > code {
        display: block;
    }

    table.parse-table tbody > tr > td:nth-child(2) > code {
        display: block;
    }
</style>

<table class="parse-table">
    <thead>
        <tr>
            <th>Позиция</th>
            <th>Вход</th>
            <th><code>inits</code></th>
            <th><code>recur</code></th>
        </tr>
    </thead>
    <tbody class="small-code">
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>⌂</code></td>
            <td><code>a[n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code> ⌂</code></td>
            <td><code>[n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>  ⌂</code></td>
            <td><code>n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>   ⌂</code></td>
            <td><code>] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>     ⌂</code></td>
            <td><code>= 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>       ⌂</code></td>
            <td><code>0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂</code></td>
            <td><code>0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                     ⌂  ⌂</code></td>
            <td><code>, 1, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code></td>
            <td></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>Внимание</em>: здесь два ⌂, потому что следующий входной токен можно
сопоставить и <em>с</em> запятой-разделителем <em>между</em> элементами в
повторении, <em>и с</em> запятой <em>после</em> повторения. Система макроса будет
помнить обе возможности, до тех пор пока не сможет определить, какую
выбрать.

            </td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂                ⌂</code></td>
            <td><code>1, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                     ⌂  ⌂ <s>⌂</s></code></td>
            <td><code>, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>Внимание</em>: третий, подчеркнутый маркер показывает, что система макроса
после обработки последнего токена удалила одну из предыдущих возможных веток.

            </td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂                ⌂</code></td>
            <td><code>..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         <s>⌂</s>                    ⌂</code></td>
            <td><code>, a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                                ⌂</code></td>
            <td><code>a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                                           ⌂</code></td>
            <td></td>
            <td><code>0</code>, <code>1</code></td>
            <td><code>a[n-1] + a[n-2]</code></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

                <em>Внимание</em>: этот конкретный шаг должен объяснить, что
                такая связь, как <tt>$recur:expr</tt>, заменит <em>все
                выражение</em>, используя знания компилятора о том, что считать
                правильным выражением. Как будет рассказано дальше, вы можете
                так делать и с другими конструкциями языка.
            </td>
        </tr>
    </tbody>
</table>

<p></p>

Понять из этого всего нужно то, что система макросов будет *пытаться*
последовательно сопоставить предложенные на входе токены с каждым правилом. Мы
ещё поговорим о том, почему именно "пытаться".

Теперь давайте напишем законченную полностью развернутую форму. Для этого
развертывания я хотел получить что-то вроде этого:

```ignore
let fib = {
    struct Recurrence {
        mem: [u64; 2],
        pos: usize,
    }
```

Это будет тип итератора. `mem` будет буфером в памяти, который будет содержать
несколько последних значений, достаточных для продолжения рекуррентных
вычислений. `pos` должен следить за значением `n`.

> **Пояснение**: Я выбрал `u64` как "достаточно большой" тип для элементов этой
последовательности. Не волнуйтесь о том, как это будет работать для
последовательностей *других* типов; мы еще вернемся к этому.

```ignore
    impl Iterator for Recurrence {
        type Item = u64;

        #[inline]
        fn next(&mut self) -> Option<u64> {
            if self.pos < 2 {
                let next_val = self.mem[self.pos];
                self.pos += 1;
                Some(next_val)
```

Нам нужна ветка, которая будет заполнять начальные значения последовательности;
ничего необычного.

```ignore
            } else {
                let a = /* something */;
                let n = self.pos;
                let next_val = (a[n-1] + a[n-2]);

                self.mem.TODO_shuffle_down_and_append(next_val);

                self.pos += 1;
                Some(next_val)
            }
        }
    }
```

Тут все немного труднее; мы еще вернемся и посмотрим, *как* именно определить
`a`. Также и `TODO_shuffle_down_and_append` - еще один временный заполнитель;
Мне нужно что-то, что поставит `next_val` в конец массива, сдвинет все остальное
вниз на одну позицию и удалит 0-й элемент.

```ignore

    Recurrence { mem: [0, 1], pos: 0 }
};

for e in fib.take(10) { println!("{}", e) }
```

В конце вернем экземпляр нашей новой структуры, который затем можно будет
итерировать. Объединяя все выше написанное, полное развертывание будет выглядеть
так:

```ignore
let fib = {
    struct Recurrence {
        mem: [u64; 2],
        pos: usize,
    }

    impl Iterator for Recurrence {
        type Item = u64;

        #[inline]
        fn next(&mut self) -> Option<u64> {
            if self.pos < 2 {
                let next_val = self.mem[self.pos];
                self.pos += 1;
                Some(next_val)
            } else {
                let a = /* something */;
                let n = self.pos;
                let next_val = (a[n-1] + a[n-2]);

                self.mem.TODO_shuffle_down_and_append(next_val.clone());

                self.pos += 1;
                Some(next_val)
            }
        }
    }

    Recurrence { mem: [0, 1], pos: 0 }
};

for e in fib.take(10) { println!("{}", e) }
```

> **Пояснение**: Да, это *действительно* означает, что мы определяем разные
структуры `Recurrence` и их реализации при каждом вызове макроса.
Большинство из этого мы оптимизируем в конце разумным использованием
атрибута `#[inline]`.

Очень полезно проверять развертывание во время его написания. Если вы заметите,
что в развертывании что-то должно быть переменным при вызове, но на текущий
момент не входит в синтаксис макроса, вы должны решить, как дать возможность
изменения данного параметра. В данном случае мы добавили `u64`, но пользователь
может захотеть что-то другое. Поэтому давайте изменим макрос.

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}

/*
let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

for e in fib.take(10) { println!("{}", e) }
*/
# fn main() {}
```

Здесь я добавил новый захват в метапеременную: `sty`, которая обозначает тип.

> **Пояснение**: Если вам интересно, кусок, идущий после двоеточия в захвате, 
может быть одним из определенных типов сопоставления синтаксиса. 
Самые распространенные - это `item`, `expr` и `ty`. Полное объяснение дается в 
[Макросы, Методическое введение; `macro_rules!` (Захват)](mbe-macro-rules.html#captures).

>
> Есть тут еще одна интересная вещь: в интересах будущего улучшения языка 
компилятор, исходя из типа сопоставления, ограничивает то, какие токены вы 
можете ставить *после* него. Обычно с этим можно столкнуться, когда 
выполняется сопоставление с выражениями и утверждениями; за ними может идти 
*только*  `=>`, `,` и `;`.
>
> Полный список можно найти в [Макросы, Методическое введение; Мелочи; Вернемся к Метапеременным и Развертыванию](mbe-min-captures-and-expansion-redux.html).

## Индексирование и сдвиг

Я не буду уделять много внимания этой части, потому что она не затрагивает
напрямую макросы. Мы хотим сделать так, чтобы пользователь смог обращаться к
предыдущим значениям в последовательности, индексируя `a`; мы хотим, чтобы это
работало как сдвигающееся окно, в котором видны только последние элементы 
последовательности (в данном случае - 2 элемента).

Используя тип-обертку, можем сделать это довольно просто:

```ignore
struct IndexOffset<'a> {
    slice: &'a [u64; 2],
    offset: usize,
}

impl<'a> Index<usize> for IndexOffset<'a> {
    type Output = u64;

    #[inline(always)]
    fn index<'b>(&'b self, index: usize) -> &'b u64 {
        use std::num::Wrapping;

        let index = Wrapping(index);
        let offset = Wrapping(self.offset);
        let window = Wrapping(2);

        let real_index = index - offset + window;
        &self.slice[real_index.0]
    }
}
```

> **Пояснение**: из-за того, что время жизни *вгоняет в ступор* новичков в Rust, 
по-быстрому объясню: `'a` и `'b` - это параметры времени жизни, которые 
используются для слежения за ссылками (*т.е.* захваченными указателями на 
какие-то данные). В этом случае `IndexOffset` захватывает ссылку на наши данные 
итератора, поэтому нам надо следить, как долго ей можно удерживать их в себе, и 
для этого используется `'a`.
>
> `'b` используется, потому что функция `Index::index`  (то, как на самом деле 
реализуется синтаксис индексации массива) *также* параметризирована временем жизни для 
обеспечения возврата захваченной ссылки. `'a` и `'b` не обязательно совпадают во 
всех возможных случаях. Анализатор заимствований должен убедиться, что, даже 
если мы явно не сопоставим `'a` и `'b` друг с другом, мы на самом деле по 
неосторожности не нарушим целостность памяти.

Меняем определение `a` на:

```ignore
let a = IndexOffset { slice: &self.mem, offset: n };
```

Единственный оставшийся вопрос - что делать с `TODO_shuffle_down_and_append`? Я
не смог найти метод в стандартной библиотеке, совпадающий по семантике с тем,
что мне нужно, но его и нетрудно сделать самому.

```ignore
{
    use std::mem::swap;

    let mut swap_tmp = next_val;
    for i in (0..2).rev() {
        swap(&mut swap_tmp, &mut self.mem[i]);
    }
}
```

Здесь новое значение перемещается в конец массива, сдвигая остальные элементы на
один вниз.

> **Пояснение**: делая так, знайте, что этот код будет работать также и для 
типов, не поддерживающих копирование.

Рабочий код теперь выглядит так:

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}

fn main() {
    /*
    let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
    */
    let fib = {
        use std::ops::Index;

        struct Recurrence {
            mem: [u64; 2],
            pos: usize,
        }

        struct IndexOffset<'a> {
            slice: &'a [u64; 2],
            offset: usize,
        }

        impl<'a> Index<usize> for IndexOffset<'a> {
            type Output = u64;

            #[inline(always)]
            fn index<'b>(&'b self, index: usize) -> &'b u64 {
                use std::num::Wrapping;

                let index = Wrapping(index);
                let offset = Wrapping(self.offset);
                let window = Wrapping(2);

                let real_index = index - offset + window;
                &self.slice[real_index.0]
            }
        }

        impl Iterator for Recurrence {
            type Item = u64;

            #[inline]
            fn next(&mut self) -> Option<u64> {
                if self.pos < 2 {
                    let next_val = self.mem[self.pos];
                    self.pos += 1;
                    Some(next_val)
                } else {
                    let next_val = {
                        let n = self.pos;
                        let a = IndexOffset { slice: &self.mem, offset: n };
                        (a[n-1] + a[n-2])
                    };

                    {
                        use std::mem::swap;

                        let mut swap_tmp = next_val;
                        for i in (0..2).rev() {
                            swap(&mut swap_tmp, &mut self.mem[i]);
                        }
                    }

                    self.pos += 1;
                    Some(next_val)
                }
            }
        }

        Recurrence { mem: [0, 1], pos: 0 }
    };

    for e in fib.take(10) { println!("{}", e) }
}
```

Заметьте, что я поменял порядок объявления `n` и `a`, а также обернул их (вместе
с рекуррентным выражением) в блок. Причина для первого довольна тривиальна (`n`
должна быть определена раньше, чтобы я мог использовать его для `a`). Причина
для второго - это то, что заимствованная ссылка `&self.mem` не дает произойти
дальнейшим сдвигам (вы не можете изменить то, что связано в другом месте).
Обертывание в блок гарантирует, что заимствование `&self.mem` заканчивается до
него.

Между прочим, единственной причиной, по которой код, делающий сдвиг `mem`,
находится внутри блока, является желание приблизить область видимости, в которой
доступен `std::mem::swap`, просто ради аккуратности.

Если мы выполним этот код, то получим:

```text
0
1
2
3
5
8
13
21
34
```

Это успех! Теперь давайте скопируем и вставим код в развертывание макроса, и
заменим развернутый код на его вызов. Получим:

```ignore
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => {
        {
            /*
                Идущий дальше код, это, *буквально*, код выше,
                вырезанный и вставленный на новую позицию. Никаких изменений
                больше не выполнялось.
            */

            use std::ops::Index;

            struct Recurrence {
                mem: [u64; 2],
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [u64; 2],
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = u64;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b u64 {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(2);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = u64;

                #[inline]
                fn next(&mut self) -> Option<u64> {
                    if self.pos < 2 {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            (a[n-1] + a[n-2])
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..2).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [0, 1], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

Очевидно, мы еще не *используем* захват в метапеременные, но можем
добавить его довольно просто. Однако, если мы попытаемся скомпилировать код,
`rustc` вернет ошибку, говорящую нам:

```text
recurrence.rs:69:45: 69:48 error: local ambiguity: multiple parsing options: built-in NTs expr ('inits') or 1 other options.
recurrence.rs:69     let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];
                                                             ^~~
```

Вот тут мы и попались в ограничение `macro_rules`. Проблемой является вторая
запятая. Когда `macro_rules` видит ее во время развертывания, он не может
решить, разобрать ему *следующее* выражение как `inits` или как `...`. К
сожалению, он не такой умный, чтобы понять, что `...` не является правильным
выражением, и поэтому он сдается. Теоретически, все должно работать, но на самом
деле нет.

> **Пояснение**: *На самом деле* я немного приврал о том, как наше правило было бы 
интерпретировано системой макросов. В общем, оно *должно было бы* работать как описано, 
но не в этом случае. Устройство `macro_rules`, как оно сейчас есть, имеет свои 
слабости, и всегда стоит помнить, что в некоторых случаях вам придется исказить код 
немного, чтобы заставить его работать.
>
> В этом *конкретном* случае две проблемы. Во-первых - система макросов не знает, 
что составляет некоторые грамматические элементы, а что нет (*например*, 
выражения); это работа парсера. Поэтому она не знает, что `...` не является 
выражением. Во-вторых - она не может попытаться захватить составной грамматический 
элемент (такой, как выражение), если он на 100% не совпадает.
>
> Другими словами, она может попросить парсер попытаться разобрать вход как 
выражение, а парсер ответит на любую проблему отменой с ошибкой. Единственным 
способом, как система макроса сможет работать - просто попытаться избежать 
ситуаций, в которых это может стать проблемой.
>
> К положительному моменту можно отнести то, что *никто* не в восторге от этого.
  Ключевое слово `macro` уже зарезервировано для будущего более строгого 
  определения системы макросов. А пока страдаем.

К счастью, решение очень простое: мы удаляем запятую из синтаксиса. Чтобы
удержать равновесие, мы удаляем *обе* запятые вокруг `...`:

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
//                                     ^~~ изменено
        /* ... */
#         // Чит :D
#         (vec![0u64, 1, 2, 3, 5, 8, 13, 21, 34]).into_iter()
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
//                                         ^~~ изменено

    for e in fib.take(10) { println!("{}", e) }
}
```

Это успех! Теперь можем заменить вещи из *развертывания* на 
*захваченные метапеременные*.

### Замена

Заменить то, что вы захватили в метапеременную, очень просто; вы заменяете
содержимое захваченного в `$sty:ty` на `$sty`. Итак, давайте пройдемся и
исправим все `u64`:

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
        {
            use std::ops::Index;

            struct Recurrence {
                mem: [$sty; 2],
//                    ^~~~ изменено
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; 2],
//                          ^~~~ изменено
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;
//                            ^~~~ изменено

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
//                                                          ^~~~ изменено
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(2);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;
//                          ^~~~ изменено

                #[inline]
                fn next(&mut self) -> Option<$sty> {
//                                           ^~~~ изменено
                    /* ... */
#                     if self.pos < 2 {
#                         let next_val = self.mem[self.pos];
#                         self.pos += 1;
#                         Some(next_val)
#                     } else {
#                         let next_val = {
#                             let n = self.pos;
#                             let a = IndexOffset { slice: &self.mem, offset: n };
#                             (a[n-1] + a[n-2])
#                         };
#     
#                         {
#                             use std::mem::swap;
#     
#                             let mut swap_tmp = next_val;
#                             for i in (0..2).rev() {
#                                 swap(&mut swap_tmp, &mut self.mem[i]);
#                             }
#                         }
#     
#                         self.pos += 1;
#                         Some(next_val)
#                     }
                }
            }

            Recurrence { mem: [1, 1], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

Давайте решим вопрос посложнее: как превратить `inits` в массив литералов 
`[0,1]` *и* в массив типов `[$sty; 2]`. Для первого мы можем сделать так:

```ignore
            Recurrence { mem: [$($inits),+], pos: 0 }
//                             ^~~~~~~~~~~ изменено
```

Это полная противоположность захвату в метапеременные: повтор `inits` один или
несколько раз, каждый из которых отделяется запятой. Это развернется в ожидаемую
последовательность токенов: `0, 1`.

Почему-то превратить `inits` в литерал `2` немного сложнее. Оказывается, нет
прямого способа это сделать, но мы *можем* исправить это, написав второй макрос.
Давайте разберёмся в этом по шагам.

```rust
macro_rules! count_exprs {
    /* ??? */
#     () => {}
}
# fn main() {}
```

Это очевидный случай: получив ноль выражений, вы ожидаете, что `count_exprs` 
развернется в литерал `0`.

```rust
macro_rules! count_exprs {
    () => (0);
//  ^~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     assert_eq!(_0, 0);
# }
```

> **Пояснение**: Вы должно быть заметили, что я использую круглые скоби вместо 
фигурных для развертывания. `macro_rules` все равно, *какие* скобки вы 
используете, если это любые из пар: `( )`, `{ }` или `[ ]`. На самом деле вы 
можете использовать любые скобки у самого макроса (*т.е.* скобки сразу после 
имени макроса), в сопоставлении с образцом в правилах синтаксиса, и в 
сопоставлении с образцом вокруг соответствующего развёртывания.
>
> Вы можете также использовать любые скобки при *вызове* макроса, но с 
ограничениями: макрос, вызываемый как  `{ ... }` или `( ... );` будет *всегда* 
разбираться как *элемент* (*т.e.*, как объявление `struct` или `fn`). Это важно
 учитывать при использовании макросов внутри тела функций; это помогает 
 устранить неоднозначность между тем, что делать - "разбирать как выражение" и 
 "разбирать как утверждение".

Что если у вас *одно* выражение? Этому должен соответствовать литерал `1`.

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
//  ^~~~~~~~~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
# }
```

Два?

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (2);
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
# }
```

Мы можем "упростить" это, по-другому выразив случай с двумя выражениями через
рекурсию.

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (1 + count_exprs!($e1));
//                           ^~~~~~~~~~~~~~~~~~~~~ изменено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
# }
```

Это работает, Rust сможет преобразовать `1 + 1` в константное значение. Что если
у нас три выражения?

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (1 + count_exprs!($e1));
    ($e0:expr, $e1:expr, $e2:expr) => (1 + count_exprs!($e1, $e2));
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     const _3: usize = count_exprs!(x, y, z);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
#     assert_eq!(_3, 3);
# }
```

> **Пояснение**: Вам должно быть интересно, можем ли мы поменять порядок 
следования правил. В данном конкретном случае, *да*, но система макросов может 
быть иногда требовательна к этому и не пожелает работать. Если вы напишете 
макрос с несколькими правилами, который, вы готовы поклясться, должен работать, но 
выдает ошибки на неожиданных токенах, попробуйте поменять порядок следования 
правил.

Мы надеемся, последовательность понятна. Мы всегда можем уменьшить список 
выражений, выполняя совпадение с одним выражением, за которым следуют ноль и 
более выражений, разворачивая это в 1 + оставшееся количество выражений.

```rust
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ изменено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     const _3: usize = count_exprs!(x, y, z);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
#     assert_eq!(_3, 3);
# }
```

> **<abbr title="Только в этом примере">ТВЭП</abbr>**: это не *единственный*, и 
даже не *лучший* способ выполнить подсчет. Лучше использовать 
[Подсчет](blk-counting.html) из следующих глав.

Наконец, теперь мы можем изменить `recurrence` так, чтобы определить необходимый
размер `mem`.

```rust
// добавлено:
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
}

macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
        {
            use std::ops::Index;

            const MEM_SIZE: usize = count_exprs!($($inits),+);
//          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ добавлено

            struct Recurrence {
                mem: [$sty; MEM_SIZE],
//                          ^~~~~~~~ изменено
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; MEM_SIZE],
//                                ^~~~~~~~ изменено
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(MEM_SIZE);
//                                        ^~~~~~~~ изменено

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;

                #[inline]
                fn next(&mut self) -> Option<$sty> {
                    if self.pos < MEM_SIZE {
//                                ^~~~~~~~ изменено
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            (a[n-1] + a[n-2])
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..MEM_SIZE).rev() {
//                                       ^~~~~~~~ изменено
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [$($inits),+], pos: 0 }
        }
    };
}
/* ... */
# 
# fn main() {
#     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
# 
#     for e in fib.take(10) { println!("{}", e) }
# }
```

Сделав это, мы можем заменить последнюю вещь: выражение `recur`.

```ignore
# macro_rules! count_exprs {
#     () => (0);
#     ($head:expr $(, $tail:expr)*) => (1 + count_exprs!($($tail),*));
# }
# macro_rules! recurrence {
#     ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
#         {
#             const MEMORY: uint = count_exprs!($($inits),+);
#             struct Recurrence {
#                 mem: [$sty; MEMORY],
#                 pos: uint,
#             }
#             struct IndexOffset<'a> {
#                 slice: &'a [$sty; MEMORY],
#                 offset: uint,
#             }
#             impl<'a> Index<uint, $sty> for IndexOffset<'a> {
#                 #[inline(always)]
#                 fn index<'b>(&'b self, index: &uint) -> &'b $sty {
#                     let real_index = *index - self.offset + MEMORY;
#                     &self.slice[real_index]
#                 }
#             }
#             impl Iterator<u64> for Recurrence {
/* ... */
                #[inline]
                fn next(&mut self) -> Option<u64> {
                    if self.pos < MEMORY {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            $recur
//                          ^~~~~~ изменено
                        };
                        {
                            use std::mem::swap;
                            let mut swap_tmp = next_val;
                            for i in range(0, MEMORY).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }
                        self.pos += 1;
                        Some(next_val)
                    }
                }
/* ... */
#             }
#             Recurrence { mem: [$($inits),+], pos: 0 }
#         }
#     };
# }
# fn main() {
#     let fib = recurrence![a[n]: u64 = 1, 1 ... a[n-1] + a[n-2]];
#     for e in fib.take(10) { println!("{}", e) }
# }
```

И, когда мы скомпилируем наш законченный макрос...

```text
recurrence.rs:77:48: 77:49 error: unresolved name `a`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:50: 77:51 error: unresolved name `n`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                  ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:57: 77:58 error: unresolved name `a`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                         ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:59: 77:60 error: unresolved name `n`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                           ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
```

... постойте, что? Так быть не должно... проверим, во что разворачивается макрос.

```shell
$ rustc -Z unstable-options --pretty expanded recurrence.rs
```

Аргумент `--pretty expanded` говорит `rustc` выполнить развертывание макроса, и
затем вернуть получившееся AST обратно в исходный код. Эта опция не считается
стабильной, поэтому надо указать `-Z unstable-options`. Вывод (после
форматирования) показан ниже; в частности, обратите внимание на то место в коде,
где `$recur` был заменен:

```ignore
#![feature(no_std)]
#![no_std]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std as std;
fn main() {
    let fib = {
        use std::ops::Index;
        const MEM_SIZE: usize = 1 + 1;
        struct Recurrence {
            mem: [u64; MEM_SIZE],
            pos: usize,
        }
        struct IndexOffset<'a> {
            slice: &'a [u64; MEM_SIZE],
            offset: usize,
        }
        impl <'a> Index<usize> for IndexOffset<'a> {
            type Output = u64;
            #[inline(always)]
            fn index<'b>(&'b self, index: usize) -> &'b u64 {
                use std::num::Wrapping;
                let index = Wrapping(index);
                let offset = Wrapping(self.offset);
                let window = Wrapping(MEM_SIZE);
                let real_index = index - offset + window;
                &self.slice[real_index.0]
            }
        }
        impl Iterator for Recurrence {
            type Item = u64;
            #[inline]
            fn next(&mut self) -> Option<u64> {
                if self.pos < MEM_SIZE {
                    let next_val = self.mem[self.pos];
                    self.pos += 1;
                    Some(next_val)
                } else {
                    let next_val = {
                        let n = self.pos;
                        let a = IndexOffset{slice: &self.mem, offset: n,};
                        a[n - 1] + a[n - 2]
                    };
                    {
                        use std::mem::swap;
                        let mut swap_tmp = next_val;
                        {
                            let result =
                                match ::std::iter::IntoIterator::into_iter((0..MEM_SIZE).rev()) {
                                    mut iter => loop {
                                        match ::std::iter::Iterator::next(&mut iter) {
                                            ::std::option::Option::Some(i) => {
                                                swap(&mut swap_tmp, &mut self.mem[i]);
                                            }
                                            ::std::option::Option::None => break,
                                        }
                                    },
                                };
                            result
                        }
                    }
                    self.pos += 1;
                    Some(next_val)
                }
            }
        }
        Recurrence{mem: [0, 1], pos: 0,}
    };
    {
        let result =
            match ::std::iter::IntoIterator::into_iter(fib.take(10)) {
                mut iter => loop {
                    match ::std::iter::Iterator::next(&mut iter) {
                        ::std::option::Option::Some(e) => {
                            ::std::io::_print(::std::fmt::Arguments::new_v1(
                                {
                                    static __STATIC_FMTSTR: &'static [&'static str] = &["", "\n"];
                                    __STATIC_FMTSTR
                                },
                                &match (&e,) {
                                    (__arg0,) => [::std::fmt::ArgumentV1::new(__arg0, ::std::fmt::Display::fmt)],
                                }
                            ))
                        }
                        ::std::option::Option::None => break,
                    }
                },
            };
        result
    }
}
```

Но все вроде в порядке! Если мы добавим несколько нехватающих атрибутов
`#![feature(...)]` и запустим под ночной сборкой `rustc`, он даже компилируется!
... *как?!*

> **Пояснение**: У вас не получится скомпилировать это на не-ночной сборке 
`rustc`. Происходит это из-за того, что развёртывание макроса `println!` 
зависит от внутренних деталей компилятора, которые еще публично *не* стабилизированы.

### Соблюдение гигиены

Проблема здесь в том, что идентификаторы в макросах Rust обладают *гигиеной*. 
Поэтому идентификаторы из двух разных контекстов *не могут* сталкиваться. 
Чтобы объяснить разницу, возьмем простой пример.

```rust
# /*
macro_rules! using_a {
    ($e:expr) => {
        {
            let a = 42i;
            $e
        }
    }
}

let four = using_a!(a / 10);
# */
# fn main() {}
```

Макрос просто принимает выражение, оборачивает его в блок и определяет
переменную `a` внутри него. Мы используем его как окольный путь вычисления `4`.
На самом деле здесь *два* контекста синтаксиса, но они невидимы. Поэтому, для
помощи вам, дадим каждому контексту свой цвет. Начнем с неразвернутого кода, в
котором только один контекст:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

Теперь развернем вызов.

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

Как видно, <code><span class="synctx-1">a</span></code>, определяемая макросом,
находится в другом контексте по отношению к <code><span
class="synctx-0">a</span></code>, которую мы подсунули в наш вызов. Поэтому
компилятор считает их абсолютно разными идентификаторами, *не принимая во
внимание, что у них одинаковое лексическое представление*.

Это то, с чем надо быть *особенно* осторожным при работе с макросами: макросы
могут сформировать AST, которые не будут компилироваться, но, которые, если
написать от руки или с использованием `--pretty expanded`, все же
*компилируются*.

Решением здесь является захватить идентификатор *с подходящим контекстом
синтаксиса*. Чтобы сделать это, надо снова улучшить синтаксис нашего макроса.
Продолжая наш простой пример:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span>:<span class="ident">ident</span>, <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span>, <span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

Это развернется в:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> </span><span class="synctx-0"><span class="ident">a</span></span><span class="synctx-1"> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

Сейчас контексты совпадают, и код компилируется. Можем аналогичным образом
изменить наш `recurrence!`, явно захватывая `a` и `n`.  После изменений
получим:

```rust
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
}

macro_rules! recurrence {
    ( $seq:ident [ $ind:ident ]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
//    ^~~~~~~~~~   ^~~~~~~~~~ изменено
        {
            use std::ops::Index;

            const MEM_SIZE: usize = count_exprs!($($inits),+);

            struct Recurrence {
                mem: [$sty; MEM_SIZE],
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; MEM_SIZE],
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(MEM_SIZE);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;

                #[inline]
                fn next(&mut self) -> Option<$sty> {
                    if self.pos < MEM_SIZE {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let $ind = self.pos;
//                              ^~~~ изменено
                            let $seq = IndexOffset { slice: &self.mem, offset: $ind };
//                              ^~~~ изменено
                            $recur
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..MEM_SIZE).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [$($inits),+], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

И он компилируется! Теперь попробуем другую последовательность.

```rust
# macro_rules! count_exprs {
#     () => (0);
#     ($head:expr) => (1);
#     ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
# }
# 
# macro_rules! recurrence {
#     ( $seq:ident [ $ind:ident ]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
#         {
#             use std::ops::Index;
#             
#             const MEM_SIZE: usize = count_exprs!($($inits),+);
#     
#             struct Recurrence {
#                 mem: [$sty; MEM_SIZE],
#                 pos: usize,
#             }
#     
#             struct IndexOffset<'a> {
#                 slice: &'a [$sty; MEM_SIZE],
#                 offset: usize,
#             }
#     
#             impl<'a> Index<usize> for IndexOffset<'a> {
#                 type Output = $sty;
#     
#                 #[inline(always)]
#                 fn index<'b>(&'b self, index: usize) -> &'b $sty {
#                     use std::num::Wrapping;
#                     
#                     let index = Wrapping(index);
#                     let offset = Wrapping(self.offset);
#                     let window = Wrapping(MEM_SIZE);
#                     
#                     let real_index = index - offset + window;
#                     &self.slice[real_index.0]
#                 }
#             }
#     
#             impl Iterator for Recurrence {
#                 type Item = $sty;
#     
#                 #[inline]
#                 fn next(&mut self) -> Option<$sty> {
#                     if self.pos < MEM_SIZE {
#                         let next_val = self.mem[self.pos];
#                         self.pos += 1;
#                         Some(next_val)
#                     } else {
#                         let next_val = {
#                             let $ind = self.pos;
#                             let $seq = IndexOffset { slice: &self.mem, offset: $ind };
#                             $recur
#                         };
#     
#                         {
#                             use std::mem::swap;
#     
#                             let mut swap_tmp = next_val;
#                             for i in (0..MEM_SIZE).rev() {
#                                 swap(&mut swap_tmp, &mut self.mem[i]);
#                             }
#                         }
#     
#                         self.pos += 1;
#                         Some(next_val)
#                     }
#                 }
#             }
#     
#             Recurrence { mem: [$($inits),+], pos: 0 }
#         }
#     };
# }
# 
# fn main() {
for e in recurrence!(f[i]: f64 = 1.0 ... f[i-1] * i as f64).take(10) {
    println!("{}", e)
}
# }
```

Получим:

```text
1
1
2
6
24
120
720
5040
40320
362880
```

Это победа!
