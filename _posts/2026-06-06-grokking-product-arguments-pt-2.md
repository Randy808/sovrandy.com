# Grokking Product Arguments, Part 2

## Introduction

After starting to draft a post about vectorized product arguments, I realized one of the concepts I was introducing would better be explained in the scalar context first. This concept is another way to make a product argument, something I wrote about in my last post. Both the method in my last post, and the method in this post, are featured in the paper I'm using as reference, 'Efficient Zero-Knowledge Argument for Correctness of a Shuffle'.

I decided to give this product argument special treatment because it's used as part of the vectorized Hadamard product argument, but it's never explained in the simpler case of scalars. Understanding this will prove crucial for understanding the vectorized argument without banging your head against the wall for hours like I did. Know that I do take some liberties with my construction in an effort to make everything easier to follow, but the final result should lead to a very similar construction to the argument presented for Hadamard products in the reference paper.


## Some things to know

Just like in my last post, I abstract away Pedersen commitments to focus on the meat of the ideas. If you're having trouble generalizing the scheme to work with Pedersen commitments, feel free to leave an issue on this blog's Github repo.

Also, in this post I use the word **tweak** a lot, so I thought I'd define what I mean when I use it before we get into the nitty-gritty. To quote Michael Folkson, who [answers what it means in the context of public keys](https://bitcoin.stackexchange.com/questions/113232/what-does-it-mean-to-tweak-a-public-key), and who in turn quoted [Bitcoin Optech](https://github.com/bitcoinops/taproot-workshop/blob/master/2.2-taptweak.ipynb), "Tweaking a public key means to alter it with a value (the tweak) such that it remains spendable with knowledge of the original private key and tweak." I use it more to mean performing an operation on a value to get another related value. It may be an abuse of the term, but I use it in this way throughout my post.

## Alternative Proof Approach to single product argument

The single product argument involves proving that the product of numbers `a[i]` for `i=1..n` equals `b` without revealing any of the `a[i]`s. I covered the mechanics for this in my post 'Grokking Product Arguments', but now, I'd like to show an alternative way we can prove the same thing.

If you recall, the general approach for the product argument presented in part 1 was:
- have prover give a commitment to "cross-terms"
- have the verifier give the prover `x`
- have the prover make derived values `a[i]'` using `a[i]`s and `x`
- have the verifier multiply tweaked terms to get derived product `b'`
- have verifier re-create derived product using commitments

In the alternative approach to this argument, the prover proposes a new tweaking mechanism. Instead of tweaking each term `a[i]` into `a[i]'` and returning each `a[i]'` separately, we can **aggregate** our tweaked terms `a[i]'` *before* sending them to the verifier. Doing this would lower the number of distinct elements we have to give the verifier and turns `n` prover-given elements into `1`. This aggregation is done by summing the tweaked terms `a[i]'` together like so: `a[1]' + a[2]' + ...`. Believe it or not, this aggregation also opens the door for *another* optimization -- removing the `r[i]` terms.

Because summing the terms together proves an effective way to hide the individual values, you can think of `a[i+1]` serving as the new `r[i]` for `a[i]`. One thing to note is that despite the prover's ability to effectively hide the individual values, we would still like to add a random element `r` to not leak the `x`-scaled sum. Adding the random term `r` also helps adhere to our definition of, what the reference paper describes as, a 'special honest zero knowledge argument'. I won't go into the definition for what that is here, but it provides some guidelines on how to structure our protocol to be a special kind of zero knowledge argument.

So is that it? Optimization complete? Well, no. Some readers may notice that *just* replacing the `r[i]`s with a single `r` and summing the tweaked terms together will break one of the checks performed by the verifier. The verifier now can't verify whether the prover knows every individual term that was committed to, or just the sum.

Usually the verifier would check each tweaked term `x*a[i] + r[i]` by using the commitments `A[i]=commit(a[i])` and `R[i]=commit(r[i])` returned by the prover like so `x*A[i] + R[i]`, but now, if the verifier is just given `a'`, the prover can fake knowing the individual preimages `a[i]`. So how do we fix this?

One way is to have the verifier give **each term** a unique coefficient `x[i]`. Doing this prevents the raw sum of `a[i]`s from being forged since the prover would have to find `a'` such that `a'*G = R + x[1]*A[1] + x[1]*A[2] + ... + x[n]*A[n]`, **without** knowing the individual `a[i]`s and only knowing the aggregate sum. To solve this we'd need to solve the elliptic curve discrete log problem for which all modern EC crypto-systems are based. So pretty unlikely to happen.

Well there's one more challenge we'd like to overcome. Requiring the verifier to choose `n` `x`s increases the number of elements the verifier will have to send the prover. In the literature this metric is known as the **communication cost**. Our next challenge is to try to reduce this communication cost, and lucky for us.

