# Private In-Round Bark Transfers

This post describes an algorithm to unlink a forfeited vtxo from its in-round output

The Bark SDK currently only supports in-round transfers between addresses of the same wallet, as part of vtxo refreshes. The process is modeled in the following diagram:

user -> server: Attests to input and declares desired destination pubkey
server -> user: Gives hash and proposed round
user -> server: Cosigns branches of vtxo tree for proposed round
server: broadcasts transaction
server -> user: Gives finalized round
user -> server: Gives forfeit signatures spending old vtxo exit to ASP hashlock
server -> user: Gives secret and new vtxo
user: Can now exit new vtxo with secret, or spend vtxo

# Overview of in-round privacy issues

When senders refresh their vtxos, they can't establish privacy across rounds -- either from on-chain observers or from the ASP.

The on-chain privacy leak is demonstrated in the case of a stubborn ASP that refuses to give the secret associated with a new round output. The participant, with no other option, will then need to exit their input vtxo so it doesn't get swept by the ASP.

If the ASP chooses to continue the swap, it will seize the exit and reveal the secret needed for the user to claim their round output. Doing so will also create an on-chan link between the 2 transactions through their shared hash.

Meanwhile the off-chain privacy leak stems from the ASP seeing the linked hashlocks through presigned transactions.

# Addressing the privacy issues

To formalize the problem, we would like to avoid the ASP learning the relationships between input vtxos and round outputs, and find a way to maintain that privacy even when an ASP claims forfeited funds.

## Attestation and output declaration

The first step I see that violates our 2 goals of on-chain privacy and ASP round linkability is the attestation and output declaration step. It violates the goal of privacy against an ASP linking old and new vtxos within a round.

The step involves a user initiating an in-round transfer by letting the ASP know they have a valid input, and declaring their output public key.

The functional requirements fulfilled by this step are vtxo-holder authentication and output binding.

Since both pieces of information are given in one message, it trivially leaks the input/output relationship. We separate these steps into 2 to circumvent this linkage.

The first step in the decomposition still requires vtxo attestation. Instead of also getting a proposed round, the user will now receive unlinkable authorization tokens as a response. Each authorization token represents an amount denomination, and the sum of all token amount denominations should equal the submitted vtxo's value.

The mechanism I'd like to use for this is a VOPRF, the steps for which are as follows:

asp: Picks secret key `asp_sk` and publishes public key `K` using generator `G`
user: choose 2 secrets, `x` and `user_sk`
user -> asp: Gives user public key `P1`, made from secret key `user_sk` and generator `X = hash_to_curve(hash(x))`
asp -> user: ASP performs Diffie-Hellman on user's `P1` with `asp_sk` and returns the result `DF_X`
asp -> user: ASP also returns 2 signatures, `(s, R1)` made with `asp_sk` and `G`, and `(s, R2)` made with `asp_sk` and `P1 = user_sk*X` (note: the same `s` is used in both signatures)
user: Multiplies Diffie-Hellman result by the inverse of `user_sk` and gets `A2 = asp_sk * hash_to_curve(hash(x))`
user -> asp: Comes back anonymously and gives the seed `x` used to make the generator and the ASP's pubkey with the user-derived generator and output address

The logic behind the ASP trusting this is that the user would not be able to obtain the ASP's pubkey with a fresh, previously unused hash output as generator. The step where the ASP generates 2 signatures with the same `s` proves the same secret key was used to make `P1` and `DF_X` under the DLEQ assumption.

## De-linking the outputs

The fundamental problem is the ASP gives a secret commitment that locks the round output and expects the user to use the same commitment in their forfeit. Requiring the commitment to be revealed in the ASP's forfeit claim is solely for the user's benefit. If the forfeit only required a signature check for the ASP's public key, it would break the atomicity of the scheme by allowing the ASP to claim the forfeit without revealing the secret needed to spend the round output.

The question we have to ask ourselves now is, "Can a user construct their forfeit transaction such that an ASP claim reveals the information needed to spend the user's round output, without the ASP learning *which* output the information is for?"

My answer to this was to ditch the hashlock on the forfeit transaction altogether and instead make the secret the ASP’s partial signature on the forfeit. With this, we can bind the reveal of secret information to the action of the ASP’s forfeiture claim. The invariant upheld is, when an ASP claims a forfeit, the participant learns a secret allowing them to spend the round output.

But how do we prove the partial signature can be used to claim the round output without revealing which round output it associates with?

We first require the ASP to commit their partial exit signature commitments and publish them.

We then ask each participant for a ring signature where each public key is the subtraction between a new static participant-provided public key `T[i]`, and the various partial signature commitments `Q[j]`, where  `j` is the slot for that ring used to index participant commitments.

The ASP doesn’t know which key is the one we know the private key for, but they know we know one of them. We also give them our token from the VOPRF to authenticate our requested denomination. Each VOPRF token is reserved a slot in each participant's ring signature and tokens originating from the same input will use the same `Q[i]`

One problem with this scheme is the reuse of the same slot.  Since each ring signature candidate key is a new difference value, since the first term `T` changes for every ring signature submission, we don't have a static value we can make a nullifier from so that the same slot for isn't used twice. What we do instead is make a connection for each denomination token to a static value `X` and use a transformation of a static `X` as a nullifier for a particular slot. The combined proof establishes knowledge of a slot’s registration secret and an offset relating T to that slot’s Q. When several slots share Q, the offset does not distinguish those slots.