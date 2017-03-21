### Abstractions and Polymorphism

I'm sure that at more than one time you have found yourself in this situation: you
have defined a great abstraction (using interfaces or something similar) for your
"business logic", and you have found the need to deal with another module over which
you have absolutely no control, and you probably were thinking of creating adapters,
proxies, and other approaches that imply a great amount of additional complexity.

Some dynamic languages allow "monkey-patching"; languages where the classes are open
and any method can be defined and redefined at any time. Also, it is well known that
this technique is a very bad practice.

We can not trust languages that allow you to silently overwrite methods that you are
using when you import third party libraries; you cannot expect consistent behavior
when this happens.

These symptoms are commonly called the "expression problem";
see http://en.wikipedia.org/wiki/Expression_problem for more details

#### Protocols

The _ClojureScript_ primitive for defining "interfaces" is called a protocol. A
protocol consists of a name and set of functions. All the functions have at least
one argument corresponding to the `this` in JavaScript or `self` in Python.

Protocols provide a type-based polymorphism, and the dispatch is always done by the
first argument (equivalent to JavaScript’s `this`, as previously mentioned).

A protocol looks like this:

```clojure
(ns myapp.testproto)

(defprotocol IProtocolName
  "A docstring describing the protocol."
  (sample-method [this] "A doc string associated with this function."))
```

NOTE: the "I" prefix is commonly used to designate the separation of protocols and
types. In the Clojure community, there are many different opinions about how the "I"
prefix should be used. In our opinion, it is an acceptable solution to avoid name
clashing and possible confusion. But not using the prefix is not considered bad
practice.

From the user perspective, protocol functions are simply plain functions defined in
the namespace where the protocol is defined. This enables an easy and simple aproach
for avoid conflicts between different protocols implemented for the same type that
have conflicting function names.

Here is an example. Let's create a protocol called `IInvertible` for data that can
be "inverted".  It will have a single method named `invert`.

```clojure
(defprotocol IInvertible
  "This is a protocol for data types that are 'invertible'"
  (invert [this] "Invert the given item."))
```

##### Extending existing types

One of the big strengths of protocols is the ability to extend existing and maybe
third party types. This operation can be done in different ways.

The majority of time you will tend to use the *extend-protocol* or the *extend-type*
macros. This is how `extend-type` syntax looks:

```clojure
(extend-type TypeA
  ProtocolA
  (function-from-protocol-a [this]
    ;; implementation here
    )

  ProtocolB
  (function-from-protocol-b-1 [this parameter1]
    ;; implementation here
    )
  (function-from-protocol-b-2 [this parameter1 parameter2]
    ;; implementation here
    ))
```

You can observe that with *extend-type* you are extending a single type with
different protocols in a single expression.

Let's play with our `IInvertible` protocol defined previously:

```clojure
(extend-type string
  IInvertible
  (invert [this] (apply str (reverse this))))

(extend-type cljs.core.List
  IInvertible
  (invert [this] (reverse this)))

(extend-type cljs.core.PersistentVector
  IInvertible
  (invert [this] (into [] (reverse this))))
```

You may note that a special symbol *string* is used instead of `js/String` for
extend the protol for string. This is because the builtin javascript types have
special treatment and if you replace the `string` with `js/String` the compiler
will emit a warning about that.

So if you want extend your protocol to javascript primitive types, instead of using
`js/Number`, `js/String`, `js/Object`, `js/Array`, `js/Boolean` and `js/Function`
you should use the respective special symbols: `number`, `string`, `object`,
`array`, `boolean` and `function`.

Now, it's time to try our protocol implementation:

```clojure
(invert "abc")
;; => "cba"

(invert 0)
;; => 0

(invert '(1 2 3))
;; => (3 2 1)

(invert [1 2 3])
;; => [3 2 1]
```

In comparison, *extend-protocol* does the inverse; given a protocol, it adds
implementations for multiple types. This is how the syntax looks:

```clojure
(extend-protocol ProtocolA
  TypeA
  (function-from-protocol-a [this]
    ;; implementation here
    )

  TypeB
  (function-from-protocol-a [this]
    ;; implementation here
    ))
```

