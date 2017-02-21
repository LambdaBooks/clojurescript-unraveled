### Functions

#### The first contact

It's time to make things happen. _ClojureScript_ has what are known as first class functions. They behave like any other type; you can pass them as parameters and you can return them as values, always respecting the lexical scope. _ClojureScript_ also has some features of dynamic scoping, but this will be discussed in another section. 

If you want to know more about scopes, this link:http://en.wikipedia.org/wiki/Scope_(computer_science)[Wikipedia article] is very extensive and explains different types of scoping.

As _ClojureScript_ is a Lisp dialect, it uses the prefix notation for calling a function:

```clojure
(inc 1)
;; => 2
```

In the example above, `inc` is a function and is part of the _ClojureScript_ runtime, and `1` is the first argument for the `inc` function.

```clojure
(+ 1 2 3)
;; => 6
```

The `+` symbol represents an `add` function. It allows multiple parameters, whereas in ALGOL-type languages, `+` is an operator and only allows two parameters.

The prefix notation has huge advantages, some of them not always obvious. _ClojureScript_ does not make a distinction between a function and an operator; everything is a function. The immediate advantage is that the prefix notation allows an arbitrary number of arguments per "operator". It also completely eliminates the problem of operator precedence.


#### Defining your own functions   

You can define an unnamed (anonymous) function with the `fn` special form. This is one type of function definition; in the following example, the function takes two parameters and returns their average. 

```clojure
(fn [param1 param2]
  (/ (+ param1 param2) 2.0))
```

You can define a function and call it at the same time (in a single expression):

```clojure
((fn [x] (* x x)) 5)
;; => 25
```

Let's start creating named functions. But what does a _named function_ really mean? It is very simple; in _ClojureScript_, functions are first-class and behave like any other value, so naming a function is done by simply binding the function to a symbol:

```clojure
(def square (fn [x] (* x x)))

(square 12)
;; => 144
```

_ClojureScript_ also offers the `defn` macro as a little syntactic sugar for making function definition more idiomatic:

```clojure
(defn square
  "Return the square of a given number."
  [x]
  (* x x))
```

The string that comes between the function name and the parameter vector is called a _docstring_ (documentation string); programs that automatically create web documentation from your source files will use these docstrings.


#### Functions with multiple arities

_ClojureScript_ also comes with the ability to define functions with an arbitrary number of arguments. (The term _arity_ means the number of arguments that a function takes.) The syntax is almost the same as for defining an ordinary function, with the difference that it has more than one body.

Let's see an example, which will explain it better:

```clojure
(defn myinc
  "Self defined version of parameterized `inc`."
  ([x] (myinc x 1))
  ([x increment]
   (+ x increment)))
```

This line: `([x] (myinc x 1))` says that if there is only one argument, call the function `myinc` with that argument and the number `1` as the second argument. The other function body `([x increment] (+ x increment))` says that if there are two arguments, return the result of adding them.

Here are some examples using the previously defined multi-arity function. Observe that if you call a function with the wrong number of arguments, the compiler will emit an error message.

```clojure
(myinc 1)
;; => 2

(myinc 1 3)
;; => 4

(myinc 1 3 3)
;; Compiler error
```

NOTE: Explaining the concept of "arity" is out of the scope of this book, however you can read about that in this link:http://en.wikipedia.org/wiki/Arity[Wikipedia article].

#### Variadic functions

Another way to accept multiple parameters is defining variadic functions. Variadic functions are functions that accept an arbitrary number of arguments:

```clojure
(defn my-variadic-set
  [& params]
  (set params))

(my-variadic-set 1 2 3 1)
;; => #{1 2 3}
```

The way to denote a variadic function is using the `&` symbol prefix on its arguments vector.


#### Short syntax for anonymous functions

_ClojureScript_ provides a shorter syntax for defining anonymous functions using the `#()` reader macro (usually leads to one-liners). Reader macros are "special" expressions that will be transformed to the appropriate language form at compile time; in this case, to some expression that uses the `fn` special form.

```clojure
(def average #(/ (+ %1 %2) 2))

(average 3 4)
;; => 3.5
```

The preceding definition is shorthand for:

```clojure
(def average-longer (fn [a b] (/ (+ a b) 2)))

(average-longer 7 8)
;; => 7.5
```

The `%1`, `%2`... `%N` are simple markers for parameter positions that are implicitly declared when the reader macro will be interpreted and converted to a `fn` expression. 

If a function only accepts one argument, you can omit the number after the `%` symbol, e.g., a function that squares a number: `#(* %1 %1))` can be written `++#++(* % %))`.

Additionally, this syntax also supports the variadic form with the `%&` symbol:

```clojure
(def my-variadic-set #(set %&))

(my-variadic-set 1 2 2)
;; => #{1 2}
```