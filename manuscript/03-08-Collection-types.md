### Collection types

#### Immutable and persistent

We mentioned before that ClojureScript collections are persistent and immutable, but we didn't explain what that meant.

An immutable data structure, as its name suggests, is a data structure that cannot be changed. In-place updates are not allowed in immutable data structures.

Let's illustrate that with an example: appending values to a vector using the `conj` (conjoin) operation.

```cljs
(let [xs [1 2 3]
      ys (conj xs 4)]
  (println "xs:" xs)
  (println "ys:" ys))

;; xs: [1 2 3]
;; ys: [1 2 3 4]
;; => nil
```

As you can see, we derived a new version of the `xs` vector appending an element to it and got a new vector `ys` with the element added. However, the `xs` vector remained unchanged because it is immutable.

A persistent data structure is a data structure that returns a new version of itself when transforming it, leaving the original unmodified. ClojureScript makes this memory and time efficient using an implementation technique called _structural sharing_, where most of the data shared between two versions of a value is not duplicated and transformations of a value are implemented by copying the minimal amount of data required.

If you want to see an example of how structural sharing works, read on. If you're not interested in more details you can skip over to the xref:the-sequence-abstraction[next section].

For illustrating the structural sharing of ClojureScript data structures, let's compare whether some  parts of the old and new versions of a data structure are actually the same object with the `identical?` predicate. We'll use the list data type for this purpose:

```cljs
(let [xs (list 1 2 3)
      ys (cons 0 xs)]
  (println "xs:" xs)
  (println "ys:" ys)
  (println "(rest ys):" (rest ys))
  (identical? xs (rest ys)))

;; xs: (1 2 3)
;; ys: (0 1 2 3)
;; (rest ys): (1 2 3)
;; => true
```

As you can see in the example, we used `cons` (construct) to prepend a value to the `xs` list and we got a new list `ys` with the element added. The `rest` of the `ys` list (all the values but the first) are the same object in memory as the `xs` list, thus `xs` and `ys` share structure. 


[[the-sequence-abstraction]]
### The sequence abstraction

One of the central ClojureScript abstractions is the _sequence_ which can be thought of as a list and can be derived from any of the collection types. It is persistent and immutable like all collection types, and many of the core ClojureScript functions return sequences.

The types that can be used to generate a sequence are called "seqables"; we can call `seq` on them and get a sequence back. Sequences support two basic operations: `first` and `rest`. They both call `seq` on the argument we provide them:

```cljs
(first [1 2 3])
;; => 1

(rest [1 2 3])
;; => (2 3)
```

Calling `seq` on a seqable can yield different results if the seqable is empty or not. It will return `nil` when empty and a sequence otherwise:

```cljs
(seq [])
;; => nil

(seq [1 2 3])
;; => (1 2 3)
```

`next` is a similar sequence operation to `rest`, but it differs from the latter in that it yields a `nil` value when called with a sequence with one or zero elements. Note that, when given one of the aforementioned sequences, the empty sequence returned by `rest` will evaluate as a boolean true whereas the `nil` value returned by `next` will evaluate as false (xref:truthiness-section[see the section on _truthiness_ later in this chapter]).

```cljs
(rest [])
;; => ()

(next [])
;; => nil

(rest [1 2 3])
;; => (2 3)

(next [1 2 3])
;; => (2 3)
```

##### nil-punning

Since `seq` returns `nil` when the collection is empty, and `nil` evaluates to false in boolean context, you can check to see if a collection is empty by using the `seq` function. The technical term for this is nil-punning.

```cljs
(defn print-coll
  [coll]
  (when (seq coll)
    (println "Saw " (first coll))
    (recur (rest coll))))

(print-coll [1 2 3])
;; Saw 1
;; Saw 2
;; Saw 3
;; => nil

(print-coll #{1 2 3})
;; Saw 1
;; Saw 3
;; Saw 2
;; => nil
```

Though `nil` is neither a seqable nor a sequence, it is supported by all the functions we saw so far:

```cljs
(seq nil)
;; => nil

(first nil)
;; => nil

(rest nil)
;; => ()
```


##### Functions that work on sequences

The ClojureScript core functions for transforming collections make sequences out of their arguments and are implemented in terms of the generic sequence operations we learned about in the preceding section. This makes them highly generic because we can use them on any data type that is seqable. Let's see how we can use `map` with a variety of seqables:

