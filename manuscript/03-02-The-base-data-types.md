### The base data types

The ClojureScript language has a rich set of data types like most programming
languages. It provides scalar data types that will be very familiar to you, such as
numbers, strings, and floats. Beyond these, it also provides a great number of
others that might be less familiar, such as symbols, keywords, regexes (regular
expressions), vars, atoms, and volatiles.

_ClojureScript_ embraces the host language, and where possible, uses the host's
provided types. For example: numbers and strings are used as is and behave in the
same way as in JavaScript.

#### Numbers

In _ClojureScript_, numbers include both integers and floating points. Keeping in
mind that _ClojureScript_ is a guest language that compiles to JavaScript, integers
are actually JavaScript's native floating points under the hood.

As in any other language, numbers in _ClojureScript_ are represented in the
following ways:

[source, clojure]
----
23
+23
-100
1.7
-2
33e8
12e-14
3.2e-4
----

#### Keywords

Keywords in _ClojureScript_ are objects that always evaluate to themselves. They are
usually used in map data structures to efficiently represent the
keys.

[source, clojure]
----
:foobar
:2
:?
----

As you can see, the keywords are all prefixed with `:`, but this character is only
part of the literal syntax and is not part of the name of the object.

You can also create a keyword by calling the `keyword` function. Don't worry if you
don't understand or are unclear about anything in the following example;
<<function-section,functions>> are discussed in a later section.

[source, clojure]
----
(keyword "foo")
;; => :foo
----

#### Symbols

Symbols in _ClojureScript_ are very, very similar to *keywords* (which you now know
about). But instead of evaluating to themselves, symbols are evaluated to something
that they refer to, which can be functions, variables, etc.

Symbols start with a non numeric character and can contain alphanumeric characters
as well as *, +, !, -, _, ', and ? such as :

[source, clojure]
----
sample-symbol
othersymbol
f1
my-special-swap!
----

Don't worry if you don't understand right away; symbols are used in almost all of
our examples, which will give you the opportunity to learn more as we go on.

#### Strings

There is almost nothing new we can explain about strings that you don't already
know. In _ClojureScript_, they work the same as in any other language. One point of
interest, however, is that they are immutable.

In this case they are the same as in JavaScript:

[source, clojure]
----
"An example of a string"
----

One peculiar aspect of strings in _ClojureScript_ is due to the language's Lisp
syntax: single and multiline strings have the same syntax:

[source, clojure]
----
"This is a multiline
      string in ClojureScript."
----

#### Characters

_ClojureScript_ also lets you write single characters using Clojure's character
literal syntax.

[source, clojure]
----
\a        ; The lowercase a character
\newline  ; The newline character
----

Since the host language doesn't contain character literals, _ClojureScript_
characters are transformed behind the scenes into single character JavaScript
strings.

#### Collections

Another big step in explaining a language is to explain its collections and
collection abstractions. _ClojureScript_ is not an exception to this rule.

_ClojureScript_ comes with many types of collections. The main difference between
_ClojureScript_ collections and collections in other languages is that they are
persistent and immutable.

Before moving on to these (possibly) unknown concepts, we'll present a high-level
overview of existing collection types in _ClojureScript_.

##### Lists

This is a classic collection type in languages based on Lisp. Lists are the simplest
type of collection in _ClojureScript_. Lists can contain items of any type,
including other collections.

Lists in _ClojureScript_ are represented by items enclosed between parentheses:

[source, clojure]
----
'(1 2 3 4 5)
'(:foo :bar 2)
----

As you can see, all list examples are prefixed with the `'` char. This is because
lists in Lisp-like languages are often used to express things like function or macro
calls. In that case, the first item should be a symbol that will evaluate to
something callable, and the rest of the list elements will be function
arguments. However, in the preceding examples, we don't want the first item as a
symbol; we just want a list of items.

The following example shows the difference between a list without and with the
preceding single quote mark:

[source, clojure]
----
(inc 1)
;; => 2

'(inc 1)
;; => (inc 1)
----

As you can see, if you evaluate `(inc 1)` without prefixing it with `'`, it will
resolve the `inc` symbol to the *inc* function and will execute it with `1` as the
first argument, returning the value `2`.

You can also explicitly create a list with the `list` function:

[source, clojure]
----
(list 1 2 3 4 5)
;; => (1 2 3 4 5)

(list :foo :bar 2)
;; => (:foo :bar 2)
----

Lists have the peculiarity that they are very efficient if you access them
sequentially or access their first elements, but a list is not a very good option if
you need random (index) access to its elements.


##### Vectors

Like lists, *vectors* store a series of values, but in this case, with very
efficient index access to their elements, as opposed to lists, which are evaluated
in order. Don't worry; in the following sections we'll go in depth with details, but
at this moment, this simple explanation is more than enough.

Vectors use square brackets for the literal syntax; let's see some examples:

[source, clojure]
----
[:foo :bar]
[3 4 5 nil]
----

Like lists, vectors can contain objects of any type, as you can observe in the
preceding example.

You can also explicitly create a vector with the `vector` function, but this is not
commonly used in ClojureScript programs:

[source, clojure]
----
(vector 1 2 3)
;; => [1 2 3]

(vector "blah" 3.5 nil)
;; => ["blah" 3.5 nil]
----

##### Maps

Maps are a collection abstraction that allow you to store key/value pairs. In other
languages, this type of structure is commonly known as a hash-map or dict
(dictionary). Map literals in _ClojureScript_ are written with the pairs between
curly braces.

[source, clojure]
----
{:foo "bar", :baz 2}
{:alphabet [:a :b :c]}
----

NOTE: Commas are frequently used to separate a key-value pair, but they are
completely optional. In _ClojureScript_ syntax, commas are treated like spaces.

Like vectors, every item in a map literal is evaluated before the result is stored
in a map, but the order of evaluation is not guaranteed.


##### Sets

And finally, *sets*.

Sets store zero or more unique items of any type and are unordered. Like maps, they
use curly braces for their literal syntax, with the difference being that they use a
`#` as the leading character. You can also use the `set` function to convert a
collection to a set:

[source, clojure]
----
#{1 2 3 :foo :bar}
;; => #{1 :bar 3 :foo 2}
(set [1 2 1 3 1 4 1 5])
;; => #{1 2 3 4 5}
----

In subsequent sections, we'll go in depth about sets and the other collection types
you've seen in this section.
