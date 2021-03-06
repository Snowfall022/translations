# Тонкости value restriction в F# [Оригинал на английском](https://blogs.msdn.microsoft.com/mulambda/2010/05/01/finer-points-of-f-value-restriction)
Одной из отличительных особенностей языка F#, по сравнению с более распространёнными языками программирования, является мощный и всеобъемлющий автоматический вывод типов. Благодаря ему в программах на F# вы почти никогда не указываете типы явно, набираете меньше текста, и получаете в итоге более краткий, фантастически элегантный код.

Алгоритмы автоматического вывода типов — захватывающая тема; за ними стоит интересная и красивая теория. Сегодня мы рассмотрим один любопытный аспект автоматического вывода типов в F#, который, возможно, даст вам представление о том, какие сложности возникают в хороших современных алгоритмах такого рода, и, надеюсь, объяснит один камень преткновения, с которым время от времени сталкиваются F#-программисты.

Нашей темой сегодня будет <abbr title="value restriction">*ограничение на значения*</abbr>. На MSDN есть [хорошая статья](http://msdn.microsoft.com/en-us/library/dd233183(v=VS.100).aspx) на тему ограничения на значения и автоматического обобщения типов в F#, которая даёт очень разумные практические советы в том случае, если вы столкнулись с ним в вашем коде. Однако эта статья просто сухо констатирует: "*Компилятор выполняет автоматическое обобщение только на полных определениях функций с явно указанными аргументами и на простых неизменяемых значениях*", и не даёт этому практически никаких объяснений, что вполне справедливо, потому что MSDN — просто справочный материал. Данный пост сфокусирован на рассуждениях, лежащих в основе ограничения на значения — я буду отвечать на вопрос "почему?", а не "что делать?".

<abbr title="automatic generalization">*Автоматическое обобщение*</abbr> — мощная возможность автоматического вывода типов F#. Определим простую функцию — например, функцию тождества: 

```fsharp
let id x = x
```

Здесь нет явных <abbr title="type annotations">аннотаций типов</abbr>, но компилятор F# выводит для этой функции тип `'a -> 'a`: функция принимает аргумент некоего типа и возвращает результат точно такого же типа. Это не особенно сложно, но заметьте, что компилятор F# вывел неявный <abbr title="generic type parameter">тип-параметр</abbr> `'a` для функции `id`.

Мы можем скомбинировать функцию `id` с функцией `List.map`, которая сама по себе <abbr title="generic function">полиморфна</abbr>:

```fsharp
let listId l = List.map id l
```

(Не очень полезная функция, но сейчас важнее наглядность). Компилятор F# даёт функции `listId` корректный тип `'a list -> 'a list` — снова произошло автоматическое обобщение. Поскольку `List.map` — каррированная функция, у нас может возникнуть искушение отбросить аргумент `l` слева и справа:

```fsharp
let listId = List.map id
```

Но F# компилятор неожиданно возмущается:

```
Program.fs(8,5): error FS0030: Value restriction. 
The value 'listId' has been inferred to have generic type
    val listId : ('_a list -> '_a list)
Either make the arguments to 'listId' explicit or,
if you do not intend for it to be generic, add a type annotation.
```

Что происходит?

Статья на MSDN предлагает 4 способа исправить `let`-определение, которое отвергается компилятором из-за ограничения на значения:
* Ограничьте тип так, чтобы он перестал быть полиморфным, добавив явную аннотацию типа к значению или параметру.
* Если проблему вызывает <abbr title="nongeneralizable construct">необобщаемая конструкция</abbr> в определении полиморфной функции (такой, как композиция функций или частичное применение аргументов к каррированной функции), попробуйте переписать определение функции на обыкновеное.
* Если проблема заключается в выражении, слишком сложном для обобщения, превратите его в функцию, добавив неиспользуемый параметр.
* Добавьте явные полиморфные типы-параметры. Это применяется редко.

Первый способ не для нас — мы хотим, чтобы функция `listId` была полиморфной.

Второй способ вернёт нас к явному указанию параметра `list` — и это канонический способ определения функций вроде `listId` в языке F#.

Третий способ применим, когда нужно определить что-то, не являющееся функцией, в нашем случае это даёт неубедительный вариант:

```fsharp
let listId () = List.map id
```

В рабочем коде я бы воспользовался вторым вариантом решения и оставил параметр функции явным. Но ради обучения давайте попробуем "редко используемый" четвёртый способ:

```fsharp
let listId<'T> : 'T list -> 'T list = List.map id
```

Это компилируется и работает так, как и ожидалось. На первый взгляд кажется, что это ошибка вывода типов — компилятор не может определить тип, поэтому мы добавили аннотацию, чтобы ему помочь. Но подождите, компилятор почти вывел этот тип — он же упоминается в сообщении об ошибке (с таинственной <abbr title="type veriable">ти́повой переменной</abbr> `'_a`)! Будто бы компилятор был ошарашен конкретно этим случаем — почему?

Причина вполне разумна. Чтобы увидеть её, давайте рассмотрим другой случай ограничения на значения. Эта <abbr title="reference cell">ссылочная ячейка</abbr> на список не скомпилируется:

```fsahrp
let v = ref []
```

```
Program.fs(16,5): error FS0030: Value restriction. 
The value 'v' has been inferred to have generic type
   val v : '_a list ref    
Either define 'v' as a simple data term, make it a function with explicit arguments or, 
if you do not intend for it to be generic, add a type annotation.
```

Давайте обойдём это с помощью явных аннотаций типов:

```
> let v<'T> : 'T list ref = ref []
val v<'T> : 'T list ref
```

Компилятор доволен. Давайте попробуем присвоить `v` какое-нибудь значение:

```
> v := [1];;
val it : unit = ()
```

Правда же, `v` теперь ссылается на список с единственным элементом `1`?

```
> let x : int list = !v;;
val x : int list = []
```

Ой! Содержимое `v` - пустой список! Куда делся наш `[1]`?

Вот что произошло. Наше присваивание на самом деле может быть переписано так:

```fsharp
(v<int>):=[1]
```

Левая сторона этого присваивания — это применение `v` к <abbr title="type argument">аргументу типа</abbr> `int`. А `v`, в свою очередь, это не ссылочная ячейка, а <abbr title="type finction">функция типа</abbr>: получив на входе аргумент типа, она вернёт ссылочную ячейку. Наше выражение создаёт новую ссылочную ячейку и присваивает ей `[1]`. Аналогично, если мы явно укажем аргумент типа в разыменовании `v`:

```fsharp
let x = !(v<int>)
```

мы увидим, что `v` снова применяется к аргументу типа, и возвращает свежую ссылочную ячейку, содержащую пустой список.

Чтобы конкретизировать разговор о функциях типа, давайте изучим полученный IL-код. Если мы скомпилируем определение `v`, наш верный Reflector покажет нам, что `v` это:

```csharp
public static FSharpRef<FSharpList<T>> v<T>(){
    return Operators.Ref<FSharpList<T>>(FSharpList<T>.get_Empty());
}
```

То, что мы воспринимаем как значение в F#, на самом деле является обобщенным методом без параметров в соответствующем IL-коде. И присваивание, и разыменование `v` вызывают IL-метод, который будет каждый раз возвращать новую ссылочную ячейку.

Однако же, ничто в выражении

```fsharp
let v = ref []
```

не намекает на подобное поведение. Имя `v` выглядит как обыкновенное значение, а вовсе не метод и даже не функция. Если бы подобное определение было разрешено, F#-разработчиков ожидал бы неприятный сюрприз. Вот поэтому здесь компилятор перестаёт выводить полиморфные параметры — ограничение на значения оберегает вас от неожиданного поведения.

Итак, когда безопасно автоматическое обобщение? Сложно привести точные критерии, но напрашивается один простой ответ: обобщение безопасно, когда правая часть `let`-выражения одновременно:

1. Не содержит *побочных эффектов* (иными словами, *чистая*)
2. Возвращает *неизменяемый объект*

Действительно, причудливое поведение `v` возникает из-за изменяемости ссылочной ячейки; именно потому, что ссылочная ячейка изменяема, нам было важно, будет ли получена одна или разные ячейки в результате разных обращений к `v`. Если правая часть `let`-выражения не содержит побочных эффектов, мы знаем, что всегда получаем эквивалентные объекты, а так как они неизменяемы, нас не волнует, получаем ли мы одну и ту же или различные их копии в результате различных вызовов.

С точки зрения компилятора трудно, даже невозможно точно установить, выполнены ли вышеупомянутые условия. Поэтому компилятор использует простое и грубое, но естественное и понятное приближение: он обобщает определение только тогда, когда может вывести чистоту и неизменяемость из синтаксической структуры выражения в правой части `let`. Поэтому выражение, для которого оригинальное определение `let listId l = List.map id l` является синтаксическим сахаром

```fsharp
let listId = fun l -> List.map id
```

 обобщается — в правой части создание замыкания; создание замыкания не содержит побочных эффектов и замыкания неизменяемы.

Аналогично с размеченными объединениями:

```fsharp
let q = None
let z = []
```

и неизменяемыми записями:

```fsharp
type 'a r = { x : 'a; y : int }
let r1 = { x = []; y = 1 }
```

Здесь `r1` получает тип `'a list r`. Однако если вы попытаетесь проинициализировать какие-либо поля неизменяемой записи результатом вызова функции: 

```fsharp
let gen =
    let u = ref 0
    fun () -> u := !u + 1; !u
let f = { x = []; y = gen() }
```

то значение `f` не будет обобщено. В примере выше `gen` — это безусловно <abbr title="non-pure">грязная</abbr> функция; она могла бы быть чистой, но компилятор не может об этом знать, поэтому он из предосторожности возвращает ошибку. По этой же причине

```fsharp
let listId = List.map id
```

не обобщается — компилятор не знает, чистая функция `List.map` или нет.

Выражения, для которых компилятор на уровне синтаксиса может определить, что они чистые и возвращают неизменяемые объекты, называются *синтаксическими значениями*. Так ограничение на значения получило своё название — *автоматическое обобщение правой части `let`-выражения ограничено синтаксическими значениями*. Описание языка F# содержит полный список синтаксических значений, но наше обсуждение даёт представление о том, что это за выражения — чистые и возвращающие неизменяемые объекты.

Задача, которую мы здесь решаем, не нова — все компиляторы языков семейства ML используют ограничение на значения в той или иной форме. Уникальной же, на мой взгляд, особенностью F# является возможность обойти ограничение на значения с помощью явных аннотаций типов, и это безопасно с точки зрения семантики F#.

Когда это может быть полезно? Классический пример — `lazy` и `lazy list`. Типичное определение `lazy` (давайте притворимся, что его нет в нашем языке)

```fsharp
type 'a LazyInner = Delayed of (unit -> 'a) | Value of 'a | Exception of exn
type 'a Lazy = 'a LazyInner ref
let create f = ref (Delayed f)
let force (l : 'a Lazy) = ...
```

на первый взгляд, полно побочных эффектов; компилятору не известен контракт между `create` и `force`. Если мы построим `lazy list` обычным способом с помощью определения `lazy`

```fsharp
type 'a cell = Nil | CCons of 'a * 'a lazylist
and 'a lazylist = 'a cell Lazy
```

и попытаемся определить пустой ленивый список:

```fsharp
let empty = create (fun () -> Nil)
```

ограничение на значения не позволит нам это сделать; однако полиморфное использование ленивого списка абсолютно законно; мы можем заявить об этом, явно указав параметр полиморфного типа:

```fsharp
let empty<'T> : 'T lazylist = create (fun () -> Nil)
```

Этого достаточно, чтобы определение `empty` скомпилировалось, но если мы попытаемся его использовать:

```fsharp
let l = empty
```

компилятор снова возмутится:

```
File1.fs(12,5): error FS0030: Value restriction. 
The value 'l' has been inferred to have generic type
    val l : '_a lazylist    
Either define 'l' as a simple data term, 
make it a function with explicit arguments or, 
if you do not intend for it to be generic, add a type annotation.
```

В самом деле, компилятор знает, что `empty` — это функция типа, которая не подвергается автоматическому обобщению, так как она не принадлежит множеству синтаксических значений. F#, однако, предоставляет здесь лазейку — мы можем указать атрибут `[<GeneralizableValue>]` в определении `empty`: 

```fsharp
[<GeneralizableValue>]
let empty<'T> : 'T lazylist = create (fun () -> Nil)
```

Это заставит компилятор считать `empty` синтаксическим значением, и выражение `let l = empty` скомпилируется.

На самом деле, иногда определения вроде нашего полиморфного `v` могут быть полезными:

```fsharp
let v<'T> : 'T list ref = ref []
```

Если вы пишете функцию, <abbr title="type-parametrized">параметризуемую типами</abbr> и возвращающую изменяемые объекты или имеющую побочные эффекты, укажите атрибут `RequiresExplicitTypeArguments`:

```fsharp
[<RequiresExplicitTypeArguments>]
let v<'T> : 'T list ref = ref []
```

Он полностью соответствует своему названию: теперь вы не можете написать `v := [1]`, только `v<int> := [1]`, и будет понятнее, что происходит на самом деле.

Если вы всё это прочувствовали, я надеюсь, что у вас теперь есть чёткое понимание ограничений на значения в F#, и теперь вы можете при необходимости контролировать его с помощью явных аннотаций типов и атрибута `GeneralizableValue`. Вместе с силой, однако, приходит и ответственность: как правильно сказано в статье на MSDN, эти возможности редко используются в повседневном программировании на F#. В моём F#-коде функции типа появляются только в случаях, аналогичных `lazylist` — базовых случаях структур данных; во всех остальных ситуациях я следую советам из статьи на MSDN:
* Ограничьте тип так, чтобы он перестал быть полиморфным, добавив явную аннотацию типа к значению или параметру.
* Если проблема заключается в использовании необобщаемой конструкции для определения полиморфной функции, такой как композиция функций или частичное применение аргументов каррированной функции, попробуйте переписать определение функции на обыкновенное.
* Если проблема заключается в том, что выражение слишком сложно для обобщения, превратите его в функцию, добавив неиспользуемый параметр.
