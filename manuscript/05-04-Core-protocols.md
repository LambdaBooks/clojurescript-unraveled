### Core protocols

One of the greatest qualities of the core ClojureScript functions is that they are implemented around protocols. This makes them open to work on any type that we extend with such protocols, be it defined by us or a third party.


#### Functions

As we've learned in previous chapters not only functions can be invoked. Vectors are functions of their indexes, maps are functions of their keys and sets are functions of their values.

We can extend types to be callable as functions implementing the `IFn` protocol. A collection that doesn't support calling it as a function is the queue, let's implement `IFn` for the `PersistentQueue` type so we're able to call queues as functions of their indexes:

```clojure
(extend-type PersistentQueue
  IFn
  (-invoke
    ([this idx]
      (nth this idx))))

(def q #queue[:a :b :c])
;; => #queue [:a :b :c]

(q 0)
;; => :a

(q 1)
;; => :b

(q 2)
;; => :c
```


#### Printing

////
TODO: IWriter?
////

For learning about some of the core protocols we'll define a `Pair` type, which will hold a pair of values.

```clojure
(deftype Pair [fst snd])
```

If we want to customize how types are printed we can implement the `IPrintWithWriter` protocol. It defines a function called `-pr-writer` that receives the value to print, a writer object and options; this function uses the writer object's `-write` function to write the desired `Pair` string representation:


```clojure
(extend-type Pair
  IPrintWithWriter
  (-pr-writer [p writer _]
    (-write writer (str "#<Pair " (.-fst p) "," (.-snd p) ">"))))
```


#### Sequences

In a xref:the-sequence-abstraction[previous section] we learned about sequences, one of ClojureScript's main abstractions. Remember the `first` and `rest` functions for working with sequences? They are defined in the `ISeq` protocol, so we can extend types for responding to such functions:

```clojure
(extend-type Pair
  ISeq
  (-first [p]
    (.-fst p))

  (-rest [p]
    (list (.-snd p))))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(first p)
;; => 1

(rest p)
;; => (2)
```

Another handy function for working with sequences is `next`. Although `next` works as long as the given argument is a sequence, we can implement it explicitly with the `INext` protocol:

```clojure
(def p (Pair. 1 2))

(next p)
;; => (2)

(extend-type Pair
  INext
  (-next [p]
    (println "Our next")
    (list (.-snd p))))

(next p)
;; Our next
;; => (2)
```

Finally, we can make our own types seqable implementing the `ISeqable` protocol. This means we can pass them to `seq` for getting a sequence back.

ISeqable
```clojure
(def p (Pair. 1 2))

(extend-type Pair
  ISeqable
  (-seq [p]
    (list (.-fst p) (.-snd p))))

(seq p)
;; => (1 2)
```

Now our `Pair` type works with the plethora of ClojureScript functions for working with sequences:

```clojure
(def p (Pair. 1 2))
;; => #<Pair 1,2>

(map inc p)
;; => (2 3)

(filter odd? p)
;; => (1)

(reduce + p)
;; => 3
```


#### Collections

Collection functions are also defined in terms of protocols. For this section examples we will make the native JavaScript string act like a collection.

The most important function for working with collection is `conj`, defined in the `ICollection` protocol.  Strings are the only type which makes sense to `conj` to a string, so the `conj` operation for strings will be simply a concatenation:

```clojure
(extend-type string
  ICollection
  (-conj [this o]
    (str this o)))

(conj "foo" "bar")
;; => "foobar"

(conj "foo" "bar" "baz")
;; => "foobarbaz"
```

Another handy function for working with collections is `empty`, which is part of the `IEmptyableCollection` protocol. Let's implement it for the string type:

```clojure
(extend-type string
  IEmptyableCollection
  (-empty [_]
    ""))

(empty "foo")
;; => ""
```

We used the `string` special symbol for extending the native JavaScript string. If you want to learn more about it check the xref:extending-javascript-types[section about extending JavaScript types].


##### Collection traits

There are some qualities that not all collections have, such as being countable in constant time or being reversible. These traits are splitted into different protocols since not all of them make sense for every collection. For illustrating these protocols we'll use the `Pair` type we defined earlier.

