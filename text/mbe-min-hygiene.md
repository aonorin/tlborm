% Гигиена

Макросы в Rust соблюдают гигиену *частично*. Говоря конкретно, они соблюдают ее,
когда дело доходит до большинства идентификаторов, но *не соблюдают ее*, когда
используются обобщенные типы или времена жизни.

Гигиена проявляется в виде добавления всем идентификаторам невидимого значения
"контекста синтаксиса". Если два идентификатора сравниваются, то для выявления
их идентичности сравниваются *как* текстовые имена идентификаторов, *так и* их
контексты синтаксиса.

Для того, чтобы это проиллюстрировать, возьмем следующий код:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

Будем использовать цвет фона, чтобы отметить контекст синтаксиса. Теперь давайте
развернем вызов макроса:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

Во-первых, напомним, что вызовы `macro_rules!` *исчезают* после
развертывания.

Во-вторых, если вы попытаетесь скомпилировать этот код, компилятор ответит что-
то навроде следующего:

```text
<anon>:11:21: 11:22 error: unresolved name `a`
<anon>:11 let four = using_a!(a / 10);
```

Заметьте, что цвет фона (*т.е.* контекст синтаксиса) для развернутого макроса
*меняется* как часть развертывания. Каждое развертывание макроса дает новый
уникальный контекст синтаксиса для своего содержимого. Как результат, здесь
*два различных `a`* в развернутом коде: один в первом контексте синтаксиса, а
второй - во втором. Другими словами, <code><span class="synctx-0">a</span></code>
- это не тот же идентификатор, что и  <code><span class="synctx-1">a</span></code>, 
какими бы похожими они не казались.

Тем не менее, токены, которые были подставлены *в* развертывание, *сохраняют* их
оригинальный контекст синтаксиса (в силу того, что были предоставлены макросу, а
не являлись частью самого макроса). Таким образом, для решения исправим макрос
следующим образом:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span>:<span class="ident">ident</span>, <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span>, <span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

Что развернется в:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> </span><span class="synctx-0"><span class="ident">a</span></span><span class="synctx-1"> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

Компилятор примет этот код, потому что используется теперь только одна  `a`.
