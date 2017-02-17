## First steps with Lisp syntax

Invented by John McCarthy in 1958, Lisp is one of the oldest programming languages
that is still around. It has evolved into many derivatives called dialects,
ClojureScript being one of them. It is a programming language written in its own
data structures — originally lists enclosed in parentheses — but Clojure(Script) has
evolved the Lisp syntax with more data structures, making it more pleasant to write
and read.

A list with a function in the first position is used for calling a function in
ClojureScript. In the example below, we apply the addition function to three
arguments. Note that unlike in other languages, `+` is not an operator but a
function. Lisp has no operators; it only has functions.

[source, clojure]
----
(+ 1 2 3)
;; => 6
----

In the example above, we're applying the addition function `+` to the arguments `1`,
`2` and `3`. ClojureScript allows many unusual characters like `?` or `-` in symbol
names, which makes it easier to read:

[source, clojure]
----
(zero? 0)
;; => true
----

To distinguish function calls from lists of data items, we can quote lists to keep
them from being evaluated.  The quoted lists will be treated as data instead of as a
function call:

[source, clojure]
----
'(+ 1 2 3)
;; => (+ 1 2 3)
----

ClojureScript uses more than lists for its syntax. The full details will be covered
later, but here is an example of the usage of a vector (enclosed in brackets) for
defining local bindings:

[source, clojure]
----
(let [x 1
      y 2
      z 3]
  (+ x y z))
;; => 6
----

This is practically all the syntax we need to know for using not only ClojureScript,
but any Lisp. Being written in its own data structures (often referred to as
_homoiconicity_) is a great property since the syntax is uniform and simple; also,
code generation via xref:macros-section[macros] is easier than in any other
language, giving us plenty of power to extend the language to suit our needs.
