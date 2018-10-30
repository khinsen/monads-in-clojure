A monad tutorial for Clojure programmers (part 4)
=================

In this fourth and last part of my monad tutorial, I will write about monad
transformers. I will deal with only one of them, but it’s a start. I will also
cover the probability monad, and how it can be extended using a monad
transformer.

Basically, a monad transformer is a function that takes a monad argument and
returns another monad. The returned monad is a variant of the one passed in to
which some functionality has been added. The monad transformer defines that
added functionality. Many of the common monads that I have presented before
have monad transformer analogs that add the monad's functionality to another
monad. This makes monads modular by permitting client code to assemble monad
building blocks into a customized monad that is just right for the task at
hand.

Consider two monads that I have discussed before: the maybe monad and the
sequence monad. The maybe monad is for computations that can fail to produce a
valid value, and return ``nil`` in that case. The sequence monad is for
computations that return multiple results, in the form of monadic values that
are sequences. A monad combining the two can take two forms: 1) computations
yielding multiple results, any of which could be ``nil`` indicating failure 2)
computations yielding either a sequence of results or ``nil`` in the case of
failure. The more interesting combination is 1), because 2) is of little
practical use: failure can be represented more easily and with no additional
effort by returning an empty result sequence.

So how can we create a monad that puts the maybe monad functionality inside
sequence monad values? Is there a way we can reuse the existing
implementations of the maybe monad and the sequence monad? It turns out that
this is not possible, but we can keep one and rewrite the other one as a monad
transformer, which we can then apply to the sequence monad (or in fact some
other monad) to get the desired result. To get the combination we want, we
need to turn the maybe monad into a transformer and apply it to the sequence
monad.

First, as a reminder, the definitions of the maybe and the sequence monads:

```clj
(defmonad maybe-m
   [m-zero   nil
    m-result (fn [v] v)
    m-bind   (fn [mv f]
               (if (nil? mv) nil (f mv)))
    m-plus   (fn [& mvs]
               (first (drop-while nil? mvs)))
    ])

(defmonad sequence-m
   [m-result (fn [v]
               (list v))
    m-bind   (fn [mv f]
               (apply concat (map f mv)))
    m-zero   (list)
    m-plus   (fn [& mvs]
               (apply concat mvs))
    ])
```

And now the definition of the maybe monad transformer:

```clj
(defn maybe-t
  [m]
  (monad [m-result (with-monad m m-result)
          m-bind   (with-monad m
                     (fn [mv f]
                       (m-bind mv
                               (fn [x]
                                 (if (nil? x)
                                   (m-result nil)
                                   (f x))))))
          m-zero   (with-monad m m-zero)
          m-plus   (with-monad m m-plus)
          ]))
```

The real definition in clojure.algo.monads is a bit more complicated, and I
will explain the differences later, but for now this basic version is good
enough. The combined monad is constructed by

```clj
(def maybe-in-sequence-m (maybe-t sequence-m))
```

which is a straightforward function call, the result of which is a
monad. Let’s first look at what ``m-result`` does. The ``m-result`` of
``maybe-m`` is the identity function, so we’d expect that our combined monad
``m-result`` is just the one from ``sequence-m``. This is indeed the case, as
``(with-monad m m-result)`` returns the ``m-result`` function from monad
``m``. We see the same construct for ``m-zero`` and ``m-plus``, meaning that
all we need to understand is ``m-bind``.

The combined ``m-bind`` calls the ``m-bind`` of the base monad ``(sequence-m``
in our case``)``, but it modifies the function argument, i.e. the function
that represents the rest of the computation. Before calling it, it first
checks if its argument would be ``nil``. If it isn't, the original function is
called, meaning that the combined monad behaves just like the base monad as
long as no computation ever returns ``nil``. If there is a ``nil`` value, the
maybe monad says that no further computation should take place and that the
final result should immediately be ``nil``. However, we can’t just return
``nil``, as we must return a valid monadic value in the combined monad (in our
example, a sequence of possibly-``nil`` values). So we feed ``nil`` into the
base monad's ``m-result``, which takes care of wrapping up ``nil`` in the
required data structure.

Let's see it in action:

```clj
(domonad maybe-in-sequence-m
  [x [1 2 nil 4]
   y [10 nil 30 40]]
  (+ x y))
```

The output is:

```clj 
(11 nil 31 41 12 nil 32 42 nil 14 nil 34 44)
```

As expected, there are all the combinations of non-``nil`` values in both
input sequences. However, it is surprising at first sight that there are four
``nil`` entries. Shouldn’t there be eight, resulting from the combinations of
a ``nil`` in one sequence with the four values in the other sequence?

