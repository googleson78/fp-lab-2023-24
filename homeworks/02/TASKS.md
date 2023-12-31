## Обвързване, интерпретация и компилация

По време на час видяхме следния тип, изразяващ структурата (по-често срещано под името абстрактно синтактично дърво в теоретичната информатика)
на "език за прости сметки":

```haskell
data Expr
  = Val Integer
  | Plus Expr Expr
  | Mult Expr Expr
  | If Expr Expr Expr
```

Конструкторите съответстваха на следните операции:
* `Val n` - връщаме числото `n`/оценява се до `n`
* `Plus e1 e2` - събираме резултатите на оценката на `e1` и `e2`
* `Mult e1 e2` - умножаваме резултатите на оценката на `e1` и `e2`
* `If cex tex eex` - оценява се до `tex`, ако `cex` се оценява до `0`, и се оценява до `eex`, в противен случай

    _Мнемоники_: `c(ond)ex(pression)`, `t(hen)ex(pression)`, `e(lse)ex(pression)`

Това съответствие по-конкретно изразихме като направихме интерпретатор за
`Expr`, пресмятащ стойност по израз:
```haskell
eval :: Expr -> Integer
eval (Val n) = n
eval (Plus e1 e2) = eval e1 + eval e2
eval (Mult e1 e2) = eval e1 * eval e2
eval (If cex tex eex) =
  eval $
    if eval cex == 0
    then tex
    else eex

-- Примери
-- > eval $ Val 5
-- 5
-- > eval $ Plus (Val 6) (Val 7)
-- 13
-- > eval $ Mult (Val 6) (Val 7)
-- 42
-- > eval $ If (Val 3) (Val 6) (Mult (Val 3) (Val 3))
-- 9
```
Интерпретатор е програма, която изпълнява код, без да се налага да го превежда до
машинен език преди това. Интерпретатор използваме всеки път когато изпълняваме програма на Lisp/Scheme, например.

Целта в това домашно ще е да направим следните разширения към нашия `Expr` език:
* добавяне на променливи, както и конструкция която "свързва" променлива
* добавяне на компилатор(/транспилатор) от `Expr` езика към друг език (в този случай лиспоподобен)
* имплементиране на оптимизация върху езика, която можем да приложим, преди да компилираме

Компилатор най-често наричаме програма, която превежда един език в друг.
По-специфично, често под "компилатор" се има предвид "превежда до машинен код(асемблер)",
а пък под "транспилатор" често се има предвид "до друг език от високо ниво"
(например `JavaScript`).

## Разширение на `Expr`

Ще разширим `Expr` типа с няколко допълнителни конструктора:

```haskell
data Expr
  = Var String
  | Val Integer
  | Oper OperType Expr Expr
  | If Expr Expr Expr
  | SumList [Expr]
  | Sum String Expr Expr
```
`Val` и `If` запазват семантиката си. Новите конструктори имат следната:
* `Var str` е **променлива** - идеята тук е, че имаме име (т.е. низ), което ще **свържем**
    със стойност, чрез някакво външно средство. При интерпретация, това ще се случва
    когато при изпълнение даваме `Integer`-и за стойности за всяка променлива, за която
    е нужно.
* `Oper` - конструктор, който абстрахира над `Plus` и `Mult`, за да минимизира дупликацията    на код - можем да видим че в `Plus` и `Mult` случаите на оригиналния `eval`,
    имплементацията е почти еднаква. Типът `OperType` ще дефинирате вие като задача.

    **Бележка**: естествено, можехме вместо `OperType` да приемаме направо двуместна функция `Integer -> Integer -> Integer`
    като поле, но има различни плюсове и минуси спрямо подхода с тип данни.

    Плюсове са, че е по-малко код и поддържа веднага **каквито и да е** двоични операции.
    Минуси са, че вече губим възможността да принтираме `Expr`-та, защото как се принтират функции?
    Губим и възможността да сравняваме `Expr`-та за равенство - равенството на функции е неразрешимо в общия случай.
    Тъй като ни се иска да имаме тестове сравняващи и принтиращи `Expr`-та, избираме `OperType` подхода тук.

    Тези два стила често се наричат  "initial encoding" (с `OperType`) и "final (tagless) encoding" (с функцията).
