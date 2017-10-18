### Transducers

#### Data transformation

ClojureScript offers a rich vocabulary for data transformation in terms of the sequence abstraction, which makes such transformations highly general and composable. Let's see how we can combine several collection processing functions to build new ones. We will be using a simple problem throughout this section: splitting grape clusters, filtering out the rotten ones, and cleaning them. We have a collection of grape clusters like the following:

```clojure
(def grape-clusters
  [{:grapes [{:rotten? false :clean? false}
             {:rotten? true :clean? false}]
    :color :green}
   {:grapes [{:rotten? true :clean? false}
             {:rotten? false :clean? false}]
    :color :black}])
```

We are interested in splittinqg the grape clusters into individual grapes, discarding the rotten ones and cleaning the remaining grapes so they are ready for eating. We are well-equipped in ClojureScript for this data transformation task; we could implement it using the familiar `map`, `filter` and `mapcat` functions:

```clojure
(defn split-cluster
  [c]
  (:grapes c))

(defn not-rotten
  [g]
  (not (:rotten? g)))

(defn clean-grape
  [g]
  (assoc g :clean? true))

(->> grape-clusters
     (mapcat split-cluster)
     (filter not-rotten)
     (map clean-grape))
;; => ({rotten? false :clean? true} {:rotten? false :clean? true})
```

In the above example we succintly solved the problem of selecting and cleaning the grapes, and we can even abstract such transformations by combining the `mapcat`, `filter` and `map` operations using partial application and function composition:

```clojure
(def process-clusters
  (comp
    (partial map clean-grape)
    (partial filter not-rotten)
    (partial mapcat split-cluster)))

(process-clusters grape-clusters)
;; => ({rotten? false :clean? true} {:rotten? false :clean? true})
```

The code is very clean, but it has a few problems. For example, each call to `map`, `filter` and `mapcat` consumes and produces a sequence that, although lazy, generates intermediate results that will be discarded. Each sequence is fed to the next step, which also returns a sequence. Wouldn't be great if we could do the transformation in a single transversal of the `grape-cluster` collection?

Another problem is that even though our `process-clusters` function works with any sequence, we can't reuse it with anything that is not a sequence. Imagine that instead of having the grape cluster collection available in memory it is being pushed to us asynchronously in a stream. In that situation we couldn't reuse `process-clusters` since usually `map`, `filter` and mapcat` have concrete implementations depending on the type.


#### Generalizing to process transformations

The process of mapping, filtering or mapcatting isn't necessarily tied to a concrete type, but we keep reimplementing them for different types. Let's see how we can generalize such processes to be context independent. We'll start by implementing naive versions of `map` and `filter` first to see how they work internally:

```clojure
(defn my-map
  [f coll]
  (when-let [s (seq coll)]
    (cons (f (first s)) (my-map f (rest s)))))

(my-map inc [0 1 2])
;; => (1 2 3)

(defn my-filter
  [pred coll]
  (when-let [s (seq coll)]
    (let [f (first s)
          r (rest s)]
      (if (pred f)
        (cons f (my-filter pred r))
        (my-filter pred r)))))

(my-filter odd? [0 1 2])
;; => (1)
```

As we can see, they both assume that they receive a seqable and return a sequence. Like many recursive functions they can be implemented in terms of the already familiar `reduce` function. Note that functions that are given to reduce receive an accumulator and an input and return the next accumulator. We'll call these types of functions reducing functions from now on.

```clojure
(defn my-mapr
  [f coll]
  (reduce (fn [acc input]         ;; reducing function
            (conj acc (f input)))
          []                      ;; initial value
          coll))                  ;; collection to reduce

(my-mapr inc [0 1 2])
;; => [1 2 3]

(defn my-filterr
  [pred coll]
  (reduce (fn [acc input]         ;; reducing function
            (if (pred input)
              (conj acc input)
              acc))
          []                      ;; initial value
          coll))                  ;; collection to reduce

