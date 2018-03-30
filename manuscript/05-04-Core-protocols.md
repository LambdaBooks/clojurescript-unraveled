### Протоколи стандартної бібліотеки

Велика перевага функцій ClojureScript полягає у тому, що вони реалізовані на базі протоколів. Це дозволяє функціям працювати з будь-яким типом, який було розширено такими протоколами. Це стосується як власних типів, так і сторонніх.

#### Функції

Як ми дізналися з попередніх розділів, у ClojureScript викликати можна не лише функції. Вектори — це функції від власних індексів, мапи — функції від власних ключів, а множини — функції від значень.

Ми можемо розширювати типи таким чином, що результат можна викликати як функції, що реалізують протокол `IFn`. Колекція, що не може бути викликана як функція, є чергою. Реалізуємо протокол `IFn` для типа `PersistentQueue` так, щоб мати змогу викликати чергу як функції від індексів:

```clojure
(extend-type PersistentQueue
  IFn
  (-invoke
    ([this idx]
      (nth this idx))))

(def q #queue[:a :b :c])
;; => #queue [:a :b :c]

(q 0)
;; => :a

(q 1)
;; => :b

(q 2)
;; => :c
```


#### Виведення на друк

Щоб познайомитися з деякими протоколами зі стандартної бібліотеки, визначимо тип `Pair`, який містить пару значень.

```clojure
(deftype Pair [fst snd])
```

Для виведення типів на друк у бажаному вигляді ми можемо реалізувати протокол `IPrintWithWriter`. Цей протокол визначає функцію під назвою `-pr-writer`, якій передається значення для друку, обʼєкт запису та опції. Ця функція використовує обʼєкт запису `-write`для запису бажаного рядкового представлення типу `Pair`:


```clojure
(extend-type Pair
  IPrintWithWriter
  (-pr-writer [p writer _]
    (-write writer (str "#<Pair " (.-fst p) "," (.-snd p) ">"))))
```


#### Послідовності

З попереднього розділу ми дізналися про послідовності — одну з основних абстракцій мови ClojureScript. Згадайте функції `first` та `rest` для роботи з послідовностями. Вони визначені у протоколі `ISeq`, тому ми можемо розширити типи та додати підтримку таких функцій:

```clojure
(extend-type Pair
  ISeq
  (-first [p]
    (.-fst p))

  (-rest [p]
    (list (.-snd p))))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(first p)
;; => 1

(rest p)
;; => (2)
```

Інша корисна функція для роботи з послідовностями — це `next`. Хоча `next` працює з будь-яким аргументом, що є послідовністю, ми можемо реалізувати це явно за допомогою протоколу `INext`:

```clojure
(def p (Pair. 1 2))

(next p)
;; => (2)

(extend-type Pair
  INext
  (-next [p]
    (println "Our next")
    (list (.-snd p))))

(next p)
;; Our next
;; => (2)
```

Врешті решт ми можемо створии власні типи, що реалізують протокол `ISeqable`. Це означає, що ми можемо передавати їх функції `seq` та отримувати послідовність.

ISeqable
```clojure
(def p (Pair. 1 2))

(extend-type Pair
  ISeqable
  (-seq [p]
    (list (.-fst p) (.-snd p))))

(seq p)
;; => (1 2)
```

Тепер тип `Pair` може працювати з великою кількістю функцій ClojureScript для обробки послідовностей:

```clojure
(def p (Pair. 1 2))
;; => #<Pair 1,2>

(map inc p)
;; => (2 3)

(filter odd? p)
;; => (1)

(reduce + p)
;; => 3
```


#### Колекції

Функції для роботи з колекціями також визначені на основі протоколів. Для прикладу у цьому розділі ми змусимо нативні рядки з JavaScript працювати як колекції.

Найважливіша функція для роботи з колекціями — це `conj`, що її визначено у протоколі `ICollection`. Рядки — це єдиний тип, до яких має сенс застосовувати операцію `conj`, тому операція `conj` для рядків буде просто конкатенацією:

```clojure
(extend-type string
  ICollection
  (-conj [this o]
    (str this o)))

(conj "foo" "bar")
;; => "foobar"

(conj "foo" "bar" "baz")
;; => "foobarbaz"
```

Інша зручна функція для роботи з колекціями — `empty`, що є частиною протоколу `IEmptyableCollection`. Реалізуємо її для рядків:

```clojure
(extend-type string
  IEmptyableCollection
  (-empty [_]
    ""))

(empty "foo")
;; => ""
```

Ми використовували спеціальний символ `string` для розширення нативних рядків з JavaScript. За більш детальною інформацією рекомендуємо звернутися до розділу про розширені типи JavaScript.


##### Особливості колекцій

