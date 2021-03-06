### Простори імен

#### Створення простору імен

Простір імен (namespace) — основна одиниця модульності в _ClojureScript_. Простори імен схожі на пакети в Java або модулі в Ruby та Python. Вони створюються за допомогою макросу `ns`. Якщо ви бачили код на ClojureScript, ви могли помітити щось подібне на початку кожного файлу:

```clojure
(ns myapp.core
  "Some docstring for the namespace.")

(def x "hello")
```

Простори імен динамічні, що дає змогу створювати їх у будь-який час. Існує домовленість, що на кожен файл повинно бути по одному простору імен. Зазвичай, простір імен створюється на початку файлу, після якого може бути рядок з документацією.

Раніше ми розглянули змінні та символи. Кожна змінна зв'язана з простором імен, в якому вона створена. Якщо створити змінну окремо від простору імен, вона автоматично буде зв'язана з простором імен `cljs.user`, що існує за замовчуванням:

```clojure
(def x "hello")
;; => #'cljs.user/x
```

#### Завантаження інших просторів імен

Створювати простори імен та змінні в них дуже просто, але в цьому не було б сенсу, якщо не можна було б використовувати символи з інших просторів імен. Макрос `ns` вміє завантажувати інші простори імен і має для цього наступний синтаксис:

```clojure
(ns myapp.main
  (:require myapp.core
            clojure.string))

(clojure.string/upper-case myapp.core/x)
;; => "HELLO"
```

Як ви можете бачити, ми використовуємо повністю кваліфіковані імена (простір імен + ім'я символу) для доступу до змінних та функцій з іншого простору імен.

Хоч так ми й можемо використовувати змінні з інших просторів імен, цей спосіб надмірно багатослівний. Буде дуже незручно, якщо простори імен матимуть довгі імена. Директива `:as` використовується для створення додаткових (звичайно коротших) імен для просторів імен. Ось як це виглядає:

```clojure
(ns myapp.main
  (:require [myapp.core :as core]
            [clojure.string :as str]))

(str/upper-case core/x)
;; => "HELLO"
```

Також в _ClojureScript_ можна посилатися лише на конкретні змінні та функції з іншого простору імен за допомогою директиви `:refer`, після якої записується вектор символів, що відповідають іменам функцій та змінних з цього простору імен. Фактично, такі змінні стають частиною поточного простору імен.

```clojure
(ns myapp.main
  (:require [clojure.string :refer [upper-case]]))
(upper-case x)
;; => "HELLO"
```

І нарешті, ви повинні знати, що все, що є у просторі імен `cljs.core`, автоматично завантажується у всі простори імен, тому змінні та функції з цього простору імен не потребують явного завантаження. Інколи ви можете створити змінну з ім'ям, яке вже є в `cljs.core`, що призведе до конфлікту. Для цього в макросі `ns` є директива, що дозволяє вказувати імена, які не будуть завантажені в поточний простір імен.

Розглянемо такий приклад:

```clojure
(ns myapp.main
  (:refer-clojure :exclude [min]))

(defn min
  [x y]
  (if (> x y)
    y
    x))
```

Також в макросі `ns` є інші директиви, як от `:import` для завантаження класів з коду батьківської платформи, та `:require-macros`, про яку ми поговоримо пізніше.

#### Простори імен та імена файлів

When you have a namespace like `myapp.core`, the code must be in a file named
_core.cljs_ inside the _myapp_ directory.  So, the preceding examples with
namespaces `myapp.core` and `myapp.main` would be found in project with a file
structure like this:

Якщо у вас є простір імен, наприклад `myapp.core`, код у ньому повинен бути у файлі _core.cljs_, що знаходиться у директорії _myapp_. Файлова структура проекту з просторами імен `myapp.core` та `myapp.main` буде виглядати наступним чином:

```
myapp
└── src
    └── myapp
        ├── core.cljs
        └── main.cljs
```