```cljs
(map inc [1 2 3])
;; => (2 3 4)

(map inc #{1 2 3})
;; => (2 4 3)

(map count {:a 41 :b 40})
;; => (2 2)

(map inc '(1 2 3))
;; => (2 3 4)
```

NOTE: When you use the `map` function on a map collection, your higher-order function will receive a two-item vector containing a key and value from the map. The following example uses xref:destructuring-section[destructuring] to access the key and value.      

```cljs
(map (fn [[key value]] (* value value))
     {:ten 10 :seven 7 :four 4})
;; => (100 49 16)
```

Obviously the same operation can be done in more idiomatic way only obtaining a seq of values:

```cljs
(map (fn [value] (* value value))
     (vals {:ten 10 :seven 7 :four 4}))
;; => (100 49 16)
```

As you may have noticed, functions that operate on sequences are safe to use with empty collections or even `nil` values since they don't need to do anything but return an empty sequence when encountering such values.

```cljs
(map inc [])
;; => ()

(map inc #{})
;; => ()

(map inc nil)
;; => ()
```

We already saw examples with the usual suspects like `map`, `filter`, and `reduce`, but ClojureScript offers a plethora of generic sequence operations in its core namespace. Note that many of the operations we'll learn about either work with seqables or are extensible to user-defined types.

We can query a value to know whether it's a collection type with the `coll?` predicate:

```cljs
(coll? nil)
;; => false

(coll? [1 2 3])
;; => true

(coll? {:language "ClojureScript" :file-extension "cljs"})
;; => true

(coll? "ClojureScript")
;; => false
```

Similar predicates exist for checking if a value is a sequence (with `seq?`) or a seqable (with `seqable?`):

```cljs
(seq? nil)
;; => false
(seqable? nil)
;; => false

(seq? [])
;; => false
(seqable? [])
;; => true

(seq? #{1 2 3})
;; => false
(seqable? #{1 2 3})
;; => true

(seq? "ClojureScript")
;; => false
(seqable? "ClojureScript")
;; => false
```

For collections that can be counted in constant time, we can use the `count` operation. This operation also works on strings, even though, as you have seen, they are not collections, sequences, or seqable. 

```cljs
(count nil)
;; => 0

(count [1 2 3])
;; => 3

(count {:language "ClojureScript" :file-extension "cljs"})
;; => 2

(count "ClojureScript")
;; => 13
```

We can also get an empty variant of a given collection with the `empty` function: 

```cljs
(empty nil)
;; => nil

(empty [1 2 3])
;; => []

(empty #{1 2 3})
;; => #{}
```

The `empty?` predicate returns true if the given collection is empty:

```cljs
(empty? nil)
;; => true

(empty? [])
;; => true

(empty? #{1 2 3})
;; => false
```

The `conj` operation adds elements to collections and may add them in different "places" depending on the type of collection. It adds them where it is most performant for the collection type, but note that not every collection has a defined order. 
We can pass as many elements as we want to add to `conj`; let's see it in action:

```cljs
(conj nil 42)
;; => (42)

(conj [1 2] 3)
;; => [1 2 3]

(conj [1 2] 3 4 5)
;; => [1 2 3 4 5]

(conj '(1 2) 0)
;; => (0 1 2)

(conj #{1 2 3} 4)
;; => #{1 3 2 4}

(conj {:language "ClojureScript"} [:file-extension "cljs"])
;; => {:language "ClojureScript", :file-extension "cljs"}
```


##### Laziness

Most of ClojureScript's sequence-returning functions generate lazy sequences instead of eagerly creating a whole new sequence. Lazy sequences generate their contents as they are requested, usually when iterating over them. Laziness ensures that we don't do more work than we need to and gives us the possibility of treating potentially infinite sequences as regular ones.

Consider the `range` function, which generates a range of integers:

```cljs
(range 5)
;; => (0 1 2 3 4)
(range 1 10)
;; => (1 2 3 4 5 6 7 8 9)
(range 10 100 15)
;; (10 25 40 55 70 85)
```