Певні риси притаманні не усім колекціям, зокрема зліченність за постійний час, здатність до обернення. Такі риси розподіляються між різними протоколами, бо не усі ці риси підходять до кожної колекції. Для демонстрації відповідних протоколів скористаємося типом `Pair`, який ми визначили раніше.

Для колекції, обсяг яких може бути злічений за постійний час за допомогою функції `count`, ми можемо визначити протокол `ICounted`. Реалізувати такий протокол для типу `Pair` нескладно:

```clojure
(extend-type Pair
  ICounted
  (-count [_]
    2))

(def p (Pair. 1 2))

(count p)
;; => 2
```

Певні типи колекцій (вектори, переліки) можуть бути проіндексовані за порядковим номером за допомогою функції `nth`. До проіндексованих типів можна застосувати протокол `IIndexed`:

```clojure
(extend-type Pair
  IIndexed
  (-nth
    ([p idx]
      (case idx
        0 (.-fst p)
        1 (.-snd p)
        (throw (js/Error. "Index out of bounds"))))
    ([p idx default]
      (case idx
        0 (.-fst p)
        1 (.-snd p)
        default))))

(nth p 0)
;; => 1

(nth p 1)
;; => 2

(nth p 2)
;; Error: Index out of bounds

(nth p 2 :default)
;; => :default
```

#### Асоціативні структури

Існує багато структур даних, що відображають ключі на значення. Такі структури називаються асоціативними.Ми вже познайомилися з великою кількістю таких структур та функцій, що працюють з ними, зокрема `get`, `assoc` та `dissoc`. Розглянемо протоколи, на яких грунтуються ці функції.

Перш за все, нам потрібен спосіб отримувати значення за ключем з асоціативних струтур даних. Протокол `ILookup` визначає функцію для цього. Додамо можливість отримувати значення за ключем до типу `Pair`, адже це асоціативна структура, що відображає індекси 0 та 1 на значення.

```clojure
(extend-type Pair
  ILookup
  (-lookup
    ([p k]
      (-lookup p k nil))
    ([p k default]
      (case k
        0 (.-fst p)
        1 (.-snd p)
        default))))

(get p 0)
;; => 1

(get p 1)
;; => 2

(get p :foo)
;; => nil

(get p 2 :default)
;; => :default
```

Для застосування функції `assoc` до струтури даних, така структура має реалізовувати протокол `IAssociative`. Для типу `Pair` дозволяється лише два значення ключів — 0 та 1. `IAssociative` також має функцію для запиту інформації щодо наявності певного ключа. 

```clojure
(extend-type Pair
  IAssociative
  (-contains-key? [_ k]
    (contains? #{0 1} k))

  (-assoc [p k v]
    (case k
      0 (Pair. v (.-snd p))
      1 (Pair. (.-fst p) v)
      (throw (js/Error. "Can only assoc to 0 and 1 keys")))))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(assoc p 0 2)
;; => #<Pair 2,2>

(assoc p 1 1)
;; => #<Pair 1,1>

(assoc p 0 0 1 1)
;; => #<Pair 0,1>

(assoc p 2 3)
;; Error: Can only assoc to 0 and 1 keys
```

Функція, що є доповненням до `assoc`, це `dissoc`. `dissoc` є частиною протоколу `IMap`. Для нашого типу `Pair` ця функція не дуже корисна, але ми її реалізуємо. `dissoc` при застосуванні до 0 чи 1 встановлює значення `nil` у таку позицію, а недійсні ключі буде проігноровано.

```clojure
(extend-type Pair
  IMap
  (-dissoc [p k]
    (case k
      0 (Pair. nil (.-snd p))
      1 (Pair. (.-fst p) nil)
      p)))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(dissoc p 0)
;; => #<Pair ,2>

(dissoc p 1)
;; => #<Pair 1,>

(dissoc p 2)
;; => #<Pair 1,2>

(dissoc p 0 1)
;; => #<Pair ,>
```

Асоціативні струтури даних складаються з ключів та значень, які попарно називаються записами. Функції `key` та `val` дозволяю робити запити за ключем або значенням такого запису і грунтуються на протоколі `IMapEntry`. Розглянемо кілька прикладів функцій `key` та `val`, та подивимося, як записи можна використовувати для побудови відображень:

```clojure
(key [:foo :bar])
;; => :foo

(val [:foo :bar])
;; => :bar

(into {} [[:foo :bar] [:baz :xyz]])
;; => {:foo :bar, :baz :xyz}
```

Пари також можуть бути відображеннями. Ми розглядаємо перші елементи як ключі, а другі — як значення:

```clojure
(extend-type Pair
  IMapEntry
  (-key [p]
    (.-fst p))

  (-val [p]
    (.-snd p)))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(key p)
;; => 1

(val p)
;; => 2

(into {} [p])
;; => {1 2}
```

