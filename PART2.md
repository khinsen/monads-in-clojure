A monad tutorial for Clojure programmers (part 2)
=================

In the first part of this tutorial, I have introduced the two most basic monads: the identity monad and the maybe monad. In this part, I will continue with the sequence monad, which will be the occasion to explain the role of the mysterious ``m-result`` function. I will also show a couple of useful generic monad operations.

One of the most frequently used monads is the sequence monad (known in the Haskell world as the list monad). It is in fact so common that it is built into Clojure as well, in the form of the for form. Let’s look at an example:

```clj
(for [a (range 5)
      b (range a)]
  (* a b))
```

A for form resembles a let form not only syntactically. It has the same structure: a list of binding expressions, in which each expression can use the bindings from the preceding ones, and a final result expressions that typically depends on all the bindings as well. The difference between let and for is that let binds a single value to each symbol, whereas for binds several values in sequence. The expressions in the binding list must therefore evaluate to sequences, and the result is a sequence as well. The for form can also contain conditions in the form of ``:when`` and ``:while`` clauses, which I will discuss later. From the monad point of view of composable computations, the sequences are seen as the results of non-deterministic computations, i.e. computations that have more than one result.

Using the monad library, the above loop is written as

```clj
(domonad sequence-m
  [a (range 5)
   b (range a)]
  (* a b))
```

Since we already know that the domonad macro expands into a chain of ``m-bind`` calls ending in an expression that calls ``m-result``, all that remains to be explained is how ``m-bind`` and ``m-result`` are defined to obtain the desired looping effect.

As we have seen before, ``m-bind`` calls a function of one argument that represents the rest of the computation, with the function argument representing the bound variable. To get a loop, we have to call this function repeatedly. A first attempt at such an ``m-bind`` function would be

```clj
(defn m-bind-first-try [sequence function]
  (map function sequence))
```

Let’s see what this does for our example:

```clj
(m-bind-first-try (range 5)  (fn [a]
(m-bind-first-try (range a)  (fn [b]
(* a b)))))
```

This yields ``(() (0) (0 2) (0 3 6) (0 4 8 12))``, whereas the for form given above yields ``(0 0 2 0 3 6 0 4 8 12)``. Something is not yet quite right. We want a single flat result sequence, but what we get is a nested sequence whose nesting level equals the number of ``m-bind`` calls. Since ``m-bind`` introduces one level of nesting, it must also remove one. That sounds like a job for concat. So let’s try again:

```clj
(defn m-bind-second-try [sequence function]
  (apply concat (map function sequence)))

(m-bind-second-try (range 5)  (fn [a]
(m-bind-second-try (range a)  (fn [b]
(* a b)))))
```

This is worse: we get an exception. Clojure tells us:

```clj 
java.lang.IllegalArgumentException: Don't know how to create ISeq from: Integer
```
Back to thinking!

Our current ``m-bind`` introduces a level of sequence nesting and also takes one away. Its result therefore has as many levels of nesting as the return value of the function that is called. The final result of our expression has as many nesting values as ``(* a b)`` – which means none at all. If we want one level of nesting in the result, no matter how many calls to ``m-bind`` we have, the only solution is to introduce one level of nesting at the end. Let’s try a quick fix:

```clj 
(m-bind-second-try (range 5)  (fn [a]
(m-bind-second-try (range a)  (fn [b]
(list (* a b))))))
```

This works! Our ``(fn [b] ...)`` always returns a one-element list. The inner ``m-bind`` thus creates a sequence of one-element lists, one for each value of ``b``, and concatenates them to make a flat list. The outermost ``m-bind`` then creates such a list for each value of ``a`` and concatenates them to make another flat list. The result of each ``m-bind`` thus is a flat list, as it should be. And that illustrates nicely why we need ``m-result`` to make a monad work. The final definition of the sequence monad is thus given by

```clj
(defn m-bind [sequence function]
  (apply concat (map function sequence)))

(defn m-result [value]
  (list value))
```

The role of ``m-result`` is to turn a bare value into the expression that, when appearing on the right-hand side in a monadic binding, binds the symbol to that value. This is one of the conditions that a pair of m-bind and ``m-result`` functions must fulfill in order to define a monad. Expressed as Clojure code, this condition reads

```clj
(= (m-bind (m-result value) function)
   (function value))
```

There are two more conditions that complete the three monad laws. One of them is

```clj
(= (m-bind monadic-expression m-result)
   monadic-expression)
```

with monadic-expression standing for any expression valid in the monad under consideration, e.g. a sequence expression for the sequence monad. This condition becomes clearer when expressed using the domonad macro:

