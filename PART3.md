A monad tutorial for Clojure programmers (part 3)
=================

Before moving on to the more advanced aspects of monads, let’s recapitulate
what defines a monad (see [part 1](PART1.md) and [part 2](PART2.md) for
explanations):

A data structure that represents the result of a computation, or the
computation itself. We haven’t seen an example of the latter case yet, but it
will come soon.  A function ``m-result`` that converts an arbitrary value to a
monadic data structure equivalent to that value.  A function ``m-bind`` that
binds the result of a computation, represented by the monadic data structure,
to a name (using a function of one argument) to make it available in the
following computational step.  Taking the sequence monad as an example, the
data structure is the sequence, representing the outcome of a
non-deterministic computation, ``m-result`` is the function list, which
converts any value into a list containing just that value, and ``m-bind`` is a
function that executes the remaining steps once for each element in a
sequence, and removes one level of nesting in the result.

The three ingredients above are what defines a monad, under the condition that
the three monad laws are respected. Some monads have two additional
definitions that make it possible to perform additional operations. These two
definitions have the names ``m-zero`` and ``m-plus``. ``m-zero`` represents a
special monadic value that corresponds to a computation with no result. One
example is ``nil`` in the maybe monad, which typically represents a failure of
some kind. Another example is the empty sequence in the sequence monad. The
identity monad is an example of a monad that has no ``m-zero``.

``m-plus`` is a function that combines the results of two or more computations
into a single one. For the sequence monad, it is the concatenation of several
sequences. For the maybe monad, it is a function that returns the first of its
arguments that is not ``nil``.

There is a condition that has to be satisfied by the definitions of ``m-zero``
and ``m-plus`` for any monad:

```clj
(= (m-plus m-zero monadic-expression)
   (m-plus monadic-expression m-zero)
   monadic-expression)
```

In words, combining ``m-zero`` with any monadic expression must yield the same
expression. You can easily verify that this is true for the two examples
(maybe and sequence) given above.

One benefit of having an ``m-zero`` in a monad is the possibility to use
conditions. In the first part, I promised to return to the ``:when`` clauses
in Clojure’s for forms, and now the time has come to discuss them. A simple
example is

```clj
(for [a (range 5)
      :when (odd? a)]
  (* 2 a))
```

The same construction is possible with ``domonad``:

```clj
(domonad sequence-m
  [a (range 5)
   :when (odd? a)]
  (* 2 a))
```

Recall that ``domonad`` is a macro that translates a let-like syntax into a
chain of calls to ``m-bind`` ending in a call to ``m-result``. The clause
``a (range 5)`` becomes

```clj
(m-bind (range 5) (fn [a] remaining-steps))
```

where remaining-steps is the transformation of the rest of the domonad form. A
``:when`` clause is of course treated specially, it becomes

```clj
(if predicate remaining-steps m-zero)
```

Our small example thus expands to

```clj
(m-bind (range 5) (fn [a]
  (if (odd? a) (m-result (* 2 a)) m-zero)))
```

Inserting the definitions of ``m-bind``, ``m-result``, and ``m-zero``, we
finally get

```clj
(apply concat (map (fn [a]
  (if (odd? a) (list (* 2 a)) (list))) (range 5)))
```

The result of ``map`` is a sequence of lists that have zero or one elements:
zero for even values (the value of ``m-zero``) and one for odd values
(produced by ``m-result``). ``concat`` makes a single flat list out of this,
which contains only the elements that satisfy the ``:when`` clause.

As for ``m-plus``, it is in practice used mostly with the maybe and sequence
monads, or with variations of them. A typical use would be a search algorithm
(think of a parser, a regular expression search, a database query) that can
succeed (with one or more results) or fail (no results). ``m-plus`` would then
be used to pursue alternative searches and combine the results into one
(sequence monad), or to continue searching until a result is found (maybe
monad). Note that it is perfectly possible in principle to have a monad with
an ``m-zero`` but no ``m-plus``, though in all common cases an ``m-plus`` can
be defined as well if an ``m-zero`` is known.

After this bit of theory, let’s get acquainted with more monads. In the
beginning of this part, I mentioned that the data structure used in a monad
does not always represent the result(s) of a computational step, but sometimes
the computation itself. An example of such a monad is the state monad, whose
data structure is a function.

The state monad’s purpose is to facilitate the implementation of stateful
algorithms in a purely functional way. Stateful algorithms are algorithms that
require updating some variables. They are of course very common in imperative
languages, but not compatible with the basic principle of pure functional
programs which should not have mutable data structures. One way to simulate
state changes while remaining purely functional is to have a special data item
(in Clojure that would typically be a map) that stores the current values of
all mutable variables that the algorithm refers to. A function that in an
imperative program would modify a variable now takes the current state as an
additional input argument and returns an updated state along with its usual
result. The changing state thus becomes explicit in the form of a data item
that is passed from function to function as the algorithm’s execution
progresses. The state monad is a way to hide the state-passing behind the
scenes and write an algorithm in an imperative style that consults and
modifies the state.

The state monad differs from the monads that we have seen before in that its
data structure is a function. This is thus a case of a monad whose data
structure represents not the result of a computation, but the computation
itself. A state monad value is a function that takes a single argument, the
current state of the computation, and returns a vector of length two
containing the result of the computation and the updated state after the
computation. In practice, these functions are typically closures, and what you
use in your program code are functions that create these closures. Such
state-monad-value-generating functions are the equivalent of statements in
imperative languages. As you will see, the state monad allows you to compose
such functions in a way that makes your code look perfectly imperative, even
though it is still purely functional!