#### Порівняння

Для перевірки еквівалентності значень через `=` слід реалізувати протокол `IEquiv`. Зробимо це для типу `Pair`:

```clojure
(def p  (Pair. 1 2))
(def p' (Pair. 1 2))
(def p'' (Pair. 1 2))

(= p p')
;; => false

(= p p' p'')
;; => false

(extend-type Pair
  IEquiv
  (-equiv [p other]
    (and (instance? Pair other)
         (= (.-fst p) (.-fst other))
         (= (.-snd p) (.-snd other)))))

(= p p')
;; => true

(= p p' p'')
;; => true
```

Типи також можна порівнювати. Функція `compare` отримує два значення та повертає відʼємне число, якщо перше значення менше за друге, 0 якщо значення рівні, та 1 якщо перше більше за друге. Для порівняння типів необхідно реалізувати протокол `IComparable`.

Порівняння пар означатиме перевірку, чи два перші значення рівні. Якщо це правда, результатом буде порівняння других значень. Інакше кінцевим результатом буде результат порівняння перших значень:

```clojure
(extend-type Pair
  IComparable
  (-compare [p other]
    (let [fc (compare (.-fst p) (.-fst other))]
      (if (zero? fc)
        (compare (.-snd p) (.-snd other))
        fc))))

(compare (Pair. 0 1) (Pair. 0 1))
;; => 0

(compare (Pair. 0 1) (Pair. 0 2))
;; => -1

(compare (Pair. 1 1) (Pair. 0 2))
;; => 1

(sort [(Pair. 1 1) (Pair. 0 2) (Pair. 0 1)])
;; => (#<Pair 0,1> #<Pair 0,2> #<Pair 1,1>)
```


#### Метадані

Функції `meta` та `with-meta` також грунтуються на протоколах, а саме `IMeta` та `IWithMeta`. Для того, щоб наші типи також підтримували можливість додання метаданих, слід реалізувати додання додаткового поля для запису метаданих та реалізацію обох протоколів.

Реалізуємо версію типу `Pair` із метаданими:

```clojure
(deftype Pair [fst snd meta]
  IMeta
  (-meta [p] meta)

  IWithMeta
  (-with-meta [p new-meta]
    (Pair. fst snd new-meta)))


(def p (Pair. 1 2 {:foo :bar}))
;; => #<Pair 1,2>

(meta p)
;; => {:foo :bar}

(def p' (with-meta p {:bar :baz}))
;; => #<Pair 1,2>

(meta p')
;; => {:bar :baz}
```


#### Взаємодія з JavaScript

ClojureScript  є гостьовою мовою у віртуальній машині JavaScript, тому часто виникає необхідність конвертації структур даних ClojureScript у відповідні структури JavaScript та навпаки. Також може зʼявитися необхідність участі типів JavaScript в абстраціях, представлених як протоколи.

##### Розширення типів JavaScript

Коли необхідно розширити обʼєкти JavaScript, слід використовувати спеціальни символи замість глобальних обʼєктів `js/String`, `js/Date`тощо. Таке обмеження захищає глобальні обʼєкти від небажаних мутацій.

Символи для розширення типів JavaScript — `object`, `array`, `number`, `string`, `function`, `boolean` та `nil`. Останній використовується для обʼєкта null. Розміщення протоколу на обʼєкти використовує функцію `goog.typeOf` бібліотеки Google Closure. Є Спеціальний символ `default` для стандартних реалізацій протоколу для будь-якого типу.

Продемонструємо розширення типів JavaScript: визначимо протокол `MaybeMutable`, що має єдину функцію — предикат `mutable?`. Змінюваність — це стандартна поведінка у JavaScript, тому розширимо стандартний тип JavaScript, що повертає `true` в результаті виклику функції `mutable?`:

```clojure
(defprotocol MaybeMutable
  (mutable? [this] "Returns true if the value is mutable."))

(extend-type default
  MaybeMutable
  (mutable? [_] true))

;; object
(mutable? #js {})
;; => true

;; array
(mutable? #js [])
;; => true

;; string
(mutable? "")
;; => true

;; function
(mutable? (fn [x] x))
;; => true
```

На щастя, не всі значення обʼєктів JavaScript є змінюваними, тому ми можемо змінити реалізацію `MaybeMutable` таким чином, що виклик буде повертати значення `false` для рядків та функцій.

```clojure
(extend-protocol MaybeMutable
  string
  (mutable? [_] false)

  function
  (mutable? [_] false))


;; object
(mutable? #js {})
;; => true

;; array
(mutable? #js [])
;; => true

;; string
(mutable? "")
;; => false

;; function
(mutable? (fn [x] x))
;; => false
```