To understand why there are four ``nil``s, let’s look again at how the
``m-bind`` definition in ``maybe-t`` handles them. At the top level, it will
be called with the vector ``[1 2 nil 4]`` as the monadic value. It hands this
to the ``m-bind`` of ``sequence-m``, which calls the anonymous function in
``maybe-t``'s ``m-bind`` four times, once for each element of the vector. For
the three non-``nil`` values, no special treatment is added. For the one
``nil`` value, the net result of the computation is ``nil`` and the rest of
the computation is never called. The ``nil`` in the first input vector thus
accounts for one ``nil`` in the result, and the rest of the computation is
called three times. Each of these three rounds produces then three valid
results and one ``nil``. We thus have 3×3 valid results, 3×1 ``nil`` from the
second vector, plus the one ``nil`` from the first vector. That makes nine
valid results and four ``nil``s.

Is there a way to get all sixteen combinations, with all the possible ``nil``
results in the result? Yes, but not using the ``maybe-t`` transformer. You
have to use the maybe and the sequence monads separately, for example like
this:

```clj
(with-monad maybe-m
  (def maybe-+ (m-lift 2 +)))

(domonad sequence-m
  [x [1 2 nil 4]
   y [10 nil 30 40]]
  (maybe-+ x y))
```

When you use ``maybe-t``, you always get the shortcutting behaviour seen
above: as soon as there is a ``nil``, the total result is ``nil`` and the rest
of the computation is never executed. In most situations, that’s what you
want.

The combination of ``maybe-t`` and ``sequence-m`` is not so useful in practice
because a much easier (and more efficient) way to handle invalid results is to
remove them from the sequences before any further processing happens. But the
example is simple and thus fine for explaining the basics. You are now ready
for a more realistic example: the use of ``maybe-t`` with the probability
distribution monad.

The probability distribution monad is made for working with finite probability
distributions, i.e. probability distributions in which a finite set of values
has a non-zero probability. Such a distribution is represented by a map from
the values to their probabilities. The monad and various useful functions for
working with finite distributions is defined in the
library
[clojure.contrib.probabilities.finite-distributions](https://github.com/richhickey/clojure-contrib/blob/master/src/main/clojure/clojure/contrib/probabilities/finite_distributions.clj) (NOTE:
this module has not yet been migrated to the new Clojure contrib library
set.).

A simple example of a finite distribution:

```clj
(use 'clojure.contrib.probabilities.finite-distributions)
(def die (uniform #{1 2 3 4 5 6}))
(prob odd? die)
```

This prints ``1/2``, the probability that throwing a single die yields an odd number. The value of die is the probability distribution of the outcome of throwing a die:

```clj
{6 1/6, 5 1/6, 4 1/6, 3 1/6, 2 1/6, 1 1/6}
```

Suppose we throw the die twice and look at the sum of the two values. What is its probability distribution? That’s where the monad comes in:

```clj
(domonad dist-m
  [d1 die
   d2 die]
  (+ d1 d2))
```

The result is:

```clj 
{2 1/36, 3 1/18, 4 1/12, 5 1/9, 6 5/36, 7 1/6, 8 5/36, 9 1/9, 10 1/12, 11 1/18, 12 1/36}
```

You can read the above ``domonad`` block as 'draw a value from the
distribution die and call it d1, draw a value from the distribution die and
call it d2, then give me the distribution of ``(+ d1 d2)``'. This is a very
simple example; in general, each distribution can depend on the values drawn
from the preceding ones, thus creating the joint distribution of several
variables. This approach is known as 'ancestral sampling'.

The monad ``dist-m`` applies the basic rule of combining probabilities: if
event A has probability p and event B has probability q, and if the events are
independent (or at least uncorrelated), then the probability of the combined
event (A and B) is p×q. Here is the definition of ``dist-m``:

```clj 
(defmonad dist-m
   [m-result (fn [v] {v 1})
    m-bind   (fn [mv f]
               (letfn [(add-prob [dist [x p]]
                         (assoc dist x (+ (get dist x 0) p)))]
                 (reduce add-prob {}
                   (for [[x p] mv  [y q] (f x)]
                     [y (* q p)]))))
    ])
```

As usual, the interesting stuff happens in ``m-bind``. Its first argument,
``mv``, is a map representing a probability distribution. Its second argument,
``f``, is a function representing the rest of the calculation. It is called
for each possible value in the probability distribution in the for form. This
for form iterates over both the possible values of the input distribution and
the possible values of the distribution returned by ``(f x)``, combining the
probabilities by multiplication and putting them into the output
distribution. This is done by reducing over the helper function ``add-prob``,
which checks if the value is already present in the map, and if so, adds the
probability to the previously obtained one. This is necessary because the
samples from the ``(f x)`` distribution can contain the same value more than
once if they were obtained for different ``x``.

