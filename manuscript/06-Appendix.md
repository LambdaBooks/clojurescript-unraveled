### Додаток A: Інтрекактивна розробка за допомогою Figwheel

#### Вступ

У цьому проекті ми не будемо у сотий раз писати “Hello World”&#8212;. Натомість створимо веб-сторінку, що запитує вік користувача та перераховує роки у дні, виходячи зі спрощення про 365 днів у році.

Ми скористаємося плагіном _figwheel_ для `leiningen`. Це плагін створює повністю інтерактивне середовище на базі REPL з функцією автоматичного перезавантаження.

#### Перші кроки

Найперший крок - створення нового проекта за допомогою шаблона _figwheel_ lein. Для створення проекту під назвою `age` введемо наступні команди:

```bash
$ lein new figwheel age
Retrieving figwheel/lein-template/0.3.5/lein-template-0.3.5.pom from clojars
Retrieving figwheel/lein-template/0.3.5/lein-template-0.3.5.jar from clojars
Generating fresh 'lein new' figwheel project.
$ cd age # move into newly created project directory
```

Проект має таку структуру:

```
> tree age      # the linux "tree" utility displays dir structure
age
├── .gitignore
├── project.clj
├── README.md
├── resources
│   └── public
│       ├── css
│       │   └── style.css
│       └── index.html
└── src
    └── age
        └── core.cljs
```

Файл `project.clj` містить інформацію, що її потребує Leiningen для завантаження залежностей та зборки проекту. Домовимося поки що вважати, що в ньому є все необхідне для правильної роботи та не розглядати його зміст детально.

Відкрийте `index.html` та додайте наступний код:

```html
<!DOCTYPE html>
<html>
  <head>
    <link href="css/style.css" rel="stylesheet" type="text/css">
    <meta http-equiv="Content-Type" content="text/html;charset=utf-8" />
  </head>
  <body>
    <div id="app">
      <h1>Age in Days</h1>
      <p>
        Enter your age in years:
        <input type="text" size="5" id="years">
        <button id="calculate">Calculate</button>
      </p>
      <p id="feedback"></p>
    </div>
    <script src="js/compiled/age.js" type="text/javascript"></script>
  </body>
</html>
```

Найцікавіше відбувається у файлі `core.cljs`. Поки що не міняйте в ньому нічого та запустіть оточення figwheel, яке завантажить велику кількість залежностей, та запустіть сервер.

```bash
$ lein figwheel
Retrieving lein-figwheel/lein-figwheel/0.5.2/lein-figwheel-0.5.2.pom from clojars
Retrieving figwheel-sidecar/figwheel-sidecar/0.5.2/figwheel-sidecar-0.5.2.pom from clojars
Retrieving org/clojure/clojurescript/1.7.228/clojurescript-1.7.228.pom from central
... # much more output
Prompt will show when Figwheel connects to your application
```

Якщо ви користуєтеся операційною системою Linux або Mac OS X, введіть команду `rlwrap lein figwheel`. У браузері перейдіть за посиланням `http://localhost:3449`. Відкривши консоль, ви маєте побачити щось подібне до наступного скріншота:

image::localhost1.png[Screenshot of web page and console]

У терміналі зʼявиться наступне поведомлення:

```
$ rlwrap lein figwheel
To quit, type: :cljs/quit
cljs.user=>
```

Виконайте цю рекомендацію, а саме у файлі `core.cljs` замініть `(println...)` та збережіть файл. Зміни будуть одразу відображені у браузері.

Зробіть навмисну помилку: додайте зайву закриваючу дужку до `println`. Після збереження файлу браузер має показати помилку компіляції.


#### Взаємодія з JavaScript

У вікні REPL введіть наступну команду для виклику функції `window.alert()`:

```clojure
(.alert js/window "It works!")
;; => nil
```

