
### Working with the REPL


#### Introduction

Although you can create a source file and compile it every time you want to try
something out in ClojureScript, it's easier to use the REPL. REPL stands for:

* Read - get input from the keyboard
* Evaluate the input
* Print the result
* Loop back for more input

In other words, the REPL lets you try out ClojureScript concepts and get immediate
feedback.

_ClojureScript_ comes with support for executing the REPL in different execution
environments, each of which has its own advantages and disadvantages. For example,
you can run a REPL in nodejs but in that environment you don't have any access to the
DOM.  Which REPL environment is best for you depends on your specific needs and
requirements.


#### Nashorn REPL

The Nashorn REPL is the easiest and perhaps most painless REPL environment because it
does not require any special stuff, just the JVM (JDK 8) that you have used in
previous examples for running the _ClojureScript_ compiler.

Let’s start creating the _repl.clj_ file with the following content:

```clojure
(require '[cljs.repl]
         '[cljs.repl.nashorn])

(cljs.repl/repl
 (cljs.repl.nashorn/repl-env)
 :output-dir "out"
 :cache-analysis true)
```

Then, execute the following command to get the REPL up and running:

```bash
$ java -cp cljs.jar:src clojure.main repl.clj
To quit, type: :cljs/quit
cljs.user=> (+ 1 2)
3
```

You may have noticed that the REPL does not have support for history and other
shell-like facilities. This is because the default REPL does not comes with
"readline" support. But this problem can be solved using a simple tool named `rlwrap`
which you should be able to find find with the package manager of your operating
system (e.g. for Ubuntu, type `sudo apt install -y rlwrap` to install).

The `rlwrap` tool gives the REPL "readline" capability, and will allow you to have
command history, code navigation, and other shell-like utilities that will make your
REPL experience much more pleasant. To use it, just prepend it to the previous
command that we used to start the REPL:

```bash
$ rlwrap java -cp cljs.jar:src clojure.main repl.clj
To quit, type: :cljs/quit
cljs.user=> (+ 1 2)
3
```


#### Node.js REPL

You must, of course, have nodejs installed on your system to use this REPL.

You may be wondering why we might want a nodejs REPL, when we already have the
nashorn REPL available which doesn't have any external dependencies. The answer is
very simple: nodejs is the most used JavaScript execution environment on the backend,
and it has a great number of community packages built around it.

The good news is that starting a nodejs REPL is very easy once you have it installed
in your system. Start writing this content to a new `repl.clj` file:

```clojure
(require '[cljs.repl]
         '[cljs.repl.node])

(cljs.repl/repl
 (cljs.repl.node/repl-env)
 :output-dir "out"
 :cache-analysis true)
````

And start the REPL like you have done it previously with nashorn REPL:

```clojure
$ rlwrap java -cp cljs.jar:src clojure.main repl.clj
To quit, type: :cljs/quit
cljs.user=> (+ 1 2)
3
```


#### Browser REPL

This REPL is the most laborious to get up and running. This is because it uses a
browser for its execution environment and it has additional requirements.

Let’s start by creating a file named `brepl.clj` with the following content:

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

This script builds the source, just as we did earlier, and then starts the REPL.

But the browser REPL also requires that some code be executed in the browser before
the REPL gets working. To do that, just re-create the application structure very
similar to the one that we have used in previous sections:

```clojure
mkdir -p src/myapp
touch src/myapp/core.cljs
```

Then, write new content to the `src/myapp/core.cljs` file:

```clojure
(ns myapp.core
 (:require [clojure.browser.repl :as repl]))

(defonce conn
  (repl/connect "http://localhost:9000/repl"))

(enable-console-print!)

(println "Hello, world!")
```

And finally, create the missing _index.html_ file that is going to be used as the
entry point for running the browser side code of the REPL:

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

Well, that was a lot of setup! But trust us, it’s all worth it when you see it in
action.  To do that, just execute the `brepl.clj` in the same way that we have done
it in previous examples:

```bash
$ rlwrap java -cp cljs.jar:src clojure.main brepl.clj
Compiling client js ...
Waiting for browser to connect ...
```

And finally, open your favourite browser and go to http://localhost:9000/. Once the
page is loaded (the page will be blank), switch back to the console where you have
run the REPL and you will see that it is up and running:

```bash
[...]
To quit, type: :cljs/quit
cljs.user=> (+ 14 28)
42
```

One of the big advantages of the browser REPL is that you have access to everything
in the browser environment. For example, type `(js/alert "hello world")` in the
REPL. This will cause the browser to display an alert box. Nice!