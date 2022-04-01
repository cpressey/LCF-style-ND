LCF-Style-ND
============

This repository shows the development, in readable pseudo-code,
of a simple LCF-style theorem prover for propositional logic
in a Natural Deduction system.

First it contains some brief notes and some (I think) interesting
observations about LCF-style theorem provers and Natural Deduction.
Then it proceeds to tie them together with some Python-like
pseudo-code.  And concludes with a couple more interesting notes.

Implementations in real programming languages of the ideas presented
here can be found in projects external to this repository, such as
[**philomath**](https://github.com/catseye/philomath) (in ANSI C).

Contents:

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
transform them to produce new valid proofs.

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

(3) Taking 1 and 2 together, one may conclude that "Parse, don't Validate"
in general doesn't rely on a static type system either.

(4) Actually preventing the mutation of such a data object on a
_real computer_ is probably impossible.  Can you not always run
the program under `gdb` and alter a few bytes here and there?
It's probably more productive to view it as a proviso: _if_ the
data object was constructed only by these operations and no
others, _then_ it represents a valid proof.  You could even have
an LCF-style theorem prover in a language with only "voluntary"
encapsulation, like Python, under such a proviso.

Natural Deduction
-----------------

[The IEP article on Natural Deduction][]
is a good overview, and I'll follow parts of it closely here.

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
    ask you to ignore the types _and_ the object-oriented bits, as
    they are both distractions;
*   Write it in pseudo-code and ask you to kind of bear with me
    and play along as I hoc up some some ad-hoc notation for it.

None of these options are fantastic.  I'll pick the first option,
because Python is pretty accessible and it's pretty easy to
pretend it has encapsulation.  (See also the proviso I mentioned
earlier.)

So, this code will deal with `Proof` objects with some
attributes whose names begin with single underscore.  Just
pretend that the operations described here are the _only_
operations that can access these attributes.  All other
operations are magically prevented from doing so.

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

Now, on to the actual operations.

The basic operation to produce a trivial valid proof takes
a propositional sentence and a label and produces a proof with
that sentence as an assumption.  Something like:

    def suppose(prop, label):
        return Proof(
            _assumptions={label: prop},
            _conclusion=prop
        )

We will also have a set of operations that represent inference
rules.  Each inference rule is a function: the inputs are the
premises and the output is the conclusion.

Inference rules such as conjunction-introduction seem
easy to formulate, and they are, but there is one aspect where
we must take care, which is this: any assumptions already made
in the premises must be carried over and preserved in the
conclusion.

Additionally, discovering two copies of the same label refer
to two different propositional formulas indicates some kind
of mistake in the proof construction, and should be flagged
as such.

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

    def conj_intro(p, q):
        """If p is proved, and q is proved, then p & q is proved."""
        assert isinstance(p, Proof)
        assert isinstance(q, Proof)
        return Proof(
            _conclusion=Conj(p._conclusion, q._conclusion),
            _assumptions=merge_assumptions(
                p._assumptions, q._assumptions
            )
        )

Note that, when writing an inference rule in natural deduction,
we would typically notate variables which take on proofs with
Greek letters.  Instead of doing that in this code, we are
using Latin letters, in this case `p` and `q`.  (Note that these
variables do represent proofs, and not particular propositional
variables that appear in the propositional statements themselves.
Those would be represented by `Var("p")` or similar.)

Conjunction-elimination is also reasonably simple.  We
assert that the conclusion has a certain structure, and then we
take it apart.  `side` is a parameter which tells which
side we want to keep: `'L'` for left, `'R'` for right.

    def conj_elim(r, side):
        """If r (of the form p & q) is proved, then p (alternately q) is proved."""
        assert isinstance(r, Proof)
        assert isinstance(r._conclusion, Conj)
        return Proof(
            _conclusion={'L': r._conclusion.lhs, 'R': r._conclusion.rhs}[side],
            _assumptions=r._assumptions
        )

We did the rules for conjunctions first because they are very
simple, but correspondingly not very exciting, and
showing a proof using only them would probably not be very
illuminating.  So, let's work towards having a selection
of inference rules that will let us write a small, but
meaningful, proof.  Here is implication-elimination, also
known as _modus ponens_:

    def impl_elim(p, r):
        """If p is proved, and r (of the form p → q) is proved, then q is proved."""
        assert isinstance(p, Proof)
        assert isinstance(r, Proof)
        assert isinstance(r._conclusion, Impl)
        assert r._conclusion.lhs == p._conclusion
        return Proof(
            _conclusion=r._conclusion.rhs,
            _assumptions=merge_assumptions(
                p._assumptions, r._assumptions
            )
        )

Implication-introduction is slightly more complex, as it will
*discharge* one of the assumptions it's given.  To identify
this assumption, we will pass in its label.  An assumption
with that label must be present on the proof step that we
also pass in.

    def impl_intro(label, q):
        """If q is proved under the assumption p, then p → q is proved."""
        assert isinstance(q, Proof)
        assert label in q._assumptions
        a = q._assumptions.copy()
        prop = a[label]
        delete a[label]
        return Proof(
            _conclusion=Impl(prop, q._conclusion),
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
                                suppose(«p», 1),
                                suppose(«p → q», 3)
                            ),
                            suppose(«q → r», 2)
                        )
                    )
                )
            )
        )

