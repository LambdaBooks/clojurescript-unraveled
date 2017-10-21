### External dependencies

In some circumstances you may found yourself that you need some library but that does not exists in _ClojureScript_ but it is already implemented in javascript and you want to use it on your project.

There are many ways that you can do it mainly depending on the library that you want to include. Let see some ways.


#### Closure Module compatible library

If you have a library that is just written to be compatible with google closure module system and you want to include it on your project you should just put it in the source (classpath) and access it like any other clojure namespace.

This is the most simplest case, because google closure modules are directly compatible and you can mix your clojure code with javascript code written using google closure module system without any additional steps.

Let play with it creating new project using *mies* template:

```shell
lein new mies myextmods
cd myextmods
```

Create a simple google closure module for experiment:

.src/myextmods/myclosuremodule.js
```javascript
goog.provide("myextmods.myclosuremodule");

goog.scope(function() {
  var module = myextmods.myclosuremodule;
  module.get_greetings = function() {
    return "Hello from google closure module.";
  };
});
```

Now, open the repl, require the namespace and try to use the exposed function:

```clojure
(require '[myextmods.myclosuremodule :as cm])
(cm/get_greetings)
;; => "Hello from google closure module."
```

NOTE: you can open the nodejs repl just executing `./scripts/repl` on the root of the repository.


#### CommonJS modules compatible libraries

Due to the Node.JS popularity the commonjs used in node is today the most used module format for javascript libraries, independently if they will be used in server side development using nodejs or using browser side applications.

Let's play with that. Start creating a simple file using commonjs module format (pretty analgous to the previous example using google closure modules):

.src/myextmods/mycommonjsmodule.js
```javascript
function getGreetings() {
  return "Hello from commonjs module.";
}

exports.getGreetings = getGreetings;
```

Later, in order to use that simple pet library you should indicate to the _ClojureScript_ compiler the path to that file and the used module type with `:foreign-libs` attribute.

Open `scripts/repl.clj` and modify it to somethig like this:

```clojure
(require
  '[cljs.repl :as repl]
  '[cljs.repl.node :as node])

(repl/repl
 (node/repl-env)
 :language-in  :ecmascript5
 :language-out :ecmascript5
 :foreign-libs [{:file "myextmods/mycommonjsmodule.js"
                 :provides ["myextmods.mycommonjsmodule"]
                 :module-type :commonjs}]
 :output-dir "out"
 :cache-analysis false)
```

NOTE: Although the direct path is used to point to this pet library you can specify a full URI to remote resource and it will be automatically downloaded.

Now, let's try to play with moment within the repl (executing the `./scripts/repl` script that uses the previously modified `./scripts/repl.clj` file):

```clojure
(require '[myextmods.mycommonjsmodule :as cm])
(cm/getGreetings)
;; => "Hello from commonjs module."
```


#### Legacy, module-less (global scope) libraries

Although today is very common have libraries packaged using some kind of modules, there are also a great amount of libraries that just exposes a global objects and does not uses any kind of modules; and you may want to use them from _ClojureScript_.

In order to  use a library that  exposes a global object, you  should follow similar steps  as  with  commojs  modules  with  the exception  that  you  should  omit  the `:module-type` attribute.

This will create a _synthetic_ namespace that you should require in order to be able to access to the global object through the `js/` namespace. The namespace is called _synthetic_ because it does not expose any object behind it, it just indicates to the compiler that you want that dependency.

Let's play with that. Start creating a simple file declaring just a global function: 

.src/myextmods/myglobalmodule.js
```javascript
function getGreetings() {
  return "Hello from global scope.";
}
```

Open `scripts/repl.clj` and modify it to somethig like this:

```clojure
(require
  '[cljs.repl :as repl]
  '[cljs.repl.node :as node])

(repl/repl
 (node/repl-env)
 :language-in  :ecmascript5
 :language-out :ecmascript5
 :foreign-libs [{:file "myextmods/mycommonjsmodule.js"
                 :provides ["myextmods.mycommonjsmodule"]
                 :module-type :commonjs}
                {:file "myextmods/myglobalmodule.js"
                 :provides ["myextmods.myglobalmodule"]}]
 :output-dir "out"
 :cache-analysis false)
```

And in the same way as in previous examples, let evaluate that in the repl:

```clojure
(require 'myextmods.myglobalmodule)
(js/getGreetings)
;; => "Hello from global scope."
```