For collections that can be counted in constant time using the `count` function we can implement the `ICounted` protocol. It should be easy to implement it for the `Pair` type:

```clojure
(extend-type Pair
  ICounted
  (-count [_]
    2))

(def p (Pair. 1 2))

(count p)
;; => 2
```

Some collection types such as vectors and lists can be indexed by a number using the `nth` function. If our types are indexed we can implement the `IIndexed` protocol:

```clojure
(extend-type Pair
  IIndexed
  (-nth
    ([p idx]
      (case idx
        0 (.-fst p)
        1 (.-snd p)
        (throw (js/Error. "Index out of bounds"))))
    ([p idx default]
      (case idx
        0 (.-fst p)
        1 (.-snd p)
        default))))

(nth p 0)
;; => 1

(nth p 1)
;; => 2

(nth p 2)
;; Error: Index out of bounds

(nth p 2 :default)
;; => :default
```

////
TODO: ISorted?
////

////
TODO: IReversible?
////


#### Associative

There are many data structures that are associative: they map keys to values. We've encountered a few of them already and we know many functions for working with them like `get`, `assoc` or `dissoc`. Let's explore the protocols that these functions build upon.

First of all, we need a way to look up keys on an associative data structure. The `ILookup` protocol defines a function for doing so, let's add the ability to look up keys in our `Pair` type since it is an associative data structure that maps the indices 0 and 1 to values.

```clojure
(extend-type Pair
  ILookup
  (-lookup
    ([p k]
      (-lookup p k nil))
    ([p k default]
      (case k
        0 (.-fst p)
        1 (.-snd p)
        default))))

(get p 0)
;; => 1

(get p 1)
;; => 2

(get p :foo)
;; => nil

(get p 2 :default)
;; => :default
```

For using `assoc` on a data structure it must implement the `IAssociative` protocol. For our `Pair` type only 0 and 1 will be allowed as keys for associating values. `IAssociative` also has a function for asking whether a key is present or not.

```clojure
(extend-type Pair
  IAssociative
  (-contains-key? [_ k]
    (contains? #{0 1} k))

  (-assoc [p k v]
    (case k
      0 (Pair. v (.-snd p))
      1 (Pair. (.-fst p) v)
      (throw (js/Error. "Can only assoc to 0 and 1 keys")))))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(assoc p 0 2)
;; => #<Pair 2,2>

(assoc p 1 1)
;; => #<Pair 1,1>

(assoc p 0 0 1 1)
;; => #<Pair 0,1>

(assoc p 2 3)
;; Error: Can only assoc to 0 and 1 keys
```

The complementary function for `assoc` is `dissoc` and it's part of the `IMap` protocol. It doesn't make much sense for our `Pair` type but we'll implement it nonetheless. Dissociating 0 or 1 will mean putting a `nil` in such position and invalid keys will be ignored.

```clojure
(extend-type Pair
  IMap
  (-dissoc [p k]
    (case k
      0 (Pair. nil (.-snd p))
      1 (Pair. (.-fst p) nil)
      p)))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(dissoc p 0)
;; => #<Pair ,2>

(dissoc p 1)
;; => #<Pair 1,>

(dissoc p 2)
;; => #<Pair 1,2>

(dissoc p 0 1)
;; => #<Pair ,>
```

Associative data structures are made of key and value pairs we can call entries. The `key` and `val` functions allow us to query the key and value of such entries and they are built upon the `IMapEntry` protocol. Let's see a few examples of `key` and `val` and how map entries can be used to build up maps:

```clojure
(key [:foo :bar])
;; => :foo

(val [:foo :bar])
;; => :bar

(into {} [[:foo :bar] [:baz :xyz]])
;; => {:foo :bar, :baz :xyz}
```

Pairs can be map entries too, we treat their first element as the key and the second as the value:

```clojure
(extend-type Pair
  IMapEntry
  (-key [p]
    (.-fst p))

  (-val [p]
    (.-snd p)))

(def p (Pair. 1 2))
;; => #<Pair 1,2>

(key p)
;; => 1

(val p)
;; => 2

(into {} [p])
;; => {1 2}
```


#### Comparison

For checking whether two or more values are equivalent with `=` we must implement the `IEquiv` protocol. Let's do it for our  `Pair` type:

