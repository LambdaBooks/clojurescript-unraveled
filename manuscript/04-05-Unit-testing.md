### Модульне тестування

Як можна було очікувати, тестування в _ClojureScript_ включає в себе ті самі концепції, що і тестування в інших мовах, таких як Clojure, Java, Python, JavaScript та ін.

Незалежно від мови основним завданням модульного тестування є реалізація певної кількості тестових випадків з метою переконатися, що код, який проходить тестування, поводиться коректним чином та не спричиняє неочікуваних винятків.

Незмінність даних в _ClojureScript_ допомагає створювати програми з потенційно меншою кількістю помилок, а також робить тестування дещо простішим. Ще одна перевага _ClojureScript_ полягає в тому, що мова заохочує до використання простиих даних замість складних об'єктів. Це дуже спрощує створення  mock-об'єктів (фіктивних об'єктів).


#### Перші кроки

Так би мовити «офіційний» фреймворк для тестування _ClojureScript_ знаходиться в просторі імен `cljs.test`. Це дуже проста бібліотека, але для наших потреб її цілком вистачить.

Існують інші бібліотеки з додатковими можливостями або зовсім іншими підходами до тестування, наприклад [test.check](https://github.com/clojure/test.check). Але в цій книзі ми не розглядаємо їх.

Почніть зі створення нового проекту за шаблоном *mies* для `leiningen`:

```bash
$ lein new mies mytestingapp
$ cd mytestingapp
```

Структура цього проекту така сама, як у розділі про управління залежностями, тому тут ми не будемо зупинятися на ній детально.

Створіть структуру директорій для тестів:

```bash
$ mkdir -p test/mytestingapp
$ touch test/mytestingapp/core_tests.cljs
```

Також слід переписати скрипт компілятора `watch.clj` наступним чином, щоб він працював з тестами:

```clojure
(require '[cljs.build.api :as b])

(b/watch (b/inputs "test" "src")
  {:main 'mytestingapp.core_tests
   :target :nodejs
   :output-to "out/mytestingapp.js"
   :output-dir "out"
   :verbose true})
```

Цей скрипт буде спостерігати за змінами та компілювати код  у директоріях `src` та `test`. Точкою входу тепер буде простір імен `mytestingapp.core_tests`.

Скопіюйте наступний код з тестами до файлу `core_tests.cljs`:

```clojure
(ns mytestingapp.core-tests
  (:require [cljs.test :as t]))

(enable-console-print!)

(t/deftest my-first-test
  (t/is (= 1 2)))

(set! *main-cli-fn* #(t/run-tests))
```

Нас цікавить лише ця частина коду тестів:

```clojure
(t/deftest my-first-test
  (t/is (= 1 2)))
```

Макрос `deftest` є базовим примітивом для створення тестів. Першим параметром він приймає назву тесту, після якого можуть бути одна або більше перевірок, записаних за допомогою макросу `is`. У цьому прикладі ми спробуємо перевірити, чи дорівнює `true` вираз `(= 1 2)`.

Давайте випробуємо наші тести. Спочатку запустимо компілятор:

```bash
$ ./scripts/watch
Building ...
Copying jar:file:/home/niwi/.m2/repository/org/clojure/_clojurescript_/1.9.36/_clojurescript_-1.9.36.jar!/cljs/core.cljs to out/cljs/core.cljs
Reading analysis cache for jar:file:/home/niwi/.m2/repository/org/clojure/_clojurescript_/1.9.36/_clojurescript_-1.9.36.jar!/cljs/core.cljs
Compiling out/cljs/core.cljs
... done. Elapsed 3.862126827 seconds
Watching paths: /home/niwi/cljsbook/playground/mytestingapp/test, /home/niwi/cljsbook/playground/mytestingapp/src
```

Післа завершення процесу компіляції спробуйте виконати скомпільований файл у `nodejs`:

```bash
$ node out/mytestingapp.js

Testing mytestingapp.core-tests

FAIL in (my-first-test) (cljs/test.js:374:14)
expected: (= 1 2)
  actual: (not (= 1 2))

Ran 1 tests containing 1 assertions.
1 failures, 0 errors.
```

Ви можете побачити, що в консоль виведене повідомлення про негативний результат перевірки, як і очікувалось. Щоб виправити тест, замініть `=` на `not=` і знову виконайте скомпільований файл:

```bash
$ node out/mytestingapp.js

Testing mytestingapp.core-tests

Ran 1 tests containing 1 assertions.
0 failures, 0 errors.
```

Звісно ми можемо писати такі прості перевірки, але вони не дуже корисні. Давайте протестуємо код застосунку. Створимо функцію, що перевіряє, чи є рік високосним. Запишіть цей код у файл `src/mytestingapp/core.clj`:

```clojure
(defn leap?
  [year]
  (and (zero? (js-mod year 4))
       (pos? (js-mod year 100))
       (pos? (js-mod year 400))))
```

Далі, у файлі `core_tests.cljs` створіть тести для перевірки нашої функції `leap?`:

```clojure
(ns mytestingapp.core-tests
  (:require [cljs.test :as t]
            [mytestingapp.core :as core]))

(enable-console-print!)

(t/deftest my-first-test
  (t/is (not= 1 2)))

(t/deftest my-second-test
  (t/is (core/leap? 1980))
  (t/is (not (core/leap? 1981))))

(set! *main-cli-fn* #(t/run-tests))
```

Знову запустіть скомпільований файл. Попередній тест проходить перевірку, як і раніше, а за ним виконуються два нових тести, що також проходять перевірку.

#### Асинхронне тестування

Однією з особливостей _ClojureScript_ є те, що код на _ClojureScript_ виконується в асинхронному, однопоточному оточенні, як і JavaScript, що має свої особливості.

В асинхронному оточенні ми повинні мати можливість тестувати асинхронні функції. Для цього в бібліотеці для тестування в _ClojureScript_ існує макрос `async`, що спрощує тестування асинхронного коду.

Для початку нам треба створити нову функцію, що виконується асинхронно. Створимо предикату  `async-leap?`, що буде виконувати те саме завадання, але асинхронно, повертаючи результат у функцію зворотнього виклику:

```clojure
(defn async-leap?
  [year callback]
  (js/setImmediate
   (fn []
     (let [result (or (zero? (js-mod year 400))
                      (and (pos? (js-mod year 100))
                           (zero? (js-mod year 4))))]
       (callback result)))))
```

Функція `setImmediate` у JavaScript використовується для емуляціі асинхронної задачі. Функція зворотнього виклику викликається з результатом обчислення предикати.

Для тестування такої функції використаємо макрос `async`:

```clojure
(t/deftest my-async-test
  (t/async done
    (core/async-leap? 1980 (fn [result]
                             (t/is (true? result))
                             (done)))))
```

Функція `done`, яку надає макрос `async`, повинна бути викликана по закінченню обчислення асинхронного коду і всіх перевірок.

Дуже важливо викликати `done` лише один раз. Не викликавши її зовсім або викликавши більше одного разу може привести до несподіваної поведінки.


#### Фікстури

Розділ ще не написано.

#### Інтеграція з СІ

Більшість інструментів для неперервної інтеграції очікують скрипт, що виконує тести і повертає стандартний код виходу з програми. Але в _ClojureScript_ неможливо повернути необхідний код виходу без спеціальної конфігурації. В JavaScript немає для цього універсального API, яке можна було б використовувати в _ClojureScript_.

Для вирішення цієї проблеми в бібліотеці для тестування є можливість конфігурації функції, що буде виконана після закінчення обчислення тестів. Саме тут слід вказати код для необхідної нам платформи (Node.js), що поверне правильний код виходу з програми: `0` якщо всі тести пройшли, `1` — якщо ні.

Додайте цей код в кінець файлу `core_tests.cljs`:

```clojure
(defmethod t/report [::t/default :end-run-tests]
  [m]
  (if (t/successful? m)
    (set! (.-exitCode js/process) 0)
    (set! (.-exitCode js/process) 1)))
```

Перевірте код виходу після виконання тестів:

```bash
$ node out/mytestingapp.js
$ echo $?
```

Звичайно це буде працювати лише для `Node.js`. Якщо ви виконуєте скрипт у іншому оточенні, вам потрібно знати, як правильно втсановлювати код виходу в цьому оточенні.
