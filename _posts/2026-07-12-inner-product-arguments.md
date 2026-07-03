# Inner Product Arguments, Part 1

Since the focus of my last post was on the mechanics of Hadamard product arguments, I was originally going to use them as a starting point for explaining inner product arguments. In the end I decided to take a different pedagogical approach rather than simply adapting the Hadamard argument to the inner product argument. Instead I'll take another first principles approach. This will allow me to break down and re-contextualize the proof mechanisms we previously established and hopefully allow us to easier identify components for optimization.

## Notation

The notation I'll use for single value commitment functions will be a little different from my other posts. In my previous posts I tried to abstract away as much information as I could and used `commit(value)` to represent commitments to 'value' (where 'value' is a scalar). Now because it will be relevant to the stuff I'll show later, I'll parametrize the commitment function to include the generator as its second argument. Thus we write `commit(val, G) = val*G` to denote a commitment to the value `val` with the generator `G`. I'd also like the reader to assume the commitment function is an elliptic curve multiplication between the scalar value being committed and a generator point on the curve.

The commitment scheme for a vector `a` is `commit(a) = sum(commit(a[i], G_a[i]) for i..len(a))` where `G_a` is a public vector of different generators with length equal to `a`. We'll use different generator vectors `G_vec`, where `vec` is the vector name, for each vector we commit to.

# Stages of a zk argument

When talking about zk arguments that prove a relation between 2 hidden values, I think it's helpful to break the process down into 5 stages. The statement, transformation, consistency check, operation, and verification stage. I'll expand on these stages below.

## Statement

The statement is the assertion being made about the "hidden" values. Hidden meaning we can only see commitments to the values we are making assertions about.

## Transformation

The transformation hides the values in the statement such that the output is algebraically linked to the hidden values in a different way than the commitments are. The difference between transformed values and commitments is that transformed values can be used in operations that would be undefined on commitments. For example, multiplication is not defined between 2 commitments whereas it *is* defined between 2 transformed values.

## Consistency Check

This step is where the verifier checks the transformed values are algebraically linked to the hidden values. This check is defined in a way that makes it nearly impossible to forge and is performed by applying a similar transformation on the statement's commitment and checking that it equals the transformed value's commitment.

## Operation

The operation is where the transformed values are run through the operation in the assertion.

## Verification

This step verifies the algebraic link between the operation's result on the transformed values and the public result of the original values included in the statement.

# The naive approach

In this section I'd like to describe what a naive inner product argument looks like when broken down into the stages listed above.

## Statement

The statement for the inner product argument informally says "I would like to prove that 2 committed vectors, `a` and `b`, have an inner-product equal to a public scalar `c`."

As part of this statement the prover needs to give the verifier:
- The commitment scheme
- `A` where `A = commit(a, G_a)`
- `B` where `B = commit(b, G_b)`
- `c` where `c` is the inner-product of `a` and `b`
- Any other pieces of information required for the proof that can't be used to leak `a` and `b`
  - This will includes vectors `R_a`, `R_b`, and `R_c`

## Transformation

We now have to find a transformation that hides vectors `a` and `b`. One requirement of this transformation is that it must be some function of an arbitrary value `x` supplied by the verifier. Another requirement is that it must be possible for the commitment of the transformation to be recreated using an expression of the commitment values provided in the statement.

The simplest transformation that allows for this adds a different random number to each value in the vector like so:

- `a' = a*x + r1` where `r1` is a vector of random numbers
- `b' = b*x + r2` where `r2` is a vector of random numbers

This transformation against `a` and `b` results in:
```
a' = [
  x*a[1] + r_a[1],
  x*a[2] + r_a[2]
]

b' = [
  x*b[1] + r_b[1],
  x*b[2] + r_b[2]
]
```

## Consistency Check

For the verifier to acknowledge that these modified values are indeed derived from `a` and `b`, we must commit the result of our transformations, `a'` and `b'`, and reconstruct each vector commitment with the commitments of the original vectors `a` and `b`. Let's see how this process works for `a`.

Our original commitment `A` is defined as:
```
A = commit(a) = a[1]*G_1[1] + a[2]*G_1[2]
```

