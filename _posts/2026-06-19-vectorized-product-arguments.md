# Vectorized Product Arguments

## Introduction

In this post I'll delve into what the vectorized product argument is and how it works. I was going back and forth trying to decide whether to group in the inner product argument here, but I now think the two concepts are different enough to warrant independent posts. Also, I suggest reading my last two posts on product arguments before reading this one since I assume knowledge of those explanations to simplify what's written here. This argument is thus framed as an extension of sorts.

## Some things to know

Maybe I should have brought this up in my previous post, but I am using `1`-based indexing when using array notation on sets or vectors. I also tend to implicitly assume the role of prover sometimes so know if you're reading a snippet written in first person, I'm probably discussing a prover action.

Given that we're working with vectors now and matrices going forward, I'd also like to mention that I refer to the `i`th **column** of matrix `A` as `a_i` and the `i`th **row** as a[i]. I'll always use array notation when indexing into those rows or columns so the first element of the first column will be denoted as `a_1[1]`, and the first element of the first row will use `a[1][1]`.

Finally, as usual, I avoid using Pedersen commitments in my writing in exchange for a simpler mental model but Pedersen commitments *are* what's used in practice. I believe once you understand the core of my post, extending to Pedersen commitments isn't too big of a leap which is why I omit it. I may use them in my explanations in the future, but I feel like they'll impede understanding in these explanatory posts more than they'll help.

## Vectorized product argument (Hadamard argument)

The difference between proving a Hadamard product as opposed to a simple product between integers is that instead of proving a product of committed values, we're now proving **multiple different** products corresponding to **different sets** of committed values.

To ease into our understanding of the vectorized product argument, let's try to create a product argument using vectors, where all vectors only have **one** element. If in the regular product argument we're proving `product(a[i]) for i=1..n` is equal to `b`, then for the vectorized approach we can make each preimage `a[i]` its own one-dimensional vector `[a[i]]`. Instead of proving `b` is the product of secret preimages `a[1]` and `a[2]` like in the original product argument, the prover will now have 2 secret preimage **vectors** `[a[1]]` and `[a[2]]`. The verifier is then given 2 vectorized commitments `[A_1]` and `[A_2]`. Everything else in the proving process for the product argument is the same.

## How does this approach extend to vectors with multiple elements?

First, let's add a restriction and say each vector has to be the same size. If the vectorized product argument can be seen as proving multiple product arguments at once, this restriction will force every product argument to have the same number of committed factors.

Our next step is to group terms of the same position across product arguments. So if `a[1][1]` and `a[1][2]` are the secret terms for one product argument, and `a[2][1]` and `a[2][2]` are the secret terms for another product argument, then the prover would respectively group the first and second terms into secret vectors like so:

`v_first_terms = [a[1][1], a[2][1]]`

`v_second_terms = [a[1][2], a[2][2]]`

`product_terms = [b[1], b[2]]`

If you think of them as column-vectors and assembled them into a matrix `[v_first_terms, v_second_terms, product_terms]`, each `row` would have the terms for its own product argument. Later in this post, whenever I refer to the `row`, I'm referring to the individual product arguments.

For each product argument, `a[1][1] * a[1][2] = b[1]` and `a[2][1] * a[2][2] = b[2]`, we run the same product argument described in the last post.

To recap, in this naive construction of a Hadamard argument, the prover's public statement to the verifier now asserts that we know preimages for commitment vectors `v_1 = [A[1][1], A[2][1]]` and `v_2 = [A[1][2], A[2][2]]` that have a public Hadamard product of `product_terms = [b[1], b[2]]`.

## Why is the Hadamard product important

Why bother with introducing vectors and calling it a 'Hadamard' product argument if we can accomplish the same thing without the whole notion of a vector? The answer lies in the fact that there is a major speedup we can leverage by aggregating terms.

### The optimization

Let's review the single value argument to help us discover how we can optimize the vectorized product argument through aggregation.

The product argument shown in `Grokking Product Arguments Part 2` goes as follows:
- Prover makes statement which includes commitments to preimages `a` and public intermediate products `b`
- Prover also gives commitments to **cross-terms**
- Verifier gives prover an integer `x`
- Prover aggregates all `a`s into one term AND all `b`s  into another term using verifier's `x`
- Prover gives aggregate terms `a'` and `b'` to verifier
- Verifier computes the product of `a'` and `b'` which should contain `a*b` as a term
- Verifier re-creates the commitment of `a'*b'` using the commitment of the public product and cross term commitments

