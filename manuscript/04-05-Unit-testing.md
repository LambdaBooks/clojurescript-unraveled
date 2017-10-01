### Unit testing

As you might expect, testing in _ClojureScript_ consists of the same concepts widely
used by other language such as Clojure, Java, Python, JavaScript, etc.

Regardless of the language, the main objective of unit testing is to run some test
cases, verifying that the code under test behaves as expected and returns without
raising unexpected exceptions.

The immutablity of _ClojureScript_ data structures helps to make programs less error
prone, and facilitates testing a little bit. Another advantage of _ClojureScript_ is
that it tends to use plain data instead of complex objects. Building "mock" objects
for testing is thus greatly simplified.


#### First steps

The "official" _ClojureScript_ testing framework is in the "cljs.test" namespace. It is a very simple library, but it should be more than enough for our purposes.

There are other libraries that offer additional features or directly different approaches to testing, such as link:https://github.com/clojure/test.check[test.check]. However, we will not cover them here.

Start creating a new project using the *mies* leiningen template for experimenting with tests:

```bash
$ lein new mies mytestingapp
$ cd mytestingapp
```

This project will contain the same layout as we have seen in the *dependency management* subchapter, so we won’t explain it again.

The next step is a creating a directory tree for our tests:

```bash
$ mkdir -p test/mytestingapp
$ touch test/mytestingapp/core_tests.cljs
```

Also, we should adapt the existing `watch.clj` script to work with this newly created test directory:

```clojure
(require '[cljs.build.api :as b])

(b/watch (b/inputs "test" "src")
  {:main 'mytestingapp.core_tests
   :target :nodejs
   :output-to "out/mytestingapp.js"
   :output-dir "out"
   :verbose true})
```

This new script will compile and watch both directories "src" and "test", and it sets the new entry point to the `mytestingapp.core_tests` namespace.

Next, put some test code in the `core_tests.cljs` file:

```clojure
(ns mytestingapp.core-tests
  (:require [cljs.test :as t]))

(enable-console-print!)

(t/deftest my-first-test
  (t/is (= 1 2)))

(set! *main-cli-fn* #(t/run-tests))
```

The relevant part of that code snippet is:

```clojure
(t/deftest my-first-test
  (t/is (= 1 2)))
```

The `deftest` macro is a basic primitive for defining our tests. It takes a name as its first parameter, followed by one or multiple assertions using the `is` macro. In this example, we try to assert that `(= 1 2)` is true.

Let's try to run this. First start the watch process:

```bash
$ ./scripts/watch
Building ...
Copying jar:file:/home/niwi/.m2/repository/org/clojure/clojurescript/1.9.36/clojurescript-1.9.36.jar!/cljs/core.cljs to out/cljs/core.cljs
Reading analysis cache for jar:file:/home/niwi/.m2/repository/org/clojure/clojurescript/1.9.36/clojurescript-1.9.36.jar!/cljs/core.cljs
Compiling out/cljs/core.cljs
... done. Elapsed 3.862126827 seconds
Watching paths: /home/niwi/cljsbook/playground/mytestingapp/test, /home/niwi/cljsbook/playground/mytestingapp/src
```

When the compilation is finished, try to run the compiled file with `nodejs`:

```bash
$ node out/mytestingapp.js

Testing mytestingapp.core-tests

FAIL in (my-first-test) (cljs/test.js:374:14)
expected: (= 1 2)
  actual: (not (= 1 2))

Ran 1 tests containing 1 assertions.
1 failures, 0 errors.
```

You can see that the expected assert failure is successfully printed in the console. To fix the test, just change the `=` with `not=` and run the file again:

```bash
$ node out/mytestingapp.js

Testing mytestingapp.core-tests

Ran 1 tests containing 1 assertions.
0 failures, 0 errors.
```

It is fine to test these kinds of assertions, but they are not very useful. Let's go to test some application code. For this, we will use a function to check if a year is a leap year or not. Write the following content to the `src/mytestingapp/core.clj` file:

```clojure
(defn leap?
  [year]
  (and (zero? (js-mod year 4))
       (pos? (js-mod year 100))
       (pos? (js-mod year 400))))
```

Next, write a new test case to check that our new `leap?` function works properly. Make the `core_tests.cljs` file look like:

```clojure
(ns mytestingapp.core-tests
  (:require [cljs.test :as t]
            [mytestingapp.core :as core]))

(enable-console-print!)

(t/deftest my-first-test
  (t/is (not= 1 2)))

(t/deftest my-second-test
  (t/is (core/leap? 1980))
  (t/is (not (core/leap? 1981))))

(set! *main-cli-fn* #(t/run-tests))
```

Run the compiled file again to see that there are now two tests running.  The first test passes as before, and our two new leap year tests pass as well.


#### Async Testing

One of the peculiarities of _ClojureScript_ is that it runs in an asynchronous, single-threaded execution environment, which has its challenges.

In an async execution environment, we should be able to test asynchronous functions. To this end, the _ClojureScript_ testing library offers the `async` macro, allowing
you to create tests that play well with asynchronous code.

First, we need to write a function that works in an asynchronous way. For this purpose, we will create the `async-leap?` predicate that will do the same operation
but asychronously return a result using a callback:

```clojure
(defn async-leap?
  [year callback]
  (js/setImmediate
   (fn []
     (let [result (or (zero? (js-mod year 400))
                      (and (pos? (js-mod year 100))
                           (zero? (js-mod year 4))))]
       (callback result)))))
```

The JavaScript function `setImmediate` is used to emulate an asynchronous task, and the callback is executed with the result of that predicate.

To test it, we should write a test case using the previously mentioned `async` macro:

```clojure
(t/deftest my-async-test
  (t/async done
    (core/async-leap? 1980 (fn [result]
                             (t/is (true? result))
                             (done)))))
```

The `done` function exposed by the `async` macro should be called after the asynchronous operation is finished and all assertions have run.

It is very important to execute the `done` function only once. Omitting it or executing it twice may cause strange behavior and should be avoided.


#### Fixtures

TBD


#### Integrating with CI

Most continuous integration tools and services expect that test scripts you provide return a standard exit code. But the ClojureScript test framework cannot customize this exit code without some configuration, because JavaScript lacks a universal exit code API for ClojureScript to use.

To fix this, the _ClojureScript_ test framework provides an avenue for executing custom code after the tests are done. This is where you are expected to set the environment-specific exit code depending on the final test status: `0` for success,
`1` for failure.

Insert this code at the end of `core_tests.cljs`:

```clojure
(defmethod t/report [::t/default :end-run-tests]
  [m]
  (if (t/successful? m)
    (set! (.-exitCode js/process) 0)
    (set! (.-exitCode js/process) 1)))
```

Now, you may check the exit code of the test script after running:

```bash
$ node out/mytestingapp.js
$ echo $?
```

This code snippet obviously assumes that you are running the tests using *nodejs*. If you are running your script in another execution environment, you should be aware
of how you can set the exit code in that environment and modify the previous snippet accordingly.
