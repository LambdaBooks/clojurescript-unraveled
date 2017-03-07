### Locals, Blocks, and Loops

#### Locals

_ClojureScript_ does not have the concept of variables as in ALGOL-like languages,
but it does have locals. Locals, as per usual, are immutable, and if you try to
mutate them, the compiler will throw an error.

Locals are defined with the `let` expression. The expression starts with a vector as
the first parameter followed by an arbitrary number of expressions. The first
parameter (the vector) should contain an arbitrary number of pairs that give a
_binding form_ (usually a symbol) followed by an expression whose value will be
bound to this new local for the remainder of the `let` expression.

```clojure
(let [x (inc 1)
      y (+ x 1)]
  (println "Simple message from the body of a let")
  (* x y))
;; Simple message from the body of a let
;; => 6
```

In the preceding example, the symbol `x` is bound to the value `(inc 1)`, which
comes out to 2, and the symbol `y` is bound to the sum of `x` and 1, which comes out
to 3. Given those bindings, the expressions `(println "Simple message from the body
of a let")` and `(* x y)` are evaluated.


#### Blocks

In JavaScript, braces `{` and `}` delimit a block of code that “belongs
together”. Blocks in _ClojureScript_ are created using the `do` expression and are
usually used for side effects, like printing something to the console or writing a
log in a logger.

A side effect is something that is not necessary for the return value.

The `do` expression accepts as its parameter an arbitrary number of other
expressions, but it returns the return value only from the last one:

```clojure
(do
  (println "hello world")
  (println "hola mundo")
  (* 3 5) ;; this value will not be returned; it is thrown away
  (+ 1 2))

;; hello world
;; hola mundo
;; => 3
```

The body of the `let` expression, explained in the previous section, is very similar
to the `do` expression in that it allows multiple expressions. In fact, the `let`
has an implicit `do`.


#### Loops

The functional approach of _ClojureScript_ means that it does not have standard,
well-known, statement-based loops such as `for` in JavaScript. The loops in
_ClojureScript_ are handled using recursion.  Recursion sometimes requires
additional thinking about how to model your problem in a slightly different way than
imperative languages.

Many of the common patterns for which `for` is used in other languages are achieved
through higher-order functions - functions that accept other functions as
parameters.


##### Looping with loop/recur

Let's take a look at how to express loops using recursion with the `loop` and
`recur` forms.  `loop` defines a possibly empty list of bindings (notice the
symmetry with `let`) and `recur` jumps execution back to the looping point with new
values for those bindings.

Let's see an example:

```clojure
(loop [x 0]
  (println "Looping with " x)
  (if (= x 2)
    (println "Done looping!")
    (recur (inc x))))
;; Looping with 0
;; Looping with 1
;; Looping with 2
;; Done looping!
;; => nil
```

In the above snippet, we bind the name `x` to the value `0` and execute the
body. Since the condition is not met the first time, it's rerun with `recur`,
incrementing the binding value with the `inc` function. We do this once more until
the condition is met and, since there aren't any more `recur` calls, exit the loop.

Note that `loop` isn't the only point we can `recur` to; using `recur` inside a
function executes the body of the function recursively with the new bindings:

```clojure
(defn recursive-function
  [x]
  (println "Looping with" x)
  (if (= x 2)
    (println "Done looping!")
    (recur (inc x))))

(recursive-function 0)
;; Looping with 0
;; Looping with 1
;; Looping with 2
;; Done looping!
;; => nil
```


##### Replacing for loops with higher-order functions

In imperative programming languages it is common to use `for` loops to iterate over
data and transform it, usually with the intent being one of the following:

- Transform every value in the iterable yielding another iterable
- Filter the elements of the iterable by certain criteria
- Convert the iterable to a value where each iteration depends on the result from
  the previous one
- Run a computation for every value in the iterable

The above actions are encoded in higher-order functions and syntactic constructs in
ClojureScript; let's see an example of the first three.

For transforming every value in an iterable data structure we use the `map`
function, which takes a function and a sequence and applies the function to every
element:

```clojure
(map inc [0 1 2])
;; => (1 2 3)
```

The first parameter for `map` can be _any_ function that takes one argument and
returns a value.  For example, if you had a graphing application and you wanted to
graph the equation `y&#160;=&#160;3x&#160;+&#160;5` for a set of _x_ values, you
could get the _y_ values like this:

```clojure
(defn y-value [x] (+ (* 3 x) 5))

(map y-value [1 2 3 4 5])
;; => (8 11 14 17 20)
```

If your function is short, you can use an anonymous function instead, either the
normal or short syntax:

```clojure
(map (fn [x] (+ (* 3 x) 5)) [1 2 3 4 5])
;; => (8 11 14 17 20)

(map #(+ (* 3 %) 5) [1 2 3 4 5])
;; => (8 11 14 17 20)
```

For filtering the values of a data structure we use the `filter` function, which
takes a predicate and a sequence and gives a new sequence with only the elements
that returned `true` for the given predicate:

```clojure
(filter odd? [1 2 3 4])
;; => (1 3)
```