In our naive vectorized approach, the statement includes commitments to **each** preimage `a`, but what if there was a way to aggregate commitments for each column vector into a single element -- reducing the cost of sending `n` commitments in a vector to just sending one commitment?

The answer to that ended up being "yes," which leads to another follow-up: whether a product argument can be constructed on the vector level using these single value vector commitments. As you can imagine, the answer to this is also "yes." We'll now focus on *how* the product argument can be done on the vector-level with single value commitments.

## The new commitment function for vectors

We define `vector_commit(v)` for a vector `v` to be the sum of all elements in `v` **after** being run through the single value commitment scheme, with each value commitment in the vector using a **different** generator. In other words, a vector commitment can be thought of as the sum of its committed elements, where each element gets its own generator.

The commitment scheme that works for our purposes looks like this (where the generator used depends on `i`):
`commit(a_vec) = sum(commit(a[i], i)) for i=1..len(a_vec)`

So now we get to figure out how all the other math follows from this new commitment scheme. Like usual, let's work through an example of vectorizing, and then proving, `2` product arguments.

## Example

Since we're proving `2` product arguments, each vector will have `2` elements. We'll also define each product argument to have `2` factors. Since we always set the number of vectors equal to the terms, you might *think* we only have `2` vectors. In actuality, it'll lead to **`3`** vectors since there's a vector for each term **and** the term we add in for randomness (referenced in part 2).

Let's take a look at what each party has to perform at each step and make notice of the changes that come with the vectorized approach.

### Statement

First, assuming the role of the prover, let's define our 3 secret column vectors `a_1 = [a_1[1], a_1[2]]`, `a_2 = [a_2[1], a_2[2]]`, and `a_3 = [a_3[1], a_3[2]]`. We also assert that the product of all elements at row index `i` across the `a` vectors equals the `i`th element of the public product vector, `b_n`. If all column vectors were appended into a matrix, we can conceptualize elements at the same index across column vectors as `rows`.

As part of our statement to the verifier, we'll make the commitments for these column vectors `A_1 = A_1[1] + A_1[2]`, `A_2 = A_2[1] + A_2[2]`, and `A_3 = A_3[1] + A_3[2]`.

### Cross terms

Since we're assuming everything else works like the original product argument, we also give cross-term commitments for each `row` as if they were being proven independently.

### Verifier `x`

Now comes the verifier's turn to choose `x`. When they do, instead of just scaling every **term** with a power of `x`, the prover now returns a vector sum where they scale every **column vector** with a power of `x` and get a resulting sum vector `a'` like so:

`a' = a_1 + x*a_2 + x^2*a_3`

This process is also done with the rolling product vectors `b_i`:

`b' = b_1*x + b_2`

To mirror the paper, I’ll put the random masking vector at the end of the `b` list. So in this example, `b_2` is the random vector and `b_1` is the intermediate rolling product vector. We have no `b_3` because that would just be the public product in the statement.

### Verifying the vector

The tweaked sum vector `a'` still needs to be checked, just like the tweaked single values did. To do this check, we can re-create the commitment of sum vector `a'`, `A'`, using the commitments of individual column vectors `A_i`. This looks like `A' = A_1 + x*A_2 + x^2*A_3`. A similar process is then done for the `b`s in `b'`, `B = x*B_1 + B_2`.

Like that, we're still able to make this commitment scheme work for verifying derived vectors `a'` and `b'`!

### Verifier computes product

The process continues with the similarities in that we compute the Hadamard product between `a'` and `b'`. The big difference is the last step where we have to re-create the commitment of the resultant vector `a'*b'` with previously committed values. As a refresher, the committed values we use for this are the column vector commitments that were in the statement, and the cross-term commitments given at the beginning of the exchange.

Also, we will be using the **zero argument** optimization introduced in `Grokking Product Arguments Pt 2` since it still improves this version of the argument (which is why the exponent on the `x`s for `b_i` counted down above).


To solidify this product operation in our head, let's work out the first row of this product vector below in terms of column vector elements:

