# Grokking Product Arguments

## Introduction

This is a follow-up post to the series I'm making about zero knowledge arguments. In this post I'll be describing a way to prove the product of 2 hidden values using something called the 'product argument'.

I think this will serve as an interesting primitive to any zk-practitioner's toolbox and is integral in understanding the Bulletproofs confidential transaction scheme that I'll describe in a future post.

Before reading this, you should know that I take the liberty of making a **commitment** of a value mean the value is being multiplied by a generator `G` on the `secp256k1` curve (e.g., `commit(a) = a*G`). In practice, we'd actually need something like a Pedersen commitment to make these schemes secure. Pedersen commitments add another randomness factor, and although I opted for simple non-Pedersen commitments, know that the math doesn't change much when this randomness vector is introduced.

## What is the product argument?

The **product argument** is a protocol between a prover and a verifier where the prover aims to convince a verifier that the preimages of two committed values, `A1` and `A2`, have a public product `b`.

## Proof Approach

The product argument works by taking a similar proof approach to the sum argument. The idea is that the prover will give the verifier constant values, create derived values `a1'` and `a2'` using verifier-sourced randomness `x`, ask the verifier to compute the product of the derived values `a1'` and `a2'`, and use `x` in an expression with our committed constants to re-create the product of derived values.

### What are the derived values?

The formulas for the derived values are `a1' = a1*x + d1` and `a2' = a2*x + d2`, where `x` is the public verifier-sourced randomness, and `d1` and `d2` are secret random values chosen by the prover.

The commitments to the values `d1` and `d2` (`D1` and `D2`) are the constants given to the verifier **before** the verifier sends us their randomness `x`. This is to prevent the prover from cheating since letting the prover arbitrarily choose `d1` and `d2` after `x` has been revealed will allow them to forge a proof.

The operations performed on the original preimages to create the derived values can all be audited by the verifier, even without knowing `d1` and `d2`. Let's take a look at how this can be done by assuming the role of the verifier.

### Proving the derivation uses `a`

As the verifier, the only term we know from the expression of the derived value `a1' = a1*x + d1` is `x`. We don't know `a1` or `d1`, since those are prover-generated secrets, but we're determined to check that `a1'` was actually derived from `a1`.

The trick to this is linking the commitment of `a1` (`A1 = a1*G`) to the commitment of `a1'` (`A1' = (x*a1 + d1)*G = x*A1 + D1`).

Because of the algebraic properties of commitments, the commitment `A1'` is computable using `A1`, which comes from the prover's original statement, and `D1`, which is one of the commitments given by the prover at the start of the interaction:
`commit(a1') == x*A1 + D1`

The same process can be done for `a2'` using one of the other commitment constants, `D2`.

## Proving we know the product of a1 and a2

After proving the derived values are indeed derived from `a1` and `a2`, we need to show how the derived product `a1'*a2'` relates back to the original product of `a1` and `a2`.

We can do this by showing the original product of `a1` and `a2` exists as a term in the derived product `a1'*a2'`. Let's take a look at what the derived product of `a1'` and `a2'` look like when expressed in terms of `a1` and `a2` (liberally grouping the terms by their power of `x`):
`(a1*x + d1) * (a2*x + d2) = x^2*(a1*a2) + x*(a1*d2 + a2*d1) + (d1*d2)`

We can see the original product `a1*a2` sitting right there as the coefficient to `x^2`. To re-create this term, the verifier just needs to multiply the public product `b` with `x^2`.

The problem is the other two terms (the cross-terms). We can't create the coefficient of the second term without `(a1*d2 + a2*d1)`, or the 3rd term without `(d1*d2)`. Since the `a`s and `d`s are all secrets held by the prover, the prover can't give these raw values to the verifier to check with.

Instead, the prover gives the **commitments** to these cross-terms, let's call them `E1 = commit(a1*d2 + a2*d1)` and `E0 = commit(d1*d2)`, in the beginning of the interaction as 'constants' to the verifier (along with `D1` and `D2`). Verifying the equality by using the commitments of each term now becomes feasible:
`commit(a1'*a2') == x^2*commit(a1*a2) + x*E1 + E0`

Why does the prover have to give these `E1` and `E0` commitments **before** `x` is chosen? Well, if the prover were allowed to wait until after they received `x`, they could just lie about the product. The problem would reduce to replacing the product commitment `commit(a1*a2)`in the following equation with an arbitrary value `fake_product` and solving for `E0` like so:

**Original equality**:

`commit(a1'*a2') == x^2*commit(a1*a2) + x*E1 + E0`

**Swap out `a1*a2` with `fake_product`**:

`commit(a1'*a2') == x^2*commit(fake_product) + x*E1 + E0`

**Set `E1` to any random value and solve for `E0`**:

`E0 = commit(a1'*a2') - x^2*commit(fake_product)`

## Rolling products

Now let’s extend this idea to a rolling product. This means I want to prove `b = a1 * a2 * a3` without revealing `a1`, `a2`, `a3`, OR the intermediate product of any chosen subset of those elements.

If I wanted to extend our concept to this, I don’t just need the commitments of `a1`, `a2`, and `a3`. I also need the commitment to the intermediate product, which we'll call `b_12 = a1*a2` (assuming I’m multiplying from left to right).

Then I could just perform the product argument twice! One argument proving `b_12 = a1*a2` while keeping `a1` and `a2` hidden, and a second argument proving `b = b_12*a3` while keeping `a3` hidden. The only changes we'd have to make to our original product argument is having the prover give the verifier the commitment `B_12 = commit(b_12)` instead of a public scalar product `b`, and giving a commitment to `b_12`, `b_12' = x*b_12 + d_b` (where `d_b` is a new constant).

Taking this approach is *much* easier than doing a direct proof where we multiply all the derived values in one go, because then we'd be dealing with a higher degree polynomial and a nightmare of terms:
```
a1'*a2'*a3' = (a1*x + d1) * (a2*x + d2) * (a3*x + d3) =
x^3*(a1*a2*a3) + x^2*(a1*a3*d2 + a2*a3*d1 + a1*a2*d3) + x*(a3*d1*d2 + a1*d2*d3 + a2*d1*d3) + d1*d2*d3
```
I had trouble just typing that. Proving intermediate product commitments and keeping each argument to just 2 terms for the sake of simplicity just makes sense.

## Rolling products optimizations

Let's review the current rolling product process. As mentioned above, the prover creates a derived value for the intermediate product using a new  blinder `d_b`:
`b_12' = b_12*x + d_b`

With the straightforward product argument, the prover has to send the cross-term commitments (`E1` and `E0`), the derived intermediate product `b_12'`, AND `B_12`. I'd like to pose the question: can we get away with the prover *never* transmitting `B_12`? This would allow us to reduce the amount of data the prover sends the verifier.

Normally, if the prover provides the cross-terms but withholds `B_12`, the verifier would be stuck stuck. When the verifier would calculate `commit(a1'*a2')`, they'd have no way to check it, because the `x^2 * B_12` could not be calculated with `B_12`.

But if the derived intermediate value is `b_12' = b_12*x + d_b`, that means its commitment is:
`commit(b_12') = x*B_12 + D_b`

Notice that this expression **contains** `B_12`, and this can be used to our advantage to eliminate `B_12` from the verifier's check. If we multiply `commit(b_12')` by `x`, we get:
`x * commit(b_12') = x^2*B_12 + x*D_b`

Now, let's look at the verifier's main equation for the multiplied derived values:
`commit(a1'*a2') = x^2*B_12 + x*E1 + E0`

If we subtract the first equation (the original derived product expression) from the second equation (the expression with the intermediate value), the `x^2 * B_12` terms completely cancel each other out:
`commit(a1'*a2') - x*commit(b_12')`
`= (x^2*B_12 + x*E1 + E0) - (x^2*B_12 + x*D_b)`
`= x*(E1 - D_b) + E0`

The prover can simply combine their commitments and send `(E1 - D_b)` as a single cross-term, along with `E0`. The verifier computes `commit(a1'*a2') - x*commit(b_12')`, and checks if it equals `x*(E1 - D_b) + E0`.

Why does this work as a replacement to giving `B_12` and checking to see if it's in the original product and the intermediate value? Well let's think about it from the angle of trying to cheat the verifier as the prover. If the prover lied about `b_12'` actually containing `a1*a2`, then the `x^2` terms in the subtraction of thr 2 equations would not match and could never be eliminated. Doing so would require the prover to commit to a value, that when multiplied by `x`, can magically get rid of the `x^2` term and equal an expression made with prover-provided pre-committed values.

By verifying this way, the prover never has to send `B_12` over the network, saving precious bytes in our proof size.

# Conclusion

After going through the mechanics of the product argument, hopefully you see that it's not too different from the sum argument. It's a little bit more involved since the derived values `x*a1 + d1` and `x*a2 + d2` are polynomials, forcing us to do polynomial multiplication, but I believe the principals used in the argument are a natural extension to the principals used to build sum arguments. My next post will somewhat build on the concepts here to explain product arguments in the vectorized setting and get into how we can prove the inner-product of vectors in zero knowledge. Stay tuned!