We must now use this commitment to re-create the commitment of `a'` which is defined as:
```
A' = commit(a') = (x*a[1] + r_a[1])*G_1[1] + (x*a[2] + r_a[2])*G_1[2]
```

With some algebraic manipulation, we can see that the terms that define `A`, `a[1]*G_1[1] + a[2]*G_1[2]`, already exist within `A'`:
```
A' =
(x*a[1] + r_a[1])*G_1[1] + (x*a[2] + r_a[2])*G_1[2] =
x*a[1]*G_1[1] + r_a[1]*G_1[1] + x*a[2]*G_1[2] + r_a[2]*G_1[2] =
x*a[1]*G_1[1] + x*a[2]*G_1[2] + r_a[1]*G_1[1] + r_a[2]*G_1[2] =
x*(a[1]*G_1[1] + a[2]*G_1[2]) + r_a[1]*G_1[1] + r_a[2]*G_1[2] =
x*(A) + R_a

where R_a = (r_a[1]*G_1[1] + r_a[2]*G_1[2])
```

We can see this transformation allows us to check the relation between the original commitments by multiplying our old commitment `A` by `x` and adding the commitment `R_a` onto it. The same process can be performed to verify `b'`.

# Operation

Let's look at what happens when we perform the inner product operation on transformed values `a'` and `b'`, and see if the result is something we can construct with the commitment for the public product `c`:

```
a' = [
  x*a[1] + r_a[1],
  x*a[2] + r_a[2]
]

b' = [
  x*b[1] + r_b[1],
  x*b[2] + r_b[2]
]

c' = inner_product(a', b') =  (
  (x*a[1] + r_a[1]) * (x*b[1] + r_b[1]) + # product of first row
  (x*a[2] + r_a[2]) * (x*b[2] + r_b[2]) # product of second row
)

=  (
  x^2(a[1]*b[1]) + x*(a[1]*r_b[1]) + x(r_a[1]*b[1]) + (r_a[1] * r_b[1]) + # Polynomial expansion of first row product
  x^2(a[2]*b[2]) + x*(a[2]*r_b[2]) + x(r_a[2]*b[2]) + (r_a[2] * r_b[2]) # Polynomial expansion of second row product
)

 =  (
  x^2*(a[1]*b[1] + a[2]*b[2]) +
  x*(a[1]*r_b[1] + a[2]*r_b[2] + r_a[1]*b[1] + r_a[2]*b[2]) +
  (r_a[1] * r_b[1] + r_a[2] * r_b[2])
)
```

This result can also be interpreted as a sum of dot products between the various components of `a'` and `b'`:
```
c' = inner_product(a', b') = (
  x^2*(a dot b)
  + x*(a dot r_b + r_a dot b)
  + (r_a dot r_b)
)
```

# Verification

Our goal for this stage is to identify how we can re-create the commitment of `c'` using the commitment of our public product `c`. Algebraically we can see the terms needed to create the original inner product `a[1]*b[1] + a[2]*b[2]` in `c` as the coefficient for `x^2`.

The other terms `(a[1]*r_b[1] + a[2]*r_b[2] + r_a[1]*b[1] + r_a[2]*b[2])` and `(r_a[1] * r_b[1] + r_a[2] * r_b[2])`, with coefficients `x^1` and `x^0` respectively, are also attached to this value. This means we'll need commitments for those as well.

Since none of these coefficients rely on the verifier-supplied `x`, we can calculate them before `x` is ever given and are what we define to be `R_c[1]` and `R_c[2]` in the statement.

With these additional terms `R_c[1]` and `R_c[2]`, we can verify the inner product by checking the following equality:
- `commit(c', G_c) = x^2*(commit(c, G_c)) + x*(R_c[1]) + R_c[2]`

# Optimizations

The rest of this post will detail optimizations for the inner-product argument presented by the paper 'Efficient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting' (aka 'BCC+16'). I took the time to break down the structure of this zk argument into stages because I wanted to frame each optimization as targeting a specific stage. Once the core optimization is understood, the rest of the actions taken can be thought of as compensating for the core change in the other stages.

# Optimization 1 - Transformation step