* `SumList` - събираме резултата от пресмятането на списък от изрази
* `Sum i limex inside` - ще означава

    ![sum-img](img/sum.png)

    т.е. оценяваме `inside` многократно, като всеки път `i` свързваме с нова стойност,
    варираща от 0 до оценката на `limex`, и сумираме резултатите на всичките оценки на
    `inside`

    Напомням, че резултатът от сумирането на празен интервал,
    (т.е. ако оценката на `limex` се окаже по-малка от 0), е 0.

## `Context`
**Контекст** наричаме списък от наредени двойки от низ и цяло число. Или с други думи:
```haskell
type Context = [(String, Integer)]
```

Елемент на контекста ще наричаме **свързване**, защото свързва име на променлива със стойност.

"Разширяване на контекст" е добавяне на ново свързване към контекста, т.е.:

```haskell
extend :: String -> Integer -> Context -> Context
extend x n = ((x, n):)
```

Това разширяване естествено може да се случва където и да е, но се оказва удобно
то да е в началото му, за да може "по-нови" ("по-скорошни") свързвания, да са по-отпред
и съответно да намираме първо тях, когато има повече от едно свързване на дадена променлива
със стойност.

## 27т. Задачи

## 13т. Интерпретация

Първо - няколко помощни функции.

### 1т. `lookup :: String -> Context -> Maybe Integer`

Имплементирайте търсене на стойност по даден ключ.

Тук специализираме ключовете до низове, и тъй като търсим в контекст,
ще връщаме цяло число.

Естествено, това име може да го няма в контекста, затова и връщаме `Maybe Integer`.

Примери:
```haskell
> lookup "x" [("x", 5), ("y", 6)]
Just 5
> lookup "z" [("x", 5), ("y", 6)]
Nothing
> lookup "x" [("x", 5), ("y", 6), ("x", 69)]
Just 5
```

### 1т. `maybeAndThen :: Maybe a -> (a -> Maybe b) -> Maybe b`

Функция, която абстрахира "работата в контекст на `Maybe`".

Често се налага, когато имаме `Maybe` стойност, да "пропагираме" `Nothing` резултатите,
а пък при `Just`, да вземем някакво решение базирано на стойността вътре.

В общия случай операцията, която ще прилагаме над стойността в `Just`-а, също може да се провали,
заради което и функцията, която се подава, връща `Maybe` също.

Тази функция ще е удобна за `eval` например, където ще се наложи работа с повече `Maybe`
стойности.

Примери:

```haskell
> maybeAndThen (lookup "x" [("x", 5)]) (\xVal -> Just $ xVal * xVal)
Just 25
> maybeAndThen (lookup "x" [("y", 5)]) (\xVal -> Just $ xVal * xVal)
Nothing
> maybeAndThen (lookup "x" [("x", 5)]) (\xVal -> if even xVal then Just xVal else Nothing)
Nothing
> maybeAndThen (lookup "x" [("x", 6)]) (\xVal -> if even xVal then Just xVal else Nothing)
Just 6
> lookup "x" [("x", 5)] `maybeAndThen` \xVal -> Just $ xVal - 1
Just 4
```

Често е удобно тази функция да се използва инфиксно:
```haskell
ctxt = [("x", 6), ("y", 9)]

-- get the values of two variables from the context, failing if *any* of them are missing
-- examples:
-- > getTwo "x" "y" ctxt
-- Just (6,9)
-- > getTwo "y" "x" ctxt
-- Just (9,6)
-- > getTwo "x" "z" ctxt
-- Nothing
-- > getTwo "z" "y" ctxt
-- Nothing
getTwo :: String -> String -> Context -> Maybe (Integer, Integer)
getTwo x y ctxt =
  lookup x ctxt `maybeAndThen` \xVal ->
  lookup y ctxt `maybeAndThen` \yVal ->
  Just (xVal, yVal)
-- with some more explicit parens:
-- lookup x ctxt `maybeAndThen` (\xVal ->
-- lookup y ctxt `maybeAndThen` (\yVal ->
-- Just (xVal, yVal)))
--
-- with some more explicit indentation:
-- lookup x ctxt `maybeAndThen` \xVal ->
--   lookup y ctxt `maybeAndThen` \yVal ->
--     Just (xVal, yVal)
--
-- without the infix syntax:
-- note that in general instead of f x y z we can write
-- f
--   x
--   y
--   z
-- so with this knowledge, we could've instead written getTwo like so:
-- maybeAndThen
--   (lookup x ctxt)
--   (\xVal ->
--     maybeAndThen
--       (lookup y ctxt)
--       (\yVal -> Just (xVal, yVal)))
-- but the "infix hanging lambda" version is nicer for indentation,
-- and it's also very close to something else we'll encounter later
```