If you just say `(range)`, you will get an infinite sequence of all the integers. Do *not* try this in the REPL, unless you are prepared to wait for a very, very long time, because the REPL wants to fully evaluate the expression. 
Here is a contrived example. Let's say you are writing a graphing program and you are graphing the equation _y_= 2 _x_ ^2^ + 5, and you want only those values of _x_ for which the _y_ value is less than 100. You can generate all the numbers 0 through 100, which will certainly be enough, and then `take-while` the condition holds:

```cljs
(take-while (fn [x] (< (+ (* 2 x x) 5) 100))
            (range 0 100))
;; => (0 1 2 3 4 5 6)
```

#### Collections in depth

Now that we're acquainted with ClojureScript's sequence abstraction and some of the generic sequence manipulating functions, it's time to dive into the concrete collection types and the operations they support.


##### Lists

In ClojureScript, lists are mostly used as a data structure for grouping symbols together into programs. Unlike in other Lisps, many of the syntactic constructs of ClojureScript use data structures different from the list (vectors and maps).  This makes code less uniform, but the gains in readability are well worth the price. 

You can think of ClojureScript lists as singly linked lists, where each node contains a value and a pointer to the rest of the list. This makes it natural (and fast!) to add items to the front of the list, since adding to the end would require traversal of the entire list. The prepend operation is performed using the `cons` function.

```cljs
(cons 0 (cons 1 (cons 2 ())))
;; => (0 1 2)
```

We used the literal `()` to represent the empty list. Since it doesn't contain any symbols, it is not treated as a function call. However, when using list literals that contain elements, we need to quote them to prevent ClojureScript from evaluating them as a function call:

```cljs
(cons 0 '(1 2))
;; => (0 1 2)
```

Since the head is the position that has constant time addition in the list collection, the `conj` operation on lists naturally adds items to the front: 

```cljs
(conj '(1 2) 0)
;; => (0 1 2)
```

Lists and other ClojureScript data structures can be used as stacks using the `peek`, `pop`, and `conj` functions. Note that the top of the stack will be the "place" where `conj` adds elements, making ` onj` equivalent to the stack's push operation. In the case of lists, `conj` adds elements to the front of the list, `peek` returns the first element of the list, and `pop` returns a list with all the elements but the first one.

Note that the two operations that return a stack (`conj` and `pop`) don't change the type of the collection used for the stack.

```cljs
(def list-stack '(0 1 2))

(peek list-stack)
;; => 0

(pop list-stack)
;; => (1 2)

(type (pop list-stack))
;; => cljs.core/List

(conj list-stack -1)
;; => (-1 0 1 2)

(type (conj list-stack -1))
;; => cljs.core/List
```

One thing that lists are not particularly good at is random indexed access. Since they are stored in a single linked list-like structure in memory, random access to a given index requires a linear traversal in order to either retrieve the requested item or throw an index out of bounds error. Non-indexed ordered collections like lazy sequences also suffer from this limitation. 


##### Vectors

Vectors are one of the most common data structures in ClojureScript. They are used as a syntactic construct in many places where more traditional Lisps use lists, for example in function argument declarations and `let` bindings.

ClojureScript vectors have enclosing brackets `[]` in their syntax literals. They can be created with `vector` and from another collection with `vec`:

```cljs
(vector? [0 1 2])
;; => true

(vector 0 1 2)
;; => [0 1 2]

(vec '(0 1 2))
;; => [0 1 2]
```

Vectors are, like lists, ordered collections of heterogeneous values. Unlike lists, vectors grow naturally from the tail, so the `conj` operation appends items to the end of a vector. Insertion on the end of a vector is effectively constant time:

```cljs
(conj [0 1] 2)
;; => [0 1 2]
```

Another thing that differentiates lists and vectors is that vectors are indexed collections and as such support efficient random index access and non-destructive updates. We can use the `nth` function to retrieve values given an index:

```cljs
(nth [0 1 2] 0)
;; => 0
```

Since vectors associate sequential numeric keys (indexes) to values, we can treat them as an associative data structure. ClojureScript provides the `assoc` function that, given an associative data structure and a set of key-value pairs, yields a new data structure with the values corresponding to the keys modified. Indexes begin at zero for the first element in a vector.

```cljs
(assoc ["cero" "uno" "two"] 2 "dos")
;; => ["cero" "uno" "dos"]
```

Note that we can only `assoc` to a key that is either contained in the vector already or if it is the last position in a vector:

