### The Closure Library

The Google Closure Library is a javascript library developed by Google. It has a
modular architecture, and provides cross-browser functions for DOM manipulations and
events, ajax and JSON, and other features.

The Google Closure Library is written specifically to take advantage of the Closure
Compiler (which is used internally by the _ClojureScript_ compiler).

_ClojureScript_ is built on the Google Closure Compiler and Closure Library. In fact,
_ClojureScript_ namespaces are Closure modules. This means that you can interact with
the Closure Library very easily:

```clojure
(ns yourapp.core
  (:require [goog.dom :as dom]))

(def element (dom/getElement "body"))
```

This code snippet shows how you can import the *dom* module of the Closure library
and use a function declared in that module.

Additionally, the closure library exposes "special" modules that behave like a class
or object. To use these features, you must use the `:import` directive in the `(ns
...)` form:

```clojure
(ns yourapp.core
  (:import goog.History))

(def instance (History.))
```

In a _Clojure_ program, the `:import` directive is used for host (Java) interop to
import Java classes.  If, however, you define types (classes) in _ClojureScript_, you
should use the standard `:require` directive and not the `:import` directive