### 2т. `traverseListMaybe :: (a -> Maybe b) -> [a] -> Maybe [b]`

Функция, която вече сме виждали на упражнение:

Имаме списък от `a`-та, както и функция, която връща `b`, но допускайки и провал `a -> Maybe b`.

Искаме да пробваме да изпълним функцията за всеки елемент на списъка,
успявайки (т.е. връщайки `Just`), само когато функцията успява върху всеки от елементите
на списъка (върнала е `Just` за всеки от тях).

За да можем да използваме и че наистина е успяла, връщаме резултатите от всичките извиквания.

Примери:
```haskell
> traverseListMaybe (\x -> if even x then Just (x * x) else Nothing) [2,4,6]
Just [4,16,36]
> traverseListMaybe (\x -> if even x then Just x else Nothing) [1,2,4]
Nothing
> traverseListMaybe (\x -> if even x then Just x else Nothing) [4,2,1]
Nothing
```

### Обяснение за свободни и свързани променливи

Интуитивно, свободни променливи са такива, които наистина са "неизвестни".

Свързаните променливи зависят от нещо - най-често "квантор".

Това, което сте срещали
вие досега като примери, са "за всяко", "съществува", "интеграл", "производна" -
във всички тези случаи, имаме променлива обвързана с квантора.

В нашия случай кванторът, който имаме, е `Sum` - в него се изисква име на променлива,
която да считаме за свързана във вътрешния израз на сумата.

Примери:
```haskell
Sum "i" (Val 5) (Var "x")
```
Тук `"x"` е свободна променлива.
```haskell
Sum "i" (Val 5) (Var "i")
```
Тук `"i"` е свързана променлива - тя даже не е истинска променлива, защото
горната граница на сумата е константа и можем да я премахнем, като
разпишем сумата до `0 + 1 + 2 + 3 + 4 + 5`.
```haskell
Sum "i" (Var "i") (Var "x")
```
Тук `"x"` е свободна променлива.
Тук `"i"`-то, което имаме в горната граница на сумата, **не е** свързана променлива - тя е свободна.

Не би било разумно да допускаме свързването в `Sum` да влияе на горната му граница,
защото тогава бихме имали много суми, които се държат странно (например са константи винаги,
 зациклят безкрайно, винаги имат само един член на сумата).
```haskell
Sum "i" (Var "i") (Var "i")
```
Тук `"i"` е **и свързана, и свободна** променлива.

Тя е свободна, тъй като участва в горната грнаица на сумата, но е и свързана, защото участва във вътрешния израз на сумата.


### 2т. `freeVars :: Expr -> [String]`

Намерете всичките свободни променливи на даден израз.

Примери
```haskell
> freeVars (If (Var "x") (Val 5) (Var "y"))
["x", "y"]
> freeVars (Sum "i" (Var "x") (If (Var "i") (Var "y") (Var "z")))
["x", "y", "z"]
> freeVars (Sum "i" (Var "i") (If (Var "i") (Var "y") (Var "z")))
["i", "y", "z"]
```

### 1т. `data OperType`

Напишете `OperType` типа, така че да поддържа поне събиране и умножение.

В следващата задача за `eval` ще имплементираме и интерпретацията му като "двуместна функция".

### 6т. `eval :: Context -> Expr -> Maybe Integer`

Тъй като вече имаме променливи, ще ни трябва някакъв начин да им даваме стойности.

Това постигаме, като приемаме контекст, в който ще интерпретираме израз:
```haskell
eval :: Expr -> Integer
-- ->
eval :: Context -> Expr -> Integer
```

Ще установим, че добавеният контекст е ключов за `Sum` случая, независимо от това
дали в началото сме имали каквито и да е свободни променливи в израза.

Тъй като нямаме гаранция, че всичките променливи в израз ще ги има в подадения контекст, трябва да позволим на `eval` да се проваля:
```haskell
eval :: Context -> Expr -> Integer
-- ->
eval :: Context -> Expr -> Maybe Integer
```

Имплементирайте новия `eval`.