```cljs
(assoc ["cero" "uno" "dos"] 3 "tres")
;; => ["cero" "uno" "dos" "tres"]

(assoc ["cero" "uno" "dos"] 4 "cuatro")
;; Error: Index 4 out of bounds [0,3]
```

Perhaps surprisingly, associative data structures can also be used as functions. They are functions of their keys to the values they are associated with. In the case of vectors, if the given key is not present an exception is thrown:

```cljs
(["cero" "uno" "dos"] 0)
;; => "cero"

(["cero" "uno" "dos"] 2)
;; => "dos"

(["cero" "uno" "dos"] 3)
;; Error: Not item 3 in vector of length 3
```

As with lists, vectors can also be used as stacks with the `peek`, `pop`, and `conj` functions. Note, however, that vectors grow from the opposite end of the collection as lists:

```cljs
(def vector-stack [0 1 2])

(peek vector-stack)
;; => 2

(pop vector-stack)
;; => [0 1]

(type (pop vector-stack))
;; => cljs.core/PersistentVector

(conj vector-stack 3)
;; => [0 1 2 3]

(type (conj vector-stack 3))
;; => cljs.core/PersistentVector
```

The `map` and `filter` operations return lazy sequences, but as it is common to need a fully realized sequence after performing those operations, vector-returning counterparts of such functions are available as `mapv` and `filterv`. They have the advantages of being faster than building a vector from a lazy sequence and making your intent more explicit:

```cljs
(map inc [0 1 2])
;; => (1 2 3)

(type (map inc [0 1 2]))
;; => cljs.core/LazySeq

(mapv inc [0 1 2])
;; => [1 2 3]

(type (mapv inc [0 1 2]))
;; => cljs.core/PersistentVector
```

##### Maps

Maps are ubiquitous in ClojureScript. Like vectors, they are also used as a syntactic construct, particularly for attaching xref:metadata-section[metadata] to vars. Any ClojureScript data structure can be used as a key in a map, although it's common to use keywords since they can also be called as functions.

ClojureScript maps are written literally as key-value pairs enclosed in braces `{}`. Alternatively, they can be created with the `hash-map` function:

```cljs
(map? {:name "Cirilla"})
;; => true

(hash-map :name "Cirilla")
;; => {:name "Cirilla"}

(hash-map :name "Cirilla" :surname "Fiona")
;; => {:name "Cirilla" :surname "Fiona"}
```

Since regular maps don't have a specific order, the `conj` operation just adds one or more key-value pairs to a map. `conj` for maps expects one or more sequences of key-value pairs as its last arguments:

```cljs
(def ciri {:name "Cirilla"})

(conj ciri [:surname "Fiona"])
;; => {:name "Cirilla", :surname "Fiona"}

(conj ciri [:surname "Fiona"] [:occupation "Wizard"])
;; => {:name "Cirilla", :surname "Fiona", :occupation "Wizard"}
```

In the preceding example, it just so happens that the order was preserved, but if you have many keys, you will see that the order is not preserved.

Maps associate keys to values and, as such, are an associative data structure. They support adding associations with `assoc` and, unlike vectors, removing them with `dissoc`. `assoc` will also update the value of an existing key. Let's explore these functions:

```cljs
(assoc {:name "Cirilla"} :surname "Fiona")
;; => {:name "Cirilla", :surname "Fiona"}
(assoc {:name "Cirilla"} :name "Alfonso")
;; => {:name "Alfonso"}
(dissoc {:name "Cirilla"} :name)
;; => {}
```

Maps are also functions of their keys, returning the values related to the given keys. Unlike vectors, they return `nil` if we supply a key that is not present in the map:

```cljs
({:name "Cirilla"} :name)
;; => "Cirilla"

({:name "Cirilla"} :surname)
;; => nil
```

ClojureScript also offers sorted hash maps which behave like their unsorted versions but preserve order when iterating over them. We can create a sorted map with default ordering with `sorted-map`:

```cljs
(def sm (sorted-map :c 2 :b 1 :a 0))
;; => {:a 0, :b 1, :c 2}

(keys sm)
;; => (:a :b :c)
```

If we need a custom ordering we can provide a comparator function to `sorted-map-by`, let's see an example inverting the value returned by the built-in `compare` function. Comparator functions take two items to compare and return -1 (if the first item is less than the second), 0 (if they are equal), or 1 (if the first item is greater than the second).

