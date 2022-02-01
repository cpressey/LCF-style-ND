LCF-Style-ND
============

This repository shows the development, in readable pseudo-code,
of a simple LCF-style theorem prover for propositional logic
in a Natural Deduction setting.

First it contains some brief notes and some (I think) interesting
observations about LCF-style theorem provers and Natural Deduction.
Then it proceeds to tie them together with some code.

Contents:

*   [LCF-style Theorem Proving](#lcf-style-theorem-proving)
*   [Natural Deduction](#natural-deduction)
*   [Code for the Theorem Prover](#code-for-the-theorem-prover) (WIP)

LCF-style Theorem Proving
-------------------------

A brief introduction to LCF-style theorem proving can be
found in [The LCF Approach to Theorem Proving][] (PDF),
slides for a presentation by John Harrison of Intel Corp. in 2001.

An LCF-style theorem prover, as far as I'm concerned, means this:

You have a data type representing valid proofs.

You have some operations that produce trivial valid proofs.

You have some other operations that take valid proofs and
transform them to produce new valid proofs.

You don't provide any other way to produce or mutate such
a data object.  In this way, these data objects are guaranteed
to represent valid proofs.

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

In particular, it proposes some criteria to distinguish Natural
Deduction proof systems from other proof systems.  In ND systems,

(1) Assumptions can be freely introduced, and subsequently
discharged under certain conditions.

(2) Reasoning proceeds by successive application of inference
rules.

(3) Rich forms of proof construction are possible.

The first corresponds to LCF-style operations which produce trivial
valid proofs.  The second corresponds to LCF-style operations which
transform valid proofs into other valid proofs.  The third
corresponds to the fact that the data type representing valid
proofs is embedded in a general-purpose programming language, in
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
This seems akin to labelled assumptions having _dynamic scope_
versus nested assumptions having _lexical scope_.  Our theorem prover
will use labels for simplicity.

Code for the Theorem Prover
---------------------------

It would seem that the ideal language in which to demonstrate the
things I'm trying to demonstrate here would be a dynamically-typed
language with readable language constructs for encapsulation.
Unfortunately, such languages are not common, and certainly not
mainstream.  So to avoid the question of what programming language
use, let's write this theorem prover in pseudo-code.

Assume we have the usual assortment of primitive types in the
language, and some way to define records containing multiple
values, and some way to hide the representation of these records
from every part of the program, except inside the definition of
our operations.

The record we will use to represent a valid proof, the
constructor for which we will denote with `ValidProof`,
contains two elements:

*   the logical expression which is the proved statement
*   a mapping from labels to logical expressions, which are
    the assumptions that have been introduced.

Logical expressions are themselves a record structure of
some sort, with constructors according to its structure
such as `Conj`, `Disj`, `Impl`, `Prop` and so forth.
However, unlike `ValidProof`, the representation of these
will not be hidden from client code.  In fact it will be
useful for client code to manipulate logical expressions
as much as it likes.

In addition, we will assume there is some kind of `Dict`
structure which can contain a mapping from values to values.

The basic operation to produce a trivial valid proof takes
a logical expression and a label and produces a proof with
that expression as an assumption.  Something like:

    function suppose(expr, label):
        local assm := new Dict{}
        assm[label] := expr
        return new ValidProof{
            statement: expr,
            assumptions: assm
        }

We will also have a set of operations that represent inference
rules.  Inference rules such as conjunction-introduction seem
easy to formulate, and they are, but there is one aspect where
we must take care, which is this: any assumptions attached to
the premises must be carried over to the conclusion.

Additionally, any label that refers to two different logical
expressions must be some kind of mistake, and should be flagged
as such.

To achieve these ends we would benefit from defining a helper
function, something like:

    function merge_assumptions(a, b):
        local c := copy_of(b)
        for each (key, value) in a:
            if key not in c:
                c[key] := value
            else if c[key] != value:
                throw Error("Inconsistent labelling")
        return c

Now we can write the operation corresponding to the inference
rule of conjunction-introduction:

    function conj_intro(p, q):
        assert p is_a ValidProof
        assert q is_a ValidProof
        return new ValidProof{
            statement: new Conj(p.statement, q.statement),
            assumptions: merge_assumptions(
                p.assumptions, q.assumptions
            )
        }

We did conjunction-introduction first because it is very
simple, but it is correspondingly not very exciting, and
showing a proof using only it will probably not be very
illuminating.  So, let's work towards having a selection
of inference rules that will let us write a small, but
meaningful, proof.  Here is implication-elimination, also
known as _modus ponens_:

    function impl_elim(p, q):
        assert p is_a ValidProof
        assert p.statement is_a Impl
        assert p.statement.lhs == q
        return new ValidProof{
            statement: p.statement.rhs,
            assumptions: merge_assumptions(
                p.assumptions, q.assumptions
            )
        }

This is a little more complex than conjunction-introduction
because we're examining the structure of one of the logical
expressions we're given as input.

Implication-introduction is more complex still, as it will
*discharge* one of the assumptions it's given.  To identify
this assumption, we will pass in its label.  An assumption
with that label must be present on the ValidProof that we
also pass in.

    function impl_intro(label, q):
        assert q is_a ValidProof
        assert label in q.assumptions
        local assm := copy_of(q.assumptions)
        local p := assm[label]
        delete assm[label]
        return new ValidProof{
            statement: new Impl(p, q),
            assumptions: assm
        )

We now have enough operations to demonstrate a simple proof.
We'll follow the proof of the statement given in section 5a
of the IEP article:

    (p → q) → ((q → r) → (p → r))

Our function application that corresponds to that proof is

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

The guillemet-quoted strings are just intended to be a convenient
way to write a logical expression (instead of something like
`Impl(Prop("p"), Prop("q")))` which would be much clumsier).

The formatting of this function application is intentionally
very tree-like, to try to make it clearer how it corresponds
with the tree presentation of the proof in the IEP article.

The glaring difference is, of course, that the nodes of the
tree in the proper tree-structured proof show the sentence of
the valid proof at each step, while in the function application,
that valid proof is merely being passed as a parameter to the
enclosing function, and not shown in the source code.

Even if we were alright with omitting explicit intermediate
logical statements, we probably want to show the result we
set out to prove, for clarity.  So, we can define another
helper function like

    function shows(proof, expr):
        assert proof is_a ValidProof
        assert proof.assumptions is empty
        assert proof.statement == expr
        return proof

Then we can say

    shows(
        impl_intro(
            3,
            (... all the rest of the above proof ...)
        ),
        «(p → q) → ((q → r) → (p → r))»
    )

and if we run this program we should not get an error, it
should just exit, indicating that the proof is indeed valid
and that it is indeed a proof of this statement, with no
outstanding assumptions.

Note that the `shows` function could also be used on
intermediate steps.

_TO BE CONTINUED_

[The IEP article on Natural Deduction]: https://iep.utm.edu/nat-ded/
[Parse, don't Validate]: https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
[Information Hiding in Scheme]: https://github.com/cpressey/Information-Hiding-in-Scheme
[The LCF Approach to Theorem Proving]: https://www.cl.cam.ac.uk/~jrh13/slides/manchester-12sep01/slides.pdf