Примери:
```haskell
> eval [] $ Var "x"
> Nothing
> eval [("x", 5), ("x", 6)] $ Var "x"
> Just 5

> eval [] (If (Val 0) (Val 6) (Val 9))
Just 6
> eval [] (If (Val 42) (Val 6) (Val 9))
Just 9
> eval [] (If (Val 0) (Val 6) (Var "x"))
Just 6
> eval [] (If (Val 42) (Var "x") (Val 9))
Just 9
> eval [] (If (Var "x") (Val 6) (Val 9))
Nothing
> eval [("x", 0)] (If (Var "x") (Val 6) (Val 9))
Just 6

> eval [] (SumList [Val 5, Val 6, Val 7])
Just 18
> eval [] (SumList [Val 5, Val 6, Var "x"])
Nothing
> eval [("x", 5)] (SumList [Val 5, Val 6, Var "x"])
Just 16

> eval [] (Sum "i" (Val 10) (Val 1))
Just 11
> eval [] (Sum "i" (Val 10) (Var "i"))
Just 55
> eval [] (Sum "i" (Val 10) (Var "x"))
Nothing
> eval [("x", 2)] (Sum "i" (Val 10) (Var "x"))
Just 22
> eval [] (Sum "i" (Val (-1)) (Var "y"))
Just 0
> eval [("i", 69)] (Sum "i" (Val 10) (Var "i"))
Just 55
> eval [] (Sum "i" (Var "i") (Var "i"))
Nothing
> eval [("i", 10)] (Sum "i" (Var "i") (Var "i"))
Just 55
> eval [] (Sum "i" (Val 2) (Sum "i" (Val 10) (Var "i")))
Just 165
```

## 9т. Компилация
Второ - още няколко помощни функции.

### 1т. `intersperse :: a -> [a] -> [a]`

Вмъкнете стойност между всеки два елемента на подадения списък:

Примери:
```haskell
> intersperse 42 [1,2,3]
[1,42,2,42,3]
> intersperse 42 [1]
[1]
> intersperse 42 []
[]
```

### 0.5т. `unwords :: [String] -> String`

Слива списък от низове, слагайки празни места между тях.

Казва се `unwords`, защото има `words`, което от низ прави списък от низове, разбити по празните места.

Примери:
```haskell
> unwords []
""
> unwords ["Pirin"]
"Pirin"
> unwords ["Pirin", "ate", "the", "pizza!"]
"Pirin ate the pizza!"
```

### 0.5т. `unlines :: [String] -> String`

Слива списък от думи, слагайки нови редове между тях.

Казва се `unlines`, защото има `lines`, което от низ прави списък от низове, разбити по нови редове.

Примери:
```haskell
> unlines []
""
> unlines ["Pirin"]
"Pirin"
> unlines ["Pirin", "ate", "the", "pizza!"]
"Pirin\nate\nthe\npizza!"
```

### Типове

Ще напишем компилатор от `Expr` типа към езика racket, с който се надявам да сте запознати.

По принцип можем да направим
```haskell
compileToRacket :: Expr -> String
```
и директно да генерираме низове за racket програми.

Оказва се по-удобно, да имаме междинно `RacketExpr`
```haskell
data RacketExpr
  = Name String
  | List [RacketExpr]
compileToRacket :: Expr -> RacketExpr
printRacketExpr :: RacketExpr -> String
```