The first step of the scheme we wish to optimize targets the transformation step. To rehash, the transformation step takes in vectors `a` and `b` and spits out *equal size* transformed vectors `a'` and `b'`. Since researchers in this field generally don't want proving steps to involve returning data proportional in size to the input, a scheme was devised to reduce `a'` and `b'` into a singular scalar each. As we'll see later on, using this optimization alone won't allow us to make a correct inner product argument, but when mixed with optimization 2, it works.

Focusing back on this optimization, we intuitively know that this mysterious new transformation needs to involve all the terms of the input vector. Otherwise, we wouldn't be able to verify the inner-product since it's a function of every element. So now we ask, "What's the simplest way to consolidate the elements of a vector into a single value?"

Hopefully your mind went to the same operation mine did. It's one of the simplest operations math has to offer -- addition. Now let's see the impact of making the transformation the sum of all elements in a vector.

One thing we get for free when we do this is the elimination of the per-element random term. If the first 2 elements are summed, we can also think of the random elements associated with each term as being summed. Thus, all random elements can be consolidated into one element.

Also, because it will make things easier later on, let's add the random value for each vector as the 1st element of the vector. Since our transformation will sum it up whether it's attached to one element or if it's another element itself, we take the route that allows us to write the simplest notation.

Here's what the output of this new transformation looks like:

```
a' = [
  a[1] + x*a[2] + x*a[3]
]

b' = [
  b[1] + x*b[2] + x*b[3]
]
```

If the above looks weird to you, you might be onto something. Simply transforming the terms like this with a simple addition is *not* secure.

If the verifier only checks the aggregate, then different underlying vectors that lead to the same aggregate can be trivially forged. For example, if the check only depends on `a[2] + a[3]`, then replacing `a[2]` with `a[2] + 1` and `a[3]` with `a[3] - 1` gives the same transformed value. If values were allowed to be forged this way, the verifier would no longer have probabilistic confidence that the original vector was committed, since it could now be a different vector that was trivially assembled to have the same sum.

Fortunately we have a fix for this. We just make sure every row has a different power of `x` applied to it before they are all summed together:

```
a' = [
  a[1] + x*a[2] + x^2*a[3]
]

b' = [
  b[1] + x*b[2] + x^2*b[3]
]
```

## Consistency check

Now that we've transformed our hidden input, it's time to check the output of the transformation for algebraic consistency. To do this, we must commit `a'`. But how do we apply `G_a[1]` to the `a1` part, `G_a[2]` to the `a2` part, and `G_a[3]` to the `a3` part if all three values have been merged into `a'` and can no longer be separated?

With some thought, someone eventually came to the realization of trying to mirror the operation performed on the elements onto the generators that correspond to those elements. Let's see what happens when we use `G_a' = G_a[1] + G_a[2] + G_a[3]` as our new generator.

I'm going to leave the new generator for `a'`, `G_a'`, broken down into a sum with the individual generator terms from the statement to allow us to see what happens to all the individual terms in the result `a'*G_a'` algebraically:

```
a' = [
  a[1] + x*a[2] + x^2*a[3]
]

