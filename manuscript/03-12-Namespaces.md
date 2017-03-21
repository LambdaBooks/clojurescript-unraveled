### Namespaces

#### Defining a namespace

The _namespace_ is ClojureScript's fundamental unit of code modularity. Namespaces
are analogous to Java packages or Ruby and Python modules and can be defined with
the `ns` macro. If you have ever looked at a little bit of ClojureScript source, you
may have noticed something like this at the beginning of the file:

```clojure
(ns myapp.core
  "Some docstring for the namespace.")

(def x "hello")
```

Namespaces are dynamic, meaning you can create one at any time. However, the
convention is to have one namespace per file. Naturally, a namespace definition is
usually at the beginning of the file, followed by an optional docstring.

Previously we have explained vars and symbols. Every var that you define will be
associated with its namespace. If you do not define a concrete namespace, then the
default one called "cljs.user" will be used:

```clojure
(def x "hello")
;; => #'cljs.user/x
```

#### Loading other namespaces

Defining a namespace and the vars in it is really easy, but it's not very useful if
we can't use symbols from other namespaces. For this purpose, the `ns` macro offers
a simple way to load other namespaces.

Observe the following:

```clojure
(ns myapp.main
  (:require myapp.core
            clojure.string))

(clojure.string/upper-case myapp.core/x)
;; => "HELLO"
```

As you can observe, we are using fully qualified names (namespace + var name) for
access to vars and functions from different namespaces.

While this will let you access other namespaces, it's also repetitive and overly
verbose. It will be especially uncomfortable if the name of a namespace is very
long. To solve that, you can use the `:as` directive to create an additional
(usually shorter) alias to the namespace.  This is how it can be done:

```clojure
(ns myapp.main
  (:require [myapp.core :as core]
            [clojure.string :as str]))

(str/upper-case core/x)
;; => "HELLO"
```

Additionally, _ClojureScript_ offers a simple way to refer to specific vars or
functions from a concrete namespace using the `:refer` directive, followed by a
sequence of symbols that will refer to vars in the namespace. Effectively, it is as
if those vars and functions are now part of your namespace, and you do not need to
qualify them at all.

```clojure
(ns myapp.main
  (:require [clojure.string :refer [upper-case]]))
(upper-case x)
;; => "HELLO"
```

And finally, you should know that everything located in the `cljs.core` namespace is
automatically loaded and you should not require it explicitly. Sometimes you may
want to declare vars that will clash with some others defined in the `cljs.core`
namespace. To do this, the `ns` macro offers another directive that allows you to
exclude specific symbols and prevent them from being automatically loaded.

Observe the following:

```clojure
(ns myapp.main
  (:refer-clojure :exclude [min]))

(defn min
  [x y]
  (if (> x y)
    y
    x))
```

The `ns` macro also has other directives for loading host classes (with `:import`)
and macros (with `:refer-macros`), but these are explained in other sections.

#### Namespaces and File Names

When you have a namespace like `myapp.core`, the code must be in a file named
_core.cljs_ inside the _myapp_ directory.  So, the preceding examples with
namespaces `myapp.core` and `myapp.main` would be found in project with a file
structure like this:

```
myapp
└── src
    └── myapp
        ├── core.cljs
        └── main.cljs
```