Для дат з JavaScript не існує спеціального символу, тому доведеться розширювати `js/Date` напряму. Те саме стосується решти типів, які можна знайти у глобальному просторі `js`.

##### Конвертація даних

Для конвертації даних з типів ClojureScript до типів JavaScript та навпаки ми використовуємо функції `clj->js` та `js->clj`, що базуються на протоколах `IEncodeJS` та `IEncodeClojure`.

Наприклад, скористаємося тип Set, що зʼявився у версії ES6. На сьогодні цей тип наявний не в кожному рантаймі.

###### З ClojureScript до JS

Перш за все розширимо тип "множина" (set) з ClojureScript так, що його можна буде конвертувати у JS. За замовчування множини конвертуються у масиви:

```clojure
(clj->js #{1 2 3})
;; => #js [1 3 2]
```

Давайте це виправимо. Функція `clj->js` має конвертувати значення рекурсивно, тому переконаємося у тому, що весь зміст множини сконвертований, та створимо нову множину із конвертованими даними:

```clojure
(extend-type PersistentHashSet
  IEncodeJS
  (-clj->js [s]
    (js/Set. (into-array (map clj->js s)))))

(def s (clj->js #{1 2 3}))
(es6-iterator-seq (.values s))
;; => (1 3 2)

(instance? js/Set s)
;; => true

(.has s 1)
;; => true
(.has s 2)
;; => true
(.has s 3)
;; => true
(.has s 4)
;; => false
```

Функція `es6-iterator-seq` — експериментальна функція в стандартній бібліотеці ClojureScript для отримання послідовності з типів ES6, які можна ітерувати.

###### З JS до ClojureScript

Час розширити тип set з JS та конвертувати його у ClojureScript. Подібно до фунції `clj->js`, функція `js->clj` рекурсивно конвертує значення структури даних:

```clojure
(extend-type js/Set
  IEncodeClojure
  (-js->clj [s options]
    (into #{} (map js->clj (es6-iterator-seq (.values s))))))

(= #{1 2 3}
   (js->clj (clj->js #{1 2 3})))
;; => true

(= #{[1 2 3] [4 5] [6]}
   (js->clj (clj->js #{[1 2 3] [4 5] [6]})))
;; => true
```

Зауважимо, що не існує однозначних відповідностей між значенням ClojureScript та JavaScript. Наприклад, ключові слова ClojureScript при конвертації за допомогою `clj->js` перетворюються на рядки.


#### Редукції

Функція `reduce` базується на протоколі `IReduce`, який дозволяє проводити редукцію сторонніх типів. Окрім використання таких типів із `reduce`, вони також будуть працювати з `transduce`, завдяки чому ми зможемо реалізувати редукцію із перетворювачем.

Масиви з JS у ClojureScript вже підтримують редукцію:

```clojure
(reduce + #js [1 2 3])
;; => 6

(transduce (map inc) conj [] [1 2 3])
;; => [2 3 4]
```

Але нові типи, що зʼявилися у версії ES6, не надають такої можливості, тому для них ми реалізуємо протокол `IReduce`. Ми отримаємо ітератор за допомогою функції `values`  множини та конвертуємо цей ітератор за допомогою функції `es6-iterator-seq` у послідовність. Після цього ми делегуємо редукцію отриманої послідовності оригінальній функції `reduce`.

```clojure
(extend-type js/Set
  IReduce
  (-reduce
   ([s f]
     (let [it (.values s)]
       (reduce f (es6-iterator-seq it))))
   ([s f init]
     (let [it (.values s)]
       (reduce f init (es6-iterator-seq it))))))

(reduce + (js/Set. #js [1 2 3]))
;; => 6

(transduce (map inc) conj [] (js/Set. #js [1 2 3]))
;; => [2 3 4]
```

До асоційовані структури даних можна застосовувати функцію `reduce-kv`, яка базується на протоколі `IKVReduce`. Основна відмінність між `reduce` та `reduce-kv` полягає у тому, що остання використуває у якості редʼєсера функцію, що очікує три аргумента. 

Розглянемо приклад. Ми проведемо редукцію мапи на вектор пар. Зауважте, що вектори поєднують вндекси та значення, тому редукцію векторів також можна проводити за допомогою `reduce-kv`.

```clojure
(reduce-kv (fn [acc k v]
             (conj acc [k v]))
           []
           {:foo :bar
            :baz :xyz})
;; => [[:foo :bar] [:baz :xyz]]
```

Розширимо новий тип map так, що він буде забезпечувати підтримку `reduce-kv`. Для цього отримаємо послідовність пар "ключ-значення" та викличемо фукнцію-редʼюсер із акумулятором, а ключі та значення передамо як позиційні аргументи: 

