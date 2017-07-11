### Host interoperability

_ClojureScript_, in the same way as its brother Clojure, is designed to be a "guest" language. This means that the design of the language works well on top of an existing ecosystem such as JavaScript for _ClojureScript_ and the JVM for _Clojure_.


#### The types

_ClojureScript_, unlike what you might expect, tries to take advantage of every type that the platform provides. This is a (perhaps incomplete) list of things that _ClojureScript_ inherits and reuses from the underlying platform:

* _ClojureScript_ strings are JavaScript *Strings*.
* _ClojureScript_ numbers are JavaScript *Numbers*.
* _ClojureScript_ `nil` is a JavaScript *null*.
* _ClojureScript_ regular expressions are JavaScript `RegExp` instances.
* _ClojureScript_ is not interpreted; it is always compiled down to JavaScript.
* _ClojureScript_ allows easy call to platform APIs with the same semantics.
* _ClojureScript_ data types internally compile to objects in JavaScript.

On top of it, _ClojureScript_ builds its own abstractions and types that do not exist in the platform, such as Vectors, Maps, Sets, and others that are explained in preceding sections of this chapter.


#### Interacting with platform types

_ClojureScript_ comes with a little set of special forms that allows it to interact with platform types such as calling object methods, creating new instances, and accessing object properties.


##### Access to the platform

_ClojureScript_ has a special syntax for access to the entire platform environment through the `js/` special namespace. This is an example of an expression to execute JavaScript's built-in `parseInt` function:

```clojure
(js/parseInt "222")
;; => 222
```


##### Creating new instances

_ClojureScript_ has two ways to create instances:

Using the `new` special form

```clojure
(new js/RegExp "^foo$")
```

Using the `.` special form

```clojure
(js/RegExp. "^foo$")
```

The last one is the recommended way to create instances. We are not aware of any real differences between the two forms, but in the ClojureScript community, the last one is used most often.


##### Invoke instance methods

To invoke methods of some object instance, as opposed to how it is done in JavaScript (e.g., `obj.method()`, the method name comes first like any other standard function in Lisp languages but with a little variation: the function name
starts with special form `.`.

Let's see how we can call the `.test()` method of a regexp instance:

```clojure
(def re (js/RegExp "^Clojure"))

(.test re "ClojureScript")
;; => true
```

You can invoke instance methods on JavaScript objects. The first example follows the pattern you have seen; the last one is a shortcut:

```clojure
(.sqrt js/Math 2)
;; => 1.4142135623730951
(js/Math.sqrt 2)
;; => 1.4142135623730951
```


##### Access to object properties

Access to an object's properties is really very similar to calling a method. The difference is that instead of using the `.` you use `.-`. Let's see an example:

```clojure
(.-multiline re)
;; => false
(.-PI js/Math)
;; => 3.141592653589793
```


##### Property access shorthand

Symbols with the `js/` prefix can contain dots to denote nested property access. Both of the following expressions invoke the same function:

```clojure
(.log js/console "Hello World")

(js/console.log "Hello World")
```

And both of the following expressions access the same property:

```clojure
(.-PI js/Math)
;; => 3.141592653589793

js/Math.PI
;; => 3.141592653589793
```


##### JavaScript objects

_ClojureScript_ has different ways to create plain JavaScript objects; each one has its own purpose. The basic one is the `js-obj` function. It accepts a variable number of pairs of keys and values and returns a JavaScript object:

```clojure
(js-obj "country" "FR")
;; => #js {:country "FR"}
```

The return value can be passed to some kind of third party library that accepts a plain JavaScript object, but you can observe the real representation of the return value of this function. It is really another form for doing the same thing.

Using the reader macro `#js` consists of prepending it to a ClojureScript map or vector, and the result will be transformed to plain JavaScript:

```clojure
(def myobj #js {:country "FR"})
```

The translation of that to plain JavaScript is similar to this:

```javascript
var myobj = {country: "FR"};
```

As explained in the previous section, you can also access the plain object properties using the `.-` syntax:

```clojure
(.-country myobj)
;; => "FR"
```

And as JavaScript objects are mutable, you can set a new value for some property using the `set!` function:

```clojure
(set! (.-country myobj) "KR")
```


##### Conversions

The inconvenience of the previously explained forms is that they do not make recursive transformations, so if you have nested objects, the nested objects will not be converted.  Consider this example that uses Clojurescript maps, then a
similar one with JavaScript objects:

```clojure
(def clj-map {:country {:code "FR" :name "France"}})
;; => {:country {:code "FR", :name "France"}}
(:code (:country clj-map)
;; => "FR"

(def js-obj #js {:country {:code "FR" :name "France"}})
;; => #js {:country {:code "FR", :name "France"}
(.-country js-obj)
;; => {:code "FR", :name "France"}
(.-code (.-country js-obj)
;; => nil
```

To solve that use case, _ClojureScript_ comes with the `clj->js` and `js->clj` functions that transform Clojure collection types into JavaScript and back. Note that the conversion to ClojureScript changes the `:country` keyword to a string.

```clojure
(clj->js {:foo {:bar "baz"}})
;; => #js {:foo #js {:bar "baz"}}
(js->clj #js {:country {:code "FR" :name "France"}}))
;; => {"country" {:code "FR", :name "France"}}
```

In the case of arrays, there is a specialized function `into-array` that behaves as expected:

```clojure
(into-array ["France" "Korea" "Peru"])
;; => #js ["France" "Korea" "Peru"]
```


##### Arrays

In the previous example, we saw how we can create an array from an existing _ClojureScript_ collection. But there is another function for creating arrays: `make-array`.

.Creating a preallocated array with length 10
```clojure
(def a (make-array 10))
;; => #js [nil nil nil nil nil nil nil nil nil nil]
```

In _ClojureScript_, arrays also play well with sequence abstractions, so you can iterate over them or simply get the number of elements with the `count` function:

```clojure
(count a)
;; => 10
```

As arrays in the JavaScript platform are a mutable collection type, you can access a concrete index and set the value at that position:

```clojure
(aset a 0 2)
;; => 2
a
;; => #js [2 nil nil nil nil nil nil nil nil nil]
```

Or access in an indexed way to get its values:

```clojure
(aget a 0)
;; => 2
```

In JavaScript, array index access is equivalent to object property access, so you can use the same functions for interacting with plain objects:

```clojure
(def b #js {:hour 16})
;; => #js {:hour 16}

(aget b "hour")
;; => 16

(aset b "minute" 22)
;; => 22

b
;; => #js {:hour 16, :minute 22}
```
