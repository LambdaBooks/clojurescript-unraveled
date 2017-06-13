
### Метадані

_ClojureScript_ дозволяє додавати до символів, варів та стійких колекцій метадані. Метадані мають форму мапи та містять інформацію про сутність, до якої відносяться. Компілятор _ClojureScript_ використовує метадані для певних цілей, зокрема для визначення типів; також система метаданих може бути корисною для розробників інструментів, бібліотек та застосунків.

У щоденній роботі нечасто зʼявляються причини скористатися метаданими, але це корисна можливість мови, про яку варто знати, бо у певний момент вона стане у нагоді. Метадані спрощують інстроспекцію коду у рантаймі та документування коду. Протягом цього розділу ми у цьому переконаємося.


#### Вари

Визначимо вар та розглянемо, метадані, що їх додано до нього за замовчуванням. Зверніть увагу на те, що код виконується у REPL, тому метадані вару, визначеного у файлі коду, можуть бути іншими. Для отримання метаданих певного значення слід викликати функцію `meta`:

```clojure
(def answer-to-everything 42)
;; => 42

#'answer-to-everything
;; => #'cljs.user/answer-to-everyhing

(meta #'answer-to-everything)
;; => {:ns cljs.user,
;;     :name answer-to-everything,
;;     :file "NO_SOURCE_FILE",
;;     :source "answer-to-everything",
;;     :column 6,
;;     :end-column 26,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (),
;;     :doc nil,
;;     :test nil}
```

Слід звернути увагу на кілька речей. Попреше, `#'answer-to-everything` надає посилання на `Var`, який містить значення символу `answer-to-everything`. Ми бачимо, що він містить інформацію про простір імен  (`:ns`), у якому був визначений, його імʼя, файл (хоча він був визначений у REPL і не має файлу вихідного коду), джерело, позицію у файлі, де він був визначений, список аргументів (це має сенс лише для функцій), документаційний рядок та тестову функцію.
 
Подивимося на метадані функції:

```clojure
(defn add
  "A function that adds two numbers."
  [x y]
  (+ x y))

(meta #'add)
;; => {:ns cljs.user,
;;     :name add,
;;     :file "NO_SOURCE_FILE",
;;     :source "add",
;;     :column 7,
;;     :end-column 10,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (quote ([x y])),
;;     :doc "A function that adds two numbers.",
;;     :test nil}
```

Видно, що список аргументів збережений у полі `:arglists`, а відповідний документаційний рядок - у полі `:doc`. Тепер визначимо тестову функцію та дізнаємося, як використовується поле  `:test`. 

```clojure
(require '[cljs.test :as t])

(t/deftest i-pass
  (t/is true))

(meta #'i-pass)
;; => {:ns cljs.user,
;;     :name i-pass,
;;     :file "NO_SOURCE_FILE",
;;     :source "i-pass",
;;     :column 12,
;;     :end-column 18,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (),
;;     :doc "A function that adds two numbers.",
;;     :test #<function (){ ... }>}
```

Атрибут `:test` (скорочений для стислості) у полі  `i-pass` представляє собою тестову функцію.  Ця функція використовується бібліотекою `cljs.test` для визначення та запуску тестів у вказаних просторах імен.


#### Значення

Ми дізналися про те, що вари можуть мати метадані і як саме такі метадані виглядають з точки зору бібліотеки для тестування  `cljs.test`. Стійкі колекції також можуть мати метадані, хоча і не мають їх за замовчуванням. Ми можемо використати функцію `with-meta` для створення похідного обʼєкта з тим самим значенням та типом, що має надані метадані:

```clojure
(def map-without-metadata {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-without-metadata)
;; => nil

(def map-with-metadata (with-meta map-without-metadata
                                  {:answer-to-everything 42}))
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:answer-to-everything 42}

(= map-with-metadata
   map-without-metadata)
;; => true

(identical? map-with-metadata
            map-without-metadata)
;; => false
```

Навряд чи стане несподіванкою те, що метадані не впливають на рівність двох структур даних, адже рівніть у _ClojureScript_ виходить зі значення. Інший цікавий аспект - функція `with-meta` створіє новий обʼєкт того ж типу та значення та додає метадані вихідного обʼєкта.

А що станеться з метаданими, якщо створити нове значення на базі стійкої структури даних?

