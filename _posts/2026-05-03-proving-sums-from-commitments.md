# Proving Sums from Commitments

## Introduction

This post builds intuition for the zero-knowledge arguments needed to understand modern range proof systems like Bulletproofs. It does this by introducing the sum argument, a simple technique for proving that hidden values add up to a known total without revealing said hidden values.

I decided to write this because, when I first encountered these schemes, I could follow the mechanics but struggled to convince myself they were actually secure. This post is my attempt to build that missing intuition.

The paper that finally helped things click for me was Efficient Zero-Knowledge Argument for Correctness of a Shuffle. I spent hours working through the same sections and trying to understand the ideas from different angles. The sum argument I’ll present here is not described explicitly in that paper, but it follows the some of the same underlying logic. The rest of this post focuses on explaining that argument as simply as possible.

## What is the sum argument?

Let’s start with a definition. The **sum argument** is a protocol between a **prover** and a **verifier** where the prover shows that two committed values, `A = commit(a)` and `B = commit(b)`, correspond to hidden values that add up to some known value `c`, without revealing what `a` and `b` are.

As a refresher, a commitment is the output of a function that is both hiding and binding. Hiding means it is hard to recover the input from the output. Binding means it is hard to find two different inputs that produce the same output. The concrete commitment function I will use is defined on the secp256k1 curve. It takes a scalar and maps it to a point on the curve, which we refer to as a **commitment**.

## Proof approach

The goal is to show that `c`, or its commitment `C = commit(c)`, can be constructed in two different ways using verifier-known values. These constructions need to be algebraically linked to the hidden values `a` and `b` in a way that would be very difficult to fake. At the same time, they cannot leak any information about `a` or `b`.

So the question becomes: how do we build expressions that are linked to `a` and `b`, but still keep them hidden?

A natural approach is to work with expressions derived from `a` and `b`. With that in mind, we want to:

- Create an expression `a'` that hides `a`, but still allows the verifier to check that it was actually used
- Create an expression `b'` that does the same for `b`
- Show the verifier how to combine `a'` and `b'` into a new expression `c'` that is linked to `c`
- Show a second way to construct `c'` using values that can be tied back to `A` and `B`

# Proving we know `a` and `b`

To build expressions that hide `a` and `b`, we also need a way to convince the verifier that these expressions were actually derived from them. Since the verifier does not know `a` or `b`, they need to check this indirectly using the commitments `A` and `B`.

This is similar to proving knowledge of a private key when the verifier only knows the corresponding public key. A private-public key pair can be viewed as a preimage-commitment pair, so we can reuse that intuition here. In particular, we can use a signature as a derived expression.

This gives us a way to link expressions back to `a` and `b` through their commitments `A` and `B`, using well-understood techniques. To make this concrete, we’ll use the Schnorr signature scheme.

## Schnorr signatures

A **Schnorr** signature consists of two components, `R` and `s`. Suppose `a` is the hidden value we want to build a derived expression from. The value `s` is defined as:

`s = e*a + r`

Here, `e` is a challenge chosen by the verifier (or derived from a hash), `a` is the secret, and `r` is a random value. The value `R` is the commitment to `r`, computed as `R = r*G`.

For our purposes, we will treat `e` as a verifier-chosen value so we can describe this in an interactive setting.

Using this construction, we can convince the verifier that we know the value `a` corresponding to the commitment `A`. The verifier checks this by computing `S = s*G` in two ways:

- Directly from `s`, by computing `s*G`
- From known values, by computing `R + e*A`

If both results match, then:

`s*G = R + e*A`

This works because `s` contains `a` inside it. When expanded, the equation becomes:

`s*G = (e*a + r)*G = e*A + R`

So any valid `s` must be consistent in being able to re-create an expression with the committed value `A`.

Now consider what happens if the prover does not know `a` and only knows `A`. Could they still produce values `R` and `s` that satisfy this equation?

If the prover could choose both `R` and `s` **after** seeing `e`, they could try to fake a valid pair by picking a random `s` and solving:

`R = s*G - e*A`

But in the protocol, the prover must send `R` **before** receiving `e`. This means `R` cannot depend on `e`. Once `e` is revealed, the prover must produce an `s` such that:

`s*G = R + e*A`

Finding such an `s` without knowing `a` would require solving a discrete logarithm problem, which is assumed to be hard. This is what gives the scheme its soundness.

## Proving we know the sum of `a` and `b`

We now extend this idea to prove a relation between two hidden values.

To show that `a + b = c` without revealing `a` or `b`, we construct derived expressions for each value:

- `a' = e*a + r1`
- `b' = e*b + r2`

These follow the same structure as the Schnorr construction. As before, the verifier must choose `e` after receiving a commitment to the randomness. In this case, we commit to the sum of the random values:

`R = (r1 + r2)*G`

The prover sends `R` first, then receives `e`, and then sends `a'` and `b'`.

The verifier can now compute:

`c' = a' + b'`

Expanding this gives:

`c' = e*a + r1 + e*b + r2`

Grouping terms:

`c' = e*(a + b) + (r1 + r2)`

Since `a + b = c`, we can rewrite this as:

`c' = e*c + r1 + r2`

Now consider the commitment to `c'`, which we call `C'`. The verifier can compute this as:

`c'*G = e*C + R`

Here, `C = commit(c)` and `R = (r1 + r2)*G`.

This gives the verifier two ways to create the derived value `c'` with known commitments. And since `c'` can be directly linked to `c` using the prover's commitments, the verifier can be confident the expressions `a'` and `b'` truly contained the values `a` and `b` to make `c`.

## Conclusion

Hopefully this post builds some intuition for how this proof approach can be generalized to other operations on committed values. In the next post, I’ll show how similar ideas can be used to prove products of committed scalars and even touch on inner-products with vector commitments.

It may seem abstract now, but these ideas are useful on their own and will come together when we get to Bulletproofs.