```clojure
(def p  (Pair. 1 2))
(def p' (Pair. 1 2))
(def p'' (Pair. 1 2))

(= p p')
;; => false

(= p p' p'')
;; => false

(extend-type Pair
  IEquiv
  (-equiv [p other]
    (and (instance? Pair other)
         (= (.-fst p) (.-fst other))
         (= (.-snd p) (.-snd other)))))

(= p p')
;; => true

(= p p' p'')
;; => true
```

////
TODO: IHash?
////

We can also make types comparable. The function `compare` takes two values and returns a negative number if the first is less than the second, 0 if both are equal and 1 if the first is greater than the second. For making our types comparable we must implement  the `IComparable` protocol.

For pairs, comparison will mean checking if the two first values are equal. If they are, the result will be the comparison of the second values. If not, we will return the result of the first comparison:

```clojure
(extend-type Pair
  IComparable
  (-compare [p other]
    (let [fc (compare (.-fst p) (.-fst other))]
      (if (zero? fc)
        (compare (.-snd p) (.-snd other))
        fc))))

(compare (Pair. 0 1) (Pair. 0 1))
;; => 0

(compare (Pair. 0 1) (Pair. 0 2))
;; => -1

(compare (Pair. 1 1) (Pair. 0 2))
;; => 1

(sort [(Pair. 1 1) (Pair. 0 2) (Pair. 0 1)])
;; => (#<Pair 0,1> #<Pair 0,2> #<Pair 1,1>)
```


#### Metadata

The `meta` and `with-meta` functions are also based upon two protocols: `IMeta` and `IWithMeta` respectively.  We can make our own types capable of carrying metadata adding an extra field for holding the metadata and implementing both protocols.

Let's implement a version of `Pair` that can have metadata:

```clojure
(deftype Pair [fst snd meta]
  IMeta
  (-meta [p] meta)

  IWithMeta
  (-with-meta [p new-meta]
    (Pair. fst snd new-meta)))


(def p (Pair. 1 2 {:foo :bar}))
;; => #<Pair 1,2>

(meta p)
;; => {:foo :bar}

(def p' (with-meta p {:bar :baz}))
;; => #<Pair 1,2>

(meta p')
;; => {:bar :baz}
```


#### Interoperability with JavaScript


Since ClojureScript is hosted in a JavaScript VM we often need to convert ClojureScript data structures to JavaScript ones and viceversa. We also may want to make native JS types participate in an abstraction represented by a protocol.


##### Extending JavaScript types

When extending JavaScript objects instead of using JS globals like `js/String`, `js/Date` and such, special symbols are used. This is done for avoiding mutating global JS objects.

The symbols for extending JS types are: `object`, `array`, `number`, `string`, `function`, `boolean` and `nil` is used for the null object. The dispatch of the protocol to native objects uses Google Closure's link:https://google.github.io/closure-library/api/namespace_goog.html#typeOf[goog.typeOf] function. There's a special `default` symbol that can be used for making a default implementation of a protocol for every type.

For illustrating the extension of JS types we are going to define a `MaybeMutable` protocol that'll have a `mutable?` predicate as its only function. Since in JavaScript mutability is the default we'll extend the default JS type returning true from `mutable?`:

```clojure
(defprotocol MaybeMutable
  (mutable? [this] "Returns true if the value is mutable."))

(extend-type default
  MaybeMutable
  (mutable? [_] true))

;; object
(mutable? #js {})
;; => true

;; array
(mutable? #js [])
;; => true

;; string
(mutable? "")
;; => true

;; function
(mutable? (fn [x] x))
;; => true
```

Since fortunately not all JS object's values are mutable we can refine the implementation of `MaybeMutable` for returning `false` for strings and functions.

```clojure
(extend-protocol MaybeMutable
  string
  (mutable? [_] false)

  function
  (mutable? [_] false))


;; object
(mutable? #js {})
;; => true

;; array
(mutable? #js [])
;; => true

;; string
(mutable? "")
;; => false

;; function
(mutable? (fn [x] x))
;; => false
```

There is no special symbol for JavaScript dates so we have to extend `js/Date` directly. The same applies to the rest of the types found in the global `js` namespace.


