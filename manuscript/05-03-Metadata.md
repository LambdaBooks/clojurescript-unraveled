
### Metadata

ClojureScript symbols, vars and persistent collections support attaching metadata to them. Metadata is a map with information about the entity it's attached to. The ClojureScript compiler uses metadata for several purposes such as type hints, and the metadata system can be used by tooling, library and application developers too.

There may not be many cases in day-to-day ClojureScript programming where you need metadata, but it is a nice language feature to have and know about; it may come in handy at some point. It makes things like runtime code introspection and documentation generation very easy. You'll see why throughout this section.


#### Vars

Let's define a var and see what metadata is attached to it by default. Note that this code is executed in a REPL, and thus the metadata of a var defined in a source file may vary. We'll use the `meta` function to retrieve the metadata of the given value:

```clojure
(def answer-to-everything 42)
;; => 42

#'answer-to-everything
;; => #'cljs.user/answer-to-everyhing

(meta #'answer-to-everything)
;; => {:ns cljs.user,
;;     :name answer-to-everything,
;;     :file "NO_SOURCE_FILE",
;;     :source "answer-to-everything",
;;     :column 6,
;;     :end-column 26,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (),
;;     :doc nil,
;;     :test nil}
```

Few things to note here. First of all, `#'answer-to-everything` gives us a reference to the `Var` that holds the value of the `answer-to-everything` symbol. We see that it includes information about the namespace (`:ns`) in which it was defined, its name, file (although, since it was defined at a REPL doesn't have a source file), source, position in the file where it was defined, argument list (which only makes sense for functions), documentation string and test function.
 
Let's take a look at a function var's metadata:

```clojure
(defn add
  "A function that adds two numbers."
  [x y]
  (+ x y))

(meta #'add)
;; => {:ns cljs.user,
;;     :name add,
;;     :file "NO_SOURCE_FILE",
;;     :source "add",
;;     :column 7,
;;     :end-column 10,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (quote ([x y])),
;;     :doc "A function that adds two numbers.",
;;     :test nil}
```

We see that the argument lists are stored in the `:arglists` field of the var's metadata and its documentation in the `:doc` field. We'll now define a test function to learn about what `:test` is used for:

```clojure
(require '[cljs.test :as t])

(t/deftest i-pass
  (t/is true))

(meta #'i-pass)
;; => {:ns cljs.user,
;;     :name i-pass,
;;     :file "NO_SOURCE_FILE",
;;     :source "i-pass",
;;     :column 12,
;;     :end-column 18,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (),
;;     :doc "A function that adds two numbers.",
;;     :test #<function (){ ... }>}
```

The `:test` attribute (truncated for brevity) in the `i-pass` var's metadata is a test function. This is used by the `cljs.test` library for discovering and running tests in the namespaces you tell it to.


#### Values

We learned that vars can have metadata and what kind of metadata is added to them for consumption by the compiler and the `cljs.test` testing library. Persistent collections can have metadata too, although they don't have any by default. We can use the `with-meta` function to derive an object with the same value and type with the given metadata attached. Let's see how:

```clojure
(def map-without-metadata {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-without-metadata)
;; => nil

(def map-with-metadata (with-meta map-without-metadata
                                  {:answer-to-everything 42}))
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:answer-to-everything 42}

(= map-with-metadata
   map-without-metadata)
;; => true

(identical? map-with-metadata
            map-without-metadata)
;; => false
```

It shouldn't come as a surprise that metadata doesn't affect equality between two data structures since equality in ClojureScript is based on value. Another interesting thing is that `with-meta` creates another object of the same type and value as the given one and attaches the given metadata to it.

Another open question is what happens with metadata when deriving new values from a persistent data structure.  Let's find out:

```clojure
(def derived-map (assoc map-with-metadata :language "Clojure"))
;; => {:language "Clojure"}

(meta derived-map)
;; => {:answer-to-everything 42}
```

As you can see in the example above, metadata is preserved in derived versions of persistent data structures. There are some subtleties, though. As long as the functions that derive new data structures return collections with the same type, metadata will be preserved; this is not true if the types change due to the transformation. To ilustrate this point, let's see what happens when we derive a seq or a subvector from a vector:

```clojure
(def v (with-meta [0 1 2 3] {:foo :bar}))
;; => [0 1 2 3]

(def sv (subvec v 0 2))
;; => [0 1]

(meta sv)
;; => nil

(meta (seq v))
;; => nil
```


#### Syntax for metadata

The ClojureScript reader has syntactic support for metadata annotations, which can be written in different ways. We can prepend var definitions or collections with a caret (`^`) followed by a map for annotating it with the given metadata map:

```clojure
(def ^{:doc "The answer to Life, Universe and Everything."} answer-to-everything 42)
;; => 42

(meta #'answer-to-everything)
;; => {:ns cljs.user,
;;     :name answer-to-everything,
;;     :file "NO_SOURCE_FILE",
;;     :source "answer-to-everything",
;;     :column 6,
;;     :end-column 26,
;;     :line 1,
;;     :end-line 1,
;;     :arglists (),
;;     :doc "The answer to Life, Universe and Everything.",
;;     :test nil}

(def map-with-metadata ^{:answer-to-everything 42} {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:answer-to-everything 42}
```

Notice how the metadata given in the `answer-to-everything` var definition is merged with the var metadata.

A very common use of metadata is to set certain keys to a `true` value. For example we may want to add to a var's metadata that the variable is dynamic or a constant. For such cases, we have a shorthand notation that uses a caret followed by a keyword. Here are some examples:

```clojure
(def ^:dynamic *foo* 42)
;; => 42

(:dynamic (meta #'*foo*))
;; => true

(def ^:foo ^:bar answer 42)
;; => 42

(select-keys (meta #'answer) [:foo :bar])
;; => {:foo true, :bar true}
```

There is another shorthand notation for attaching metadata. If we use a caret followed by a symbol it will be added to the metadata map under the `:tag` key. Using tags such as `^boolean` gives the ClojureScript compiler hints about the type of expressions or function return types.

```clojure
(defn ^boolean will-it-blend? [_] true)
;; => #<function ... >

(:tag (meta #'will-it-blend?))
;; => boolean

(not ^boolean (js/isNaN js/NaN))
;; => false
```

#### Functions for working with metadata

We've learned about `meta` and `with-meta` so far but ClojureScript offers a few functions for transforming metadata. There is `vary-meta` which is similar to `with-meta` in that it derives a new object with the same type and value as the original but it doesn't take the metadata to attach directly. Instead, it takes a function to apply to the metadata of the given object to transform it for deriving
new metadata. This is how it works:

```clojure
(def map-with-metadata ^{:foo 40} {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:foo 40}

(def derived-map (vary-meta map-with-metadata update :foo + 2))
;; => {:language "ClojureScript"}

(meta derived-map)
;; => {:foo 42}
```

If instead we want to change the metadata of an existing var or value we can use `alter-meta!` for changing it by applying a function or `reset-meta!` for replacing it with another map:

```clojure
(def map-with-metadata ^{:foo 40} {:language "ClojureScript"})
;; => {:language "ClojureScript"}

(meta map-with-metadata)
;; => {:foo 40}

(alter-meta! map-with-metadata update :foo + 2)
;; => {:foo 42}

(meta map-with-metadata)
;; => {:foo 42}

(reset-meta! map-with-metadata {:foo 40})
;; => {:foo 40}

(meta map-with-metadata)
;; => {:foo 40}
```
