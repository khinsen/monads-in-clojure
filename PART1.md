A monad tutorial for Clojure programmers (part 1)
=================

Monads in functional programming are most often associated with the Haskell
language, where they play a central role in I/O and have found numerous other
uses. Most introductions to monads are currently written for Haskell
programmers. However, monads can be used with any functional language, even
languages quite different from Haskell. Here I want to explain monads in the
context of Clojure, a modern Lisp dialect with strong support for functional
programming. A monad implementation for Clojure is available in the library
clojure.algo.monads. Before trying out the examples given in this tutorial,
type (use 'clojure.algo.monads) into your Clojure REPL. You also have to
install the monad library, of course, which you can do by hand, or using build
tools such as leiningen.

Monads are about composing computational steps into a bigger multi-step
computation. Let’s start with the simplest monad, known as the identity monad
in the Haskell world. It’s actually built into the Clojure language, and you
have certainly used it: it’s the let form.

Consider the following piece of code:

```clj
(let [a  1
      b  (inc a)]
  (* a b))
```
This can be seen as a three-step calculation:

Compute 1 (a constant), and call the result a.
Compute ``(inc a)``, and call the result `b`.
Compute ``(* a b)``, which is the result of the multi-step computation.
Each step has access to the results of all previous steps through the symbols to which their results have been bound.

Now suppose that Clojure didn’t have a let form. Could you still compose
computations by binding intermediate results to symbols? The answer is yes,
using functions. The following expression is in fact equivalent to the
previous one:

```clj ( (fn [a] ( (fn [b] (* a b)) (inc a) ) ) 1 ) ``` The outermost level
defines an anonymous function of a and calls with with the argument 1 – this
is how we bind 1 to the symbol ``a``. Inside the function of ``a``, the same
construct is used once more: the body of ``(fn [a] ...)`` is a function of
``b`` called with argument ``(inc a)``. If you don’t believe that this
somewhat convoluted expression is equivalent to the original let form, just
paste both into Clojure!

Of course the functional equivalent of the let form is not something you would
want to work with. The computational steps appear in reverse order, and the
whole construct is nearly unreadable even for this very small example. But we
can clean it up and put the steps in the right order with a small helper
function, bind. We will call it ``m-bind`` (for monadic bind) right away,
because that’s the name it has in Clojure’s monad library. First, its
definition:

```clj
(defn m-bind [value function]
  (function value))
```

As you can see, it does almost nothing, but it permits to write a value before
the function that is applied to it. Using ``m-bind``, we can write our example
as

```clj
(m-bind 1        (fn [a]
(m-bind (inc a)  (fn [b]
        (* a b)))))
```

That’s still not as nice as the let form, but it comes a lot closer. In fact,
all it takes to convert a let form into a chain of computations linked by
``m-bind`` is a little macro. This macro is called ``domonad``, and it permits
us to write our example as

```clj
(domonad identity-m
  [a  1
   b  (inc a)]
  (* a b))
```

This looks exactly like our original let form. Running ``macroexpand-1`` on it
yields

```clj
(clojure.algo.monads/with-monad identity-m
  (m-bind 1 (fn [a] (m-bind (inc a) (fn [b] (m-result (* a b)))))))
```

This is the expression you have seen above, wrapped in a ``(with-monad
identity-m ...)`` block (to tell Clojure that you want to evaluate it in the
identity monad) and with an additional call to ``m-result`` that I will
explain later. For the identity monad, ``m-result`` is just identity – hence
its name.

As you might guess from all this, monads are generalizations of the let form
that replace the simple ``m-bind`` function shown above by something more
complex. Each monad is defined by an implementation of ``m-bind`` and an
associated implementation of ``m-result``. A ``with-monad`` block simply binds
(using a let form!) these implementations to the names ``m-bind`` and
``m-result``, so that you can use a single syntax for composing computations
in any monad. Most frequently, you will use the ``domonad`` macro for this.

As our second example, we will look at another very simple monad, but one that
adds something useful that you don’t get in a let form. Suppose your
computations can fail in some way, and signal failure by producing nil as a
result. Let’s take our example expression again and wrap it into a function:

```clj
(defn f [x]
  (let [a  x
        b  (inc a)]
    (* a b)))
```

In the new setting of possibly-failing computations, you want this to return
``nil`` when ``x`` is ``nil``, or when ``(inc a)`` yields ``nil``. (Of course
``(inc a)`` will never yield ``nil``, but that’s the nature of examples…)
Anyway, the idea is that whenever a computational step yields ``nil``, the
final result of the computation is ``nil``, and the remaining steps are never
executed. All it takes to get this behaviour is a small change:

```clj
(defn f [x]
  (domonad maybe-m
    [a  x
     b  (inc a)]
    (* a b)))
```

The maybe monad represents computations whose result is maybe a valid value,
but maybe ``nil``. Its ``m-result`` function is still identity, so we don’t
have to discuss ``m-result`` yet (be patient, we will get there in the second
part of this tutorial). All the magic is in the m-bind function:

```clj
(defn m-bind [value function]
  (if (nil? value)
      nil
      (function value)))
```

If its input value is non-nil, it calls the supplied function, just as in the
identity monad. Recall that this function represents the rest of the
computation, i.e. all following steps. If the value is ``nil``, then
``m-bind`` returns ``nil`` and the rest of the computation is never
called. You can thus call ``(f 1)``, yielding 2 as before, but also ``(f
nil)`` yielding ``nil``, without having to add nil-detecting code after every
step of your computation, because ``m-bind`` does it behind the scenes.

In [part 2](PART2.md), I will introduce some more monads, and look at some
generic functions that can be used in any monad to aid in composing
computations.