But before I get into that, I'd like to modify the protocol a little bit to hopefully simplify the next explanation. Including the `r` in the derivation can be a bit tedious and generally serves as another piece of information we have to keep track of. What the reference paper does is prepend `r` into the list of preimages `a[i]`, turning `r` to `a[1]`. This would make the existing protocol stay mostly the same, except now, we can implicitly **re-define** `n` to be the number of elements in `a` *including* the `r` element. Alright, now let's get back to how we're going to tackle reducing the communication cost of the `x`s.

Instead of having the verifier pick `n` `x`s where `n` is the number of `a`s, the verifier can give the prover one verifier-chosen `x` and have the prover derive the other coefficients from it. More concretely, the verifier asks the prover to define `x[i]` as `x[i]=x^(i - 1)` for `i=1..n`. That way the verifier only needs to give one value to the prover, `x`. Problem solved!

Ok, one last thing. In our previous post we had the distinction of the simple case and the case of rolling products. This product argument has no such distinction given the `2` element case balloons the number of `a`s to `3` because of the implicit `r`. Since every case turns into the rolling case of `len(a[i]) > 2`, we'll be using the same concept of intermediate products `b[i]` introduced in the previous post. `b[i]` being defined as `b[i]=a[1]*...*a[i]`.

Also similar to our previous post, we will need to tweak these intermediate products to keep them secret. To do this, we'll tweak them the same way we tweaked our list of preimages `a`. We'll include a random element for `b[1]` and create a sum of intermediate products scaled by different powers of `x`.

Now, let's run our explanation of the product argument back with this new and improved protocol.

## What is the product argument?

To reiterate from my first post, a **product argument** is a protocol between a prover and a verifier where the prover aims to convince a verifier that the preimages of `n` committed values, `a[i]` for `i<=n`, have a public product `b`. Just like in my previous post, I'll be focusing on a simple case to better cement the idea with an example. This simple case will be when `n=3` since we'll need a random term to be included even when we're just interested in the product of 2 committed values.

I'll also add that as part of the statement being made, for which we need to make this argument for, the prover gives the verifier commitments for every `a[i]` and commitments to the rolling intermediate products `b[i]`.

If we're focusing on the case where `n=3`, we'll have 2 `b`s, `b[1]` and `b[2]`.

## Proof Approach

The idea for this new scheme is that the prover will give the verifier commitments to constant values, create derived values `a'` and `b'` using verifier-sourced randomness `x`, ask the verifier to compute the product of the derived value `a'` with `b'`, and have the verifier use `x` in an expression with the pre-committed constants to re-create the product of the derived values. Let's start the explanation of this argument by diving into what the constants are.

### What are the constants given at the beginning of the protocol?

The constants given to the verifier at the start of the argument are commitments to the same **cross terms** `d[i]` we described in part 1. These constants are given to the verifier **before** the verifier sends the prover their randomness `x`.

To make a product argument, we'll eventually have to compute the product of tweaked values `a'` and `b'`. Since these values are tweaked, the tweaked product won't only contain the public product we're trying to prove as a term with an `x` coefficient, it'll also include a bunch of extra terms. The coefficients of these extra terms can be defined before the verifier sends the prover `x`, and then later scaled by powers of `x` when the verifier re-creates the derived product.

If the committed coefficients do not describe the right polynomial structure, the prover is reduced to hoping the verifier's random `x` still fits with their fake values. This is very unlikely, which makes this construct effective in preventing the prover from cheating. Just like in the previous product argument, allowing these constants to be chosen freely after seeing `x` can lead to a forged proof, so it's imperative that the prover coughs up these values at the very start of the argument.

### What are the derived values?

Once the cross term commitments have been sent by the prover, the verifier can now send them a random `x` term the prover can use to make derived values `a'` and `b'`. These derived values can be made using the following expressions:
`a' = sum(x^(i - 1) * a[i])` for `i=1..n`
and `b'=sum(x^(i - 1) * b[i])` for `i=1..n`

### Proving the derivation uses `a`

As the verifier, the only term we know from the expression of the derived `a' = sum(x^[i]*a[i])` is `x`. We don't know any of the `a[i]` prover secrets. Since the verifier has given commitments to `a[i]` as part of the original statement, those commitments can now be used to link the secret preimages `a[i]`s to the public derived `a'`.

We do this by acknowledging that the commitment `A'` is computable using the commitments given to us at the start of the protocol, `A[1]`, `A[2]`, and `A[3]`. We then compute `A[1] + x*A[2] + x^2*A[3]` and do an equality check with `A'` to finish proving the link between the `a[i]`s and `a'`.

## Proving we know the product of a[1], a[2], and a[3]

