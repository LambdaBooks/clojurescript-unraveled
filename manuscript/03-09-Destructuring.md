### Destructuring

Destructuring, as its name suggests, is a way of taking apart structured data such
as collections and focusing on individual parts of them. ClojureScript offers a
concise syntax for destructuring both indexed sequences and associative data
structures that can be used any place where bindings are declared.

Let's see an example of what destructuring is useful for that will help us
understand the previous statements better. Imagine that you have a sequence but are
only interested in the first and third item. You could get a reference to them
easily with the `nth` function:

```clojure
(let [v [0 1 2]
      fst (nth v 0)
      thrd (nth v 2)]
  [thrd fst])
;; => [2 0]
```

However, the previous code is overly verbose. Destructuring lets us extract values
of indexed sequences more succintly using a vector on the left-hand side of a
binding:

```clojure
(let [[fst _ thrd] [0 1 2]]
  [thrd fst])
;; => [2 0]
```

In the above example, `[fst _ thrd]` is a destructuring form. It is represented as a
vector and used for binding indexed values to the symbols `fst` and `thrd`,
corresponding to the index `0` and `2`, respectively. The `_` symbol is used as a
placeholder for indexes we are not interested in — in this case `1`.

Note that destructuring is not limited to the `let` binding form; it works in almost
every place where we bind values to symbols such as in the `for` and `doseq` special
forms or in function arguments. We can write a function that takes a pair and swaps
its positions very concisely using destructuring syntax in function arguments:

```clojure
(defn swap-pair [[fst snd]]
  [snd fst])

(swap-pair [1 2])
;; => [2 1]

(swap-pair '(3 4))
;; => [4 3]
```

Positional destructuring with vectors is quite handy for taking indexed values out
of sequences, but sometimes we don't want to discard the rest of the elements in the
sequence when destructuring.  Similarly to how `&` is used for accepting variadic
function arguments, the ampersand can be used inside a vector destructuring form for
grouping together the rest of a sequence:

```clojure
(let [[fst snd & more] (range 10)]
  {:first fst
   :snd snd
   :rest more})
;; => {:first 0, :snd 1, :rest (2 3 4 5 6 7 8 9)}
```

Notice how the value in the `0` index got bound to `fst`, the value in the `1` index
got bound to `snd`, and the sequence of elements from `2` onwards got bound to the
`more` symbol.

We may still be interested in a data structure as a whole even when we are
destructuring it. This can be achieved with the `:as` keyword. If used inside a
destructuring form, the original data structure is bound to the symbol following
that keyword:

```clojure
(let [[fst snd & more :as original] (range 10)]
  {:first fst
   :snd snd
   :rest more
   :original original})
;; => {:first 0, :snd 1, :rest (2 3 4 5 6 7 8 9), :original (0 1 2 3 4 5 6 7 8 9)}
```

Not only can indexed sequences be destructured, but associative data can also be
destructured. Its destructuring binding form is represented as a map instead of a
vector, where the keys are the symbols we want to bind values to and the values are
the keys that we want to look up in the associative data structure. Let's see an
example:

```clojure
(let [{language :language} {:language "ClojureScript"}]
  language)
;; => "ClojureScript"
```

In the above example, we are extracting the value associated with the `:language`
key and binding it to the `language` symbol. When looking up keys that are not
present, the symbol will get bound to `nil`:

```clojure
(let [{name :name} {:language "ClojureScript"}]
  name)
;; => nil
```

Associative destructuring lets us give default values to bindings which will be used
if the key isn't found in the data structure we are taking apart. A map following
the `:or` keyword is used for default values as the following examples show:

```clojure
(let [{name :name :or {name "Anonymous"}} {:language "ClojureScript"}]
  name)
;; => "Anonymous"

(let [{name :name :or {name "Anonymous"}} {:name "Cirilla"}]
  name)
;; => "Cirilla"
```

Associative destructuring also supports binding the original data structure to a
symbol placed after the `:as` keyword:

```clojure
(let [{name :name :as person} {:name "Cirilla" :age 49}]
  [name person])
;; => ["Cirilla" {:name "Cirilla" :age 49}]
```

Keywords aren't the only things that can be the keys of associative data structures.
Numbers, strings, symbols and many other data structures can be used as keys, so we
can destructure using those, too. Note that we need to quote the symbols to prevent
them from being resolved as a var lookup:

```clojure
(let [{one 1} {0 "zero" 1 "one"}]
  one)
;; => "one"

(let [{name "name"} {"name" "Cirilla"}]
  name)
;; => "Cirilla"

(let [{lang 'language} {'language "ClojureScript"}]
  lang)
;; => "ClojureScript"
```

Since the values corresponding to keys are usually bound to their equivalent symbol
representation (for example, when binding the value of `:language` to the symbol
`language`) and keys are usually keywords, strings, or symbols, ClojureScript offers
shorthand syntax for these cases.

We'll show examples of all of these, starting with destructuring keywords using
`:keys`:

```clojure
(let [{:keys [name surname]} {:name "Cirilla" :surname "Fiona"}]
  [name surname])
;; => ["Cirilla" "Fiona"]
```

As you can see in the example, if we use the `:keys` keyword and associate it with a
vector of symbols in a binding form, the values corresponding to the keywordized
version of the symbols will be bound to them. The `{:keys [name surname]}`
destructuring is equivalent to `{name :name surname :surname}`, only shorter.

The string and symbol shorthand syntax works exactly like `:keys`, but using the
`:strs` and `:syms` keywords respectively:

```clojure
(let [{:strs [name surname]} {"name" "Cirilla" "surname" "Fiona"}]
  [name surname])
;; => ["Cirilla" "Fiona"]

(let [{:syms [name surname]} {'name "Cirilla" 'surname "Fiona"}]
  [name surname])
;; => ["Cirilla" "Fiona"]
```

An interesting property of destructuring is that we can nest destructuring forms
arbitrarily, which makes code that accesses nested data on a collection very easy to
understand, as it mimics the collection's structure:

```clojure
(let [{[fst snd] :languages} {:languages ["ClojureScript" "Clojure"]}]
  [snd fst])
;; => ["Clojure" "ClojureScript"]
```