Thus, the previous example could have been written equally well with this way:

```clojure
(extend-protocol IInvertible
  string
  (invert [this] (apply str (reverse this)))

  cljs.core.List
  (invert [this] (reverse this))

  cljs.core.PersistentVector
  (invert [this] (into [] (reverse this))))
```

##### Participate in ClojureScript abstractions

ClojureScript itself is built up on abstractions defined as protocols. Almost all
behavior in the _ClojureScript_ language itself can be adapted to third party
libraries. Let's look at a real life example.

In previous sections, we have explained the different kinds of built-in
collections. For this example we will use a *set*. See this snippet of code:

```clojure
(def mynums #{1 2})

(filter mynums [1 2 4 5 1 3 4 5])
;; => (1 2 1)
```

What happened? In this case, the _set_ type implements the _ClojureScript_ internal
`IFn` protocol that represents an abstraction for functions or anything
callable. This way it can be used like a callable predicate in filter.

OK, but what happens if we want to use a regular expression as a predicate function
for filtering a collection of strings:

```clojure
(filter #"^foo" ["haha" "foobar" "baz" "foobaz"])
;; TypeError: Cannot call undefined
```

The exception is raised because the `RegExp` type does not implement the `IFn`
protocol so it cannot behave like a callable, but that can be easily fixed:

```clojure
(extend-type js/RegExp
  IFn
  (-invoke
   ([this a]
     (re-find this a))))
```

Let’s analyze this: we are extending the `js/RegExp` type so that it implements the
`invoke` function in the `IFn` protocol. To invoke a regular expression `a` as if it
were a function, call the `re-find` function with the object of the function and the
pattern.

Now, you will be able use the regex instances as predicates in a filter operation:

```clojure
(filter #"^foo" ["haha" "foobar" "baz" "foobaz"])
;; => ("foobar" "foobaz")
```

##### Introspection using Protocols

_ClojureScript_ comes with a useful function that allows runtime introspection:
`satisfies?`. The purpose of this function is to determine at runtime if some object
(instance of some type) satisfies the concrete protocol.

So, with the previous examples, if we check if a `set` instance satisfies an *IFn*
protocol, it should return `true`:

```clojure
(satisfies? IFn #{1})
;; => true
```

#### Multimethods

We have previously talked about protocols which solve a very common use case of
polymorphism: dispatch by type. But in some circumstances, the protocol approach can
be limiting. And here, *multimethods* come to the rescue.

These *multimethods* are not limited to type dispatch only; instead, they also offer
dispatch by types of multiple arguments and by value. They also allow ad-hoc
hierarchies to be defined. Also, like protocols, multimethods are an "Open System",
so you or any third parties can extend a multimethod for new types.

The basic constructions of *multimethods* are the `defmulti` and `defmethod`
forms. The `defmulti` form is used to create the multimethod with an initial
dispatch function. This is a model of what it looks like:

```clojure
(defmulti say-hello
  "A polymorphic function that return a greetings message
  depending on the language key with default lang as `:en`"
  (fn [param] (:locale param))
  :default :en)
```

The anonymous function defined within the `defmulti` form is a dispatch function. It
will be called in every call to the `say-hello` function and should return some kind
of marker object that will be used for dispatch. In our example, it returns the
contents of the `:locale` key of the first argument.

And finally, you should add implementations. That is done with the `defmethod` form:

```clojure
(defmethod say-hello :en
  [person]
  (str "Hello " (:name person "Anonymous")))

(defmethod say-hello :es
  [person]
  (str "Hola " (:name person "Anónimo")))
```

So, if you execute that function over a hash map containing the `:locale` and
optionally the `:name` key, the multimethod will first call the dispatch function to
determine the dispatch value, then it will search for an implementation for that
value. If an implementation is found, the dispatcher will execute it. Otherwise, the
dispatch will search for a default implementation (if one is specified) and execute
it.

```clojure
(say-hello {:locale :es})
;; => "Hola Anónimo"

(say-hello {:locale :en :name "Ciri"})
;; => "Hello Ciri"

(say-hello {:locale :fr})
;; => "Hello Anonymous"
```