- `a'[1]*b'[1]` =
- `(a_1[1] + x*a_2[1] + x^2*a_3[1]) * (x*b_1[1] + b_2[1])` =
- `x*a_1[1]*b_1[1] + x^2*a_2[1]*b_1[1] + x^3*a_3[1]*b_1[1] + a_1[1]*b_2[1] + x*a_2[1]*b_2[1] + x^2*a_3[1]*b_2[1]`
- `x^3*a_3[1]*b_1[1] + x^2*(a_2[1]*b_1[1] + a_3[1]*b_2[1]) + x*(a_1[1]*b_1[1] + a_2[1]*b_2[1]) + a_1[1]*b_2[1]`

To verify this product contains the public product in the old single value approach, we would just re-create the commitment of `a'*b'` using the commitments we were given for each row. This would work, but there IS a more efficient way to do this. Let's transition into our last optimization.

### The cross terms

If we look at the breakdown of each row `i` in `a'*b'`, we'll see they're ALL of the form:
- `x^3*a_3[i]*b_1[i] + x^2*(a_2[i]*b_1[i] + a_3[i]*b_2[i]) + x*(a_1[i]*b_1[i] + a_2[i]*b_2[i]) + a_1[i]*b_2[i]`

Every row has terms with the same powers of x, `x^0`, `x^1`, `x^2`, and `x^3`. If we re-defined our protocol to skip doing a **per-row** reconstruction of `a'*b'`, and instead summed every row, we can group cross terms with the same `x` factor which means the prover could also aggregate the grouped cross terms sent at the beginning of the argument. This would lessen the number of cross terms the prover has to send, therefore reducing the **communication cost**.

### Attacking our aggregate scheme

One problem that we've run into before is the vector for a prover to cheat if they only know the aggregate and not the individual terms. In our example, there is a way to only know the sum of rows for `a'*b'` and not know the individual Hadamard products for each individual row. By forgoing individual row checks, we are skipping validation of the individual product arguments and a malicious prover could forge a proof.

For example, the prover could assert that the product of row `a[1]` is `b[1]` and the product of row `a[2]` is `b[2]` but actually make the product of `a[1]` be `b[1] - 1` and the product of `a[2]` be `b[2] + 1`. Since we're only checking the sum of the product arguments in the naive approach, this would validate even though we lied about what the individual products were!

To harden our scheme against this attack, we ask the verifier to choose a new value `y`. We'll then incorporate `y` into our sum by doing `(a'*b')[i] *y^i`. Once we weight the rows with `y`, the fake aggregate becomes something like `(b[1] - 1)*y + (b[2] + 1)*y^2`, which no longer matches `b[1]*y + b[2]*y^2` except for unlucky choices of y.

The big question we have to ask now is **where** in our protocol should the verifier give us this `y` value?

If we asked the verifier to give the new `y` at the same time they gave `x`, this would stop us from aggregating the cross-terms since these coefficients to the `x` terms in the row sum now include a `y` term.

The verifier must now give `y` **before** the prover gives the cross term commitments for the aggregated row check. That way the prover can incorporate `y` into the cross term commitments and still have the verifier use `x` to scale what the product terms in the final sum `(a'*b')[i] * y^i` should be.

## Link to inner product argument

If you look closely at the sum we performed in the step to reduce the number of final cross terms, you'll see that it was an addition of all the elements in a vector that already represented an element-wise multiplication. Meanwhile, an inner-product is also the sum of elements in a vector resulting from an element-wise product. To prove we know the product of committed vectors `a_i`, we have computed the `y`-weighted **inner product** of `a'` and `b'`. We'll leverage this fact in our next post.

## Conclusion

The main idea behind the Hadamard argument is that it lets us take a bunch of product arguments with the same structure and prove them together. Each row can be thought of as its own product argument, but instead of sending every commitment and cross-term for every row separately, we stack matching terms into vectors and use vector commitments to keep communication small.

From there, the argument starts to look very similar to the single-value product argument. We still use `x` to aggregate terms inside each product argument, but after vectorizing, we now have multiple row-level derived products to account for. If we just added those row results together directly, a prover could potentially fake the split across rows while keeping the final aggregate looking correct. This is where `y` comes in. By weighting each row-derived product with a different power of `y`, the verifier makes it much harder for fake row values to cancel out in the aggregate.

This is also where the connection to inner products starts to show up. Once we multiply matching entries, scale them by powers of `y`, and add everything together, we are effectively doing a weighted inner product. So while the Hadamard argument starts as a way to prove element-wise products, the final aggregation step naturally leads us toward inner-product arguments.

In the next post, I’ll dig more directly into that inner-product view and show how the same aggregation ideas can be used as their own standalone argument.