##### Converting data

For converting values from ClojureScript types to JavaScript ones and viceversa we use the `clj->js` and `js->clj` functions, which are based in the `IEncodeJS` and `IEncodeClojure` protocols respectively.

For the examples we'll use the Set type introduced in ES6. Note that is not available in every JS runtime.


###### From ClojureScript to JS

First of all we'll extend ClojureScript's set type for being able to convert it to JS. By default sets are converted to JavaScript arrays:

```clojure
(clj->js #{1 2 3})
;; => #js [1 3 2]
```

Let's fix it, `clj->js` is supposed to convert values recursively so we'll make sure to convert all the set contents to JS and creating the set with the converted values:

```clojure
(extend-type PersistentHashSet
  IEncodeJS
  (-clj->js [s]
    (js/Set. (into-array (map clj->js s)))))

(def s (clj->js #{1 2 3}))
(es6-iterator-seq (.values s))
;; => (1 3 2)

(instance? js/Set s)
;; => true

(.has s 1)
;; => true
(.has s 2)
;; => true
(.has s 3)
;; => true
(.has s 4)
;; => false
```

The `es6-iterator-seq` is an experimental function in ClojureScript core for obtaining a seq from an ES6 iterable.


###### From JS to ClojureScript


Now it's time to extend the JS set to convert to ClojureScript. As with `clj->js`, `js->clj` recursively converts the value of the data structure:

```clojure
(extend-type js/Set
  IEncodeClojure
  (-js->clj [s options]
    (into #{} (map js->clj (es6-iterator-seq (.values s))))))

(= #{1 2 3}
   (js->clj (clj->js #{1 2 3})))
;; => true

(= #{[1 2 3] [4 5] [6]}
   (js->clj (clj->js #{[1 2 3] [4 5] [6]})))
;; => true
```

Note that there is no one-to-one mapping between ClojureScript and JavaScript values. For example, ClojureScript keywords are converted to JavaScript strings when passed to `clj->js`.


#### Reductions

The `reduce` function is based on the `IReduce` protocol, which enables us to make our own or third-party types reducible. Apart from using them with `reduce` they will automatically work with `transduce` too, which will allow us to make a reduction with a transducer.

The JS array is already reducible in ClojureScript:

```clojure
(reduce + #js [1 2 3])
;; => 6

(transduce (map inc) conj [] [1 2 3])
;; => [2 3 4]
```

However, the new ES6 Set type isn't so let's implement the `IReduce` protocol. We'll get an iterator using the Set's `values` method and convert it to a seq with the `es6-iterator-seq` function; after that we'll delegate to the original `reduce` function to reduce the obtained sequence.

```clojure
(extend-type js/Set
  IReduce
  (-reduce
   ([s f]
     (let [it (.values s)]
       (reduce f (es6-iterator-seq it))))
   ([s f init]
     (let [it (.values s)]
       (reduce f init (es6-iterator-seq it))))))

(reduce + (js/Set. #js [1 2 3]))
;; => 6

(transduce (map inc) conj [] (js/Set. #js [1 2 3]))
;; => [2 3 4]
```

Associative data structures can be reduced with the `reduce-kv` function, which is based in the `IKVReduce` protocol. The main difference between `reduce` and `reduce-kv` is that the latter uses a three-argument function as a reducer, receiving the accumulator, key and value for each item.

Let's look at an example, we will reduce a map to a vector of pairs. Note that, since vectors associate indexes to values, they can also be reduced with `reduce-kv`.

```clojure
(reduce-kv (fn [acc k v]
             (conj acc [k v]))
           []
           {:foo :bar
            :baz :xyz})
;; => [[:foo :bar] [:baz :xyz]]
```

We'll extend the new ES6 map type to support `reduce-kv`, we'll do this by getting a sequence of key-value pairs and calling the reducing function with the accumulator, key and value as positional arguments:

```clojure
(extend-type js/Map
  IKVReduce
  (-kv-reduce [m f init]
   (let [it (.entries m)]
     (reduce (fn [acc [k v]]
               (f acc k v))
             init
             (es6-iterator-seq it)))))

(def m (js/Map.))
(.set m "foo" "bar")
(.set m "baz" "xyz")

(reduce-kv (fn [acc k v]
             (conj acc [k v]))
           []
           m)
;; => [["foo" "bar"] ["baz" "xyz"]]
```