Again, you can use any function that returns `true` or `false` as the first argument
to `filter`.  Here is an example that keeps only words less than five characters
long. (The `count` function returns the length of its argument.)

```clojure
(filter (fn [word] (< (count word) 5)) ["ant" "baboon" "crab" "duck" "echidna" "fox"])
;; => ("ant" "crab" "duck" "fox")
```

Converting an iterable to a single value, accumulating the intermediate result at
every step of the iteration can be achieved with `reduce`, which takes a function
for accumulating values, an optional initial value and a collection:

```clojure
(reduce + 0 [1 2 3 4])
;; => 10
```

Yet again, you can provide your own function as the first argument to `reduce`, but
your function must have _two_ parameters. The first one is the "accumulated value"
and the second parameter is the collection item being processed. The function
returns a value that becomes the accumulator for the next item in the list.  For
example, here is how you would find the sum of squares of a set of numbers (this is
an important calculation in statistics). Using a separate function:

```clojure
(defn sum-squares
  [accumulator item]
  (+ accumulator (* item item)))

(reduce sum-squares 0 [3 4 5])
;; => 50
```

...and with an anonymous function:

```clojure
(reduce (fn [acc item] (+ acc (* item item))) 0 [3 4 5])
;; => 50
```

Here is a `reduce` that finds the total number of characters in a set of words:

```clojure
(reduce (fn [acc word] (+ acc (count word))) 0 ["ant" "bee" "crab" "duck"])
;; => 14
```

We have not used the short syntax here because, although it requires less typing, it
can be less readable, and when you are starting with a new language, it's important
to be able to read what you wrote! If you are comfortable with the short syntax,
feel free to use it.

Remember to choose your starting value for the accumulator carefully. If you wanted
to use `reduce` to find the product of a series of numbers, you would have to start
with one rather than zero, otherwise all the numbers would be multiplied by zero!

```clojure
;; wrong starting value
(reduce * 0 [3 4 5])
;; => 0

;; correct starting accumulator
(reduce * 1 [3 4 5])
;; => 60
```


##### `for` sequence comprehensions

In ClojureScript, the `for` construct isn't used for iteration but for generating
sequences, an operation also known as "sequence comprehension". In this section
we'll learn how it works and use it to declaratively build sequences.

`for` takes a vector of bindings and an expression and generates a sequence of the
result of evaluating the expression. Let's take a look at an example:

```clojure
(for [x [1 2 3]]
  [x (* x x)])
;; => ([1 1] [2 4] [3 9])
```

In this example, `x` is bound to each of the items in the vector `[1 2 3]` in turn,
and returns a new sequence of two-item vectors with the original item squared.

`for` supports multiple bindings, which will cause the collections to be iterated in
a nested fashion, much like nesting `for` loops in imperative languages. The
innermost binding iterates “fastest.”

```clojure
(for [x [1 2 3]
      y [4 5]]
  [x y])

;; => ([1 4] [1 5] [2 4] [2 5] [3 4] [3 5])
```

We can also follow the bindings with three modifiers: `:let` for creating local
bindings, `:while` for breaking out of the sequence generation, and `:when` for
filtering out values.

Here's an example of local bindings using the `:let` modifier; note that the
bindings defined with it will be available in the expression:

```clojure
(for [x [1 2 3]
      y [4 5]
      :let [z (+ x y)]]
  z)
;; => (5 6 6 7 7 8)
```

We can use the `:while` modifier for expressing a condition that, when it is no
longer met, will stop the sequence generation. Here's an example:

```clojure
(for [x [1 2 3]
      y [4 5]
      :while (= y 4)]
  [x y])

;; => ([1 4] [2 4] [3 4])
```

For filtering out generated values, use the `:when` modifier as in the following
example:

```clojure
(for [x [1 2 3]
      y [4 5]
      :when (= (+ x y) 6)]
  [x y])

;; => ([1 5] [2 4])
```

We can combine the modifiers shown above for expressing complex sequence generations
or more clearly expressing the intent of our comprehension:

```clojure
(for [x [1 2 3]
      y [4 5]
      :let [z (+ x y)]
      :when (= z 6)]
  [x y])

;; => ([1 5] [2 4])
```

When we outlined the most common usages of the `for` construct in imperative
programming languages, we mentioned that sometimes we want to run a computation for
every value in a sequence, not caring about the result. Presumably we do this for
achieving some sort of side-effect with the values of the sequence.

ClojureScript provides the `doseq` construct, which is analogous to `for` but
executes the expression, discards the resulting values, and returns `nil`.

```clojure
(doseq [x [1 2 3]
        y [4 5]
       :let [z (+ x y)]]
  (println x "+" y "=" z))

;; 1 + 4 = 5
;; 1 + 5 = 6
;; 2 + 4 = 6
;; 2 + 5 = 7
;; 3 + 4 = 7
;; 3 + 5 = 8
;; => nil
```

If you want just iterate and apply some side effectfull operation (like `println`)
over each item in the collection, you can just use the specialized function `run!`
that internally uses fast reduction:

```clojure
(run! println [1 2 3])
;; 1
;; 2
;; 3
;; => nil
```

This function explicitly returns `nil`.