For a more interesting example, let’s consider the
famous
[Monty Hall problem](http://en.wikipedia.org/wiki/Monty_Hall_problem). In a
game show, the player faces three doors. A prize is waiting for him behind one
of them, but there is nothing behind the two other ones. If he picks the right
door, he gets the prize. Up to there, the problem is simple: the probability
of winning is ``1/3``.

But there is a twist. After the player makes his choice, the game host open
one of the two other doors, which shows an empty space. He then asks the
player if he wants to change his mind and choose the last remaining door
instead of his initial choice. Is this a good strategy?

To make this a well-defined problem, we have to assume that the game host
knows where the prize is and that he would not open the corresponding
door. Then we can start coding:

```clj
(def doors #{:A :B :C})

(domonad dist-m
  [prize  (uniform doors)
   choice (uniform doors)]
  (if (= choice prize) :win :loose))
```

Let's go through this step by step. First, we choose the prize door by drawing
from a uniform distribution over the three doors ``:A``, ``:B``, and
``:C``. That represents what happens before the player comes in. Then the
player's initial choice is made, drawing from the same distribution. Finally,
we ask for the distribution of the outcome of the game, code>``:win`` or
``:loose``. The answer is, unsurprisingly, ``{:win 1/3, :loose 2/3}``.

This covers the case in which the player does not accept the host's
proposition to change his mind. If he does, the game becomes more complicated:

```clj
(domonad dist-m
  [prize  (uniform doors)
   choice (uniform doors)
   opened (uniform (disj doors prize choice))
   choice (uniform (disj doors opened choice))]
  (if (= choice prize) :win :loose))
```

The third step is the most interesting one: the game host opens a door which
is neither the prize door nor the initial choice of the player. We model this
by removing both prize and choice from the set of doors, and draw uniformly
from the resulting set, which can have one or two elements depending on prize
and choice. The player then changes his mind and chooses from the set of doors
other than the open one and his initial choice. With the standard three-door
game, that set has exactly one element, but the code above also works for a
larger number of doors - try it out yourself!

Evaluating this piece of code yields ``{:loose 1/3, :win 2/3}``, indicating
that the change-your-mind strategy is indeed the better one.

Back to the ``maybe-t`` transformer. The finite-distribution library defines a
second monad by

```clj
(def cond-dist-m (maybe-t dist-m))
```

This makes ``nil`` a special value in distributions, which is used to
represent events that we don't want to consider as possible ones. With the
definitions of ``maybe-t`` and ``dist-m``, you can guess how ``nil`` values
are propagated when distributions are combined: for any ``nil`` value, the
distributions that potentially depend on it are never evaluated, and the
``nil`` value's probability is transferred entirely to the probability of
``nil`` in the output distribution. But how does ``nil`` ever get into a
distribution? And, most of all, what is that good for?

Let's start with the last question. The goal of this ``nil``-containing
distributions is to eliminate certain values. Once the final distribution is
obtained, the ``nil`` value is removed, and the remaining distribution is
normalized to make the sum of the probabilities of the remaining values equal
to one. This ``nil``-removal and normalization is performed by the utility
function normalize-cond. The ``cond-dist-m`` monad is thus a sophisticated way
to compute conditional probabilities, and in particular to facilitate Bayesian
inference, which is an important technique in all kinds of data analysis.

As a first exercice, let's calculate a simple conditional probability from an
input distribution and a predicate. The output distribution should contain
only the values satisfying the predicate, but be normalized to one:

```clj
(defn cond-prob [pred dist]
  (normalize-cond (domonad cond-dist-m
                    [v dist
                     :when (pred v)]
                    v))))
```

The important line is the one with the ``:when`` condition. As I have explained in parts 1 and 2, the ``domonad`` form becomes

```clj
(m-bind dist
        (fn [v]
          (if (pred v)
            (m-result v)
             m-zero)))
```

If you have been following carefully, you should complain now: with the
definitions of ``dist-m`` and ``maybe-t`` I have given above, cond-``dist-m``
should not have any ``m-zero``! But as I said earlier, the ``maybe-t`` shown
here is a simplified version. The real one checks if the base monad has
``m-zero``, and if it hasn't, it substitutes its own, which is ``(with-monad m
(m-result nil))``. Therefore the ``m-zero`` of ``cond-dist-m`` is ``{nil 1}``,
the distribution whose only value is ``nil``.

The net effect of the ``domonad`` form in this example is thus to keep all
values that satisfy the predicate with their initial probabilities, but to
transfer the probability of all values to ``nil``. The call to normalize-cond
then takes out the ``nil`` and re-distributes its probability to the other
values. Example:

```clj
(cond-prob odd? die)
-> {5 1/3, 3 1/3, 1 1/3}
```