In both examples we ended up delegating to the `reduce` function, which is aware of reduced values and terminates when encountering one. Take into account that if you don't implement these protocols in terms of `reduce` you will have to check for
reduced values for early termination.


#### Asynchrony

There are some types that have the notion of asynchronous computation, the value they represent may not be realized yet. We can ask whether a value is realized using the `realized?` predicate.

Let's ilustrate it with the `Delay` type, which takes a computation and executes it when the result is needed.  When we dereference a delay the computation is run and the delay is realized:

```clojure
(defn computation []
  (println "running!")
  42)

(def d (Delay. computation nil))

(realized? d)
;; => false

(deref d)
;; running!
;; => 42

(realized? d)
;; => true

@d
;; => 42
```

Both `realized?` and `deref` sit atop two protocols: `IPending` and `IDeref`.

ES6 introduced a type that captures the notion of an asynchronous computation that might fail: the Promise.  A Promise represents an eventual value and can be in one of three states:

 - pending: there is still no value available for this computation.
 - rejected: an error occurred and the promise contains a value that indicates
   the error.
 - resolved: the computation succesfully executed and the promise contains a
   value with the result.


Since the ES6 defined interface for promises doesn't support querying its state we'll use Bluebird library's Promise type for the examples. You can use Bluebird's promise type with the link:https://github.com/funcool/promesa[Promesa] library.

First of all we'll add the ability to check if a promise is realized (either resolved or rejected) using the `realized?` predicate. We just have to implement the `IPending` protocol:

```clojure
(require '[promesa.core :as p])

(extend-type js/Promise
  IPending
  (-realized? [p]
    (not (.isPending p))))


(p/promise (fn [resolve reject]))
;; => #<Promise {:status :pending}>

(realized? (p/promise (fn [resolve reject])))
;; => false

(p/resolved 42)
;; => #<Promise {:status :resolved, :value 42}>

(realized? (p/resolved 42))
;; => true

(p/rejected (js/Error. "OH NO"))
;; => #<Promise {:status :rejected, :error #object[Error Error: OH NO]}>

(realized? (p/rejected (js/Error. "OH NO")))
;; => true
```

Now we'll extend promises to be derefable. When a promise that is still pending is dereferenced we will return a special keyword: `:promise/pending`. If it's not we'll just return the value it contains, be it an error or a result:

```clojure
(require '[promesa.core :as pro])

(extend-type js/Promise
  IDeref
  (-deref [p]
    (cond
      (.isPending p)
      :promise/pending

      (.isRejected p)
      (.reason p)

      :else
      (.value p))))

@(p/promise (fn [resolve reject]))
;; => :promise/pending

@(p/resolved 42)
;; => 42

@(p/rejected (js/Error. "OH NO"))
;; => #object[Error Error: OH NO]
```


#### State

The ClojureScript state constructs such as the Atom and the Volatile have different characteristics and semantics, and the operations on them like `add-watch`, `reset!` or `swap!` are backed by protocols.


##### Atom

For ilustrating such protocols we will implement our own simplified version of an `Atom`. It won't support validators nor metadata, but we will be able to:

- `deref` the atom for getting its current value
- `reset!` the value contained in the atom
- `swap!` the atom with a function for transforming its state

`deref` is based on the `IDeref` protocol. `reset!` is based on the `IReset` protocol and `swap!` on `ISwap`. We'll start by defining a data type and a constructor for our atom implementation:

```clojure
(deftype MyAtom [^:mutable state ^:mutable watches]
  IPrintWithWriter
  (-pr-writer [p writer _]
    (-write writer (str "#<MyAtom " (pr-str state) ">"))))

(defn my-atom
  ([]
    (my-atom nil))
  ([init]
    (MyAtom. init {})))

(my-atom)
;; => #<MyAtom nil>

(my-atom 42)
;; => #<MyAtom 42>
```

Note that we've marked both the current state of the atom (`state`) and the map of watchers (`watches`) with the `{:mutable true}` metadata. We'll be modifying them and we're making this explicit with the annotations.