```clojure
(extend-type js/Map
  IKVReduce
  (-kv-reduce [m f init]
   (let [it (.entries m)]
     (reduce (fn [acc [k v]]
               (f acc k v))
             init
             (es6-iterator-seq it)))))

(def m (js/Map.))
(.set m "foo" "bar")
(.set m "baz" "xyz")

(reduce-kv (fn [acc k v]
             (conj acc [k v]))
           []
           m)
;; => [["foo" "bar"] ["baz" "xyz"]]
```

В обох прикладах ми здійснили делегування до функції `reduce`, що знає про отримані значення та завершує виконання, коли доходить до таких значень. Зверніть увагу на наступне. Якщо ви не реалізуєте такі протоколи, для використання функції `reduce` вам доведеться перевіряти значення для передчасного припинення виконання.


#### Асинхронність

Існують типи, що передбачають асинхронні обчислення. Значення, яке представляють такі типи, може не бути реалізованим у певний момент часу. Дізнатися стан реалізації значення можна за допомогою предиката `realized?`.

Продемонструємо це на прикладі типу `Delay`, який отримує обчислення та виконує його у той момент, коли зʼявляється потреба у результаті. При запиті значення обчислення відбувається, і відстрочене значення реалізується.

```clojure
(defn computation []
  (println "running!")
  42)

(def d (Delay. computation nil))

(realized? d)
;; => false

(deref d)
;; running!
;; => 42

(realized? d)
 ;; => true

@d
;; => 42
```

Обидві функції базуються на протоколах, а саме `IPending` та `IDeref`.

Стандарт ES6 представив тип, що уособлює поняття асинхронного обчислення, що може привести до помилки: проміс. Проміс представляє значення, що буде реалізоване через невідомий час, і може знаходитися в одному з трьох станів:

 - `pending`(очікування): значення недоступне для обчислення
 - `rejected`(відмова): сталася помилка, проміс містить значення, що вказує на помилку
 - `resolved`(вирішення): обчислення пройшло успішно, проміс містить значення результату

