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

Code for the Theorem Prover
---------------------------

To avoid the question of what programming language to write this
theorem prover in, let's write it in pseudo-code.

Assume we have the usual assortment of primitive types in the
language, and some way to define records containing multiple
values, and some way to hide the representation of these records
from every part of the program, except inside the definition of
our operations.

The record we will use to represent a valid proof contains
two elements:

*   the logical expression which is the proved statement
*   a mapping from labels to logical expressions, which are
    the assumptions that have been introduced.

The basic operation to produce a trivial valid proof takes
a logical expression and a label and produces a proof with
that expression as an assumption.  Something like:

    def suppose(expr, label) =
        return ValidProof(
            statement=expr,
            assumptions={ label: expr }
        )

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

    def merge_assumptions(a, b) =
        local c := copy_of(b)
        for each (key, value) in a:
            if key not in c:
                c[key] := value
            else if c[key] != value:
                raise Error("Inconsistent labelling")
        return c

Now we can write the operation corresponding to the inference
rule of conjunction-introduction:

    def conj_intro(p, q) =
        assert p is_a ValidProof
        assert q is_a ValidProof
        return ValidProof(
            statement=Conj(p.statement, q.statement),
            assumptions=merge_assumptions(
                p.assumptions, q.assumptions
            )
        )

Conj is some sort of constructor that takes two
logical expressions and returns a new logical expression
that is their conjunction; unlike ValidProof, there is no
harm for users of these operations to know about the
structure of (and thus be able to manipulate) logical
expressions.

We did conjunction-introduction first because it is very
simple, but it is correspondingly not very exciting, and
showing a proof using only it will probably not be very
illuminating.  So, let's work towards having a selection
of inference rules that will let us write a small, but
meaningful, proof.  Here is implication-elimination, also
known as _modus ponens_:

    def impl_elim(p, q) =
        assert p is_a ValidProof
        assert p.statement is_a Impl
        assert p.statement.lhs == q
        return ValidProof(
            statement=p.statement.rhs,
            assumptions=merge_assumptions(
                p.assumptions, q.assumptions
            )
        )

This is a little more complex than conjunction-introduction
because we're examining the structure of one of the logical
expressions we're given as input.

Implication-introduction is more complex still, as it will
*discharge* one of the assumptions it's given.  To identify
this assumption, we will pass in its label.  An assumption
with that label must be present on the ValidProof that we
also pass in.

    def impl_intro(p, label) =
        assert p is_a ValidProof
        assert label in p.assumptions
        local a := copy_of(p.assumptions)
        del a[label]
        return ValidProof(
            statement=Impl(p.assumptions[label], p)
            assumptions=a
        )

We now have enough operations to demonstrate a simple proof.

_TO BE CONTINUED_

[The IEP article on Natural Deduction]: https://iep.utm.edu/nat-ded/
[Parse, don't Validate]: https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
[Information Hiding in Scheme]: https://github.com/cpressey/Information-Hiding-in-Scheme
[The LCF Approach to Theorem Proving]: https://www.cl.cam.ac.uk/~jrh13/slides/manchester-12sep01/slides.pdf
