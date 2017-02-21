### Flow control

_ClojureScript_ has a very different approach to flow control than languages like
JavaScript, C, etc.

#### Branching with `if`

Let's start with a basic one: `if`. In _ClojureScript_, the `if` is an expression
and not a statement, and it has three parameters: the first one is the condition
expression, the second one is an expression that will be evaluated if the condition
expression evaluates to logical true, and the third expression will be evaluated
otherwise.

```clojure
(defn discount
  "You get 5% discount for ordering 100 or more items"
  [quantity]
  (if (>= quantity 100)
    0.05
    0))

(discount 30)
;; => 0

(discount 130)
;; => 0.05
```

The block expression `do` can be used to have multiple expressions in an `if`
branch. `do` is explained in the next section.

#### Branching with `cond`

Sometimes, the `if` expression can be slightly limiting because it does not have the
"else if" part to add more than one condition. The `cond` macro comes to the rescue.

With the `cond` expression, you can define multiple conditions:

```clojure
(defn mypos?
  [x]
  (cond
    (> x 0) "positive"
    (< x 0) "negative"
    :else "zero"))

(mypos? 0)
;; => "zero"

(mypos? -2)
;; => "negative"
```

Also, `cond` has another form, called `condp`, that works very similarly to the
simple `cond` but looks cleaner when the condition (also called a predicate) is the
same for all conditions:

```clojure
(defn translate-lang-code
  [code]
  (condp = (keyword code)
    :es "Spanish"
    :en "English"
    "Unknown"))

(translate-lang-code "en")
;; => "English"

(translate-lang-code "fr")
;; => "Unknown"
```

The line `condp = (keyword code)` means that, in each of the following lines,
_ClojureScript_ will apply the `=` function to the result of evaluating `(keyword
code)`.


#### Branching with `case`

The `case` branching expression has a similar use as our previous example with
`condp`. The main differences are that `case` always uses the `=` predicate/function
and its branching values are evaluated at compile time. This results in a more
performant form than `cond` or `condp` but has the disadvantage that the condition
value must be static.

Here is the previous example rewritten to use `case`:

```clojure
(defn translate-lang-code
  [code]
  (case code
    "es" "Spanish"
    "en" "English"
    "Unknown"))

(translate-lang-code "en")
;; => "English"

(translate-lang-code "fr")
;; => "Unknown"
```
