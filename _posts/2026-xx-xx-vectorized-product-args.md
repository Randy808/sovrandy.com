# Vectorized product arguments

## Introduction

I was originally going to just describe the Hadamard product argument here, but I decided to also group in inner product arguments in this post. Looking at `Eﬃcient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting`, a paper that describes an inner-product optimization I'll write about in my next post, they mention the paper I've been referencing in my last posts in the context of inner products. Turns out the method they claim is to prove Hadamard products can be proved with what essentially is an inner-product argument.

## Vectorized product argument (Hadamard argument)

So the idea behind proving the products of **vectors** instead of single values, is that instead of proving **one** product of committed values, we're now proving multiple products of different committed values.

So instead of asking someone to prove the preimages of A1 and A2 have a product of `b_`, we're proving the preimages of `A1_1` and `A1_2` have a product of `b_1` **AND** `A2_1` and `A2_2` have a product of `b_2`. The trick to vectorization is that every set of commitments we prove the preimage product for **must** have the same size. When they do, we can stack preimages of the same index across commitment sets.

E.g. we're proving the element-wise/Hadamard product of these 2 vectors:

[a1_1, a2_1] * [a1_2, a2_2]

## How does our approach change from the previous single value product argument?

To be completely honest, it doesn't have to change at all. We could just prove the product of the first row of all vectors as if each were a term in the single value argument, then tackle the second element of each vector, etc. Each column will all have the same `x` applied to them before they're summed with the other `x`-scaled columns and this would work. But one question always remains: Is there a better way?

## The optimization

Let's review the single val argument again, since the process is **almost** the same:
- Prover makes statement which includes commitments to preimages `a` and intermediate products `b`
- Prover also gives verifier commitments to **cross-terms**
- Verifier gives prover a value they choose, `x`
- Prover modifies all `a`s into one term and all  `b`s  into another term using `x`
- Prover gives modified terms `a'` and `b'` to verifier
- Verifier get the product of `a'` and `b'` which should contain `a*b` as a term
- Verifier re-creates the commitment of `a'*b'` using the commitment of the product and the cross term commitments


# TODO: Also talk about the communication savings from vector commitment scheme
The optimization comes from asking the question, "is there a way we can reduce the number of cross terms sent at the beginning?" The answer lies in another strategy for aggregation.

Imagine we're making an argument for the vectorized version where there are multiple sets of terms. We start proving every one of them using the product argument above, one by one, but we stop before we do the last step. We can then add all the modified products `a[i]'*b[i]'` into a single value consolidating cross terms with the same `x` co-efficient across products. With this consolidation, we see that we didn't even need to send individual cross terms at the beginning. Instead we could have sent the aggregated cross terms resulting from the sum of `a[i]'*b[i]'`s.

One problem that we've run into before is the vector for a prover to cheat  if they only know the aggregate and not the individual terms. This vector also exists with this approach. There is a way to only know the sum of `a[i]'*b[i]'`s and not know the individual `a[i]'*b[i]'` terms. To combat this we have a similar fix to past approaches.

Before we calculate the product of every `a[i]'*b[i]'`, we ask the verifier to choose a new value `y` for which we'll now incorporate into our sum by doing `a[i]'*b[i]'*y^i`. Normally this would stop us from aggregating the cross-terms if we asked the verifier to give the new `y` at the same time they gave `x`. But instead, the verifier gives `y` **before** the prover gives any cross terms. That way the prover can incorporate `y` into the cross term commitments but still have an arbitrary coefficient to scale what the product terms in the final sum `a[i]'*b[i]'*y^i` should be.

Also, we will be using the **zero argument** optimization in our lost post here since it still improves this version of the argument.