```cljs
(defn reverse-compare [a b] (compare b a))

(def sm (sorted-map-by reverse-compare :a 0 :b 1 :c 2))
;; => {:c 2, :b 1, :a 0}

(keys sm)
;; => (:c :b :a)
```


##### Sets

Sets in ClojureScript have literal syntax as values enclosed in `#{}` and they can be created with the `set` constructor. They are unordered collections of values without duplicates.

```cljs
(set? #{\a \e \i \o \u})
;; => true

(set [1 1 2 3])
;; => #{1 2 3}
```

Set literals cannot contain duplicate values. If you accidentally write a set literal with duplicates an error will be thrown:

```cljs
#{1 1 2 3}
;; clojure.lang.ExceptionInfo: Duplicate key: 1
```

There are many operations that can be performed with sets, although they are located in the `clojure.set` namespace and thus need to be imported. You'll learn xref:namespace-section[the details of namespacing] later; for now, you only need to know that we are loading a namespace called `clojure.set` and binding it to the `s`
symbol.

```cljs
(require '[clojure.set :as s])

(def danish-vowels #{\a \e \i \o \u \æ \ø \å})
;; => #{"a" "e" "å" "æ" "i" "o" "u" "ø"}

(def spanish-vowels #{\a \e \i \o \u})
;; => #{"a" "e" "i" "o" "u"}

(s/difference danish-vowels spanish-vowels)
;; => #{"å" "æ" "ø"}

(s/union danish-vowels spanish-vowels)
;; => #{"a" "e" "å" "æ" "i" "o" "u" "ø"}

(s/intersection danish-vowels spanish-vowels)
;; => #{"a" "e" "i" "o" "u"}
```

A nice property of immutable sets is that they can be nested. Languages that have mutable sets can end up containing duplicate values, but that can't happen in ClojureScript. In fact, all ClojureScript data structures can be nested arbitrarily due to immutability.

Sets also support the generic `conj` operation just like every other collection does.

```cljs
(def spanish-vowels #{\a \e \i \o \u})
;; => #{"a" "e" "i" "o" "u"}

(def danish-vowels (conj spanish-vowels \æ \ø \å))
;; => #{"a" "e" "i" "o" "u" "æ" "ø" "å"}

(conj #{1 2 3} 1)
;; => #{1 3 2}
```

Sets act as read-only associative data that associates the values it contains to themselves. Since every value except `nil` and `false` is truthy in ClojureScript, we can use sets as predicate functions:

```cljs
(def vowels #{\a \e \i \o \u})
;; => #{"a" "e" "i" "o" "u"}

(get vowels \b)
;; => nil

(contains? vowels \b)
;; => false

(vowels \a)
;; => "a"

(vowels \z)
;; => nil

(filter vowels "Hound dog")
;; => ("o" "u" "o")
```

Sets have a sorted counterpart like maps do that are created using the functions `sorted-set` and `sorted-set-by` which are analogous to map's `sorted-map` and `sorted-map-by`.

```cljs
(def unordered-set #{[0] [1] [2]})
;; => #{[0] [2] [1]}

(seq unordered-set)
;; => ([0] [2] [1])

(def ordered-set (sorted-set [0] [1] [2]))
;; =># {[0] [1] [2]}

(seq ordered-set)
;; => ([0] [1] [2])
```



##### Queues

ClojureScript also provides a persistent and immutable queue. Queues are not used as pervasively as other collection types.  They can be created using the `#queue []` literal syntax, but there are no convenient constructor functions for them.

```cljs
(def pq #queue [1 2 3])
;; => #queue [1 2 3]
```

Using `conj` to add values to a queue adds items onto the rear:

```cljs
(def pq #queue [1 2 3])
;; => #queue [1 2 3]

(conj pq 4 5)
;; => #queue [1 2 3 4 5]
```

A thing to bear in mind about queues is that the stack operations don't follow the usual stack semantics (pushing and popping from the same end). `pop` takes values from the front position, and `conj` pushes (appends) elements to the back.

```cljs
(def pq #queue [1 2 3])
;; => #queue [1 2 3]

(peek pq)
;; => 1

(pop pq)
;; => #queue [2 3]

(conj pq 4)
;; => #queue [1 2 3 4]
```

Queues are not as frequently used as lists or vectors, but it is good to know that they are available in ClojureScript, as they may occasionally come in handy. 