```clojure
(def derived-map (assoc map-with-metadata :language "Clojure"))
;; => {:language "Clojure"}

(meta derived-map)
;; => {:answer-to-everything 42}
```

Як ви можете побачити у цьому прикладі, у похідних версіях стійких структур даних зберігаються метадані оригіналів. Але треба памʼятати про певні нюанси. Метадані зберігаться лише у тих випадках, коли функція, що створює похідну структуру, повертає колекцію того ж типу. Зовсім інший ефект буде при трансформації, що призводить до зміни типу. Продемонструємо таку поведінку на прикладі створення послідовності або субвектора з вектора: 

```clojure
(def v (with-meta [0 1 2 3] {:foo :bar}))
;; => [0 1 2 3]

(def sv (subvec v 0 2))
;; => [0 1]

(meta sv)
;; => nil

(meta (seq v))
;; => nil
```


#### Синтаксис метаданих

Рідер ClojureScrip має синтаксичну підтримку різних типів запису метаданих. Можна ставити перед визначенням варів та колекцій знак «карет» (`^`) та мапи з відповідною анотацією.

```clojure
(def ^{:doc "The answer to Life, Universe and Everything."} answer-to-everything 42)
;; => 42

(meta #'answer-to-everything)
;; => {:ns cljs.user,
;;     :name answer-to-everything,
;;     :file "NO_SOURCE_FILE",
;;     :source "answer-to-everything",
;;     :column 6,
;;     :end-column 26,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (),
;;     :doc "The answer to Life, Universe and Everything.",
;;     :test nil}

(def map-with-metadata ^{:answer-to-everything 42} {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:answer-to-everything 42}
```

Зверніть увагу на те, що метадані, що їх було надано при визначенні вару `answer-to-everything`, поєднані з метаданими вару.

Розповсюдженим прикладом використання метаданих є присвоєння значення `true` певним ключам. Наприклад, нам потрібно вказати у метаданих вару, чи відповідна змінна є динамічною змінною або константною. Для таких випадків є скорочений запис із використання знаку «карет» та ключового слова. Ось деякі приклади:

```clojure
(def ^:dynamic *foo* 42)
;; => 42

(:dynamic (meta #'*foo*))
;; => true

(def ^:foo ^:bar answer 42)
;; => 42

(select-keys (meta #'answer) [:foo :bar])
;; => {:foo true, :bar true}
```

Інше скорочення для зпису метаданих - використання "карет" та символу, що їх буде додано до метаданих мапи за ключем `:tag`. Використання тегів, зокрема `^boolean`, надає компілятору _ClojureScript_ інформацію щодо типу виразу або типу значення, що його повертає функція.

```clojure
(defn ^boolean will-it-blend? [_] true)
;; => #<function ... >

(:tag (meta #'will-it-blend?))
;; => boolean

(not ^boolean (js/isNaN js/NaN))
;; => false
```

#### Функції для роботи з метаданими

Ми вже дізналися про функцї  `meta` та `with-meta`, але _ClojureScript_ має цілу низку функцій для роботи з метаданими. Є функція `vary-meta`, подібна до  `with-meta` у тому, що створює новий похідний обʼєкт того ж типу та значення, що й оригінальний, але не приймає метадані, щоб додати їх напряму. Натомість, ця функція приймає функцію, яка буде застосована до метаданих вказаного обʼєкта з метою трансформування для отримання нових метаданих. Ось як це відбувається:

```clojure
(def map-with-metadata ^{:foo 40} {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:foo 40}

(def derived-map (vary-meta map-with-metadata update :foo + 2))
;; => {:language "ClojureScript"}

(meta derived-map)
;; => {:foo 42}
```

Якщо ми натомість хочемо змінити метадані існуючого вара чи його значення, ми можемо скористатися функцією `alter-meta!` або замінити на іншу мапу за допомогою `alter-meta!`:

```clojure
(def map-with-metadata ^{:foo 40} {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:foo 40}

(alter-meta! map-with-metadata update :foo + 2)
;; => {:foo 42}

(meta map-with-metadata)
;; => {:foo 42}

(reset-meta! map-with-metadata {:foo 40})
;; => {:foo 40}

(meta map-with-metadata)
;; => {:foo 40}
```
