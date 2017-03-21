### Reader Conditionals

This language feature allows different dialects of Clojure to share common code that
is mostly platform independent but need some platform dependent code.

To use reader conditionals, all you need is to rename your source file with
`.cljs` extension to one with `.cljc`, because reader conditionals only work if
they are placed in files with `.cljc` extension.

#### Standard (`#?`)

There are two types of reader conditionals, standard and splicing. The standard
reader conditional behaves similarly to a traditional cond and the syntax looks
like this:

```clojure
(defn parse-int
  [v]
  #?(:clj  (Integer/parseInt s)
     :cljs (js/parseInt s)))
```

As you can observe, `#?` reading macro looks very similar to cond, the difference is
that the condition is just a keyword that identifies the platform, where `:cljs` is
for _ClojureScript_ and `:clj` is for _Clojure_. The advantage of this approach, is
that it is evaluated at compile time so no runtime performance overhead exists for
using this.

#### Splicing (`#?@`)

The splicing reader conditional works in the same way as the standard and allows
splice lists into the containing form. The `#?@` reader macro is used for that
and the code looks like this:

```clojure
(defn make-list
  []
  (list #?@(:clj  [5 6 7 8]
            :cljs [1 2 3 4])))

;; On ClojureScript
(make-list)
;; => (1 2 3 4)
```

The _ClojureScript_ compiler will read that code as this:

```clojure
(defn make-list
  []
  (list 1 2 3 4))
```

The splicing reader conditional can't be used to splice multiple top level forms,
so the following code is ilegal:

```clojure
#?@(:cljs [(defn func-a [] :a)
           (defn func-b [] :b)])
;; => #error "Reader conditional splicing not allowed at the top level."
```

If you need so, you can use multiple forms or just use `do` block for group
multiple forms together:

```clojure
#?(:cljs (defn func-a [] :a))
#?(:cljs (defn func-b [] :b))

;; Or

#?(:cljs
   (do
     (defn func-a [] :a)
     (defn func-b [] :b)))
```


#### More readings

- http://clojure.org/guides/reader_conditionals
- https://danielcompton.net/2015/06/10/clojure-reader-conditionals-by-example
- https://github.com/funcool/cuerdas (example small project that uses
  reader conditionals)