If the default implementation is not specified, an exception will be raised
notifying you that some value does not have an implementation for that multimethod.

#### Hierarchies

Hierarchies are _ClojureScript_’s way to let you build whatever relations that your
domain may require. Hierarchies are defined in term of relations between named
objects, such as symbols, keywords, or types.

Hierarchies can be defined globally or locally, depending on your needs. Like
multimethods, hierarchies are not limited to a single namespace. You can extend a
hierarchy from any namespace, not only from the one in which it is defined.

The global namespace is more limited, for good reasons. Keywords or symbols that are
not namespaced can not be used in the global hierarchy. That behavior helps prevent
unexpected situations when two or more third party libraries use the same symbol for
different semantics.

##### Defining a hierarchy

The hierarchy relations should be established using the `derive` function:

```clojure
(derive ::circle ::shape)
(derive ::box ::shape)
```

We have just defined a set of relationships between namespaced keywords. In this
case the `::circle` is a child of `::shape`, and `::box` is also a child of
`::shape`.

TIP: The `::circle` keyword syntax is a shorthand for `:current.ns/circle`. So if
you are executing it in a REPL, `::circle` will be evaluated as `:cljs.user/circle`.

##### Hierarchies and introspection

_ClojureScript_ comes with a little toolset of functions that allows runtime
introspection of globally or locally defined hierarchies. This toolset consists of
three functions: `isa?`, `ancestors`, and `descendants`.

Let's see an example of how it can be used with the hierarchy defined in the
previous example:

```clojure
(ancestors ::box)
;; => #{:cljs.user/shape}

(descendants ::shape)
;; => #{:cljs.user/circle :cljs.user/box}

(isa? ::box ::shape)
;; => true

(isa? ::rect ::shape)
;; => false
```

##### Locally defined hierarchies

As we mentioned previously, in _ClojureScript_ you also can define local
hierarchies. This can be done with the `make-hierarchy` function. Here is an example
of how you can replicate the previous example using a local hierarchy:

```clojure
(def h (-> (make-hierarchy)
           (derive :box :shape)
           (derive :circle :shape)))
```

Now you can use the same introspection functions with that locally defined
hierarchy:

```clojure
(isa? h :box :shape)
;; => true

(isa? :box :shape)
;; => false
```

As you can observe, in local hierarchies we can use normal (not namespace qualified)
keywords, and if we execute the `isa?` without passing the local hierarchy
parameter, it returns `false` as expected.

##### Hierarchies in multimethods

One of the big advantages of hierarchies is that they work very well together with
multimethods.  This is because multimethods by default use the `isa?` function for
the last step of dispatching.

Let's see an example to clearly understand what that means. First, we define the
multimethod with the `defmulti` form:

```clojure
(defmulti stringify-shape
  "A function that prints a human readable representation
  of a shape keyword."
  identity
  :hierarchy #'h)
```

With the `:hierarchy` keyword parameter, we indicate to the multimethod what
hierarchy we want to use; if it is not specified, the global hierarchy will be used.

Second, we define an implementation for our multimethod using the `defmethod` form:

```clojure
(defmethod stringify-shape :box
  [_]
  "A box shape")

(defmethod stringify-shape :shape
  [_]
  "A generic shape")

(defmethod stringify-shape :default
  [_]
  "Unexpected object")
```

Now, let's see what happens if we execute that function with a box:

```clojure
(stringify-shape :box)
;; => "A box shape"
```

Now everything works as expected; the multimethod executes the direct matching
implementation for the given parameter. Next, let's see what happens if we execute
the same function but with the `:circle` keyword as the parameter which does not
have the direct matching dispatch value:

```clojure
(stringify-shape :circle)
;; => "A generic shape"
```

The multimethod automatically resolves it using the provided hierarchy, and since
`:circle` is a descendant of `:shape`, the `:shape` implementation is executed.

Finally, if you give a keyword that isn't part of the hierarchy, you get the
`:default` implementation:

```clojure
(stringify-shape :triangle)
;; => "Unexpected object"
```