The ``cond-dist-m`` monad really becomes interesting for Bayesian inference
problems. Bayesian inference is technique for drawing conclusions from
incomplete observations. It has a wide range of applications, from spam
filters to weather forecasts. For an introduction to the technique and its
mathematical basis, you can start with
the [Wikipedia article](http://en.wikipedia.org/wiki/Bayesian_inference).

Here I will discuss a very simple inference problem and its solution in
Clojure. Suppose someone has three dice, one with six faces, one with eight,
and one with twelve. This person picks one die, throws it a few times, and
gives us the numbers, but doesn't tell us which die it was. Given these
observations, we would like to infer the probabilities for each of the three
dice to have been picked. We start by defining a function that returns the
distribution of a die with n faces:

```clj
(defn die-n [n] (uniform (range 1 (inc n))))
```

Next, we come to the core of Bayesian inference. One central ingredient is the
probability for throwing a given number under the assumption that die X was
used. We thus need the probability distributions for each of our three dice:

```clj
(def dice {:six     (die-n 6)
           :eight   (die-n 8 )
           :twelve  (die-n 12)})
```

The other central ingredient is a distribution representing our 'prior
knowledge' about the chosen die. We actually know nothing at all, so each die
has the same weight in this distribution:

```clj
(def prior (uniform (keys dice)))
```

Now we can write the inference function. It takes as input the prior-knowledge
distribution and a number that was obtained from the die. It returns the a
posteriori distribution that combines the prior information with the
information from the observation.

```clj
(defn add-observation [prior observation]
  (normalize-cond
    (domonad cond-dist-m
      [die    prior
       number (get dice die)
       :when  (= number observation)]
      die)))
```

Let's look at the ``domonad`` form. The first step picks one die according to
the prior knowledge. The second line "throws" that die, obtaining a
number. The third line eliminates the numbers that don't match the
observation. And then we ask for the distribution of the die.

It is instructive to compare this function with the mathematical formula for
Bayes' theorem, which is the basis of Bayesian inference. Bayes' theorem is
P(H|E) = P(E|H) P(H) / P(E), where H stands for the hypothesis ("the die
chosen was X") and E stands for the evidence ("the number thrown was N"). P(H)
is the prior knowledge. The formula must be evaluated for a fixed value of E,
which is the observation.

The first line of our ``domonad`` form implements P(H), the second line
implements P(E|H). These two lines together thus sample P(E, H) using
ancestral sampling, as we have seen before. The ``:when`` line represents the
observation; we wish to apply Bayes' theorem for a fixed value of E. Once E
has been fixed, P(E) is just a number, required for normalization. This is
handled by ``normalize-cond`` in our code.

Let's see what happens when we add a single observation:

```clj
(add-observation prior 1)
-> {:twelve 2/9, :eight 1/3, :six 4/9}
```

We see that the highest probability is given to ``:six``, then ``:eight``, and
finally ``:twelve``. This happens because 1 is a possible value for all dice,
but it is more probable as a result of throwing a six-faced die (1/6) than as
a result of throwing an eight-faced die (1/8) or a twelve-faced die
(1/12). The observation thus favours a die with a small number of faces.

If we have three observations, we can call add-observation repeatedly:

```clj
(-> prior (add-observation 1)
          (add-observation 3)
          (add-observation 7))
-> {:twelve 8/35, :eight 27/35}
```

Now we see that the candidate ``:six`` has disappeared. In fact, the observed
value of 7 rules it out completely. Moreover, the observed numbers strongly
favour ``:eight`` over ``:twelve``, which is again due to the preference for
the smallest possible die in the game.

This inference problem is very similar to how a spam filter works. In that
case, the three dice are replaced by the choices ``:spam`` or
``:no-spam``. For each of them, we have a distribution of words, obtained by
analyzing large quantities of e-mail messages. The function add-observation is
strictly the same, we'd just pick different variable names. And then we'd call
it for each word in the message we wish to evaluate, starting from a prior
distribution defined by the total number of ``:spam`` and ``:no-spam``
messages in our database.

To end this introduction to monad transformers, I will explain the ``m-zero``
problem in ``maybe-t``. As you know, the maybe monad has an ``m-zero``
definition (``nil``) and an ``m-plus`` definition, and those two can be
carried over into a monad created by applying ``maybe-t`` to some base
monad. This is what we have seen in the case of ``cond-dist-m``. However, the
base monad might have its own ``m-zero`` and ``m-plus``, as we have seen in
the case of ``sequence-m``. Which set of definitions should the combined monad
have? Only the user of ``maybe-t`` can make that decision, so ``maybe-t`` has
an optional parameter for this (see its documentation for the details). The
only clear case is a base monad without ``m-zero`` and ``m-plus``; in that
case, nothing is lost if ``maybe-t`` imposes its own.