У загальному випадку виклик функцій JavaScript з контексту ClojureScript передбачає введення імені функції (перед яким вводиться крапка), обʼєкту, який містить цю функцію, та будь-яких необхідних параметрів. Після введеня команди ви маєте побачити вікно повідомлення у браузері. Натисніть "відміна", і побачите у терміналі `nil` та запит наступної команди. Інший спосіб:

```clojure
(js/alert "It works!")
;; => nil
```

Перший варіант працює у будь-якому випадку, тому для консистентності ми будемо користуватися ним у цьому уроці.

Обʼєкти JavaScript можна створювати з контексту ClojureScript за допомогою того ж самого синтаксису - вводити імʼя класу та крапку. Методи JavaScript також можна викликати:

```clojure
> (def d (js/Date.))
;; => #'cljs.user/d
> d
;; => #inst "2016-04-03T21:04:29.908-00:00"
> (.getFullYear d)
;; => 2016

> (.toUpperCase "doh!")
;; => "DOH!"

> (.getElementById js/document "years")
;; => #object[HTMLInputElement [object HTMLInputElement]]
```

Настіпний приклад має зробити наші кроки більш зрозумілими. Для читання властивості обʼєкта слід використовувати синтаксис з крапкою та дефісом: вводити `.-` перед назвою властивості. У вікні браузера введіть число у поле для вводу (24 у нашому прикладі), потім поверніться до терміналу.

```clojure
(def year-field (.getElementById js/document "years"))
;; => #'cljs.user/year-field

(.-value year-field)
;; => "24"

(set! (.-value year-field) "25")
;; => "25"
```

Все працює, але поки що ми побачили лише трохи більше, ніж прямий переклад з JavaScript на ClojureScript. Наступним кроком додамо обробник подій на кнопку. Обробка подій ускладнюється особливостями крос-платформної взаємодії, тому виконувати її на на простому ClojureScript не бажано.

Рішенням є використання бібліотеки Google Closure library. Для підключення цієї бібіліотеки внесіть зміни до `:require` на початку файла `core.cljs`:

```clojure
(ns ^:figwheel-always age.core
  (:require [goog.dom :as dom]
            [goog.events :as events]))
```

Тепер отримати доступ до елемента та задати його значення стало простіше. Виконайте наступні кроки у терміналі REPL та спостерігайте за змінами у вікні браузера:

```clojure
(in-ns 'age.core)
(def y (dom/getElement "years"))
;; => #'age.core/y

(set! (.-value y) "26")
;; => "26"

(dom/setTextContent (dom/getElement "feedback") "This works!")
;; => nil
```

Щоб додати подію, слід визначити функцію від одного аргумента (а саме подію, яку треба оборобити), та наказати відповідному елементу HTML очікувати на потрібну подію. Функція `events/listen` приймає три аргументи: елемент, подію та функцію-обробник.

```clojure
(defn testing [evt] (js/alert "Responding to click"))
;; => #'age.core/testing

(events/listen (dom/getElement "calculate") "click" testing)
;; => #<[object Object]>
```

Після цього браузер має реагувати на натискання на кнопку. Для видалення обробника подій слід викликати функцію `unlisten`.

```clojure
(events/unlisten (dom/getElement "calculate") "click" testing)
;; => true
```

Зберемо усе разом у файлі `core.cljs`:

```clojure
(ns ^:figwheel-always age.core
  (:require [goog.dom :as dom]
            [goog.events :as events]))

(enable-console-print!)

(defn calculate
  [event]
  (let [years (.parseInt js/window (.-value (dom/getElement "years")))
        days (* 365 years)]
    (dom/setTextContent (dom/getElement "feedback")
                        (str "That is " days " days old."))))

(defn on-js-reload [])

(events/listen (dom/getElement "calculate") "click" calculate)
```


### Додаток B: Налаштування оточення для розробки на ClojureScript

#### Cursive

Розділ не завершено.

#### Emacs

Розділ не завершено.

#### Vim

Розділ не завершено.
