
### Threading Macros

Threading macros, also known as arrow functions, enables one to write more readable code
when multiple nested function calls are performed.

Imagine you have `(f (g (h x)))` where a function `f` receives as its first parameter the
result of executing function `g`, repeated multiple times. With the most basic `->` threading
macro you can convert that into `(\-> x (h) (g) (f))` which is easier to read.

The result is syntactic sugar, because the arrow functions are defined as macros
and it does not imply any runtime performance. The `(\-> x (h) (g) (f))` is
automatically converted to (f (g (h x))) at compile time.

Take note that the parenthesis on `h`, `g` and `f` are optional, and can be ommited:
`(f (g (h x)))` is the same as `(\-> x h g f)`.


#### The thread-first macro (`\->`)

This is called *thread first* because it threads the first argument throught the
different expressions as first arguments.

Using a more concrete example, this is how the code looks without using threading
macros:

```clojure
(def book {:name "Lady of the Lake"
           :readers 0})

(update (assoc book :age 1999) :readers inc)
;; => {:name "Lady of the lake" :age 1999 :readers 1}
```

We can rewrite that code to use the `\->` threading macro:

```clojure
(-> book
    (assoc :age 1999)
    (update :readers inc))
;; => {:name "Lady of the lake" :age 1999 :readers 1}
```

This threading macro is especially useful for transforming data structures, because
_ClojureScript_ (and _Clojure_) functions for data structures transformations
consistently uses the first argument for receive the data structure.


#### The thread-last macro (`\->>`)

The main difference between the thread-last and thread-first macros is that instead
of threading the first argument given as the first argument on the following expresions,
it threads it as the last argument.

Let's look at an example:

```clojure
(def numbers [1 2 3 4 5 6 7 8 9 0])

(take 2 (filter odd? (map inc numbers)))
;; => (3 5)
```

The same code written using `\->>` threading macro:

```clojure
(->> numbers
     (map inc)
     (filter odd?)
     (take 2))
;; => (3 5)
```

This threading macro is especially useful for transforming sequences or collections
of data because _ClojureScript_ functions that work with sequences and collections
consistently use the last argument position to receive them.


#### The thread-as macro (`as\->`)

Finally, there are cases where neither `\->` nor `\->>` are applicable. In these
cases, you’ll need to use `as\->`, the more flexible alternative, that allows you to
thread into any argument position, not just the first or last.

It expects two fixed arguments and an arbitrary number of expressions. As with
`\->`, the first argument is a value to be threaded through the following forms. The
second argument is the name of a binding. In each of the subsequent forms, the bound
name can be used for the prior expression's result.

Let's see an example:

```clojure
(as-> numbers $
  (map inc $)
  (filter odd? $)
  (first $)
  (hash-map :result $ :id 1))
;; => {:result 3 :id 1}
```


#### The thread-some macros (`some\->` and `some\->>`)

Two of the more specialized threading macros that _ClojureScript_ comes with. They work
in the same way as their analagous `\->` and `\->>` macros with the additional
support for short-circuiting the expression if one of the expresions evaluates
to `nil`.

Let's see another example:

```clojure
(some-> (rand-nth [1 nil])
        (inc))
;; => 2

(some-> (rand-nth [1 nil])
        (inc))
;; => nil
```

This is an easy way avoid null pointer exceptions.


#### The thread-cond macros (`cond\->` and `cond\->>`)

The `cond\->` and `cond\->>` macros are analgous to `\->` and `\->>` that offers
the ability to conditionally skip some steps from the pipeline. Let see an example:

```clojure
(defn describe-number
  [n]
  (cond-> []
    (odd? n) (conj "odd")
    (even? n) (conj "even")
    (zero? n) (conj "zero")
    (pos? n) (conj "positive")))

(describe-number 3)
;; => ["odd" "positive"]

(describe-number 4)
;; => ["even" "positive"]
```

The value threading only happens when the corresponding condition evaluates to
logical true.

#### Additional Readings

- http://www.spacjer.com/blog/2015/11/09/lesser-known-clojure-variants-of-threading-macro/
- http://clojure.org/guides/threading_macros
