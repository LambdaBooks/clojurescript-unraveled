### Бібліотека Closure

Google Closure Library — це бібліотека на JavaScript, розроблена в Google. Це модульна бібліотека з набором функцій для роботи з DOM та подіями в DOM, Ajax і JSON та багатьма іншими функціями.

Бібліотека Google Closure Library написана з орієнтацією на максимальну сумісність із компілятором Closure Compiler (компілятор _ClojureScript_ потребує його для роботи).

Мова _ClojureScript_ та її компілятор використовують Google Closure Compiler та бібліотеку Closure. Простори імен _ClojureScript_ — це фактично модулі, сумісні з Closure. Це означає, що з бібліотекою Closure можна працювати майже так само, як зі звичайними просторами імен в _ClojureScript_:

```clojure
(ns yourapp.core
  (:require [goog.dom :as dom]))

(def element (dom/getElement "body"))
```

Цей приклад показує, як імпортувати модуль *dom* з бібліотеки Closure і використовувати оголошені в ньому функції.

Також бібліотека Closure надає доступ до спеціальних модулів, що поводяться як класи або об'єкти. Щоб імпортувати такі модулі в формі `(ns...)`, використовують директиву `:import`:

```clojure
(ns yourapp.core
  (:import goog.History))

(def instance (History.))
```

В програмах на _Clojure_ директива `:import` використовується для імпортування класів з бібліотек батьківської платформи (Java), так само, як і в _ClojureScript_ з JavaScript. Але для імпортування типів (класів), написаних на _ClojureScript_, замість `:import` слід використовувати звичайний `:require`.