Our `MyAtom` type is not very useful yet, we'll start by implementing the `IDeref` protocol so we can dereference its current value:

```clojure
(extend-type MyAtom
  IDeref
  (-deref [a]
    (.-state a)))

(def a (my-atom 42))

@a
;; => 42
```

Now that we can dereference it we'll implement the `IWatchable` protocol, which will let us add and remove watches to our custom atom. We'll store the watches in the `watches` map of `MyAtom`, associating keys to callbacks.

```clojure
(extend-type MyAtom
  IWatchable
  (-add-watch [a key f]
    (let [ws (.-watches a)]
      (set! (.-watches a) (assoc ws key f))))

  (-remove-watch [a key]
    (let [ws (.-watches a)]
      (set! (.-watches a) (dissoc ws key))))

  (-notify-watches [a oldval newval]
    (doseq [[key f] (.-watches a)]
      (f key a oldval newval))))
```

We can now add watches to our atom but is not very useful since we still can't change it. For incorporating change we have to implement the `IReset` protocol and make sure we notify the watches every time we reset the atom's value.

```clojure
(extend-type MyAtom
  IReset
  (-reset! [a newval]
    (let [oldval (.-state a)]
      (set! (.-state a) newval)
      (-notify-watches a oldval newval)
      newval)))
```

Now let's check that we got it right. We'll add a watch, change the atom's value making sure the watch gets called and then remove it:

```clojure
(def a (my-atom 41))
;; => #<MyAtom 41>

(add-watch a :log (fn [key a oldval newval]
                    (println {:key key
                              :old oldval
                              :new newval})))
;; => #<MyAtom 41>

(reset! a 42)
;; {:key :log, :old 41, :new 42}
;; => 42

(remove-watch a :log)
;; => #<MyAtom 42>

(reset! a 43)
;; => 43
```

Our atom is still missing the swapping functionality so we'll add that now, let's implement the `ISwap` protocol. There are four arities for the `-swap!` method of the protocol since the function passed to `swap!` may take one, two, three or more
arguments:


```clojure
(extend-type MyAtom
  ISwap
  (-swap!
   ([a f]
    (let [oldval (.-state a)
          newval (f oldval)]
      (reset! a newval)))

   ([a f x]
     (let [oldval (.-state a)
           newval (f oldval x)]
       (reset! a newval)))

   ([a f x y]
     (let [oldval (.-state a)
           newval (f oldval x y)]
       (reset! a newval)))

   ([a f x y more]
     (let [oldval (.-state a)
           newval (apply f oldval x y more)]
       (reset! a newval)))))
```

We now have a custom implementation of the atom abstraction, let's test it in the REPL and see if it behaves like we expect:

```clojure
(def a (my-atom 0))
;; => #<MyAtom 0>

(add-watch a :log (fn [key a oldval newval]
                    (println {:key key
                              :old oldval
                              :new newval})))
;; => #<MyAtom 0>

(swap! a inc)
;; {:key :log, :old 0, :new 1}
;; => 1

(swap! a + 2)
;; {:key :log, :old 1, :new 3}
;; => 3

(swap! a - 2)
;; {:key :log, :old 3, :new 1}
;; => 1

(swap! a + 2 3)
;; {:key :log, :old 1, :new 6}
;; => 6


(swap! a + 4 5 6)
;; {:key :log, :old 6, :new 21}
;; => 21

(swap! a * 2)
;; {:key :log, :old 21, :new 42}
;; => 42

(remove-watch a :log)
;; => #<MyAtom 42>
```

We did it! We implemented a version of ClojureScript Atom without support for metadata nor validators, extending it to support such features is left as an exercise for the reader. Note that you'll need to modify the `MyAtom` type for being able to store metadata and a validator.


##### Volatile

Volatiles are simpler than atoms in that they don't support watching for changes. All changes override the previous value much like the mutable variables present in almost every programming language. Volatiles are based on the `IVolatile` protocol that only defines a method for `vreset!`, since `vswap!` is implemented as a macro.

Let's start by creating our own volatile type and constructor:

```clojure
(deftype MyVolatile [^:mutable state]
  IPrintWithWriter
  (-pr-writer [p writer _]
    (-write writer (str "#<MyVolatile " (pr-str state) ">"))))

(defn my-volatile
  ([]
    (my-volatile nil))
  ([v]
    (MyVolatile. v)))

(my-volatile)
;; => #<MyVolatile nil>

(my-volatile 42)
;; => #<MyVolatile 42>
```

Our `MyVolatile` still needs to support dereferencing and reseting it, let's implement `IDeref` and `IVolatile`, which will enable use to use `deref`, `vreset!` and `vswap!` in our custom volatile:

```clojure
(extend-type MyVolatile
  IDeref
  (-deref [v]
    (.-state v))

  IVolatile
  (-vreset! [v newval]
    (set! (.-state v) newval)
    newval))

(def v (my-volatile 0))
;; => #<MyVolatile 42>

(vreset! v 1)
;; => 1

@v
;; => 1

(vswap! v + 2 3)
;; => 6

@v
;; => 6
```


#### Mutation

In the xref:transients[section about transients] we learned about the mutable counterparts of the immutable and persistent data structures that ClojureScript provides. These data structures are mutable, and the operations on them end with a bang (`!`) to make that explicit. As you may have guessed every operation on transients is based on protocols.


##### From persistent to transient and viceversa

We've learned that we can transform a persistent data structure with the `transient` function, which is based on the `IEditableCollection` protocol; for transforming a transient data structure to a persistent one we use `persistent!`, based on
`ITransientCollection`.

Implementing immutable and persistent data structures and their transient counterparts is out of the scope of this book but we recommend taking a look at ClojureScript's data structure implementation if you are curious.


##### Case study: the hodgepodge library

link:https://github.com/funcool/hodgepodge[Hodgepodge] is a ClojureScript library for treating the browser's local and session storage as if it were a transient data structure. It allows you to insert, read and delete ClojureScript data structures without worrying about encoding and decoding them.

Browser's storage is a simple key-value store that only supports strings. Since all of ClojureScript data structures can be dumped into a string and reified from a string using xref:the-reader[the reader] we can store arbitrary ClojureScript data in storage. We can also extend the reader for being able to read custom data types so we're able to put our types in storage and `hodgepodge` will handle the encoding and decoding for us.

We'll start by wrapping the low-level storage methods with functions. The following operations are supported by the storage:

- getting the value corresponding to a key
- setting a key to a certain value
- removing a value given its key
- counting the number of entries
- clearing the storage

Let's wrap them in a more idiomatic API for ClojureScript:

```clojure
(defn contains-key?
  [^js/Storage storage ^string key]
  (let [ks (.keys js/Object storage)
        idx (.indexOf ks key)]
    (>= idx 0)))

(defn get-item
  ([^js/Storage storage ^string key]
     (get-item storage key nil))
  ([^js/Storage storage ^string key ^string default]
     (if (contains-key? storage key)
       (.getItem storage key)
       default)))

(defn set-item
  [^js/Storage storage ^string key ^string val]
  (.setItem storage key val)
  storage)

(defn remove-item
  [^js/Storage storage ^string key]
  (.removeItem storage key)
  storage)

(defn length
  [^js/Storage storage]
  (.-length storage))

(defn clear!
  [^js/Storage storage]
  (.clear storage))
```

Nothing too interesting going on there, we just wrapped the storage methods in a nicer API. Now we will define a couple functions for serializing and deserializing ClojureScript data structures to strings:

```clojure
(require '[cljs.reader :as reader])

(defn serialize [v]
  (binding [*print-dup* true
            *print-readably* true]
    (pr-str v)))

(def deserialize
  (memoize reader/read-string))
```

The `serialize` function is used for converting a ClojureScript data structure into a string using the `pr-str` function, configuring a couple dynamic variables for obtaining the desired behaviour:

- `*print-dup*` is set to `true` for a printed object to preserve its type when read later 
- `*print-readably*` is set to `true` for not converting non-alphanumeric characters to their escape sequences

The `deserialize` function simply invokes the reader's function for reading a string into a ClojureScript data structure: `read-string`. It's memoized for not having to call the reader each time we deserialize a string since we can assume that a repeated string always corresponds to the same data structure.