```clj
(= (domonad
     [x monadic-expression]
      x)
   monadic-expression)
```

The final monad law postulates associativity:

```clj
(= (m-bind (m-bind monadic-expression
                   function1)
           function2)
   (m-bind monadic-expression
           (fn [x] (m-bind (function1 x)
                           function2))))
```

Again this becomes a bit clearer using domonad syntax:

```clj
(= (domonad
     [y (domonad
          [x monadic-expression]
          (function1 x))]
     (function2 y))
   (domonad
     [x monadic-expression
      y (m-result (function1 x))]
     (function2 y)))
```

It is not necessary to remember the monad laws for using monads, they are of importance only when you start to define your own monads. What you should remember about ``m-result`` is that ``(m-result x)`` represents the monadic computation whose result is ``x``. For the sequence monad, this means a sequence with the single element ``x``. For the identity monad and the maybe monad, which I have presented in the first part of the tutorial, there is no particular structure to monadic expressions, and therefore ``m-result`` is just the identity function.

Now it’s time to relax: the most difficult material has been covered. I will return to monad theory in the next part, where I will tell you more about the ``:when`` clauses in for loops. The rest of this part will be of a more pragmatic nature.

You may have wondered what the point of the identity and sequence monads is, given that Clojure already contains fully equivalent forms. The answer is that there are generic operations on computations that have an interpretation in any monad. Using the monad library, you can write functions that take a monad as an argument and compose computations in the given monad. I will come back to this later with a concrete example. The monad library also contains some useful predefined operations for use with any monad, which I will explain now. They all have names starting with the prefix m-.

Perhaps the most frequently used generic monad function is ``m-lift``. It converts a function of n standard value arguments into a function of n monadic expressions that returns a monadic expression. The new function contains implicit ``m-bind`` and ``m-result`` calls. As a simple example, take

```clj
(def nil-respecting-addition
  (with-monad maybe-m
    (m-lift 2 +)))
```

This is a function that returns the sum of two arguments, just like ``+`` does, except that it automatically returns ``nil`` when either of its arguments is ``nil``. Note that ``m-lift`` needs to know the number of arguments that the function has, as there is no way to obtain this information by inspecting the function itself.

To illustrate how ``m-lift`` works, I will show you an equivalent definition in terms of ``domonad``:

```clj
(defn nil-respecting-addition
  [x y]
  (domonad maybe-m
    [a x
     b y]
    (+ a b)))
```

This shows that ``m-lift`` implies one call to ``m-result`` and one ``m-bind`` call per argument. The same definition using the sequence monad would yield a function that returns a sequence of all possible sums of pairs from the two input sequences.

Exercice: The following function is equivalent to a well-known built-in Clojure function. Which one?

```clj
(with-monad sequence-m
  (defn mystery
    [f xs]
    ( (m-lift 1 f) xs )))
```

Another popular monad operation is ``m-seq``. It takes a sequence of monadic expressions, and returns a sequence of their result values. In terms of domonad, the expression ``(m-seq [a b c])`` becomes

```clj
(domonad
  [x a
   y b
   z c]
  '(x y z))
```

Here is an example of how you might want to use it:

```clj
(with-monad sequence-m
   (defn ntuples [n xs]
      (m-seq (replicate n xs))))
```

Try it out for yourself!

The final monad operation I want to mention is ``m-chain``. It takes a list of one-argument computations, and chains them together by calling each element of this list with the result of the preceding one. For example, ``(m-chain [a b c])`` is equivalent to

```clj
(fn [arg]
  (domonad
    [x (a arg)
     y (b x)
     z (c y)]
    z))
```

A usage example is the traversal of hierarchies. The Clojure function parents yields the parents of a given class or type in the hierarchy used for multimethod dispatch. When given a Java class, it returns its base classes. The following function builds on parents to find the n-th generation ascendants of a class:

```clj
(with-monad sequence-m
  (defn n-th-generation
    [n cls]
    ( (m-chain (replicate n parents)) cls )))

(n-th-generation 0 (class []))
(n-th-generation 1 (class []))
(n-th-generation 2 (class []))
```

You may notice that some classes can occur more than once in the result, because they are the base class of more than one class in the generation below. In fact, we ought to use sets instead of sequences for representing the ascendants at each generation. Well… that’s easy. Just replace ``sequence-m`` by ``set-m`` and run it again!


In [part 3](PART3.md), I will come back to the ``:when`` clause in for loops, and show how it is implemented and generalized in terms of monads. I will also explain another monad or two. Stay tuned!