Let’s start with a simple but frequent situation: the state that your code
deals with takes the form of a map. You may consider that map to be a
namespace in an imperative languages, with each key defining a variable. Two
basic operations are reading the value of a variable, and modifying that
value. They are already provided in the Clojure monad library, but I will show
them here anyway because they make nice examples.

First, we look at ``fetch-val``, which retrieves the value of a variable:

```clj
(defn fetch-val [key]
  (fn [s]
    [(key s) s]))
```

Here we have a simple state-monad-value-generating function. It returns a
function of a state variable s which, when executed, returns a vector of the
return value and the new state. The return value is the value corresponding to
the key in the map that is the state value. The new state is just the old one
– a lookup should not change the state of course.

Next, let’s look at ``set-val``, which modifies the value of a variable and
returns the previous value:

```clj
(defn set-val [key val]
  (fn [s]
    (let [old-val (get s key)
      new-s   (assoc s key val)]
      [old-val new-s])))
```

The pattern is the same again: ``set-val`` returns a function of state s that,
when executed, returns the old value of the variable plus an updated state map
in which the new value is the given one.

With these two ingredients, we can start composing statements. Let’s define a
statement that copies the value of one variable into another one and returns
the previous value of the modified variable:

```clj
(defn copy-val [from to]
  (domonad state-m
    [from-val   (fetch-val from)
     old-to-val (set-val to from-val)]
    old-to-val))
```

What is the result of ``copy-val?`` A state-monad value, of course: a function
of a state variable s that, when executed, returns the old value of variable
to plus the state in which the copy has taken place. Let’s try it out:

```clj
(let [initial-state        {:a 1 :b 2}
      computation          (copy-val :b :a)
      [result final-state] (computation initial-state)]
  final-state)
```

We get ``{:a 2, :b 2}``, as expected. But how does it work? To understand the
state monad, we need to look at its definitions for ``m-result`` and
``m-bind``, of course.

First, ``m-result``, which does not contain any surprises: it returns a
function of a state variable s that, when executed, returns the result value
``v`` and the unchanged state ``s``:

```clj
(defn m-result [v] (fn [s] [v s]))
```

The definition of ``m-bind`` is more interesting:

```clj
(defn m-bind [mv f]
  (fn [s]
    (let [[v ss] (mv s)]
      ((f v) ss))))
```

Obviously, it returns a function of a state variable ``s``. When that function
is executed, it first runs the computation described by mv (the first
'statement' in the chain set up by ``m-bind``) by applying it to the state
``s``. The return value is decomposed into result ``v`` and new state
``ss``. The result of the first step, ``v``, is injected into the rest of the
computation by calling ``f`` on it (like for the other ``m-bind`` functions
that we have seen). The result of that call is of course another state-monad
value, and thus a function of a state variable. When we are inside our ``(fn
[s] ...)``, we are already at the execution stage, so we have to call that
function on the state ss, the one that resulted from the execution of the
first computational step.

The state monad is one of the most basic monads, of which many variants are in
use. Usually such a variant adds something to ``m-bind`` that is specific to
the kind of state being handled. An example is the the stream monad in
``clojure.contrib.stream-utils``. (NOTE: the stream monad has not been
migrated to the new Clojure contrib library set.) Its state describes a stream
of data items, and the ``m-bind`` function checks for invalid values and for
the end-of-stream condition in addition to what the basic ``m-bind`` of the
state monad does.

A variant of the state monad that is so frequently used that has itself become
one of the standard monads is the writer monad. Its state is an accumulator
(any type implementing th e protocol writer-monad-protocol, for example
strings, lists, vectors, and sets), to which computations can add something by
calling the function write. The name comes from a particularly popular
application: logging. Take a basic computation in the identity monad, for
example (remember that the identity monad is just Clojure’s built-in let). Now
assume you want to add a protocol of the computation in the form of a list or
a string that accumulates information about the progress of the
computation. Just change the identity monad to the writer monad, and add calls
to write where required!

Here is a concrete example: the well-known Fibonacci function in its most
straightforward (and most inefficient) implementation:

```clj
(defn fib [n]
  (if (< n 2)
    n
    (let [n1 (dec n)
          n2 (dec n1)]
      (+ (fib n1) (fib n2)))))
```

Let’s add some protocol of the computation in order to see which calls are
made to arrive at the final result. First, we rewrite the above example a bit
to make every computational step explicit:

```clj
(defn fib [n]
  (if (< n 2)
    n
    (let [n1 (dec n)
          n2 (dec n1)
          f1 (fib n1)
          f2 (fib n2)]
      (+ f1 f2))))
```

Second, we replace let by domonad and choose the writer monad with a vector accumulator:

```clj
(with-monad (writer-m [])

  (defn fib-trace [n]
    (if (< n 2)
      (m-result n)
      (domonad
        [n1 (m-result (dec n))
         n2 (m-result (dec n1))
         f1 (fib-trace n1)
         _  (write [n1 f1])
         f2 (fib-trace n2)
         _  (write [n2 f2])]
        (+ f1 f2))))

)
```

Finally, we run ``fib-trace`` and look at the result:

```clj
(fib-trace 3)
[2 [[1 1] [0 0] [2 1] [1 1]]]
```

The first element of the return value, 2, is the result of the function
fib. The second element is the protocol vector containing the arguments and
results of the recursive calls.

Note that it is sufficient to comment out the lines with the calls to write
and change the monad to ``identity-m`` to obtain a standard ``fib`` function
with no protocol – try it out for yourself!

[Part 4](PART4.md) will show you how to define your own monads by combining
monad building blocks called monad transformers. As an illustration, I will
explain the probability monad and how it can be used for Bayesian estimates
when combined with the maybe-transformer.