After proving the derived value `a'` is indeed derived from `a[i]`, we need to show how the public product `b` can be derived from `a'` given `a[1]` and `a[2]` are both contained within the expression. In the previous product argument, we had a derivation for each term and had a way to link the product of the derived `a[i]'`s to the public product via reconstructing the commitment of the product between the derived `a'`s. In this model, we're just given a single sum of tweaks and have yet to find an operation on this sum that will yield the product of `a[1]`, `a[2]`, and `a[3]`. So the question is "what operation can we make on `a'` that will yield an expression with the product of `a[1]`, `a[2]`, and `a[3]` as a term?"

This is where the derived value `b'` comes in. We can multiply the derived sum `a'` with the sum of derived intermediate products `b'` to get an expression that suits our needs. If we think about the breakdown of the product from these derived sums, we see:
- `a'*b'` =
- `(a[1] + x*a[2] + x^2*a[3]) * (b[1] + x*b[2])` =
- `(a[1] + x*a[2] + x^2*a[3]) * (a[1] + x*a[1]*a[2])` =
- `a[1]*b[1] + x*a[1]*b[2] + x*a[2]*b[1] + x^2*a[2]*b[2] + x^2*a[3]*b[1] + x^3*a[3]*b[2]`


We can see the original product `a[1]*a[2]*a[3]` sitting right there as the coefficient to `x^3`, since `a[3]*b[2] = a[3]*a[1]*a[2]`.

To verify the expression above, we then take the **commitment** of `a'*b'` and recreate it using the cross-term commitments sent at the beginning of the argument and other verifier-known values. For the case of `n=2` (with the random term `a[1]`), the verifier wants to recreate commitments to the following terms that were coefficients to `x^i` in the expression above:
- `a[1]*b[1]`
- `x*(a[1]*b[2] + a[2]*b[1])`
- `x^2*(a[2]*b[2] + a[3]*b[1])`
- `x^3*b`

The constant value commitments mentioned earlier represent commitments to the extra coefficient terms in this derived product. By "extra coefficient terms", I mean everything that's not directly the final public product. Meanwhile, the final `x^3*b` term is computable by the verifier because `b` is public.

## Running through the protocol end to end

Let's recap this new protocol again to summarize the steps.

The prover starts by sending over the commitments to all possible cross terms between the secret intermediate values `b[i]` and the secret preimages `a[i]`. If `n=3`, the following represents all cross term products between `a'` and `b'`:

- `a[1]*b[1]`
- `(a[1]*b[2] + a[2]*b[1])`
- `(a[2]*b[2] + a[3]*b[1])`

You might notice that **included** in these cross terms are some the intermediate products `b[i]`. For example, the `a[2]*b[1]` in the second term actually evaluates to `b[2]` This is because of the relationship between `a[i]` and `b[i]`:
`b[i]=b[i-1]*a[i]` for `i > 1`

Now that the verifier has these values, they pick a random value `x` to give to the prover.

The prover responds with the derived values `a'` and `b'`:
- `a'` will be defined as `a' = a[1] + x*a[2] + x^2*a[3]`.
- `b'` will be defined as `b' = b[1] + x*b[2]`.

It's now the verifier's job to multiply these values together and verify the constants we got at the beginning can be used to make the derived product `a'*b'` when plugged into our expected algebraic structure. Here, our expected algebraic structure is:
- `a'*b'`
- `(a[1] + x*a[2] + x^2*a[3]) * (b[1] + x*b[2])`
- `a[1]*b[1] + x*a[1]*b[2] + x*a[2]*b[1] + x^2*a[2]*b[2] + x^2*a[3]*b[1] + x^3*a[3]*b[2]`
- `x^3*a[3]*b[2] + x^2*(a[2]*b[2] + a[3]*b[1]) + x*(a[1]*b[2] + a[2]*b[1]) + a[1]*b[1]`

So we need to multiply:
- `x^3` by the commitment for the final product `b`, since `a[3]*b[2] = a[1]*a[2]*a[3]`
- `x^2` by the commitment for `a[2]*b[2] + a[3]*b[1]`
- `x` by the commitment for `a[1]*b[2] + a[2]*b[1]`

Then sum all of those values together with `a[1]*b[1]` to make sure it's equal to `a'*b'`. And that's it!

# Optimizing away a term

There is one more optimization on the protocol we can make by changing the definition of `b'`. This alternate expression for `b'` uses the derivation `b'=sum(x^(m - i - 1) * b[i])`, with the exponent on the `x` counting **down** against the increasing indices of `b`. For our `n=3` example, where we only have `b[1]` and `b[2]` as intermediate products, this means:

`b' = x*b[1] + b[2]`

So how does this help us?

In the paper I'm using as reference, 'Efficient Zero-Knowledge Argument for Correctness of a Shuffle', something called the **zero argument** is introduced. In the scalar version we're using here, the zero argument can be understood as a cancellation trick. Aside from making the powers of `x` in `b'` count down instead of up, we also add one extra value to `b` side that equals the negative sum of the rest of the terms in `b`. This will cause one of the terms in the product `a'*b'` to equal the sum of intermediate products scaled by a power of `x`. Since the extra `b` value was chosen as the negative of the others, the coefficient on that `x` term now cancels to 0. This lets the verifier replace one reconstructed coefficient with a zero check.

This is easier to appreciate in the full vector argument because the zero argument is doing real work across many values at once. But even in this scalar walkthrough, the key idea is changing the direction of the powers lines terms up so that one part of the derived product can be compared to 0 instead of being reconstructed as another committed coefficient.