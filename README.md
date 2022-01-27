LCF-Style-ND
============

This repository aims to contain a simple LCF-style theorem prover
for propositional logic in a [Natural Deduction][] setting.

For now, it contains some brief notes and some (I think) interesting
observations about [LCF-style theorem provers][].

LCF-style Theorem Provers
-------------------------

An LCF-style theorem prover, as far as I'm concerned, means this:

You have a data type representing valid proofs.

You have some operations that produce trivial valid proofs.

You have some other operations that take valid proofs and
transform them to produce new valid proofs.

You don't provide any other way to produce or mutate such
a data object.  In this way, these data objects are guaranteed
to represent valid proofs.

Some Observations
-----------------

Now, some observations about LCF-style theorem provers:

(1) This is fundamentally an application of "[Parse, don't Validate]()".

(2) This doesn't rely on a static type system.  Rather, it relies
on _encapsulation_ a.k.a. _information hiding_.
[You could write it in Scheme][]], even though a type theorist
would call Scheme an "untyped" language.  (The rest of the world would
probably call it "dynamically typed" or "latently typed".)

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

[LCF-style theorem provers]: https://www.cl.cam.ac.uk/~jrh13/slides/manchester-12sep01/slides.pdf
[Natural Deduction]: https://iep.utm.edu/nat-ded/
[Parse, don't Validate]: https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
[You could write it in Scheme]: https://github.com/cpressey/Information-Hiding-in-Scheme