Інтерфейс промісів, визначений у стандарті ES6, не дозволя. дізнатися стан промісу у певний час, ми скористаємося промісами з бібліотеки Bluebird. Проміси Bluebird можна використовувати з бібліотекою [Promesa](https://github.com/funcool/promesa).

Почнемо з того, що додамо можливість перевірки, чи проміс здійснився (має стан resolved або rejected) за допомогою предиката `realized?`. Необхідно реалізувати протокол `IPending`:

```clojure
(require '[promesa.core :as p])

(extend-type js/Promise
  IPending
  (-realized? [p]
    (not (.isPending p))))


(p/promise (fn [resolve reject]))
;; => #<Promise {:status :pending}>

(realized? (p/promise (fn [resolve reject])))
;; => false

(p/resolved 42)
;; => #<Promise {:status :resolved, :value 42}>

(realized? (p/resolved 42))
;; => true

(p/rejected (js/Error. "OH NO"))
;; => #<Promise {:status :rejected, :error #object[Error Error: OH NO]}>

(realized? (p/rejected (js/Error. "OH NO")))
;; => true
```

Тепер розширимо тип промісів таким чином, що матимемо змогу отримувати значення проміса. Якщо проміс знаходиться у нереалізованому стані, результатом буде спеціальне ключове слово `:promise/pending`. Інакше отримаємо значення проміса, тобто помилку або результат:

```clojure
(require '[promesa.core :as pro])

(extend-type js/Promise
  IDeref
  (-deref [p]
    (cond
      (.isPending p)
      :promise/pending

      (.isRejected p)
      (.reason p)

      :else
      (.value p))))

@(p/promise (fn [resolve reject]))
;; => :promise/pending

@(p/resolved 42)
;; => 42

@(p/rejected (js/Error. "OH NO"))
;; => #object[Error Error: OH NO]
```


#### Стан

Конструкти стану у ClojureScript(атоми, волатайли) мають різні характеристики та семантику, а операції на таих конструктах, такі як `add-watch`, `reset!` або `swap!` визначаються протоколами.


##### Атом

Для демонстрації роботи таких протоколів реалізуємо власну спрощену версію `Atom`. Наша версія не матиме підтримки валідаторів та метаданих. Натомість будуть такі функції:

- `deref` - отримати актуально значення атома
- `reset!` - присвоїти попереднє значення
- `swap!` - замінити на функцію для зміни стану

Функція `deref` базується на протоколі `IDeref`, `reset!` — на `IReset`, а `swap!`, відповідно, на `ISwap`. Почнемо з визначення типу даних та конструктора для нашої реалізації атома:

```clojure
(deftype MyAtom [^:mutable state ^:mutable watches]
  IPrintWithWriter
  (-pr-writer [p writer _]
    (-write writer (str "#<MyAtom " (pr-str state) ">"))))

(defn my-atom
  ([]
    (my-atom nil))
  ([init]
    (MyAtom. init {})))

(my-atom)
;; => #<MyAtom nil>

(my-atom 42)
;; => #<MyAtom 42>
```

Зауважте, що ми позначили поточний стан атома (`state`) та мапу спостерігачів  (`watches`) метаданими `{:mutable true}`. Ми будемо змінювати ці дані, і вказуємо це явно за допомогою анотацій.

Наш тип `MyAtom` поки що не дуже корисний. Почнемо з реалізації протоколу `IDeref` для отримання поточного значення атому:

```clojure
(extend-type MyAtom
  IDeref
  (-deref [a]
    (.-state a)))

(def a (my-atom 42))

@a
;; => 42
```

Тепер ми можемо отримати значення атом, тому перейдемо до реалізації протоколу `IWatchable`, завдяки якому ми зможемо додавати та видаляти спостерігачі до нашого атома. Зберагати спостерігачі будемо у мапі `watches`, поєднуючі ключі та функції-колбеки.

```clojure
(extend-type MyAtom
  IWatchable
  (-add-watch [a key f]
    (let [ws (.-watches a)]
      (set! (.-watches a) (assoc ws key f))))

  (-remove-watch [a key]
    (let [ws (.-watches a)]
      (set! (.-watches a) (dissoc ws key))))

  (-notify-watches [a oldval newval]
    (doseq [[key f] (.-watches a)]
      (f key a oldval newval))))
```

Ми можемо додавати нові спостерігачі до атома, але це не дуже корисно, бо ми не можемо його змінювати. Для змін необхідно реалізувати протокол `IReset` та подбати про те, щоб повідомляти спостерігачів про зміну значення атома.

```clojure
(extend-type MyAtom
  IReset
  (-reset! [a newval]
    (let [oldval (.-state a)]
      (set! (.-state a) newval)
      (-notify-watches a oldval newval)
      newval)))
```

Переконаємося у правильності результату. Додамо спостерігач, змінимо значення атома, викличемо спостерігач та видалимо його:

```clojure
(def a (my-atom 41))
;; => #<MyAtom 41>

(add-watch a :log (fn [key a oldval newval]
                    (println {:key key
                              :old oldval
                              :new newval})))
;; => #<MyAtom 41>

(reset! a 42)
;; {:key :log, :old 41, :new 42}
;; => 42

(remove-watch a :log)
;; => #<MyAtom 42>

(reset! a 43)
;; => 43
```

Реалізуємо також протокол `ISwap`. Метод `-swap!` може отримувати один, два, три або чотири аргумента:


```clojure
(extend-type MyAtom
  ISwap
  (-swap!
   ([a f]
    (let [oldval (.-state a)
          newval (f oldval)]
      (reset! a newval)))

   ([a f x]
     (let [oldval (.-state a)
           newval (f oldval x)]
       (reset! a newval)))

   ([a f x y]
     (let [oldval (.-state a)
           newval (f oldval x y)]
       (reset! a newval)))

   ([a f x y more]
     (let [oldval (.-state a)
           newval (apply f oldval x y more)]
       (reset! a newval)))))
```

ми отримали власну реалізацію абстракції атома. Перевіримо її роботу у REPL та переконаємося, що вона має очікувану поведінку:

```clojure
(def a (my-atom 0))
;; => #<MyAtom 0>

(add-watch a :log (fn [key a oldval newval]
                    (println {:key key
                              :old oldval
                              :new newval})))
;; => #<MyAtom 0>

(swap! a inc)
;; {:key :log, :old 0, :new 1}
;; => 1

(swap! a + 2)
;; {:key :log, :old 1, :new 3}
;; => 3

(swap! a - 2)
;; {:key :log, :old 3, :new 1}
;; => 1

(swap! a + 2 3)
;; {:key :log, :old 1, :new 6}
;; => 6


(swap! a + 4 5 6)
;; {:key :log, :old 6, :new 21}
;; => 21

(swap! a * 2)
;; {:key :log, :old 21, :new 42}
;; => 42

(remove-watch a :log)
;; => #<MyAtom 42>
```

Вийшло! Ми реалізували версію атома ClojureScript без підтримки метаданих чи валідаторів. Пропонуємо нашим читачам додати ці можливості самостійно для вправи. Зауважте, що вам доведеться змінювати тип `MyAtom` для збереження метаданих та валідатора.

##### Волатайли

Волатайли простіші за атоми, бо не підтримують спостереження за змінами. Усі зміни переписують попередні значення як змінювані сутнотсті, що присутні майже у кожній мові. Волатайли грунтуються на протоколі `IVolatile`, що визначає лише метод для `vreset!`, бо `vswap!` реалізований як макрос.

Створимо власний тип та конструктор:

```clojure
(deftype MyVolatile [^:mutable state]
  IPrintWithWriter
  (-pr-writer [p writer _]
    (-write writer (str "#<MyVolatile " (pr-str state) ">"))))

(defn my-volatile
  ([]
    (my-volatile nil))
  ([v]
    (MyVolatile. v)))

(my-volatile)
;; => #<MyVolatile nil>

(my-volatile 42)
;; => #<MyVolatile 42>
```

`MyVolatile` все ще має забезпечити підтримку дереферування та повернення до попереднього значення. Реалізуємо `IDeref` та `IVolatile`, що дозволить нам використовувати `deref`, `vreset!` та `vswap!`.

```clojure
(extend-type MyVolatile
  IDeref
  (-deref [v]
    (.-state v))

  IVolatile
  (-vreset! [v newval]
    (set! (.-state v) newval)
    newval))

(def v (my-volatile 0))
;; => #<MyVolatile 42>

(vreset! v 1)
;; => 1

@v
;; => 1

(vswap! v + 2 3)
;; => 6

@v
;; => 6
```


#### Мутація

З розділу про перехідні струтури даних ми дізналися про змінювані аналоги незмінних та стіфких стуртур даних, що існують у ClojureScript. такі структури даних є змінюваними, а опеарції з такими стуртурами позначаються окличним знаком (`!`) на кінці назви. Як ви могли здогадатися, кожна операція зі змінюваними структурами даних грунтується на протоколі.

##### Від стійких структур до перехідних та навпаки

Ми дізналися, що можна трансформувати структури даних за допомогою функції `transient`, яка грунтується на протоколі `IEditableCollection`. Для трансформації у стійку структуру використовується `persistent!`, що грунтується на `ITransientCollection`.

Реалізація незмінюваних та стійких структур даних та відповідних змінюваних аналогів не входить до тем, що їх буде розглянуто у цій книзі, але ми рекомендуємо ознайомитися із реалізацією структур даних ClojureScript, якщо ця тема вас зацікавила.


##### Приклад: бібліотека hodgepodge

Бібліотека hodgepodge призначена для управління локальним сховищем та сесіями браузера як перехідними структурами даних у ClojureScript. Ця бібліотека дозволяє вставляти, читати та видаляти структури даних ClojureScript без необхідності кодування та декодування таких даних.

Сховище браузера — це просте сховище, що зберігає пари "ключ-значення" у вигляді рядків. Завдяки тому, що будь-які струкутри даних ClojureScript можна зберегти як рядки та читати за допомогою the reader, ми можемо зберігати дані на ClojureScript у сховищі. Також можливо розширювати reader таким чином, що стає можливим читання користувацьких типів даних. Завдяки цьому ми можемо зберігати дані, а кодування та декодування довірити бібліотеці `hodgepodge`.

Почнемо з того, що огорнемо низькорівневі методи сховища у функції. Сховище дозволяє виконання наступних операцій:

- отримання знаення, що відповідає ключеві
- призначення ключеві певного значення
- видалення значення за ключем
- підрахунок кількості записів
- очищення сховища

Створимо для цих функцій більш ідіоматичний для ClojureScript інтерфейс:

```clojure
(defn contains-key?
  [^js/Storage storage ^string key]
  (let [ks (.keys js/Object storage)
        idx (.indexOf ks key)]
    (>= idx 0)))

(defn get-item
  ([^js/Storage storage ^string key]
     (get-item storage key nil))
  ([^js/Storage storage ^string key ^string default]
     (if (contains-key? storage key)
       (.getItem storage key)
       default)))

(defn set-item
  [^js/Storage storage ^string key ^string val]
  (.setItem storage key val)
  storage)

(defn remove-item
  [^js/Storage storage ^string key]
  (.removeItem storage key)
  storage)

(defn length
  [^js/Storage storage]
  (.-length storage))

(defn clear!
  [^js/Storage storage]
  (.clear storage))
```

Нічого цікавого, ми просто огорнули методи сховища у більш прийнятний інтерфейс. Тепер визначимо кілька функцій для серіалізації стурктур даних ClojureScript у рядки:

```clojure
(require '[cljs.reader :as reader])

(defn serialize [v]
  (binding [*print-dup* true
            *print-readably* true]
    (pr-str v)))

(def deserialize
  (memoize reader/read-string))
```

Функція `serialize` використовується для конвертації структури даних з ClojureScript у рядок за допомогою функції `pr-str` та конфігурування певних динамійних змінних для забезпечення бажаної поведінки:

- `*print-dup*` призначено значення `true` для збереження типу обʼєктів при подальшому читанні 
- `*print-readably*` призначено значення `true` для запобігання конвертації не-альфачислових знаків

Функція `deserialize` викликає функцію читання рядка та збереження результату у структуру ClojureScript: `read-string`. Ця функція мемоізована, тому не буде викликати читання щоразу, коли необхідно серіалізувати рядок, тому що ми виходимо з того, що рядок, що зустрічається більше одного разу, відповідає одній структурі даних.

Тепер ми можем розширювати вбудований тип `Storage` браузера до поведінки перехідної структури. Не забувайте про те, що тип `Storage` доступний лише у середовищі браузера. Почнемо з реалізації протокола `ICounted` для підрахунку сутностей у сховищі. Звернемося до фукнції `length`, що її ми визначили раніше.

```clojure
(extend-type js/Storage
  ICounted
  (-count [^js/Storage s]
   (length s)))
```

Ми б хотіли застосовувати функцію `assoc!` and `dissoc!` для додання та видалення пар "ключ-значення" зі сховища, а також для забезпечення можливості читання. Реалізуємо протокол `ITransientAssociative` для `assoc!`, `ITransientMap` для `dissoc!` та `ILookup`  для читання ключів зі сховища.

```clojure
(extend-type js/Storage
  ITransientAssociative
  (-assoc! [^js/Storage s key val]
    (set-item s (serialize key) (serialize val))
    s)

  ITransientMap
  (-dissoc! [^js/Storage s key]
    (remove-item s (serialize key))
    s)

  ILookup
  (-lookup
    ([^js/Storage s key]
       (-lookup s key nil))
    ([^js/Storage s key not-found]
       (let [sk (serialize key)]
         (if (contains-key? s sk)
           (deserialize (get-item s sk))
           not-found)))))
```

Тепер ми можем виконати деякі операції зі сховищем сесії та з локальним сховищем. Спробуємо це зробити:

```clojure
(def local-storage js/localStorage)
(def session-storage js/sessionStorage)

(assoc! local-storage :foo :bar)

(:foo local-storage)
;; => :bar

(dissoc! local-storage :foo)

(get local-storage :foo)
;; => nil

(get local-storage :foo :default)
;; => :default
```

Насамкінець ми хочемо використовувати функцію `conj!` та `persistent!` із локальним сховищем, тому слід реалізувати протокол `ITransientCollection`:

```clojure
(extend-type js/Storage
  ITransientCollection
  (-conj! [^js/Storage s ^IMapEntry kv]
    (assoc! s (key kv) (val kv))
    s)

  (-persistent! [^js/Storage s]
    (into {}
          (for [i (range (count s))
                :let [k (.key s i)
                      v (get-item s k)]]
            [(deserialize k) (deserialize v)]))))
```

Функція `conj!` отримує ключ та значення з запису та передає ці дані функції `assoc!`. Функція `persistent!` десеріалізує кожну пару "ключ-значення" у сховищі та повертає незмінну копію сховища у вигляді мапи.

```clojure
(clear! local-storage)

(persistent! local-storage)
;; => {}

(conj! local-storage [:foo :bar])
(conj! local-storage [:baz :xyz])

(persistent! local-storage)
;; => {:foo :bar, :baz :xyz}
```


##### Перехідні вектори та множини

Ми познайомилися з більшістю протоколів для перехідних даних, окрім двох — `ITransientVector`, що дозволяє використовувати функцію `assoc!` зі змінюваними векторами, та `ITransientSet` для використання `disj!` з множинами.

Продемонструємо протокол `ITransientVector`: розширимо тип масиву так, що він стане перехідною структурою даних:

```clojure
(extend-type array
  ITransientAssociative
  (-assoc! [arr key val]
    (if (number? key)
      (-assoc-n! arr key val)
      (throw (js/Error. "Array's key for assoc! must be a number."))))

  ITransientVector
  (-assoc-n! [arr n val]
    (.splice arr n 1 val)
    arr))

(def a #js [1 2 3])
;; => #js [1 2 3]

(assoc! a 0 42)
;; => #js [42 2 3]

(assoc! a 1 43)
;; => #js [42 43 3]

(assoc! a 2 44)
;; => #js [42 43 44]
```

Для демонстрації протоколу `ITransientSet` розширимо тип `Set` зі стандарту ES6 до перехідної структури, що підтримує операції `conj!`, `disj!` та `persistent!`. Зауважимо, що попередньо ми розширили тип Set таким чином, що його можна перетворити на структуру ClojureScript.

```clojure
(extend-type js/Set
  ITransientCollection
  (-conj! [s v]
    (.add s v)
    s)

  (-persistent! [s]
   (js->clj s))

  ITransientSet
  (-disjoin! [s v]
    (.delete s v)
    s))

(def s (js/Set.))

(conj! s 1)
(conj! s 1)
(conj! s 2)
(conj! s 2)

(persistent! s)
;; => #{1 2}

(disj! s 1)

(persistent! s)
;; => #{2}
```
