### Керування залежностями

До цього момента для компіляції коду у JavaScript ми користувалися готовим набором інструментів _ClojureScript_. Це мінімум, необхідний для компіляції та для розуміння основ роботи компілятора. Але для більш-менш великих проектів нам знадобиться більш потужний інструмент для зборки та керування залежностями від сторонніх бібліотек.

Решта цього розділу присвячена поясненню роботи інструмента *Leiningen*, який фактично є стандартним інструментом для зборки проектів на _ClojureScript_ та для керування залежностями. Інший інструмент, *boot*, також набирає популярність, але в цій книзі ми обмежимося розглядом *Leiningen*. 


#### Інсталяція leiningen

Процес істаляції _leiningen_ доволі простий. Виконайте наступні кроки:

```bash
mkdir ~/bin
cd ~/bin
wget https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein
chmod a+x ./lein
export PATH=$PATH:~/bin
```

Переконайтеся, що директорія `~/bin` завжди присутня у `path`. Можна додати рядок, що починається з `export`, до файлу `~/.bashrc` (для оболонки bash).

Відкрийте новий термінал та виконайте команду `lein version`. Результат має бути приблизно такий:

```bash
$ lein version
Leiningen 2.5.1 on Java 1.8.0_45 OpenJDK 64-Bit Server VM
```

ЗАУВАЖЕННЯ. Ми виходимо з того, що наші читачі користуються юніксоподібними системами, як Linux або BSD. Користувачам  Windows рекомендуємо відвідати сторінку [Leiningen](http://leiningen.org/) та скористуватися наданими там інструкціями. Також на іказаному веб-сайті можна знайти код скрипту leiningen для Linux, Mac OS X, або BSD тощо.


#### Перший проект

Найкарщий спосіб продемонструвати роботу інструменту - використати його для створення некладного проекту. Ми напишемо застосунок, який визначає, чи є певний рік високосним. Для початку ми скористуємося шаблоном *mies* для leiningen.

ЗАУВАЖЕННЯ: Шаблони - це функція leiningen, що дозволяє створювати певну вихідну структуру проекту. Спільнота розробників на  clojure створила багато зручних шаблонів. В цьому випадку ми скористаємося шаблоном *mies*, який був започаткований розробником стандартної бібліотеки clojurescript. Рекомендуємо звернутися до документації leiningen по подробиці.

Почнемо зі створення основи проекту:

```bash
$ lein new mies leapyears
$ cd leapyears # move into newly created project directory
```

The project has the following structure:
Проект має таку структуру:

```
leapyears
├── index.html
├── project.clj
├── README.md
├── scripts
│   ├── build
│   ├── release
│   ├── watch
│   ├── repl
│   └── brepl
└── src
    └── leapyears
        └── core.cljs
```

Файл `project.clj` містить інформацію, яка потрібна Leiningen для завантаження залежностей та зборки проекту. Поки що домовимося, що зміст цього файлу саме такий, яким має бути. 

Відкрийте файл `index.html` та додайте наступний код напочатку елементу body:

```html
<section class="viewport">
  <div id="result">
    ----
  </div>
  <form action="" method="">
    <label for="year">Enter a year</label>
    <input id="year" name="year" />
  </form>
</section>
```

Наступний крок - зробити форму інтерактивною. Додайте наступний код у файл `src/leapyears/core.cljs`:

```clojure
(ns leapyears.core
  (:require [goog.dom :as dom]
            [goog.events :as events]
            [cljs.reader :refer (read-string)]))

(enable-console-print!)

(def input (dom/getElement "year"))
(def result (dom/getElement "result"))

(defn leap?
  [year]
  (or (zero? (js-mod year 400))
      (and (pos? (js-mod year 100))
           (zero? (js-mod year 4)))))

(defn on-change
  [event]
  (let [target (.-target event)
        value (read-string (.-value target))]
    (if (leap? value)
      (set! (.-innerHTML result) "YES")
      (set! (.-innerHTML result) "NO"))))

(events/listen input "keyup" on-change)
```

Тепер скомпілюємо код наступною командою:

```bash
$ ./scripts/watch
```

За лаштунками скрипт `watch` використовує інструмент для зборки `lein` для виконання команд подібним чином до команди `java` з минулого розділу:

```bash
rlwrap lein trampoline run -m clojure.main scripts/watch.clj
```

УВАГА: у вашій операційній системі має бути встановлений `rlwrap`.

Насамкінець відркийте файл `index.html` у браузері. Якщо ввести рік у форму, на екрані бути виведено, чи є вказаний рік високосним.

Як ви могли помітити, у директорії зі скриптами присутні інші файли, зокрема `build` та `release`. Це ті скрипти, про які ми згадували у попередньому розділі, але поки що ми розглядатимемо лише `watch`.


#### Керування залежностями

Справжня цінність використання Leiningen у процесі компіляції ClojureScript полягає в автоматизації запиту залежностей. Це значно спрощує процес у порівняння з ручним доданням бібліотек.

Залежності оголошуються разом з іншими параметрами проекту у файлі `project.clj` наступним чином (приклад з шаблону *mies*):

```clojure
(defproject leapyears "0.1.0-SNAPSHOT"
  :description "FIXME: write this!"
  :url "http://example.com/FIXME"
  :dependencies [[org.clojure/clojure "1.8.0"]
                 [org.clojure/clojurescript "1.9.36"]
                 [org.clojure/data.json "0.2.6"]]
  :jvm-opts ^:replace ["-Xmx1g" "-server"]
  :node-dependencies [[source-map-support "0.3.2"]]
  :plugins [[lein-npm "0.5.0"]]
  :source-paths ["src" "target/classes"]
  :clean-targets ["out" "release"]
  :target-path "target")
```

Далі наведемо коротке пояснення властивосей, що мають значення у контексті ClojureScript:

* `:dependencies`: вектор залежностей вашого проекту
* `:clean-targets`: вектор шляхів, які має видалити `lein clean`.

Залежності у ClojureScript пакуються у файли типу `jar`. Якщо ви писали на Clojure або на іншій мові, що працює з JVM, файли `jar` вам мають бути знайомі. Якщо це не так, не переймайтеся: файл `jar` схожий на звичайний zip-файл, який містить `project.clj` для бібліотек, певні метадані та сирцеві файли на ClojureScript. Процес пакування ми пояснимо в іншому розділі.

Пакети на Clojure часто публікують за посиланням  [Clojars](http://clojars.org). Також багато бібліотек можна знайти на [ClojureScript Wiki](https://github.com/clojure/clojurescript/wiki#libraries)
