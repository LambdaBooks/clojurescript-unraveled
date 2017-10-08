### Dependency management

Until now, we have used the builtin _ClojureScript_ toolchain to compile our source files to JavaScript.  This is the minimal setup required for working with and understanding the compiler. For larger projects, however, we often want to use a more powerful build tool that can manage a project's dependencies on other libraries.

For this reason, the remainder of this chapter will explain how to use *Leiningen*, the de facto clojure build and dependency management tool, for building _ClojureScript_ projects. The *boot* build tool is also growing in popularity, but for the purposes of this book we will limit ourselves to Leiningen.


#### Installing leiningen

The installation process of leiningen is quite simple; just follow these steps:

```bash
mkdir ~/bin
cd ~/bin
wget https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein
chmod a+x ./lein
export PATH=$PATH:~/bin
```

Make sure that the `~/bin` directory is always set on your path. To make it permanent, add the line starting with `export` to your `~/.bashrc` file (assuming you are using the bash shell).

Now, open another clean terminal and execute `lein version`.  You should see something like the following:

```bash
$ lein version
Leiningen 2.5.1 on Java 1.8.0_45 OpenJDK 64-Bit Server VM
```

NOTE: We assume here that you are using a Unix-like system such as Linux or BSD. If you are a Windows user, please check the instructions on the link:http://leiningen.org/[Leiningen homepage]. You can also get the Linux/Mac OS X/BSD version of the leiningen script at the web site.


#### First project

The best way to show how a tool works is by creating a toy project with it. In this case, we will create a small application that determines if a year is a leap year or not. To start, we will use the *mies* leiningen template.

NOTE: Templates are a facility in leiningen for creating an initial project structure.  The clojure community has a great many of them. In this case we'll use the *mies* template that was started by the clojurescript core developer.  Consult the leiningen docs to learn more about templates.

Let's start creating the project layout:

```bash
$ lein new mies leapyears
$ cd leapyears # move into newly created project directory
```

The project has the following structure:

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

The `project.clj` file contains information that Leiningen uses to download dependencies and build the project. For now, just trust that everything in that file is exactly as it should be.

Open the `index.html` file and add the following content at the beginning of body:

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

The next step is adding some code to make the form interactive. Put the following code into the `src/leapyears/core.cljs`:

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

Now, compile the clojurescript code with:

```bash
$ ./scripts/watch
```

Behind the scenes, the `watch` script uses the `lein` build tool to execute a command similar to the `java` build command from the previous sections:

```bash
rlwrap lein trampoline run -m clojure.main scripts/watch.clj
```

WARNING: You must have `rlwrap` installed on your system.

Finally, open the `index.html` file in a browser.  Typing a year in the textbox should display an indication of its leap year status.

You may have noticed other files in the scripts directory, like `build` and `release`.  These are the same build scripts mentioned in the previous section, but we will stick with `watch` here.


#### Managing dependencies

The real purpose of using Leiningen for the ClojureScript compilation process is to automate the retrieval of dependencies.  This is dramatically simpler than retrieving them manually.

The dependencies, among other parameters, are declared in the `project.clj` file and have this form (from the *mies* template):

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

And here is a brief explanation of the properties relevant for ClojureScript:

* `:dependencies`: a vector of dependencies that your project needs.
* `:clean-targets`: a vector of paths that `lein clean` should delete.

The dependencies in ClojureScript are packaged using `jar` files. If you are coming from Clojure or any JVM language, `jar` files will be very familiar to you. But if you aren't familiar with them, do not worry: a .jar file is like a plain zip file that contains the `project.clj` for the library, some metadata, and the ClojureScript sources. The packaging will be explained in another section.

Clojure packages are often published on link:http://clojars.org[Clojars]. You can also find many third party libraries on the link:https://github.com/clojure/clojurescript/wiki#libraries[ClojureScript Wiki].