(my-filterr odd? [0 1 2])
;; => [1]
```

We've made the previous versions more general since using `reduce` makes our functions work on any thing that is reducible, not just sequences. However you may have noticed that, even though `my-mapr` and `my-filterr` don't know anything about the source (`coll`) they are still tied to the output they produce (a vector) both with the initial value of the reduce (`[]`) and the hardcoded `conj` operation in the body of the reducing function. We could have accumulated results in another data structure, for example a lazy sequence, but we'd have to rewrite the functions in order to do so.

How can we make these functions truly generic? They shouldn't know about either the source of inputs they are transforming nor the output that is generated. Have you noticed that `conj` is just another reducing function? It takes an accumulator and an input and returns another accumulator. So, if we parameterise the reducing function that `my-mapr` and `my-filterr` use, they won't know anything about the type of the result they are building. Let's give it a shot:

```clojure
(defn my-mapt
  [f]                         ;; function to map over inputs
  (fn [rfn]                   ;; parameterised reducing function
    (fn [acc input]           ;; transformed reducing function, now it maps `f`!
      (rfn acc (f input)))))

(def incer (my-mapt inc))

(reduce (incer conj) [] [0 1 2])
;; => [1 2 3]

(defn my-filtert
  [pred]                      ;; predicate to filter out inputs
  (fn [rfn]                   ;; parameterised reducing function
    (fn [acc input]           ;; transformed reducing function, now it discards values based on `pred`!
      (if (pred input)
        (rfn acc input)
        acc))))

(def only-odds (my-filtert odd?))

(reduce (only-odds conj) [] [0 1 2])
;; => [1]
```

That's a lot of higher-order functions so let's break it down for a better understanding of what's going on. We'll examine how `my-mapt` works step by step. The mechanics are similar for `my-filtert`, so we'll leave it out for now.

First of all, `my-mapt` takes a mapping function; in the example we are giving it `inc` and getting another function back. Let's replace `f` with `inc` to see what we are building:

```clojure
(def incer (my-mapt inc))
;; (fn [rfn]
;;   (fn [acc input]
;;     (rfn acc (inc input))))
;;               ^^^
```

The resulting function is still parameterised to receive a reducing function to which it will delegate, let's see what happens when we call it with `conj`:

```clojure
(incer conj)
;; (fn [acc input]
;;   (conj acc (inc input)))
;;    ^^^^
```

We get back a reducing function which uses `inc` to transform the inputs and the `conj` reducing function to accumulate the results. In essence, we have defined map as the transformation of a reducing function.  The functions that transform one reducing function into another are called transducers in ClojureScript.

To ilustrate the generality of transducers, let's use different sources and destinations in our call to `reduce`:

```clojure
(reduce (incer str) "" [0 1 2])
;; => "123"

