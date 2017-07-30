
### State management

We've learned that one of ClojureScript's fundamental ideas is immutability. Both scalar values and collections are immutable in ClojureScript, except those mutable types present in the JS host like `Date`.

Immutability has many great properties but we are sometimes faced with the need to model values that change over time. How can we achieve this if we can't change data structures in place?


#### Vars

Vars can be redefined at will inside a namespace but there is no way to know *when* they change. The inability to redefine vars from other namespaces is a bit limiting; also, if we are modifying state, we're probably interested in knowing when it occurs.


#### Atoms

ClojureScript gives us the `Atom` type, which is an object containing a value that can be altered at will. Besides altering its value, it also supports observation through watcher functions that can be attached and detached from it and validation for ensuring that the value contained in the atom is always valid.

If we were to model an identity corresponding to a person called Ciri, we could wrap an immutable value containing Ciri's data in an atom. Note that we can get the atom's value with the `deref` function or using its shorthand `@` notation:

```clojure
(def ciri (atom {:name "Cirilla" :lastname "Fiona" :age 20}))
;; #<Atom: {:name "Cirilla", :lastname "Fiona", :age 20}>

(deref ciri)
;; {:name "Cirilla", :lastname "Fiona", :age 20}

@ciri
;; {:name "Cirilla", :lastname "Fiona", :age 20}
```

We can use the `swap!` function on an atom to alter its value with a function. Since Ciri's birthday is today, let's increment her age count:

```clojure
(swap! ciri update :age inc)
;; {:name "Cirilla", :lastname "Fiona", :age 21}

@ciri
;; {:name "Cirilla", :lastname "Fiona", :age 21}
```

The `reset!` functions replaces the value contained in the atom with a new one:

```clojure
(reset! ciri {:name "Cirilla", :lastname "Fiona", :age 22})
;; {:name "Cirilla", :lastname "Fiona", :age 22}

@ciri
;; {:name "Cirilla", :lastname "Fiona", :age 22}
```

##### Observation

We can add and remove watcher functions for atoms. Whenever the atom's value is changed through a `swap!` or `reset!`, all the atom's watcher functions will be called. Watchers are added with the `add-watch` function. Notice that each watcher has a key associated (`:logger` in the example) to it which is later used to remove the watch from the atom.

```clojure
(def a (atom))

(add-watch a :logger (fn [key the-atom old-value new-value]
                       (println "Key:" key "Old:" old-value "New:" new-value)))

(reset! a 42)
;; Key: :logger Old: nil New: 42
;; => 42

(swap! a inc)
;; Key: :logger Old: 42 New: 43
;; => 43

(remove-watch a :logger)
```


#### Volatiles

Volatiles, like atoms, are objects containing a value that can be altered. However, they don't provide the observation and validation capabilities that atoms provide. This makes them slightly more performant and a more suitable mutable container to use inside stateful functions that don't need observation nor validation.

Their API closely resembles that of atoms. They can be dereferenced to grab the value they contain and support swapping and resetting with `vswap!` and `vreset!` respectively:

```clojure
(def ciri (volatile! {:name "Cirilla" :lastname "Fiona" :age 20}))
;; #<Volatile: {:name "Cirilla", :lastname "Fiona", :age 20}>

(volatile? ciri)
;; => true

(deref ciri)
;; {:name "Cirilla", :lastname "Fiona", :age 20}

(vswap! ciri update :age inc)
;; {:name "Cirilla", :lastname "Fiona", :age 21}

(vreset! ciri {:name "Cirilla", :lastname "Fiona", :age 22})
;; {:name "Cirilla", :lastname "Fiona", :age 22}
```

Note that another difference with atoms is that the constructor of volatiles uses a bang at the end. You create volatiles with `volatile!` and atoms with `atom`.
