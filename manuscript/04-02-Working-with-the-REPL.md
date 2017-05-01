
### Робота з REPL


#### Вступ

Коли потрібно спробувати щось на ClojureScript, можна щоразу створювати та компілювати необхідні сирцеві файли, але для невеликих обчислень зручніше використовувати REPL. Абревіатура REPL розшифровується наступним чином:

* Read - отримання вводу з клавіатури
* Evaluate - обчислення вводу
* Print - друк результату
* Loop - повтор циклу для отримання нового вводу

Іншими словами, REPL дозволяє випробовувати концепції ClojureScript та негайно отримувати результат.

У мові _ClojureScript_ передбачена підтримка виконання REPL у різних середовищах, кожне з яких має свої переваги та недоліки. Наприклад, можливо запустити REPL у nodejs, але це середовище не надає доступу до DOM. Вірний вибір середовища для виконання REPL залежить від конкретних потреб та вимог.


#### Nashorn REPL

Nashorn REPL - найпростіше та, мабуть, найбільш безпроблемне середовище REPL завдяки тому, що для його роботи потрібен лише JVM (JDK 8), яким ви користувалися у минулих прикладах для запуску компілятора _ClojureScript_.

Почнемо зі створення файлу  _repl.clj_ із наступним змістом:

```clojure
(require '[cljs.repl]
         '[cljs.repl.nashorn])

(cljs.repl/repl
 (cljs.repl.nashorn/repl-env)
 :output-dir "out"
 :cache-analysis true)
```

Виконайте наступну команду для запуску REPL:

```bash
$ java -cp cljs.jar:src clojure.main repl.clj
To quit, type: :cljs/quit
cljs.user=> (+ 1 2)
3
```

Ви могли помітити, що REPL не підтримує збереження історії та деяких інших можливостей програмної оболонки. Це пояснюється тим, що звичайний REPL не має підтримки функції "readline". Цю проблему легко подолати використанням інструменту `rlwrap`, який ви можете знайти за допомогою встановленого на вашій системі менеджеру пакетів (наприклад, відповідна команда для встановлення `rlwrap` на Ubuntu - це `sudo apt install -y rlwrap`).

Інструмент `rlwrap` надає REPL можливість для виконання "readline" та дозволяє зберігати історію команд, переміщатися по коду та користуватися деякими іншими можливостями програмної оболонки, завдяки яким робота у REPL буде більш приємною. Для використання `rlwrap` вкажіть його на початку команди запуску REPL:

```bash
$ rlwrap java -cp cljs.jar:src clojure.main repl.clj
To quit, type: :cljs/quit
cljs.user=> (+ 1 2)
3
```


#### Node.js REPL

Звичайно, для використання цього REPL на вашій операційній системі має бути встановлений nodejs.

Можливо, у вас виникло питання, навіщо використовувати nodejs REPL, якщо ми вже маємо nashorn REPL, який не має зовнішніх залежностей. Відповідь дуже проста: nodejs найбільш розповсюджене середовище для виконання JavaScript-коду  на сервері.

Зручним є те, що для запуску nodejs REPL достатньо просто встановити його на свою операційну систему. Почніть з написання наступного коду у файлі `repl.clj`:

```clojure
(require '[cljs.repl]
         '[cljs.repl.node])

(cljs.repl/repl
 (cljs.repl.node/repl-env)
 :output-dir "out"
 :cache-analysis true)
```

Запустіть REPL, як робили раніше із nashorn REPL:

```clojure
$ rlwrap java -cp cljs.jar:src clojure.main repl.clj
To quit, type: :cljs/quit
cljs.user=> (+ 1 2)
3
```


#### Browser REPL

Цей REPL вимагає найбільших зусиль для запуску. Це пов'язано з використанням браузера як середовища виконання та приводить до появи додаткових вимог.

Створіть файл під назвою `brepl.clj` із наступним змістом:

```clojure
(require
  '[cljs.build.api :as b]
  '[cljs.repl :as repl]
  '[cljs.repl.browser :as browser])

(b/build "src"
 {:output-to "main.js"
  :output-dir "out/"
  :source-map true
  :main 'myapp.core
  :verbose true
  :optimizations :none})

(repl/repl (browser/repl-env)
  :output-dir "out")
```

Скрипт виконує компіляцію сирцевих файлів, як ми робили раніше, після чого запускає REPL.

Але browser REPL потребує виконання певного коду у середовищі браузера перед початком роботи REPL. Для цього слід відтворити структуру застосунку, який ми використовували у попередніх розділах:

```clojure
mkdir -p src/myapp
touch src/myapp/core.cljs
```

Заповніть файл `src/myapp/core.cljs` наступним кодом:

```clojure
(ns myapp.core
 (:require [clojure.browser.repl :as repl]))

(defonce conn
  (repl/connect "http://localhost:9000/repl"))

(enable-console-print!)

(println "Hello, world!")
```

Насамкінець створіть файл _index.html_, який буде точкою входу для запуску коду REPL на стороні браузера: 

```html
<!DOCTYPE html>
<html>
  <header>
    <meta charset="utf-8" />
    <title>Hello World from ClojureScript</title>
  </header>
  <body>
    <script src="main.js"></script>
  </body>
</html>
```

Так, налаштувань насправді багато, але результат переконає вас у виправданності докладених зусиль. Виконайте файл `brepl.clj` так, як робили у попередніх прикладах:

```bash
$ rlwrap java -cp cljs.jar:src clojure.main brepl.clj
Compiling client js ...
Waiting for browser to connect ...
```

І, нарешті, відкрийте ваш улюблений браузер та перейдіть за посиланням http://localhost:9000/. Після завантаження сторінки (сторінка буде порожня) поверніться до консолі, у якій запущено REPL, і переконайтеся, що він продовжує роботу.

```bash
[...]
To quit, type: :cljs/quit
cljs.user=> (+ 14 28)
42
```
Одна з найбільш суттєвих переваг використання REPL у браузері - це доступ до відповідного оточення. Для прикладу наберіть `(js/alert "hello world")` у REPL, і браузер відобразить вікно повідомлення. Чудово!