commit(a') = (a[1] + x*a[2] + x^2*a[3]) * (G_1[1] + G_1[2] + G_1[3])
= a[1]*G_1[1] + a[1]*G_1[2] + a[1]*G_1[3] + x*a[2]*G_1[1] + x*a[2]*G_1[2] + x*a[2]*G_1[3] + x^2*a[3]*G_1[1] + x^2*a[3]*G_1[2] + x^2*a[3]*G_1[3]

```

Remember our goal to prove the expression for  `commit(a')` contains the commitment to the original vector `commit(a) = a[1]*G_a[1] + a[2]*G_a[2] + a[3]*G_a[3]`. To work towards this goal, let's try to at least group the relevant terms together and keep it at the front of the expression:
```
commit(a') = a[1]*G_a[1] + a[1]*G_a[2] + a[1]*G_a[3] + x*a[2]*G_a[1] + x*a[2]*G_a[2] + x*a[2]*G_a[3] + x^2*a[3]*G_a[1] + x^2*a[3]*G_a[2] + x^2*a[3]*G_a[3]
= (a[1]*G_a[1] + x*a[2]*G_a[2] + x^2*a[3]*G_a[3]) + a[1]*G_a[2] + a[1]*G_a[3] + x*a[2]*G_a[1]  + x*a[2]*G_a[3] + x^2*a[3]*G_a[1] + x^2*a[3]*G_a[2]
```

The problem with this expression is that even though the first grouping, `(a[1]*G_1[1] + x*a[2]*G_1[2] + x^2*a[3]*G_1[3])`, has the same terms of our commitment, it *also* adds a different power of `x` to each term. This is a problem because we don't have individual elements needed to re-create that expression. The whole point of this new transformation is so we could eliminate the need for per-element commitments! This leaves us at an impasse.

The brilliance of the paper 'Efficient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting' is that it found a way to tackle this problem. Their insight is that even though we don't know the individual commitments of the original `a` vector, there might be modifications they could make to the **individual** elements in the `G` vector to force the expression into a checkable form.

Their answer to this was to have the verifier pre-multiply each G_i[i] with the inverse of the `x` that would be attached to the corresponding element in the first grouping to make the `x`s cancel out. Concretely, they would modify `G_1 = [G_1[1] + G_1[2] + G_1[3]]` to `G_1' = G_1[1] + (x^-1)*G_1[2] + (x^-2)*G_1[3]`.

With `G_1'`, the first grouping would turn into the original commitment and allow the verifier to re-construct `commit(a')` using `commit(a)`. The other terms impacted are cross-terms anyway so we can just give the commitments of the `x` co-efficients at the beginning and have the verifier adapt it with the `x` they choose during the transformation step for verification.

Let's see how this works in practice:

```
a' = [
  a[1] + x*a[2] + x^2*a[3]
]
commit(a') = (a[1] + x*a[2] + x^2*a[3]) * (G_1'[1] + G_1'[2] + G_1'[3])

= (a[1] + x*a[2] + x^2*a[3]) * (G_1[1]*(x^0) + G_1[2]*(x^-1) + G_1[3]*(x^-2))

= a[1]*G_1[1] + a[1]*G_1[2]*(x^-1) + a[1]*G_1[3]*(x^-2) + x*a[2]*G_1[1] + x*a[2]*G_1[2]*(x^-1) + x*a[2]*G_1[3]*(x^-2) + x^2*a[3]*G_1[1] + x^2*a[3]*G_1[2]*(x^-1) + x^2*a[3]*G_1[3]*(x^-2)

= (a[1]*G_1[1] +  x*a[2]*G_1[2]*(x^-1) + x^2*a[3]*G_1[3]*(x^-2)) + a[1]*G_1[2]*(x^-1) + a[1]*G_1[3]*(x^-2) + x*a[2]*G_1[1] + x*a[2]*G_1[3]*(x^-2) + x^2*a[3]*G_1[1] + x^2*a[3]*G_1[2]*(x^-1)

# Let's simplify the first grouping
= (a[1]*G_1[1] +  a[2]*G_1[2] + a[3]*G_1[3]) + (x^-1)*a[1]*G_1[2] + (x^-2)*a[1]*G_1[3] + x*a[2]*G_1[1] + x^-1*a[2]*G_1[3] + x^2*a[3]*G_1[1] + x*a[3]*G_1[2]

# Now let's group the cross terms by their `x` coefficient
= (a[1]*G_1[1] +  a[2]*G_1[2] + a[3]*G_1[3]) + (x^-2)*a[1]*G_1[3] + (x^-1)*(a[1]*G_1[2] + a[2]*G_1[3]) + x*(a[2]*G_1[1] + a[3]*G_1[2]) + x^2*a[3]*G_1[1]

```

As you can see from the calculation above, the first grouping is the commitment to the original vector `a`.

## Operation

```
a' = [
  a[1] + x*a[2] + x^2*a[3]
]

b' = [
  b[1] + x*b[2] + x^2*b[3]
]

a'*b' = (a[1] + x*a[2] + x^2*a[3]) * (b[1] + x*b[2] + x^2*b[3])
= a[1]*b[1] + a[1]*x*b[2] + a[1]*x^2*b[3] + x*a[2]*b[1] + x*a[2]*x*b[2] + x*a[2]*x^2*b[3] + x^2*a[3]*b[1] + x^2*a[3]*x*b[2]  + x^2*a[3]*x^2*b[3]
= x^4*(a[3]*b[3]) + x^3*(a[2]*b[3] + a[3]*b[2]) + x^2(a[1]*b[3] + a[2]*b[2] + a[3]*b[1])+ x*(a[1]*b[2] + a[2]*b[1]) + a[1]*b[1]
```

One may think a similar verification operation to the consistency check where the verifier replaces `G` with `G'` can now be performed to eliminate the `x` coefficients that prevent the original dot product (`a*b = a[1]*b[1] + a[2]*b[2] + a[3]*b[3]`) from forming, but they'd be understandably incorrect. Using a new generator `G_c'` for the commitment of the public product is not sufficient. Let's see why.

## The issue

Now that we have `= x^4*(a[3]*b[3]) + x^3*(a[2]*b[3] + a[3]*b[2]) + x^2(a[1]*b[3] + a[2]*b[2] + a[3]*b[1])+ x*(a[1]*b[2] + a[2]*b[1]) + a[1]*b[1]`, you'd think we could eliminate the coefficients of the `x^2` on `a[2]*b[2]` and `x^4` on `a[3]*b[3]`, and in fact we can! All we have to do is define `G_c'` to be `G_c' = G_c + x^-2*G_c + x^-4*G_c`. Each generator term will generate a copy of all terms in `c'` scaled by the coefficient on that generator term. Each term in the resultant expression of `c'*G_c'` that *isn't* a term of the dot product of `a` and `b` should therefore be treated as a cross-term.

The problem with this is that this will generate 2 additional terms **without** an `x` coefficient that we'd have to commit to. To better visualize this process, let's work it out:
```
c' * G_c' =
  (x^4*(a[3]*b[3]) +
  x^3*(a[2]*b[3] + a[3]*b[2]) +
  x^2(a[1]*b[3] + a[2]*b[2] + a[3]*b[1]) +
  x*(a[1]*b[2] + a[2]*b[1])
  + a[1]*b[1]) * (
    G_c + x^-2*G_c + x^-4*G_c
  )

= ... honestly an expression that's too long for me to want to work out

Let's just see which terms would be left over without an `x` coefficient:

(a[1]*b[1])*G_c +  x^2(a[1]*b[3] + a[2]*b[2] + a[3]*b[1])*x^-2*G_c  + (x^4*(a[3]*b[3]))*x^-4*G_c =

a[1]*b[1]*G_c +  a[1]*b[3]*G_c + a[2]*b[2]*G_c + a[3]*b[1]*G_c + a[3]*b[3]*G_c =
(a[1]*b[1]*G_c + a[2]*b[2]*G_c + a[3]*b[3]*G_c) + a[1]*b[3]*G_c + a[3]*b[1]*G_c  =
```

Since we have additional terms `a[1]*b[3]*G_c` and `a[3]*b[1]*G_c` not scaled by `x`, this introduces a security concern. If the statement claims the dot product is 5, a prover could supply malicious vector values whose actual dot product is 3, while the additional unscaled cross-terms contribute the missing 2. The verifier would only see the combined value 5 and could not distinguish it from the claimed dot product.

# Optimization 2 - Negative exponents on `b`

The next optimization is small but powerful. It allows us to fix the problem preventing us from running with the previous optimization. The insight for this optimization is using **negative** `x` exponents for the transformation step of `b`, and only `b`.

Instead of `b'` being defined as:

```
b' = [
  b[1] + x*b[2] + x^2*b[3]
]
```

We make it:
```
b' = [
  b[1] + (x^-1)*b[2] + (x^-2)*b[3]
]
```

To accommodate this change we also need to change how we modify the generator for the consistency check of `b'`. Instead of `G_b' = G_b[1] + (x^-1)*G_b[2] + (x^-2)*G_b[3]`, we'll define `G_b'` as `G_b' = G_b[1] + x*G_b[2] + x^2*G_b[3]`. Now, because the `x`s of `a'` and `b'` will cancel out, we'll get the following as the dot product:
```
a' = [
  a[1] + x*a[2] + x^2*a[3]
]

b' = [
  b[1] + (x^-1)*b[2] + (x^-2)*b[3]
]

a'*b' = (a[1] + x*a[2] + x^2*a[3]) * (b[1] + (x^-1)*b[2] + (x^-2)*b[3])
= a[1]*b[1] + a[1]*(x^-1)*b[2] + a[1]*(x^-2)*b[3] + x*a[2]*b[1] + x*a[2]*(x^-1)*b[2] + x*a[2]*(x^-2)*b[3] + x^2*a[3]*b[1] + x^2*a[3]*(x^-1)*b[2]  + x^2*a[3]*(x^-2)*b[3]
# Now let's group the inner-product terms in the front of the expression
= (a[1]*b[1] + x*a[2]*(x^-1)*b[2] + x^2*a[3]*(x^-2)*b[3] ) + a[1]*(x^-1)*b[2] + a[1]*(x^-2)*b[3] + x*a[2]*b[1] + x*a[2]*(x^-2)*b[3] + x^2*a[3]*b[1] + x^2*a[3]*(x^-1)*b[2]
= (a[1]*b[1] + a[2]*b[2] + a[3]*b[3] ) + (x^-1)*(a[1]*b[2]) + (x^-2)*(a[1]*b[3]) + x*(a[2]*b[1]) + (x^-1)*(a[2]*b[3]) + x^2*(a[3]*b[1]) + x*a[3]*b[2]
= (a[1]*b[1] + a[2]*b[2] + a[3]*b[3]) + (x^-2)*(a[1]*b[3]) + (x^-1)*(a[1]*b[2] + a[2]*b[3]) + x*(a[2]*b[1] + a[3]*b[2]) + x^2*(a[3]*b[1])
```

In our verification step we just need to provide 4 cross terms (the coefficients on `x^-2`, `x^-1`, `x`, and `x^2`) and no modification of the committed dot product is needed!

# Optimization 3 - Recursion

This optimization provides the biggest reduction in communication cost and it's only possible with a slight modification of optimization 1. In this modified version, we don't smush all the elements into 1 expression, we smush them into groups instead.
So this:
```
a' = [
  a[1] + x*a[2] + x^2*a[3]
]

```

Becomes this:
```
a' = [
  a[1] + x*a[2],
  a[3] + x*0
]
```

Keeping that in mind, the main idea of this recursion-focused optimization is having the prover replace giving `a'` and `b'`, and instead ask the verifier to **create** the commitments of `A' = commit(a')` and `B' = commit(b')` like they would in the consistency check. The prover then withholds the values of `a'` and `b'` and *just* gives the verifier `c'`. The prover also commits to c' as C'. Using C, the cross-term commitments fixed before x, and the verifier's challenge x, the verifier checks that C' is the correct transformed inner-product commitment.

The assertion of whether the inner-product of `a'` and `b'` equals the prover-supplied `c'` is still unproven. The prover then asks the verifier to perform *another* inner-product argument using `A'`, `B'`, and `c'` to prove this assertion instead of having the verifier receive and perform the operation on `a'` and `b'` directly.


## Why skip the operation step?

Because we group each cross-term by their `x` co-efficient, and the number of different cross-terms with different `x` coefficients scales with the size of `a'` and `b'`, this step aims to reduce the number of cross-terms by reducing `a'` and `b'`.

In our 3 element example, since we only send the cross-terms needed to make `A'` instead of `a'` directly, we don't have to worry about the increase in the number of elements of `a'` with the grouping scheme. The idea is we can defer the inner-product operation of `a'` and `b'` and prove it with the operation of smaller `a''` and `b''` vectors instead.

It may be easier to reason why this works using an inductive framework. If we can assert the product of preimages for `A` and `B` with knowledge of `c = inner_product(a, b)`, we should also be able to assert the product of preimages for `A'` and `B'` with knowledge of `c' = inner_product(a', b')`. The only part we need to be careful about is properly linking A, B, and C to A', B', and C'.

# Conclusion

I'm over explaining the inner-product argument from the cited paper here so I'm going to wrap it up here. In a future post I'll probably try to make a part 2 where I explain the bulletproofs version of the inner-product argument.