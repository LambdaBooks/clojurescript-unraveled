## Introduction

> Why are we doing this? Because Clojure _rocks_, and JavaScript _reaches_.
>
> — <cite>Rich Hickey</cite>

_ClojureScript_ is an implementation of the Clojure programming language that
targets JavaScript. Because of this, it can run in many different execution
environments including web browsers, Node.js, io.js and Nashorn.

Unlike other languages that intend to _compile_ to JavaScript (like TypeScript,
FunScript, or CoffeeScript), ClojureScript is designed to use JavaScript like
bytecode.  It embraces functional programming and has very safe and consistent
defaults. Its semantics differ significantly from those of JavaScript.

Another big difference (and in our opinion an advantage) over other languages is
that Clojure is designed to be a guest. It is a language without its own virtual
machine that can be easily adapted to the nuances of its execution environment. This
has the benefit that Clojure (and hence ClojureScript) has access to all the
existing libraries written for the host language.

Before we jump in, let us summarize some of the core ideas that ClojureScript brings
to the table. Don't worry if you don't understand all of them right now, they'll
become clear throughout the book.

* ClojureScript enforces the functional programming paradigm with its design
  decisions and idioms. Although being strongly opinionated about functional
  programming it's a pragmatic language rather than pursuing theoretical purity.
* Encourages programming with immutable data, offering highly performant and
  state of the art immutable collection implementations.
* It makes a clear distinction of identity and its state, with explicit constructs
  for managing change as a series of immutable values over time.
* It has type-based and value-based polymorphism, elegantly solving the expression
  problem.
* It is a Lisp dialect so programs are written in the programming language's own
  data structures, a property known as _homoiconicity_ that makes metaprogramming
  (programs that write programs) as simple as it can be.

These ideas together have a great influence in the way you design and implement
software, even if you are not using ClojureScript. Functional programming, decoupling
of data (which is immutable) from the operations to transform it, explicit idioms for
managing change over time and polymorphic constructs for programming to abstractions
greatly simplify the systems we write.

> We can make the same exact software we are making today with dramatically simpler
stuff — dramatically simpler languages, tools, techniques, approaches.
>
> — <cite>Rich Hickey</cite>

We hope you enjoy the book and ClojureScript brings the same joy and inspiration that
has brought to us.