Now we can start extending the browser's `Storage` type for acting like a transient data structure. Take into account that the `Storage` type is only available in browsers. We'll start by implementing the `ICounted` protocol for counting the items in the storage, we'll simply delegate to our previously defined `length` function:

```clojure
(extend-type js/Storage
  ICounted
  (-count [^js/Storage s]
   (length s)))
```

We want to be able to use `assoc!` and `dissoc!` for inserting and deleting key-value pairs in the storage, as well as the ability to read from it. We'll implement the `ITransientAssociative` protocol for `assoc!`, `ITransientMap` for `dissoc!` and `ILookup`  for reading storage keys.

```clojure
(extend-type js/Storage
  ITransientAssociative
  (-assoc! [^js/Storage s key val]
    (set-item s (serialize key) (serialize val))
    s)

  ITransientMap
  (-dissoc! [^js/Storage s key]
    (remove-item s (serialize key))
    s)

  ILookup
  (-lookup
    ([^js/Storage s key]
       (-lookup s key nil))
    ([^js/Storage s key not-found]
       (let [sk (serialize key)]
         (if (contains-key? s sk)
           (deserialize (get-item s sk))
           not-found)))))
```

Now we're able to perform some operations on either session or local storage, let's give them a try:

```clojure
(def local-storage js/localStorage)
(def session-storage js/sessionStorage)

(assoc! local-storage :foo :bar)

(:foo local-storage)
;; => :bar

(dissoc! local-storage :foo)

(get local-storage :foo)
;; => nil

(get local-storage :foo :default)
;; => :default
```

Finally, we want to be able to use `conj!` and `persistent!` on local storage so we must implement the `ITransientCollection` protocol, let's give it a go:

```clojure
(extend-type js/Storage
  ITransientCollection
  (-conj! [^js/Storage s ^IMapEntry kv]
    (assoc! s (key kv) (val kv))
    s)

  (-persistent! [^js/Storage s]
    (into {}
          (for [i (range (count s))
                :let [k (.key s i)
                      v (get-item s k)]]
            [(deserialize k) (deserialize v)]))))
```

`conj!` simply obtains the key and value from the map entry and delegates to `assoc!`. `persistent!` deserializes every key-value pair in the storage and returns an immutable snapshot of the storage as a ClojureScript map. Let's try it out:

```clojure
(clear! local-storage)

(persistent! local-storage)
;; => {}

(conj! local-storage [:foo :bar])
(conj! local-storage [:baz :xyz])

(persistent! local-storage)
;; => {:foo :bar, :baz :xyz}
```


##### Transient vectors and sets

We've learned about most of the protocols for transient data structures but we're missing two: `ITransientVector` for using `assoc!` on transient vectors and `ITransientSet` for using `disj!` on transient sets.

For illustrating the `ITransientVector` protocol we'll extend the JavaScript array type for making it an associative transient data structure:

```clojure
(extend-type array
  ITransientAssociative
  (-assoc! [arr key val]
    (if (number? key)
      (-assoc-n! arr key val)
      (throw (js/Error. "Array's key for assoc! must be a number."))))

  ITransientVector
  (-assoc-n! [arr n val]
    (.splice arr n 1 val)
    arr))

(def a #js [1 2 3])
;; => #js [1 2 3]

(assoc! a 0 42)
;; => #js [42 2 3]

(assoc! a 1 43)
;; => #js [42 43 3]

(assoc! a 2 44)
;; => #js [42 43 44]
```

For illustrating the `ITransientSet` protocol we'll extend the ES6 Set type for making it a transient set, supporting the `conj!`, `disj!` and `persistent!` operations. Note that we've extended the Set type previously for being able to convert it to ClojureScript and we'll take advantage of that fact.

```clojure
(extend-type js/Set
  ITransientCollection
  (-conj! [s v]
    (.add s v)
    s)

  (-persistent! [s]
   (js->clj s))

  ITransientSet
  (-disjoin! [s v]
    (.delete s v)
    s))

(def s (js/Set.))

(conj! s 1)
(conj! s 1)
(conj! s 2)
(conj! s 2)

(persistent! s)
;; => #{1 2}

(disj! s 1)

(persistent! s)
;; => #{2}
```