(reduce (only-odds str) "" '(0 1 2))
;; => "1"
```

The transducer versions of `map` and `filter` transform a process that carries inputs from a source to a destination but don't know anything about where the inputs come from and where they end up. In their implementation they contain the _essence_ of what they accomplish, independent of context.

Now that we know more about transducers we can try to implement our own version of `mapcat`. We already have a fundamental piece of it: the `map` transducer. What `mapcat` does is map a function over an input and flatten the resulting structure one
level. Let's try to implement the catenation part as a transducer:

```clojure
(defn my-cat
  [rfn]
  (fn [acc input]
    (reduce rfn acc input)))

(reduce (my-cat conj) [] [[0 1 2] [3 4 5]])
;; => [0 1 2 3 4 5]
```

The `my-cat` transducer returns a reducing function that catenates its inputs into the accumulator. It does so reducing the `input` reducible with the `rfn` reducing function and using the accumulator (`acc`) as the initial value for such reduction. `mapcat` is simply the composition of `map` and `cat`. The order in which transducers are composed may seem backwards but it'll become clear in a moment.

```clojure
(defn my-mapcat
  [f]
  (comp (my-mapt f) my-cat))

(defn dupe
  [x]
  [x x])

(def duper (my-mapcat dupe))

(reduce (duper conj) [] [0 1 2])
;; => [0 0 1 1 2 2]
```

#### Transducers in ClojureScript core

Some of the ClojureScript core functions like `map`, `filter` and `mapcat` support an arity 1 version that returns a transducer. Let's revisit our definition of `process-cluster` and define it in terms of transducers:

```clojure
(def process-clusters
  (comp
    (mapcat split-cluster)
    (filter not-rotten)
    (map clean-grape)))
```

A few things changed since our previous definition `process-clusters`. First of all, we are using the transducer-returning versions of `mapcat`, `filter` and `map` instead of partially applying them for working on sequences.

Also you may have noticed that the order in which they are composed is reversed, they appear in the order they are executed. Note that all `map`, `filter` and `mapcat` return a transducer. `filter` transforms the reducing function returned by `map`, applying the filtering before proceeding; `mapcat` transforms the reducing function returned by `filter`, applying the mapping and catenation before proceeding.

One of the powerful properties of transducers is that they are combined using regular function composition.  What's even more elegant is that the composition of various transducers is itself a transducer! This means that our `process-cluster` is a transducer too, so we have defined a composable and context-independent algorithmic transformation.

Many of the core ClojureScript functions accept a transducer, let's look at some examples with our newly created `process-cluster`:

```clojure
(into [] process-clusters grape-clusters)
;; => [{:rotten? false, :clean? true} {:rotten? false, :clean? true}]

(sequence process-clusters grape-clusters)
;; => ({:rotten? false, :clean? true} {:rotten? false, :clean? true})

(reduce (process-clusters conj) [] grape-clusters)
;; => [{:rotten? false, :clean? true} {:rotten? false, :clean? true}]
```

Since using `reduce` with the reducing function returned from a transducer is so common, there is a function for reducing with a transformation called `transduce`. We can now rewrite the previous call to `reduce` using `transduce`:

```clojure
(transduce process-clusters conj [] grape-clusters)
;; => [{:rotten? false, :clean? true} {:rotten? false, :clean? true}]
```


#### Initialisation

In the last example we provided an initial value to the `transduce` function (`[]`) but we can omit it and get the same result:

```clojure
(transduce process-clusters conj grape-clusters)
;; => [{:rotten? false, :clean? true} {:rotten? false, :clean? true}]
```

What's going on here? How can `transduce` know what initial value use as an accumulator when we haven't specified it? Try calling `conj` without any arguments and see what happens:

```clojure
(conj)
;; => []
```

The `conj` function has a arity 0 version that returns an empty vector but is not the only reducing function that supports arity 0. Let's explore some others:

```clojure
(+)
;; => 0

(*)
;; => 1

(str)
;; => ""

(= identity (comp))
;; => true
```

The reducing function returned by transducers must support the arity 0 as well, which will typically delegate to the transformed reducing function. There is no sensible implementation of the arity 0 for the transducers we have implemented so far, so we'll simply call the reducing function without arguments. Here's how our modified `my-mapt` could look like:

```clojure
(defn my-mapt
  [f]
  (fn [rfn]
    (fn
      ([] (rfn))                ;; arity 0 that delegates to the reducing fn
      ([acc input]
        (rfn acc (f input))))))
```

The call to the arity 0 of the reducing function returned by a transducer will call the arity 0 version of every nested reducing function, eventually calling the outermost reducing function. Let's see an example with our already defined `process-clusters` transducer:

```clojure
((process-clusters conj))
;; => []
```

The call to the arity 0 flows through the transducer stack, eventually calling
`(conj)`.


#### Stateful transducers

So far we've only seen purely functional transducer; they don't have any implicit state and are very predictable. However, there are many data transformation functions that are inherently stateful, like `take`. `take` receives a number `n` of elements to keep and a collection and returns a collection with at most `n` elements.

```clojure
(take 10 (range 100))
;; => (0 1 2 3 4 5 6 7 8 9)
```

Let's step back for a bit and learn about the early termination of the `reduce` function. We can wrap an accumulator in a type called `reduced` for telling `reduce` that the reduction process should terminate immediately. Let's see an example of a reduction that aggregates the inputs in a collection and finishes as soon as there are 10 elements in the accumulator:

```clojure
(reduce (fn [acc input]
          (if (= (count acc) 10)
            (reduced acc)
            (conj acc input)))
         []
         (range 100))
;; => [0 1 2 3 4 5 6 7 8 9]
```

Since transducers are modifications of reducing functions they also use `reduced` for early termination.  Note that stateful transducers may need to do some cleanup before the process terminates, so they must support an arity 1 as a "completion" step. Usually, like with arity 0, this arity will simply delegate to the transformed reducing function's arity 1.

Knowing this we are able to write stateful transducers like `take`. We'll be using mutable state internally for tracking the number of inputs we have seen so far, and wrap the accumulator in a `reduced` as soon as we've seen enough elements:

```clojure
(defn my-take
  [n]
  (fn [rfn]
    (let [remaining (volatile! n)]
      (fn
        ([] (rfn))
        ([acc] (rfn acc))
        ([acc input]
          (let [rem @remaining
                nr (vswap! remaining dec)
                result (if (pos? rem)
                         (rfn acc input)   ;; we still have items to take
                         acc)]             ;; we're done, acc becomes the result
            (if (not (pos? nr))
              (ensure-reduced result)      ;; wrap result in reduced if not already
              result)))))))
```

This is a simplified version of the `take` function present in ClojureScript core. There are a few things to note here so let's break it up in pieces to understand it better.

The first thing to notice is that we are creating a mutable value inside the transducer. Note that we don't create it until we receive a reducing function to transform. If we created it before returning the transducer we couldn't use `my-take` more than once. Since the transducer is handed a reducing function to transform each time it is used, we can use it multiple times and the mutable variable will be created in every use.

```clojure
(fn [rfn]
  (let [remaining (volatile! n)] ;; make sure to create mutable variables inside the transducer
    (fn
      ;; ...
)))

(def take-five (my-take 5))

(transduce take-five conj (range 100))
;; => [0 1 2 3 4]

(transduce take-five conj (range 100))
;; => [0 1 2 3 4]
```

Let's now dig into the reducing function returned from `my-take`. First of all we `deref` the volatile to get the number of elements that remain to be taken and decrement it to get the next remaining value.  If there are still remaining items to take, we call `rfn` passing the accumulator and input; if not, we already have the final result.

```clojure
([acc input]
  (let [rem @remaining
        nr (vswap! remaining dec)
        result (if (pos? rem)
                 (rfn acc input)
                 acc)]
    ;; ...
))
```

The body of `my-take` should be obvious by now. We check whether there are still items to be processed using the next remainder (`nr`) and, if not, wrap the result in a `reduced` using the `ensure-reduced` function. `ensure-reduced` will wrap the value in a `reduced` if it's not reduced already or simply return the value if it's already reduced. In case we are not done yet, we return the accumulated `result` for further processing.

```clojure
(if (not (pos? nr))
  (ensure-reduced result)
  result)
```

We've seen an example of a stateful transducer but it didn't do anything in its completion step. Let's see an example of a transducer that uses the completion step to flush an accumulated value. We'll implement a simplified version of `partition-all`, which given a `n` number of elements converts the inputs in vectors of size `n`. For understanding its purpose better let's see what the arity 2 version gives us when providing a number and a collection:

```clojure
(partition-all 3 (range 10))
;; => ((0 1 2) (3 4 5) (6 7 8) (9))
```

The transducer returning function of `partition-all` will take a number `n` and return a transducer that groups inputs in vectors of size `n`. In the completion step it will check if there is an accumulated result and, if so, add it to the result. Here's a simplified version of ClojureScript core `partition-all` function, where `array-list` is a wrapper for a mutable JavaScript array:

```clojure
(defn my-partition-all
  [n]
  (fn [rfn]
    (let [a (array-list)]
      (fn
        ([] (rfn))
        ([result]
          (let [result (if (.isEmpty a)                  ;; no inputs accumulated, don't have to modify result
                         result
                         (let [v (vec (.toArray a))]
                           (.clear a)                    ;; flush array contents for garbage collection
                           (unreduced (rfn result v))))] ;; pass to `rfn`, removing the reduced wrapper if present
            (rfn result)))
        ([acc input]
          (.add a input)
          (if (== n (.size a))                           ;; got enough results for a chunk
            (let [v (vec (.toArray a))]
              (.clear a)
              (rfn acc v))                               ;; the accumulated chunk becomes input to `rfn`
            acc))))))

(def triples (my-partition-all 3))

(transduce triples conj (range 10))
;; => [[0 1 2] [3 4 5] [6 7 8] [9]]
```


#### Eductions

Eductions are a way to combine a collection and one or more transformations that can be reduced and iterated over, and that apply the transformations every time we do so. If we have a collection that we want to process and a transformation over it that we want others to extend, we can hand them a eduction, encapsulating the source collection and our transformation. We can create an eduction with the `eduction` function:

```clojure
(def ed (eduction (filter odd?) (take 5) (range 100)))

(reduce + 0 ed)
;; => 25

(transduce (partition-all 2) conj ed)
;; => [[1 3] [5 7] [9]]
```


#### More transducers in ClojureScript core

We learned about `map`, `filter`, `mapcat`, `take` and `partition-all`, but there are a lot more transducers available in ClojureScript. Here is an incomplete list of some other intersting ones:

- `drop` is the dual of `take`, dropping up to `n` values before passing inputs to the reducing function
- `distinct` only allows inputs to occur once
- `dedupe` removes succesive duplicates in input values

We encourage you to explore ClojureScript core to see what other transducers are out there.

#### Defining our own transducers

There a few things to consider before writing our own transducers so in this section we will learn how to properly implement one. First of all, we've learned that the general structure of a transducer is the following:

```clojure
(fn [xf]
  (fn
    ([]          ;; init
      ...)
    ([r]         ;; completion
      ...)
    ([acc input] ;; step
      ...)))
```

Usually only the code represented with `...` changes between transducers, these are the invariants that must be preserved in each arity of the resulting function:

 * arity 0 (init): must call the arity 0 of the nested transform `xf`
 * arity 1 (completion): used to produce a final value and potentially flush state,
   must call the arity 1 of the nested transform `xf` *exactly once*
 * arity 2 (step): the resulting reducing function which will call the arity 2 of
   the nested transform `xf` zero, one or more times


#### Transducible processes

A transducible process is any process that is defined in terms of a succession of steps ingesting input values.  The source of input varies from one process to another. Most of our examples dealt with inputs from a collection or a lazy sequence, but it could be an asynchronous stream of values or a `core.async` channel. The outputs produced by each step are also different for every process; `into` creates a collection with every output of the transducer, `sequence` yields a lazy sequence, and asynchronous streams would probably push the outputs to their listeners.

In order to improve our understanding of transducible processes, we're going to implement an unbounded queue, since adding values to a queue can be thought in terms of a succession of steps ingesting input. First of all we'll define a protocol and a
data type that implements the unbounded queue:

```clojure
(defprotocol Queue
  (put! [q item] "put an item into the queue")
  (take! [q] "take an item from the queue")
  (shutdown! [q] "stop accepting puts in the queue"))

(deftype UnboundedQueue [^:mutable arr ^:mutable closed]
  Queue
  (put! [_ item]
    (assert (not closed))
    (assert (not (nil? item)))
    (.push arr item)
    item)
  (take! [_]
    (aget (.splice arr 0 1) 0))
  (shutdown! [_]
    (set! closed true)))
```

We defined the `Queue` protocol and as you may have noticed the implementation of `UnboundedQueue` doesn't know about transducers at all. It has a `put!` operation as its step function and we're going to implement the transducible process on top of that interface:

```clojure
(defn unbounded-queue
  ([]
   (unbounded-queue nil))
  ([xform]
   (let [put! (completing put!)
         xput! (if xform (xform put!) put!)
         q (UnboundedQueue. #js [] false)]
     (reify
       Queue
       (put! [_ item]
         (when-not (.-closed q)
           (let [val (xput! q item)]
             (if (reduced? val)
               (do
                 (xput! @val)  ;; call completion step
                 (shutdown! q) ;; respect reduced
                 @val)
               val))))
       (take! [_]
         (take! q))
       (shutdown! [_]
         (shutdown! q))))))
```

As you can see, the `unbounded-queue` constructor uses an `UnboundedQueue` instance internally, proxying the `take!` and `shutdown!` calls and implementing the transducible process logic in the `put!` function. Let's deconstruct it to understand what's going on.

```clojure
(let [put! (completing put!)
      xput! (if xform (xform put!) put!)
      q (UnboundedQueue. #js [] false)]
  ;; ...
)
```

First of all, we use `completing` for adding the arity 0 and arity 1 to the `Queue` protocol's `put!` function.  This will make it play nicely with transducers in case we give this reducing function to `xform` to derive another.  After that, if a
transducer (`xform`) was provided, we derive a reducing function applying the transducer to `put!`.  If we're not handed a transducer we will just use `put!`. `q` is the internal instance of `UnboundedQueue`.

```clojure
(reify
  Queue
  (put! [_ item]
    (when-not (.-closed q)
      (let [val (xput! q item)]
        (if (reduced? val)
          (do
            (xput! @val)  ;; call completion step
            (shutdown! q) ;; respect reduced
            @val)
          val))))
  ;; ...
)
```

The exposed `put!` operation will only be performed if the queue hasn't been shut down. Notice that the `put!` implementation of `UnboundedQueue` uses an assert to verify that we can still put values to it and we don't want to break that invariant. If the queue isn't closed we can put values into it, we use the possibly transformed `xput!` for doing so.

If the put operation gives back a reduced value it's telling us that we should terminate the transducible process.  In this case that means shutting down the queue to not accept more values. If we didn't get a reduced value we can happily continue accepting puts.

Let's see how our queue behaves without transducers:

```clojure
(def q (unbounded-queue))
;; => #<[object Object]>

(put! q 1)
;; => 1
(put! q 2)
;; => 2

(take! q)
;; => 1
(take! q)
;; => 2
(take! q)
;; => nil
```

Pretty much what we expected, let's now try with a stateless transducer:

```clojure
(def incq (unbounded-queue (map inc)))
;; => #<[object Object]>

(put! incq 1)
;; => 2
(put! incq 2)
;; => 3

(take! incq)
;; => 2
(take! incq)
;; => 3
(take! incq)
;; => nil
```

To check that we've implemented the transducible process, let's use a stateful transducer. We'll use a transducer that will accept values while they aren't equal to 4 and will partition inputs in chunks of 2 elements:

```clojure
(def xq (unbounded-queue (comp
                           (take-while #(not= % 4))
                           (partition-all 2))))

(put! xq 1)
(put! xq 2)
;; => [1 2]
(put! xq 3)
(put! xq 4) ;; shouldn't accept more values from here on
(put! xq 5)
;; => nil

(take! xq)
;; => [1 2]
(take! xq) ;; seems like `partition-all` flushed correctly!
;; => [3]
(take! xq)
;; => nil
```

The example of the queue was heavily inspired by how `core.async` channels use transducers in their internal step. We'll discuss channels and their usage with transducers in a later section.

Transducible processes must respect `reduced` as a way for signaling early termination. For example, building a collection stops when encountering a `reduced` and `core.async` channels with transducers are closed.  The `reduced` value must be unwrapped with `deref` and passed to the completion step, which must be called exactly once.

Transducible processes shouldn't expose the reducing function they generate when calling the transducer with their own step function since it may be stateful and unsafe to use from elsewhere.