което е минимална версия на синтактичното дърво на racket-ски(`lisp`-ски) програми
(т.нар. [s-expressions](https://en.wikipedia.org/wiki/S-expression)).

Идеята тук е да отделим разделим компилацията на "превеждане до racket израз"
и "принтиране на racket израз".

Това е полезно, защото иначе трябва постоянно
да мислим как да слепваме низове, както и да пишем много пъти неща като например
"сложи скоби тук" или "сложи празни места между тези низове".

Ето примери за как бихме изразили някои изрази от racket чрез този тип данни:
```haskell
5 -> Name "5"
x -> Name "x"
+ -> Name "+"
(+ 1 2 3) -> List [Name "+", Name "1", Name "2", Name "3"]
(define (f x) (+ 5 x)) ->
  List
    [ Name "define"
    , List [Name "f", Name "x"]
    , List [Name "+", Name "5", Name "x"]
    ]
```

Допълнително защото
* изглежда по-яко да можем да имаме повече от един израз едновременно
* ще ни се наложи да слагаме някакви "служебни" неща като `#lang racket` или
    помощни дефиниции на които да разчита компилацията

сме направили и
```haskell
newtype RacketProgram = MkRacketProgram [RacketExpr]
```
което да съдържа много изрази.

#### Бележка за `newtype`

По принцип, когато имаме `data` тип, стойностите му са имплементирани така, че винаги имат "тагче" в тях,
по което да може програмата, докато работи, да разбере точно кой конструктор е ползван.

Например за
```haskell
data Animal = Dog String | Cat Int
```
за дадена стойност от тип `Animal`, освен `String` или `Int` (зависимост от конструктора),
в паметта, която заема, има и допълнителна информация (реално е число), което индикира дали стойността е `Dog` или `Cat`.

Допълнително, поради ленивостта на езика (както ще разгледаме на упражнения), `String`/`Int`-ът вътре
всъщност е указател към `String`/`Int`.

`newtype` е същото като `data`, в това че създава нов тип данни за типовата система.
Целта, с която е измислен `newtype`, е новият тип да има **същото представяне в паметта** като това, което съдържа.
Това се реализира чрез следното ограничение:
* `newtype` типовете винаги имат точно един конструктор с точно едно поле

Поради това, не се нуждаем вече да имаме таг, защото няма алтернативи, които да трябва да различаваме.
Допълнително, той не държи указател към стойността в конструктора, а е направо самата стойност.

`newtype` ползваме, когато не искаме да хабим повече ресурси,
но искаме да различим в типовата система (съответно по време на компилация) между две неща, които иначе са еднакви.
Това е удобно, когато имаме много аргументи от един и същ тип, за да дадем повече яснота кой какъв е:
```haskell
-- easy to forget which Int is for what
mkArray :: Int -> Int -> Array
-- ->
newtype ArrayStart = MkArrayStart Int
newtype ArraySize = MkArraySize Int

mkArray :: ArrayStart -> ArraySize -> Array
```
Допълнително, можем да скрием конструкторите на `Array{Start,Size}` от външния свят (т.е. да не излизат извън сегашния модул)
и по този начин да гарантираме, че ще са валидни, като ги използваме по-нататък:
```haskell
-- arrays can't start from negative addresses
mkArrayStart :: Int -> Maybe ArrayStart
mkArrayStart x = if x < 0 then Nothing else Just x
-- arrays can't have a negative size
mkArraySize :: Int -> Maybe ArraySize
mkArraySize x = if x < 0 then Nothing else Just x
```


#### Бележка за тестовете
След като имаме интерпретатор за `Expr` под формата на `eval`, можем да тестваме
коректността на компилатора ни.

Това постигаме, като за произволни `Expr`-та сравняваме дали при изпълнение с racket
на компилирания ни код получаваме същия резултат, какъвто получаваме при изпълнение
му с `eval`. Тестовете за компилатора правят именно това.

За целите на тестовете се разчита, че имате в `PATH` програмата racket.
За да проверите това, можете да отворите терминал и да видите в него пуска ли се racket.

Има булева константа `solvingCompiler`, която в момента е `False`.

Когато искате да се пускат тестовете за компилатора, трябва да я направите `True`.

Целта на това е да се намали "шумът", който получавате от тестовете, преди да сте започнали
да имплементирате компилатор частта.

### 1т. `printRacketExpr :: RacketExpr -> String`

Имплементирайте принтенето на единичен racket израз.

* `Name x` искаме да се преврежда до `x`.
* `List [x0, x1, ... xn]` искаме да се превежда до `(x0 x1 ... xn)`.

### 2т. `printRacketProgram :: Context -> RacketProgram -> String`

Имплементирайте принтенето на racket програма.

Направете всеки отделен израз да се принти на нов ред.

Не забравяйте, че racket програмите имат `#lang racket` най-отгоре.

Тук можете да добавяте всякакви допълнителни декларации.

Например, ако искате да използвате някаква функция в `compileToRacket`
за имплементацията на превеждането на `Expr` конструкция,
тук е хубаво място да я сложите.

Тъй като някои `Expr`-та могат да имат свободни променливи, например `Var "x"`,
при превеждането от `Expr` към `RacketExpr` ще получим изрази, които имат недефинирани имена.

За да заобиколим това, имаме няколко опции. Една би била, ако при превеждане бихме имали
недефинирани имена (т.е. има свободни променливи), вместо израз, да генерираме декларация на функция,
взимаща тези свободни променливи като аргументи.

Тук съм избрал при `printRacketProgram` да се приема отново контекст, като идеята е
за всяко свързване от контекста да генерираме `define` декларация, даваща му стойност.

Например за `[("x", 6), ("y", 9)]` бихме получили
```scheme
(define x 6)
(define y 9)
```
и така бихме могли да компилираме и изпълним `Var "x"`

### 4т. `compileToRacket :: Expr -> RacketExpr`

Имплементирайте главната функция, грижеща се за компилация.

Отново, искаме резултатните racket изрази да се оценяват до същото, до което биха се,
ако ги бяхме изпълнили с `eval`.

За удобство, с `writeFile <файл> <низ>` можете да запишете
`<низ>` във `<файл>`, като ползвате за `<низ>` резултат на `printRacketProgram`.

#### `Sum`
Не е нужно функцията да поддържа `Sum` конструктора, това ще е бонус задача.

Можете да оставите за дефиниция `error "not supported"`.

## 5т. Частично оценяване

След като имаме компилатор, защо да не направим и оптимизационна стъпка за него.

"Поддървета" или "подизрази" на даден `e :: Expr` ще наричам всички `Expr`, които се съдържат в `e`, рекурсивно.

Например:
* `Oper op (Val 5) (Var "x")` има за подизрази
    * `Val 5`
    * `Var "x"`
* `If (Val 5) (Val 6) (Var "x")` има за подизрази
    * `Val 5`
    * `Val 6`
    * `Var "x"`
* `Oper op (If (Val 5) (Val 6) (Var "x")) (Var "y")` има за подизрази
    * ! `If (Val 5) (Val 6) (Var "x")` - забележете че подизразите включват не само листа на синтактичното дърво
    * `Val 5`
    * `Val 6`
    * `Var "x"`
    * `Var "y"`

Това, което ще имплементираме, традиционно се нарича "partial evaluation" и/или "constant folding".

Целта на оптимизацията е да "опростим" всички поддървета, за които се знае, без да се гледат
стойностите на промелниви, как биха се оценили.
С други думи това са неща, чиято стойност се знае, преди да се изпълни програмата.

---

Да разгледаме няколко израза, за да демонстрираме идеята с примери:

```haskell
If (Val 0) (Val 3) (Val 9)
```
Този израз е направен изцяло от константи. Знаем още преди да се опитаме да го изпълним,
че той ще се оцени до `3`, защото знаем, че условието на `if`-а е константата 0,
според което трябва да изберем първия му клон.

Тогава, за да спестим сметки на racket след това, можем още преди да го компилираме,
да го заместим с `Val 3`.

---

```haskell
SumList [SumList [Val 1, Val 2], Val 3, SumList [Val 4, Val 5]]
```

Подобно на горния пример, тук всичките ни изрази са константи и съотвенто можем да сведем
този израз до `Val 15`.

---

```haskell
If (Val 0) (Var "x") (Var "y")
```
Въпреки че оценката на този израз винаги зависи от променливи, ние все пак можем
да опростим израза, защото знаем до какво се оценява условието на `If`-а, заменяйки го с `Var  "x"`.

---

```haskell
SumList [SumList [Val 1, Val 2], Var "x", SumList [Val 4, Val 5]]
```

За разлика от миналия пример, тук имаме променлива в най-външния `SumList` и съответно
няма как да се отървем от него. Въпреки това, виждаме че има други константи,
които можем да съберем в една предварително, получавайки нещо от вида на

```haskell
SumList [Val 12, Var "x"]
```

---

```haskell
If
  (Var "x")
  (SumList [Val 3, Val 6])
  (Val 9)
```
Тук въпреки че условието на `if`-а е неизвестно и няма как изцяло да махнем `if`-а,
все още има подизраза `SumList [Val 3, Val 6]`, който можем да заменим с
`Val 9`, тъй като знаем всичките неща, които ще събираме, получавайки
```haskell
If
  (Var "x")
  (Val 9)
  (Val 9)
```

Тук изскача допълнителна идея:
Двата случая на `if`-а са еднакви - защо да не го махнем, заменяйки го с `Val 9`?

Това обаче **не запазва семантиката на програмата** в общия случай - няма как да знаем,
че условието на `If` е програма която ще **успее**.

По-конкретно, тук може да нямаме `"x"` в контекста, когато се опитваме да оценим израза, съответно това би трябвало да се оцени до `Nothing`.

За нашия език това реално е възможно само когато нямаме променлива в контекста, но
ако имахме странични ефекти, би било възможно условието да се провали по други
начини (например опитваме се да четем несъществуващ файл).

### 5т. `partialEval :: Expr -> Expr`

Имплементирайте функцията `partialEval`, която да прави описаната
по-горе оптимизация.

#### `Sum`
Не е нужно функцията да поддържа `Sum` конструктора, това ще е бонус задача.

Можете да оставите за дефиниция `error "not supported"`.

#### Тестове
За да пускате тестовете за `partialEval`, променете
`solvingPartialEval` на `True`.

Примери:
```haskell
> partialEval (If (Val 42) (Val 6) (Val 9))
Val 9

> partialEval (If (Val 42) (Var "y") (Var "x"))
Var "x"

> partialEval (If (Val 0) (Val 0) (Var "x"))
Val 0

> partialEval (If (Var "x") (Val 0) (Val 69))
If (Var "x") (Val 0) (Val 69)

-- следващите два примера са форматирани на няколко реда за четимост
-- но това означава, че няма да можете да ги пейстнете директно в ghci
> partialEval
    (If
      (If (Val 0) (Var "x") (Val 69))
      (Val 0)
      (Val 69))
If (Var "x") (Val 0) (Val 69)

> partialEval
    (If
      (Var "x")
      (If (Val 0) (Var "x") (Val 69))
      (If (Val 1) (Var "x") (Val 69)))
If (Var "x") (Var "x") (Val 69)
```

## 10т. `Sum` бонуси

### Тестове

За да пускате тестовете за `Sum`, сменете `solvingSum` на `True`.

### 3т. Компилация

Разширете `compileToRacket` да поддържа и `Sum`.

Тук най-вероятно ще ви се наложи да си направите допълнителна racket-ска функция,
която да използвате в оценяването на `Sum`.

### 7т. `partialEval`

Разширете `partialEval` да поддържа и `Sum`.

Да разгледаме още няколко примера за "частично оценяване", за да видим какво
ще ни е нужно.
```haskell
Sum "x" (Val 5) (Var "x")
```
Тук знаем откъде докъде ще варира `"x"`. Освен това знаем и че изразът, който сумираме,
е константа **спрямо `"x"`** - няма други неизвестни променливи в него.

Заради това, можем да го заменим с `Val 15`

```haskell
Sum "x" (Val 2) (If (Var "x") (Var "y") (Var "x"))
```
Тук знаем откъде докъде ще варира `"x"`. Въпреки това, вътрешният израз **не е константа "спрямо `"x"`"** - в случая в който `"x"` се оценява до 0, имаме все още променлива.

Заради това, няма как да заместим целия израз с `Val`, но пък можем вместо това да го заменим с дърво от `Oper` събирания, или алтернативно със
```haskell
SumList [Var "y", Val 1, Val 2]
```
изразявайки сумата като списък от събираеми, всяко от които няма вече `"x"` в себе си.

За да поддържаме примерите отгоре ще ни е нужно някакси да следим в момента дали
някои променливи са свързани със стойности, когато правим частичното ни оценяване.

Това е именно контекст - ще трябва да модифицираме досегашната ни `partialEval` функция
да поддържа контекст, като в началото винаги почваме с празен контекст и той се разширява,
всеки път като срещнем `Sum`.

Препоръчвам да си изнесете реалната дефиниция на `partialEval` в локална "хелпър" функция
приемаща и контекст, която да викате с празен контекст:
```haskell
partialEval = go []
  where
    go :: Context -> Expr -> Expr
    go = ...
```

Примери:
```haskell
> partialEval (Sum "i" (If (Val 0) (Var "x") (Val 69)) (Var "x"))
Sum "i" (Var "x") (Var "x")

> partialEval (Sum "i" (Var "x") (If (Val 0) (Var "x") (Val 69)))
Sum "i" (Var "x") (Var "x")

> partialEval (Sum "i" (Val 10) (Val 1))
Val 11

> partialEval (Sum "i" (Val 10) (Var "i"))
Val 55

-- нека имаме
-- isSum :: Expr -> Bool
-- isSum (Sum _ _ _) = True
-- isSum _ = False
> isSum $ partialEval (Sum "i" (Val 10) (Var "x"))
False
```
