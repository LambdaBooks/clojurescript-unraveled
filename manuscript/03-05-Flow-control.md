### Керування потоком виконання

Підхід до керування потоком виконання програми у ClojureScript відрізняється від того, який зазвичай використовується в мовах JavaScript, C, та ін.

#### Розгалуження з `if`

Почнемо з основного: `if`. В ClojureScript `if` — це вираз з трьома параметрами: умова, вираз, що буде обчислений, якщо умова — логічне `true`, та вираз, що буде обчислений, коли умова — логічне `false`.

```clojure
(defn discount
  "You get 5% discount for ordering 100 or more items"
  [quantity]
  (if (>= quantity 100)
    0.05
    0))

(discount 30)
;; => 0

(discount 130)
;; => 0.05
```

Блок виразів у формі `do` можна використати, в даному випадку, для того, щоб мати декілька виразів в одній із гілок `if`. Більше про `do` в наступній секції.

#### Розгалуження з `cond`

Деколи простого `if` може бути недостатньо, оскільки в ньому немає `else if`. Макрос `cond` вирішує цю проблему.

У виразі `cond` можна записати декілька умов:

```clojure
(defn mypos?
  [x]
  (cond
    (> x 0) "positive"
    (< x 0) "negative"
    :else "zero"))

(mypos? 0)
;; => "zero"

(mypos? -2)
;; => "negative"
```

Також `cond` має іншу, дуже схожу, форму — `condp`, яка підходить краще, коли усі умови мають однакову предикату:

```clojure
(defn translate-lang-code
  [code]
  (condp = (keyword code)
    :es "Spanish"
    :en "English"
    "Unknown"))

(translate-lang-code "en")
;; => "English"

(translate-lang-code "fr")
;; => "Unknown"
```

Рядок `condp = (keyword code)` означає, що функція `=` буде викликана на кожній умові з результатом виразу `(keyword code)`.

#### Розгалуження з `case`

Вираз розгалуження `case` схожий на `condp`. Головною відмінністю є те, що в `case` предикатою завжди буде `=`, а значення умов обчислюється під час компіляції. Саме тому ця форма ефективніша за `cond` та `condp`, але недоліком є те, що ці значення повинні бути статичні.

Ось попередній приклад переписаний з використанням `case`:

```clojure
(defn translate-lang-code
  [code]
  (case code
    "es" "Spanish"
    "en" "English"
    "Unknown"))

(translate-lang-code "en")
;; => "English"

(translate-lang-code "fr")
;; => "Unknown"
```
