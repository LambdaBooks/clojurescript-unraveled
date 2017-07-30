### Getting Started with the Compiler

At this point, you are surely very bored with the constant theoretical explanations about the language itself and will want to write and execute some code. The goal of this section is to provide a little practical introduction to the _ClojureScript_ compiler.

The _ClojureScript_ compiler takes the source code that has been split over numerous directories and namespaces and compiles it down to JavaScript. Today, JavaScript has a great number of different environments where it can be executed - each with its own peculiarities.

This chapter intends to explain how to use _ClojureScript_ without any additional tooling.  This will help you understand how the compiler works and how you can use it when other tooling is not available (such as link:http://leiningen.org/[leiningen] + link:https://github.com/emezeske/lein-cljsbuild[cljsbuild] or link:http://boot-clj.com/[boot]).


#### Execution environments

What is an execution environment? An execution environment is an engine where JavaScript can be executed. For example, the most popular execution environment is a browser (Chrome, Firefox, ...) followed by the second most popular - link:https://nodejs.org/[nodejs].

There are others, such as Rhino (JDK 6+), Nashorn (JDK 8), QtQuick (QT),... but none of them have significant differences from the first two. So, _ClojureScript_ at the moment may compile code to run in the browser or in nodejs-like environments out of the box.


#### Download the compiler

Although the _ClojureScript_ is self hosted, the best way to use it is just using the JVM implementation. To use it, you should have jdk8 installed. _ClojureScript_ itself only requires JDK 7, but the standalone compiler that we are going to use in this chapter requires JDK 8, which can be found at http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

You can download the latest _ClojureScript_ compiler using `wget`:

```bash
wget https://github.com/clojure/clojurescript/releases/download/r1.9.36/cljs.jar
```

The _ClojureScript_ compiler is packaged in a standalone executable jar file, so this is the only file (along with JDK 8) that you need to compile your _ClojureScript_ source code to JavaScript.


#### Compile for Node.js

Let’s start with a practical example compiling code that will target *Node.js* (hereafter simply "nodejs"). For this example, you should have nodejs installed.

There are different ways to install nodejs, but the recommended way is using nvm ("Node.js Version Manager"). You can read the instructions on how to install and use nvm on its link:https://github.com/creationix/nvm[home page].

When you have installed nvm, follow installing the latest version of nodejs:

```shell
nvm install v6.2.0
nvm alias default v6.2.0
```

You can test if *nodejs* is installed in your system with this command:

```shell
$ node --version
v6.2.0
```


##### Create the example application

For the first step of our practical example, we will create our application directory structure and populate it with example code.

Start by creating the directory tree structure for our “hello world” application:

```bash
mkdir -p myapp/src/myapp
touch myapp/src/myapp/core.cljs
```

Resulting in this directory tree:

```text
myapp
└── src
    └── myapp
        └── core.cljs
```


Second, write the example code into the previously created
`myapp/src/myapp/core.cljs` file:

```clojure
(ns myapp.core
  (:require [cljs.nodejs :as nodejs]))

(nodejs/enable-util-print!)

(defn -main
  [& args]
  (println "Hello world!"))

(set! *main-cli-fn* -main)
```

NOTE: It is very important that the declared namespace in the file exactly matches the directory structure. This is the way _ClojureScript_ structures its source code.


##### Compile the example application

In order to compile that source code, we need a simple build script that tells the _ClojureScript_ compiler the source directory and the output file. _ClojureScript_ has a lot of other options, but at this moment we can ignore that.

Let’s create the _myapp/build.clj_ file with the following content:

```clojure
(require '[cljs.build.api :as b])

(b/build "src"
 {:main 'myapp.core
  :output-to "main.js"
  :output-dir "out"
  :target :nodejs
  :verbose true})
```

This is a brief explanation of the compiler options used in this example:

* The `:output-to` parameter indicates to the compiler the destination of the compiled code, in this case to the "main.js" file.
* The `:main` property indicates to the compiler the namespace that will act as the entry point of your application when it's executed.
* The `:target` property indicates the platform where you want to execute the compiled code. In this case, we are going to use *nodejs*. If you omit this parameter, the source will be compiled to run in the browser environment.

To run the compilation, just execute the following command:

```bash
cd myapp
java -cp ../cljs.jar:src clojure.main build.clj
```

And when it finishes, execute the compiled file using *node*:

```shell
$ node main.js
Hello world!
```


#### Compile for the Browser

In this section we are going to create an application similar to the "hello world" example from the previous section to run in the browser environment. The minimal requirement for this application is just a browser that can execute JavaScript.

The process is almost the same, and the directory structure is the same. The only things that changes is the entry point of the application and the build script. So, start re-creating the directory tree from previous example in a different directory.

```bash
mkdir -p mywebapp/src/mywebapp
touch mywebapp/src/mywebapp/core.cljs
```

Resulting in this directory tree:

```text
mywebapp
└── src
    └── mywebapp
        └── core.cljs
```

Then, write new content to the `mywebapp/src/mywebapp/core.cljs` file:

```clojure
(ns mywebapp.core)

(enable-console-print!)

(println "Hello world!")
```

In the browser environment we do not need a specific entry point for the application, so the entry point is the entire namespace.


##### Compile the example application

In order to compile the source code to run properly in a browser, overwrite the _mywebapp/build.clj_ file with the following content:

```clojure
(require '[cljs.build.api :as b])

(b/build "src"
 {:output-to "main.js"
  :output-dir "out/"
  :source-map true
  :main 'mywebapp.core
  :verbose true
  :optimizations :none})
```

This is a brief explanation of the compiler options we're using:

* The `:output-to` parameter indicates to the compiler the destination of the compiled code, in this case the "main.js" file.
* The `:main` property indicates to the compiler the namespace that will act as the  entry point of your application when it's executed.
* `:source-map` indicates the destination of the source map. (The source map connects the ClojureScript source to the generated JavaScript so that error messages can point you back to the original source.)
* `:output-dir` indicates the destination directory for all file sources used in a compilation. It is just for making source maps work properly with the rest of the code, not only your source.
* `:optimizations` indicates the compilation optimization. There are different values for this option, but that will be covered in subsequent sections in more detail.

To run the compilation, just execute the following command:

```bash
cd mywebapp;
java -cp ../cljs.jar:src clojure.main build.clj
```

This process can take some time, so do not worry; wait a little bit. The JVM bootstrap with the Clojure compiler is slightly slow. In the following sections, we will explain how to start a watch process to avoid constantly starting and stopping this slow process.

While waiting for the compilation, let's create a dummy HTML file to make it easy to execute our example app in the browser. Create the _index.html_ file with the following content; it goes in the main _mywebapp_ directory.

```html
<!DOCTYPE html>
<html>
  <header>
    <meta charset="utf-8" />
    <title>Hello World from ClojureScript</title>
  </header>
  <body>
    <script src="main.js"></script>
  </body>
</html>
```

Now, when the compilation finishes and you have the basic HTML file you can just open it with your favorite browser and take a look in the development tools console. The "Hello world!" message should appear there.


#### Watch process

You may have already noticed the slow startup time of the _ClojureScript_ compiler. To solve this, the _ClojureScript_ standalone compiler comes with a tool to watch for changes in your source code, and re-compile modified files as soon as they are written to disk

Start by creating another build script, but this time name it _watch.clj_:

```clojure
(require '[cljs.build.api :as b])

(b/watch "src"
 {:output-to "main.js"
  :output-dir "out/"
  :source-map true
  :main 'mywebapp.core
  :optimizations :none})
```

Now, execute the script just like you have in previous sections:

```bash
$ java -cp ../cljs.jar:src clojure.main watch.clj
Building ...
Reading analysis cache for jar:file:/home/niwi/cljsbook/playground/cljs.jar!/cljs/core.cljs
Compiling src/mywebapp/core.cljs
Compiling out/cljs/core.cljs
Using cached cljs.core out/cljs/core.cljs
... done. Elapsed 0.754487937 seconds
Watching paths: /home/niwi/cljsbook/playground/mywebapp/src
```

Go back to the `mywebapp.core` namespace, and change the print text to `"Hello World, Again!"`.  You'll see that the file `src/mywebapp/core.cljs` the file is immediately recompiled, and if you reload `index.html` in your browser the new text is displayed in the developer console.  Another advantage of this method is that it gives a little bit more output.


#### Optimization levels

The _ClojureScript_ compiler has different levels of optimization. Behind the scenes, those compilation levels are coming from the Google Closure Compiler.

A simplified overview of the compilation process is:

1. The reader reads the code and does some analysis. This compiler may raise some warnings during this phase.
2. Then, the _ClojureScript_ compiler emits JavaScript code. The result is one JavaScript output file for each ClojureScript input file.
3. The generated JavaScript files are passed through the Google Closure Compiler which, depending on the optimization level and other options (sourcemaps, output dir output to, ...), generates the final output file(s).

The final output format depends on the optimization level chosen:


##### none

This optimization level causes the generated JavaScript to be written into separate output files for each namespace, without any additional transformations to the code.


##### whitespace

This optimization level causes the generated JavaScript files to be concatenated into a single output file, in dependency order.  Line breaks and other whitespace are removed.

This reduces compilation speed somewhat, resulting in a slower compilations. However, it is not terribly slow and it is quite usable for small-to-medium sized applications.

##### simple

The simple compilation level builds on the work from the `whitespace` optimization level, and additionally performs optimizations within expressions and functions, such as renaming local variables and function parameters to have shorter names.

Compilation with the `:simple` optimization always preserves the functionality of syntactically valid JavaScript, so it does not interfere with the interaction between the compiled _ClojureScript_ and other JavaScript.


##### advanced

The advanced compilation level builds on the `simple` optimization level, and additionally performs more aggressive optimizations and dead code elimination. This results in a significantly smaller output file.

The `:advanced` optimizations only work for a strict subset of JavaScript which follows the Google Closure Compiler rules.  _ClojureScript_ generates valid JavaScript within this strict subset, but if you are interacting with third party JavaScript code, some additional work is required to make everything work as
expected.

This interaction with third party javascript libraries will be explained in later sections.