The guillemet-quoted strings are just intended to be a convenient
way to write a logical expression (instead of something like
`Impl(Prop("p"), Prop("q")))` which would be much clumsier).

(And note well that `proof1a` is *not* an operation and may
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

    def shows(p, conclusion):
        assert isinstance(p, Proof)
        assert len(p._assumptions) == 0
        assert p._conclusion == conclusion
        return p

Then we can say

    def proof1b():
        return shows(
            impl_intro(
                3,
                (... all the rest of the above proof ...)
            ),
            «(p → q) → ((q → r) → (p → r))»
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
        s1 = suppose(«p», 1)
        s2 = suppose(«p → q», 3)
        s3 = impl_elim(s1, s2)
        s4 = suppose(«q → r», 2)
        s5 = impl_elim(s3, s4)
        s6 = impl_intro(1, s5)
        s7 = impl_intro(2, s6)
        s8 = impl_intro(3, s7)
        return shows(s8, «(p → q) → ((q → r) → (p → r))»)

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
        s1 = suppose(«p → q», 3)
        s2 = suppose(«q → r», 2)
        s3 = suppose(«p», 1)
        s4 = impl_elim(s3, s1)
        s5 = impl_elim(s4, s2)
        s6 = impl_intro(1, s5)
        s7 = impl_intro(2, s6)
        s8 = impl_intro(3, s7)
        return shows(s8, «(p → q) → ((q → r) → (p → r))»)

Now let's pretend we've rewritten `suppose` to be a context
manager.  Instead of taking a label as an argument, `suppose`
now generates its own unique label and pushes it onto a
stack.  And now, also instead of taking a label as an
argument, `impl_intro` pops the label off that stack.
This lets us write:

    def proof1c3():
        with suppose(«p → q») as s1:
            with suppose(«q → r») as s2:
                with suppose(«p») as s3:
                    s4 = impl_elim(s3, s1)
                    s5 = impl_elim(s4, s2)
                    s6 = impl_intro(s5)
                s7 = impl_intro(s6)
            s8 = impl_intro(s7)
        return shows(s8, «(p → q) → ((q → r) → (p → r))»)

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
                s6 = inner3(suppose(«p»))
                return impl_intro(s6, lab2)
            s7 = inner2(suppose(«q → r»))
            return impl_intro(s7, lab1)
        s8 = inner1(suppose(«p → q»))
        return shows(s8, «(p → q) → ((q → r) → (p → r))»)

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

From this, we know we need to take a proof (call it `r`)
and assert that its conclusion has a certain form (`Disj`);
and we will need to take another proof, and a label (call
them `s` and `l1`); and yet another proof and a label
(call them `t` and `l2`).

At the labels we will locate `p` and `q` and assert that
those are what `r` breaks up into.

We will assert that `s` and `t` also represent the same
conclusion, and that is what we will return.  We will
discharge the assumptions `p` and `q`, but we will also
need to carry forward any undischarged assumptions that
may still be in `r`, `s`, and `t`.

Which means the function must look like this:

    def disj_elim(r, s, l1, t, l2):
        assert isinstance(r, Proof)
        assert isinstance(r._conclusion, Disj)

        assert isinstance(s, Proof)
        assert l1 in s._assumptions
        a_s = s._assumptions.copy()
        p1 = a_s[l1]
        delete a_s[l1]

        assert isinstance(t, Proof)
        assert l2 in t._assumptions
        a_t = t._assumptions.copy()
        q1 = a_t[l1]
        delete a_t[l1]

        assert s._conclusion == t._conclusion
        assert r._conclusion.lhs == p
        assert r._conclusion.rhs == q

        return Proof(
            _conclusion=s._conclusion,
            _assumptions=merge_assumptions(
                r._assumptions,
                merge_assumptions(a_s, a_t)
            )
        )

Whew!

_(TO BE CONTINUED)_

[The IEP article on Natural Deduction]: https://iep.utm.edu/nat-ded/
[Parse, don't Validate]: https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
[Information Hiding in Scheme]: https://github.com/cpressey/Information-Hiding-in-Scheme
[employ some contortions to achieve encapsulation]: https://github.com/cpressey/Information-Hiding-in-Scheme
[The LCF Approach to Theorem Proving]: https://www.cl.cam.ac.uk/~jrh13/slides/manchester-12sep01/slides.pdf
