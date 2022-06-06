LCF-style ND
============

_See also:_ [Philomath](https://github.com/catseye/Philomath#readme)
∘ [Destructorizers](https://github.com/cpressey/Destructorizers#readme)
∘ [Nested Modal Transducers](https://github.com/cpressey/Nested-Modal-Transducers#readme)

- - - -

This article shows a (perhaps unorthodox) development
of a simple LCF-style theorem prover for propositional logic
in a Natural Deduction system.

First it contains some brief notes and some (I think) interesting
observations about LCF-style theorem provers and Natural Deduction.
It then proceeds to tie them together with some Python-like
pseudo-code, and concludes with a handful of further interesting observations.

This article contains only pseudo-code; for implementations of these
ideas in real programming languages, have a look at projects
such as [**Philomath**](https://github.com/catseye/philomath) (in ANSI C).

### Table of Contents

*   [LCF-style Theorem Proving](#lcf-style-theorem-proving)
*   [Natural Deduction](#natural-deduction)
*   [Pseudo-Code for the Theorem Prover](#pseudo-code-for-the-theorem-prover)

LCF-style Theorem Proving
-------------------------

A brief introduction to LCF-style theorem proving can be
found in [The LCF Approach to Theorem Proving][] (PDF),
slides for a presentation by John Harrison of Intel Corp. in 2001.

An LCF-style theorem prover, as far as I'm concerned, means this:

You have a programming language, and you will write your proofs
directly in this language, as executable programs.

To do this, you have a data type representing valid proofs.

You have some operations that produce trivial valid proofs.

You have some other operations that take valid proofs and
transform them in some way to produce new valid proofs.

You don't provide any other way to produce or mutate such
a data object.  In this way, these data objects are guaranteed
to represent valid proofs, and a program that runs and produces
such a data object as its output, has proved a theorem.

### Some Observations

Now, some observations about LCF-style theorem provers:

(1) This is fundamentally an application of "[Parse, don't Validate][]".

(2) This doesn't rely on a static type system.  Rather, it relies
on _encapsulation_ a.k.a. _information hiding_.  You could write it in
Scheme (see, e.g., "[Information Hiding in Scheme][]"), even though
a type theorist would call Scheme an "untyped" language.  (The rest of
the world would probably call it "dynamically typed" or "latently typed".)

(3) Incidentally, taking 1 and 2 together, one may conclude that
"Parse, don't Validate" _in general_ doesn't rely on a static type
system, either.

(4) Actually preventing the mutation of such a data object on a
_real computer_ is probably impossible.  Can you not always run
the program under `gdb` and alter a few bytes here and there?
It's probably more productive to view it as a proviso: _if_ the
data object was constructed only by these operations and no
others, _then_ it represents a valid proof.  Under such a proviso,
one could even have an LCF-style theorem prover in a language with
only "voluntary" encapsulation, such as Python.

Natural Deduction
-----------------

A good tutorial on Natural Deduction is the first chapter of the book
[Logic in Computer Science][] by Huth and Ryan, which is available
for borrowing from archive.org.

[The IEP article on Natural Deduction][] by Andrzej Indrzejczak
is also a good overview, and I'll follow parts of it closely here.

In particular, it proposes some criteria for distinguishing Natural
Deduction proof systems from other proof systems.  In ND systems,

(1) Assumptions can be freely introduced, and subsequently
discharged under certain conditions.

(2) Reasoning proceeds by successive application of inference
rules.

(3) Rich forms of proof construction are possible.

The first corresponds to LCF-style operations which produce trivial
valid proofs.  The second corresponds to LCF-style operations which
transform valid proofs into other valid proofs.  The third
corresponds to the fact that the data objects representing valid
proofs are processed by a general-purpose programming language, in
which we can build arbitrary contrivances to help us work with them.

### Some Observations

An ND proof may be presented in a tree format or in a list format.
An LCF-style proof in an applicative language is a set of function
applications, which has the shape of a tree.  If we bind each
application to an identifier (as in a `let`-type construct) we obtain
a flattened list, where each proof step (application) may refer to
earlier proof steps.

In the tree format, assumptions are labelled when introduced, so
that when an assumption is discharged, we know which one it is.
In the list format, nested boxes can be used to serve the same purpose.
This is akin to labelled assumptions having _dynamic scope_
versus nested assumptions having _lexical scope_, which will be
demonstrated in one of the sections below.  To simplify the exposition,
our theorem prover will use labelled assumptions.

Pseudo-Code for the Theorem Prover
----------------------------------

It would seem that the ideal language in which to demonstrate the
things I'm trying to demonstrate here would be a dynamically-typed
language with good support for encapsulation.

Unfortunately, such a combination is rarely found in a
programming language (which could be a topic unto itself).
This leaves me a few options:

*   Write it in a dynamically-typed language (like Python or Scheme)
    and ask you to pretend there's actually encapsulation happening;
*   Write it in an encapsulation-supporting language (like C or
    Modula-2) and ask you to pretend the static types aren't actually
    happening;
*   Write it in a dynamically-typed language but
    [employ some contortions to achieve encapsulation][] and
    ask you to ignore them, and hope that they do not distract you;
*   Write it in an object-oriented language with encapsulation, and
    ask you to ignore the types _and_ the object-oriented features
    (such as inheritance and polymorphism), as they are _both_
    distractions;
*   Write it in pseudo-code and ask you to kind of bear with me
    and play along as I hoc up some some ad-hoc notation for it.

None of these options are fantastic.  I'll pick the first option,
because Python is pretty accessible and it's pretty easy to
pretend it has encapsulation.  (See also the proviso I mentioned
earlier.)

So, this code will deal with `Proof` objects with some
attributes whose names begin with single underscore.  Just
pretend that the functions we define here are the _only_
functions that can access these attributes.  All other
code is just magically prevented from accessing them.

These attributes are:

*   `_conclusion` -- a propositional formula which
    is the proved statement
*   `_assumptions` -- a dict mapping labels to
    propositional formulas, which are the assumptions
    under which the `_conclusion` is proved true.

Propositional formulas are represented with an abstract syntax
tree structure, whose nodes are objects from classes
such as `Conj`, `Disj`, `Impl`, `Var` and so forth.  When
these are binary operators we assume they have attributes
called `lhs` and `rhs` containing their child nodes.

Unlike our proof objects, the representation of propositional
formulas will not be hidden from client code.  In fact it
will be useful for client code to manipulate propositional
formulas as much as it likes.

In particular, in order to avoid writing clumsy
formula-tree-building code like `Impl(Var("p"), Var("q")))`,
assume we have a helper function `wff` which parses strings
containing propositional formulas, like so: `wff("p → q")`.

Now, on to the actual operations.

The basic operation to produce a trivial valid proof takes
a propositional sentence and a label and produces a proof with
that sentence as an assumption.  Something like:

    def suppose(formula, label):
        return Proof(
            _assumptions={label: formula},
            _conclusion=formula
        )

We will also have a set of operations that represent inference
rules.  Each inference rule is a function: the inputs are the
premises and the output is the conclusion.

Inference rules such as conjunction-introduction seem
easy to formulate, and they are, but there is one aspect where
we must take care, which is this: any assumptions already made
in the premises must be carried over and preserved in the
conclusion.

Additionally, if we ever discover that two copies of the same label
refer to two different propositional formulas, this indicates some kind
of mistake in the proof construction, and we should take care to
flag it as such.

To achieve these ends we would benefit from defining a helper
function, something like:

    def merge_assumptions(a, b):
        c = b.copy()
        for (key, value) in a.items():
            if key not in c:
                c[key] = value
            elif c[key] != value:
                raise Error("Inconsistent labelling")
        return c

Now we can write the operation corresponding to the inference
rule of conjunction-introduction:

    def conj_intro(x, y):
        """If x is proved, and y is proved, then x & y is proved."""
        assert isinstance(x, Proof)
        assert isinstance(y, Proof)
        return Proof(
            _conclusion=Conj(x._conclusion, y._conclusion),
            _assumptions=merge_assumptions(
                x._assumptions, y._assumptions
            )
        )

(Note that traditionally, Greek letters are used for the variables
representing proofs in the premises and conclusion of an inference rule.
We will avoid them here, because they don't work as well in pseudo-code
as they do in typeset mathematics; they don't flow quite as naturally.
Writing them out seems awkward too (`phi` and `psi` are too similar),
so we will use Latin letters instead.  But also, though it might make
sense to use the letter `p` for a variable for "proof", it would be
too easy to confuse with `p` used as a propositional variable (which
it commonly is.)  So we will use `x`, `y`, `z` for variables that
represent proofs.)

Conjunction-elimination is also reasonably simple.  We
assert that the conclusion has a certain structure, and then we
take it apart.  `side` is a parameter which tells which
side we want to keep: `'L'` for left, `'R'` for right.

    def conj_elim(z, side):
        """If z (of the form x & y) is proved, then x (alternately y) is proved."""
        assert isinstance(z, Proof)
        assert isinstance(z._conclusion, Conj)
        return Proof(
            _conclusion={'L': z._conclusion.lhs, 'R': z._conclusion.rhs}[side],
            _assumptions=z._assumptions
        )

We did the rules for conjunctions first because they are very
simple, but correspondingly not very exciting, and
showing a proof using only them would probably not be very
illuminating.  So, let's work towards having a selection
of inference rules that will let us write a small, but
meaningful, proof.  Here is implication-elimination, also
known as _modus ponens_:

    def impl_elim(x, y):
        """If x is proved, and y (of the form x → z) is proved, then z is proved."""
        assert isinstance(x, Proof)
        assert isinstance(y, Proof)
        assert isinstance(y._conclusion, Impl)
        assert y._conclusion.lhs == x._conclusion
        return Proof(
            _conclusion=y._conclusion.rhs,
            _assumptions=merge_assumptions(
                x._assumptions, y._assumptions
            )
        )

Implication-introduction is slightly more complex, as it will
*discharge* one of the assumptions it's given.  To identify
this assumption, we will pass in its label.  An assumption
with that label must be present on the proof step that we
also pass in.

    def impl_intro(label, y):
        """If y is proved under the assumption x, then x → y is proved."""
        assert isinstance(y, Proof)
        assert label in y._assumptions
        a = y._assumptions.copy()
        fx = a[label]
        delete a[label]
        return Proof(
            _conclusion=Impl(fx, y._conclusion),
            _assumptions=a
        )

We now have enough operations to demonstrate a simple proof.
We'll follow the proof of the statement given in section 5a
of the IEP article:

    (p → q) → ((q → r) → (p → r))

Our function application that corresponds to that proof is

    def proof1a():
        return (
            impl_intro(
                3,
                impl_intro(
                    2,
                    impl_intro(
                        1,
                        impl_elim(
                            impl_elim(
                                suppose(wff("p"), 1),
                                suppose(wff("p → q"), 3)
                            ),
                            suppose(wff("q → r"), 2)
                        )
                    )
                )
            )
        )

(Note well that `proof1a` is *not* an operation and may
*not* directly access or manipulate the `_conclusion`
and `_assumptions` fields of the object it creates and returns.)

The formatting of this function application is intentionally
very tree-like, to try to make it clearer how it corresponds
with the tree presentation of the proof in the IEP article.

The glaring difference between this and a tree-structured proof
is, of course, that the nodes of the tree-structured proof show
the conclusion at each step, while in the function application,
the valid proof is merely being passed as a parameter to the
enclosing function, and not shown in the source code.

Even if we found it acceptable to omit those explicit intermediate
conclusions, we probably would still want to show the final
conclusion that we set out to prove, for clarity.  So, we can
define another helper function like

    def shows(x, conclusion):
        assert isinstance(x, Proof)
        assert len(x._assumptions) == 0
        assert x._conclusion == conclusion
        return x

Then we can say

    def proof1b():
        return shows(
            impl_intro(
                3,
                (... all the rest of the above proof ...)
            ),
            wff("(p → q) → ((q → r) → (p → r))")
        )

and if we call this function we should not get an error; it
should just return, indicating that the proof is indeed valid
and that it is indeed a proof of this statement, with no
outstanding assumptions.

Note that the `shows` function could also be used on
intermediate steps, if desired.

But even if you used it on every intermediate step, this
layout seems a bit awkward.  It corresponds with the tree
proof in the article, but it's oriented sideways.

Natural deduction proofs can also be written out linearly,
as a list of steps.  We can do that with our big nest
of function applications, by binding each application to
a local name.  Like so:

    def proof1c1():
        s1 = suppose(wff("p"), 1)
        s2 = suppose(wff("p → q"), 3)
        s3 = impl_elim(s1, s2)
        s4 = suppose(wff("q → r"), 2)
        s5 = impl_elim(s3, s4)
        s6 = impl_intro(1, s5)
        s7 = impl_intro(2, s6)
        s8 = impl_intro(3, s7)
        return shows(s8, wff("(p → q) → ((q → r) → (p → r))"))

This layout is immediately, in some ways, more readable.  It
comes at the cost of having to introduce and manage all those
intermediate names, though.  And we still need to manage
the bookkeeping of the assumption labels.

In a Fitch-style proof, the assumption labels are replaced
by nested boxes.  The proof steps outside the boxes cannot
make reference directly to the proof steps inside the boxes.
This suggests the proof steps in the boxes are in
an _inner lexical scope_.

First let's re-arrange the proof steps to show that they
can indeed nest.

    def proof1c2():
        s1 = suppose(wff("p → q"), 3)
        s2 = suppose(wff("q → r"), 2)
        s3 = suppose(wff("p"), 1)
        s4 = impl_elim(s3, s1)
        s5 = impl_elim(s4, s2)
        s6 = impl_intro(1, s5)
        s7 = impl_intro(2, s6)
        s8 = impl_intro(3, s7)
        return shows(s8, wff("(p → q) → ((q → r) → (p → r))"))

Now let's pretend we've rewritten `suppose` to be a context
manager.  Instead of taking a label as an argument, `suppose`
now generates its own unique label and pushes it onto a
stack.  And now, also instead of taking a label as an
argument, `impl_intro` pops the label off that stack.
This lets us write:

    def proof1c3():
        with suppose(wff("p → q")) as s1:
            with suppose(wff("q → r")) as s2:
                with suppose(wff("p")) as s3:
                    s4 = impl_elim(s3, s1)
                    s5 = impl_elim(s4, s2)
                    s6 = impl_intro(s5)
                s7 = impl_intro(s6)
            s8 = impl_intro(s7)
        return shows(s8, wff("(p → q) → ((q → r) → (p → r))"))

...which matches pretty closely with a Fitch-style exposition.
It's still lacking a few things; mainly, the lines after a
`with` block shouldn't be able to see the local variables
defined within it; they should only be able to refer to the
final line.  We'll have to pretend they are magically prevented
from referring to any of the other lines.

A more thorough way to do this would be to use local function
definitions, although it does begin to look a bit less Fitch-like:

    def proof1c4():
        def inner1((s1, lab1)):
            def inner2((s2, lab2):
                def inner3((s3, lab3)):
                    s4 = impl_elim(s3, s1)
                    s5 = impl_elim(s4, s2)
                    return impl_intro(s5, lab3)
                s6 = inner3(suppose(wff("p")))
                return impl_intro(s6, lab2)
            s7 = inner2(suppose(wff("q → r")))
            return impl_intro(s7, lab1)
        s8 = inner1(suppose(wff("p → q")))
        return shows(s8, wff("(p → q) → ((q → r) → (p → r))"))

You can, however, probably imagine an alternate syntax for this
that is more properly Fitch-like.  It would be simply an exercise
in syntactic sugar, so I won't dwell on it.

Now, to go back to the inference rules.  We haven't given
a full set, but I think we've given enough to get the idea
across.  I could say the rest are left as an exercise for
the reader.  But, one in particular is perhaps still worth
doing, and that's disjunction-elimination, and the reason
it's perhaps worth doing, is because it's complex.  If you
can translate it into a function, you can probably translate
any inference rule into a function.

Pictorially, disjunction-elimination is usually shown as
something like

                        φ   ψ
                        :   :
                        :   :
                φ ∨ ψ   χ   χ
            -------------------- ∨e
                      χ

In prose, we can read this as "If φ ∨ ψ is proved, and if
we can prove χ under the assumption φ, and we can prove
χ under the assumption ψ, then χ is proved."

From this, we know we need to take a proof (call it `z`)
and assert that its conclusion has a certain form (`Disj`);
and we will need to take another proof, and a label (call
them `s` and `l1`); and yet another proof and a label
(call them `t` and `l2`).

At the labels we will locate `x` and `y` and assert that
those are what `z` breaks up into.

We will assert that `s` and `t` also represent the same
conclusion, and that is what we will return.  We will
discharge the assumptions `x` and `y`, but we will also
need to carry forward any undischarged assumptions that
may still be in `z`, `s`, and `t`.

Which means the function must look like this:

    def disj_elim(z, s, l1, t, l2):
        assert isinstance(z, Proof)
        assert isinstance(z._conclusion, Disj)

        assert isinstance(s, Proof)
        assert l1 in s._assumptions
        a_s = s._assumptions.copy()
        fx = a_s[l1]
        delete a_s[l1]

        assert isinstance(t, Proof)
        assert l2 in t._assumptions
        fy = t._assumptions.copy()
        q1 = a_t[l1]
        delete a_t[l1]

        assert s._conclusion == t._conclusion
        assert z._conclusion.lhs == fx
        assert z._conclusion.rhs == fy

        return Proof(
            _conclusion=s._conclusion,
            _assumptions=merge_assumptions(
                z._assumptions,
                merge_assumptions(a_s, a_t)
            )
        )

Whew!

### Conclusion

We'll conclude with a couple of more observations, ones which
suggest avenues for future work.

The first is that, while you can certainly build a proof with
these functions, you have to actually _run_ the constructed
program to check if the proof is valid.  And confirming that
the proof is valid is _all_ the program does.  And as
computational tasks go, that's really not a very complex one --
there aren't any loops, or even any conditionals, in these
inference rules we've written.

So you might (very reasonably) wonder if the proof can be checked
without going all the way to executing the program.  You might
(very reasonably) think that you could apply concepts from static
analysis, such as constant folding and abstract interpretation,
to statically analyze the steps, and obtain the confirmation of
validity of the proof at compile-time rather than runtime.

Similarly, you might think of writing it as a macro -- another
compile-time thing.

Having not tried it, I cannot say exactly how it does end up,
but I would be surprised if the result isn't parallel in some
way to "propositions as types", since the compile-time-computable
component can arguably be viewed as a type system.

I would guess the resulting system would be fairly trivial for
propositional logic, but would start to resemble dependent types
(which is what is more conventionally used for theorem provers)
if one were to upgrade to first-order logic.

The other observation is that our program which expresses and
checks a proof has a simple structure: a tree of function
applications.  There is a straightforward way of "refactoring"
such a program into a data structure: [defunctionalization][].

In this way, the proof could be expressed as "plain old data"
(think JSON) and "interpreted" by the program.  In this case,
the programming language needn't support information hiding;
the program is prevented from producing an incorrect proof
object by virtue of the fact that the "interpreter" is closed,
and does not allow any external code (i.e. code that could
potentially modify the proof object in process in invalid ways)
to be executed before the proof checking process terminates.

It would admittedly be a stretch to call the resulting
theorem prover "LCF-style" though, as it would no longer be
possible to write arbitrary proof-manipulating code (e.g.
tactics) in the host language.

[The IEP article on Natural Deduction]: https://iep.utm.edu/nat-ded/
[Parse, don't Validate]: https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
[Logic in Computer Science]: https://archive.org/details/logicincomputers0000huth
[Information Hiding in Scheme]: https://github.com/cpressey/Information-Hiding-in-Scheme
[employ some contortions to achieve encapsulation]: https://github.com/cpressey/Information-Hiding-in-Scheme
[The LCF Approach to Theorem Proving]: https://www.cl.cam.ac.uk/~jrh13/slides/manchester-12sep01/slides.pdf
[defunctionalization]: https://en.wikipedia.org/wiki/Defunctionalization
