
# Confidential transactions for dummies pt 2 - Borromean Range Proofs

- Why are range proofs important?
In the previous post, we discussed Pedersen commitments and how they hide transaction amounts while still allowing balances to add up correctly. We then touched on the topic of how one can fake 2 outputs that cancel out to spend more money than they have. Range proofs are the patch for this. They let you prove that committed values stay within sane bounds without revealing the values themselves. Without range proofs, confidential transactions don’t work.

- Simply explained, what's a high level technical summary of how this is done?
With the flavor of range proofs we're using, borromean range proofs, we'll need bring in a new cryptographic construct -- ring signatures. Ring signatures let a signer within a group of public keys sign on behalf of the group without exposing themself as the signer. If you like analogies, you can think of this as a police lineup: the verifier knows the real signer is in the lineup, but can’t tell which individual it was. To motivate its use within a range proof, let's take a look at a somewhat naive solution.

Imagine we have a list of pedersen commitments where each commitment is a curve point `C` generated like so:
`C = vG + bH`

What's cool about curve points is that any point can be treated as a public key in secp256k1, even `C`, but to sign for `C` I would need to know its discrete log. Unfortunately I don't have a scalar `c` in `C = c*G` I can use as my private key so I can't actually sign for `C`, but I *can* sign for the public key generated from vG or bH if I were to isolate the terms (since I know the private keys are 'v' and 'b' respectively). We'll use this to our advantage later on. Now let's use this concept to make a rudimentary range proof where we'll prove C is in the **range** of 1 to 2.

- How can I use this to my advantage to prove C is in the **range** of 1 to 2?
One way I can prove that my pedersen commitment is either '1' or '2' is to make a scheme with ring signatures using public keys derived from `C`. What I'll do is I'll make a separate public key for each possible value `pv` where the public key is `C - pv*G` for that value. For the case of proving my commitment contains a value `v` that's 1 or 2 as part of my commitment, I'll make 2 public keys -- `C - 1*G` and `C - 2*G`. 

What does this scheme do for us?

If I'm not lying about my value being 1 or 2, one of my *derived* public keys will subtract the correct value, allowing me to sign with the pure H multiple b*H. Since we're utilizing a ring signature scheme, I can sign for all the public keys as a group and not reveal to an outside observer which public key subtracted the correct value.

 For example, if my value `v` is 1, then the first term in the list of public keys would evaluate to `bH` (since `C - (1*G) = (1*G + bH) - 1*G = bH`). Meanwhile I could never sign for the public key `C - 2*G` because I don't know the discrete log for the resulting point `X`. `2*G` won't cancel out one of the terms I know, and I'd need to solve the elliptic curve discrete log problem to find `x` in `x*G = X`.

Thinking more broadly, we can use this strategy for an arbitrary number of values from `1` to `n`, instead of `1` to `2` like in our toy example. The downside to this scheme is that it requires additional data for every possible value added. The bigger the range, the bigger the signature size grows. This led researchers to the question of whether the size for this scheme could be shrunk down and, lucky for us, the answer was yes.

# Binary decomposition
The path to doing this is through arithmetic. Researchers figured the difficulty of mathematically identifying the signer in a ring signature doesn't get reduced if we reduce the number of partipants. So lets do that.

Imagine we have a ring signature for a naive range proof where we're trying to prove `C` contains a `v` with either `0` or `1`. The public keys for this would be `C - 0*G` or `C - 1*G`. Now imagine we made a **second** ring where the possible values are `0` or `2`. Is there a way to represent a commitment with an internal value `v=3` using **two** ring signatures?

Well, why not?! We can think of each ring signature as bits that can commit to a power of 2 being 'on' or 'off' and sign for the 'on' value for powers of 2 that would be in the **binary representation** of `3`. So instead of one ring proving "v is one of {0,1,2,3}", we're using 2 to prove "the 2^0 bit is 0 or 1" *and* "the 2^1 bit is 0 or 1.

We will also have to split b, but b doesn't have to get split into its binary representation. We just need to make sure we split `b` up into `n` parts, where `n` is the number of rings we would like to use. These `b`s should actually be chosen somewhat randomly the method I previously described when choosing blinding coefficients for commitments on a transaction's outputs. We can make all numbers random except the last one, and the last number can be chosen to make the sum equal `b`. If the `b`s were chosen to be factors of 2 as well, an attacker would notice the `2^n*H` public key with a motivated attack.

For intuition about this multi-ring approach, we're saving space because the number of values we're able to prove exponentially scales with the number of rings (just like adding a bit to series of bits lets us represent twice the amount of numbers). The key observation is that it's easier to represent bits than exhaustively represent every number.

With separate independent ring signatures, someone could mix-and-match valid signatures from different signing sessions. Borromean structure chains the rings together so they're all bound to a single signing event. With separate independent ring signatures, someone could mix-and-match valid signatures from different signing sessions (akin to vulnerabilities to CBC-style encryption).