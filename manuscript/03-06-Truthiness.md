### Truthiness

This is the aspect where each language has its own semantics (mostly wrongly). The
majority of languages consider empty collections, the integer 0, and other things
like this to be false.  In _ClojureScript_, unlike in other languages, only two
values are considered as false: `nil` and `false`. Everything else is treated as
logical `true`.

Jointly with the ability to implement the callable protocol (the `IFn`, explained
more in detail later), data structures like sets can be used just as predicates,
without need of additional wrapping them in a function:

```clojure
(def valid? #{1 2 3})

(filter valid? (range 1 10))
;; => (1 2 3)
```

This is because set returns the a value if it exists or nil in case contrary:

```clojure
(valid? 1)
;; => 1

(valid? 4)
;; => nil
```
