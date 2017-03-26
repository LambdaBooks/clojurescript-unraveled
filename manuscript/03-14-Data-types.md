#### Data types

Until now, we have used maps, sets, lists, and vectors to represent our data. And in most cases, this is a really great approach. But sometimes we need to define our own types, and in this book we will call them *data types*.

A data type provides the following:

* A unique host-backed type, either named or anonymous.
* The ability to implement protocols (inline).
* Explicitly declared structure using fields or closures.
* Map-like behavior (via records, see below).


#### Deftype

The most low-level construction in _ClojureScript_ for creating your own types is the `deftype` macro. As a demonstration, we will define a type called `User`:

```cljs
(deftype User [firstname lastname])
```

Once the type has been defined, we can create an instance of our `User`. In the following example, the `.` after `User` indicates that we are calling a constructor.

```cljs
(def person (User. "Triss" "Merigold"))
```

Its fields can be accessed using the prefix dot notation:

```cljs
(.-firstname person)
;; => "Triss"
```

Types defined with `deftype` (and `defrecord`, which we will see later) create a host-backed class-like object associated with the current namespace. For convenience, _ClojureScript_ also defines a constructor function called `->User` that can be imported using the `:require` directive.

We personally do not like this type of function, and we prefer to define our own constructors with more idiomatic names:

```cljs
(defn make-user
  [firstname lastname]
  (User. firstname lastname))
```

We use this in our code instead of `->User`.


#### Defrecord

The record is a slightly higher-level abstraction for defining types in _ClojureScript_ and should be the preferred way to do it.

As we know, _ClojureScript_ tends to use plain data types such as maps, but in most cases we need a named type to represent the entities of our application. Here come the records.

A record is a data type that implements the map protocol and therefore can be used like any other map.  And since records are also proper types, they support type-based polymorphism through protocols.

In summary: with records, we have the best of both worlds, maps that can play in different abstractions.

Let's start defining the `User` type but using records:

```cljs
(defrecord User [firstname lastname])
```

It looks really similar to the `deftype` syntax; in fact, it uses `deftype` behind the scenes as a low-level primitive for defining types.

Now, look at the difference with raw types for access to its fields:

```cljs
(def person (User. "Yennefer" "of Vengerberg"))

(:firstname person)
;; => "Yennefer"

(get person :firstname)
;; => "Yennefer"
```

As we mentioned previously, records are maps and act like them:

```
(map? person)
;; => true
```

And like maps, they support extra fields that are not initially defined:

```
(def person2 (assoc person :age 92))

(:age person2)
;; => 92
```

As we can see, the `assoc` function works as expected and returns a new instance of the same type but with new key value pair. But take care with `dissoc`! Its behavior with records is slightly different than with maps; it will return a new record if the field being dissociated is an optional field, but it will return a plain map if you dissociate a mandatory field.

Another difference with maps is that records do not act like functions:

```cljs
(def plain-person {:firstname "Yennefer", :lastname "of Vengerberg"})

(plain-person :firstname)
;; => "Yennefer"

(person :firstname)
;; => person.User does not implement IFn protocol.
```

For convenience, the `defrecord` macro, like `deftype`, exposes a `->User` function, as well as an additional `map->User` constructor function. We have the same opinion about that constructor as with `deftype` defined ones: we recommend defining your own instead of using the other ones. But as they exist, let’s see how they can be used:

```
(def cirilla (->User "Cirilla" "Fiona"))
(def yen (map->User {:firstname "Yennefer"
                     :lastname "of Vengerberg"}))
```


#### Implementing protocols

Both type definition primitives that we have seen so far allow inline implementations for protocols (explained in a previous section). Let's define one for example purposes:

```
(defprotocol IUser
  "A common abstraction for working with user types."
  (full-name [_] "Get the full name of the user."))
```

Now, you can define a type with inline implementation for an abstraction, in our case the `IUser`:

```
(defrecord User [firstname lastname]
  IUser
  (full-name [_]
    (str firstname " " lastname)))

;; Create an instance.
(def user (User. "Yennefer" "of Vengerberg"))

(full-name user)
;; => "Yennefer of Vengerberg"
```


#### Reify

The `reify` macro is an _ad hoc constructor_ you can use to create objects without pre-defining a type.  Protocol implementations are supplied the same as `deftype` and `defrecord`, but in contrast, `reify` does not have accessible fields.

This is how we can emulate an instance of the user type that plays well with the `IUser` abstraction:

```cljs
(defn user
  [firstname lastname]
  (reify
    IUser
    (full-name [_]
      (str firstname " " lastname))))

(def yen (user "Yennefer" "of Vengerberg"))
(full-name yen)
;; => "Yennefer of Vengerberg"
```

#### Specify

`specify!` is an advanced alternative to `reify`, allowing you to add protocol implementations to an existing JavaScript object.  This can be useful if you want to graft protocols onto a JavaScript library's components.

```cljs
(def obj #js {})

(specify! obj
  IUser
  (full-name [_]
    "my full name"))

(full-name obj)
;; => "my full name"
```

`specify` is an immutable version of `specify!` that can be used on immutable, copyable values implementing `ICloneable` (e.g. ClojureScript collections).

```
(def a {})

(def b (specify a
         IUser
         (full-name [_]
           "my full name")))

(full-name a)
;; Error: No protocol method IUser.full-name defined for type cljs.core/PersistentArrayMap: {}

(full-name b)
;; => "my full name"
```