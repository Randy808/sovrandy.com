# Confidential Transactions for Dummies Part 1 - Pedersen Commitments

## Prerequisite knowledge
The requirements are knowledge of the elliptic curve discrete log problem and elliptic curve field operations.

## Why should you care about privacy on the blockchain?
I want you to think about every financial transaction made within the past year. Political donations, pharmacy purchases, food delivery orders, etc. Now imagine all that information made public. Or worse—made available to a blackmailer to abuse. Every piece of information about you in the wrong hands can be used against you. Information about what you're purchasing, who you're purchasing from, or your purchasing amount is no exception, yet this is the case we find ourselves in with most blockchains today. So what do we do about this privacy problem?

## Addressing the privacy problem
Having good privacy hygiene and not reusing addresses does some of the heavy lifting in pseudonymizing recipients, but one fundamental component to privacy is being able to hide transaction amounts. Even if I can't see what you're purchasing, knowing that you spent $500 at a pharmacy tells me plenty. At first, hiding amounts on the blockchain may seem like an impossible task, but there *is* a way! This is what's accomplished by **confidential transactions**.

## What are confidential transactions?
Confidential transactions are transactions where the output amounts are hidden to everyone but me, my recipient, and anyone we decide to share it with. The scheme takes the amount I'm spending and 'hides' it with a function that allows everyone to verify I'm not spending more money than I've received. The output of this 'hiding' function is what we refer to as a **commitment**, and given a commitment, there is *no feasible way* to recover the original input amount. The original amount is instead hidden in a different part of the transaction for the recipient to recover so they can verify it corresponds with the commitment. If the amount didn't correspond with the commitment, no future recipients would accept the transfer.

In this post I'll be focusing on the function that transforms the amount into a commitment, which in this context is a "Pedersen Commitment."

## What are Pedersen commitments?
Let's break this term down. The first thing I'll explain is what a commitment is. A commitment is an output value of a function f that takes in a secret, or multiple secrets, as input. What makes it a *commitment* is the property that the input can't easily be derived from the output (which we refer to as **hiding**), and that another input can't easily be found that creates that same output (what we call **binding**). 

**Pedersen** commitments rely on the assumption that the 'discrete log problem' is hard. With this assumption, it's not computationally feasible to find another input that equals that output. **Opening** a commitment means obtaining the input from the committer that produces the commitment.

The paper that introduced Pedersen commitments, written by Torben Pryds **Pedersen**, used it as part of a larger secret sharing scheme and originally formed the commitment like so:

`g^a · h^b mod p`

Where `a` represents the number we want to 'commit' to, and `b` represents a per-commitment **nonce**, called the **blinding factor**, that makes sure we can make different commitments for the same committed value.

In the context of elliptic curves, the expression becomes:

`a*G + b*H`

The original scheme worked in a multiplicative group where `g^a` and `h^b` are both integers for which multiplication is defined. For elliptic curves, `a*G` and `b*H` are **points** for which point multiplication is **not** defined—we only have point addition. This naturally causes a change in the expression to use point addition instead.

## How do Pedersen commitments help us hide transaction amounts?
The main goal of designing a confidential transaction scheme is to maintain the integrity of the blockchain so that no money is unjustly created. Pedersen commitments allow us to do that by allowing us to perform *arithmetic operations on commitments*. 

Because I can perform arithmetic operations on commitments, I can subtract the sum of all output commitments from the sum of all input commitments and verify that the result is 0 to verify no money is being created (or destroyed in a way that's unaccounted for). In short, using `f` to represent the function that turns values into Pedersen commitments, our check looks like:

`f(inputs) - f(outputs) == 0`

Let's try an example. Imagine we have one input value of 3 represented by the commitment `in_0 = (3*G) + (7*H)`. With this input we make 2 outputs—one with the value of 2 represented by the commitment `out_0 = (2*G) + (4*H)`, and another with the value of 1 represented by the commitment `out_1 = (1*G) + (3*H)`. An observer to our transaction is not able to break our commitments into the additive terms, but they can perform the following comparison to show the inputs are equal to the outputs:

`in_0 - (out_0 + out_1) == 0`

Inserting our example values for this, we get:

`(3*G) + (7*H) - ((2*G) + (4*H)) - ((1*G) + (3*H)) == 0`

If we move terms so that all the `G`s and `H`s are grouped together (since addition is commutative on elliptic curves) we can reframe the expression as:

`(3*G) - (2*G) - (1*G) + (7*H) - (4*H) - (3*H)`

Both the amounts (3 = 2 + 1) and the blinding factors (7 = 4 + 3) cancel out independently, which is what makes this work.

Notice that the output blinding factor isn't completely random since both the input and output coefficients of H need to cancel out to 0. To minimize the disruption of this restriction, we usually pick all the output commitment blinding factors *except* the last one randomly, and pick whatever value cancels out the sum of the other blinding factors as the last one. So basically:

`last_output_blind = sum_of_input_blinds - sum_of_other_random_output_blinds`

Unfortunately we have a big problem with our scheme. Namely, the problem of 'negative' valued outputs. I say 'negative' in quotes because there are no negative values on elliptic curves in the traditional sense. Instead, we refer to a number `a` as negative with respect to another number `b` if it is large enough to make the sum `a + b` wrap around the modulus and produce a result less than `b`. See my last blog post for more information about "negative" numbers on elliptic curves. For now, let's keep conceptualizing these numbers as negative.

So here's the problem. Imagine I have 1 input value of 4 and 2 output values of 1 and 3 respectively. Let's also say I make commitments for them:

```
in_0  = 4*G + 2*H
out_0 = 1*G + 1*H
out_1 = 3*G + 1*H
```

Let's think like an attacker here. What's stopping me from adding an extra output back to myself for a value of 30? If I did, subtracting output commitments from input commitments would now equal -30. Well how about if I added *another* output with a value of -30 (where -30 is `modulus - 30`)? These 2 outputs would cancel out in the input/output equality check and no one would know because the values are hidden.

Addressing this problem necessitates the addition of a new tool in our confidential transaction toolbox which we'll refer to as **range proofs**. Range proofs are a way to prove that all outputs are within a range of values that won't cause an overflow, and will be the topic of my next blog post "Confidential Transactions for Dummies Part 2."