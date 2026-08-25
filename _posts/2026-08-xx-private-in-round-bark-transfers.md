# Private In-Round Bark Transfers

This post describes an algorithm to unlink a forfeited vtxo from its in-round output

The Bark SDK currently supports in-round transfers between the same-owner as part of a vtxo refresh. The process is modeled in the following diagram:

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

The on-chain privacy leak is demonstrated in the case of a stubborn ASP that refuses to give the secret associated with a new round output. The participant, with no other option, then exits their input vtxo used for the round so it doesn't get swept by the ASP.

The ASP will see this and, if they choose to continue the swap, will seize the exit and reveal the secret needed to claim the user round output in the process. The privacy issue here arises when/if the user then exits their round output with the revealed secret. Doing so will create an on-chan link between the 2 transactions since both transactions will have the same hash on them.

The off-chain privacy leak refers to the information the ASP can glean from coordinating these swaps. This stems from the current mechanism of a forfeit transaction using a hashlock sharing a hash with the desired round outputs.

# Addressing the privacy issues

The risk of on-chain linkability can be tackled in a straightforward way. Maintaining privacy against the ASP's ability to link round inputs and outputs is a bit harder.

To formalize the problem, we identify the steps that may link the old vtxo and the new, and stay mindful of the invariants that need to be kept.

## Attestation and output declaration

The first step I see that violates our 2 goals of on-chain privacy and ASP round linkability is the attestation and output declaration step. Mainly, it violates the goal of privacy against an ASP linking old vtxo and new vtxos within a round.

The step involves a user initiating an in-round transfer by letting the ASP know they have a valid input, and declaring their destination public key.

The functional requirements for this step are therefore to authenticate the user as a valid vtxo-holder, and to ensure the associated output address was given by the authenticated participant.

Effectively, there is no way to give both pieces of information in one message without leaking the input/output relationship. To circumvent this challenge we separate these steps into 2. One where the participant is authenticated via their input attestation and given a token, and one where the user brings an unlinkable authorization token to the ASP to declare their round output.

The mechanism I'd like to use for this is a VOPRF, the steps for which are as follows:

asp: Picks secret key `asp_sk` and publishes public key `K` using generator `G`
user: choose 2 secrets, `x` and `user_sk`
user -> asp: Gives user public key `P1`, made from secret key `user_sk` and generator `X = hash_to_curve(hash(x))`
asp -> user: ASP performs Diffie-Hellman on user's `P1` with `asp_sk` and returns the result `DF_X`
asp -> user: ASP also returns 2 signatures, `(s, R1)` made with `asp_sk` and `G`, and `(s, R2)` made with `asp_sk` and `P1 = user_sk*X` (note: the same `s` is used in both signatures)
user: Multiplies Diffie-Hellman result by the inverse of `user_sk` and gets `A2 = asp_sk * hash_to_curve(hash(x))`
user -> asp: Comes back anonymously and gives the seed `x` used to make the generator and the ASP's pubkey with the user-derived generator and output address

The logic behind the ASP trusting this is that the user would not be able to obtain the ASP's pubkey with a fresh, previously unused hash output as generator. The step where the ASP generates 2 signatures with the same `s` proves the same secret key was used to make `P1` and `DF_X` under the DLEQ assumption.


##  Gives hash and proposed round output

The next step to tackle is the 'Gives hash and proposed round' step. Since the ASP creates the hash for the user to use as proof of atomic forfeiture, the user will always have to show the ASP an output-identifying value in their forfeit transaction. As mentioned before, an on-chain observer can link an exited forfeit to an exited round output. Thus the first goal for this step is to break the on-chain link.

### Addressing linking output values

Rather than our old vtxo's exit hash determining whether the ASP can claim the forfeit, we'll make the secret something that isn't published on-chain. We'll make an adaptor signature secret.

Adapator signatures are a simple drop-in replacement for hashlocks (whenever musig is used) and thankfully don't invalidate any protocol invariants. The secret can still only be learned by the ASP upon user exit, and the ASP's exit seizure still reveals the secret to the user's round output.

## De-linking the outputs

Now that we erased the potential on-chain link, we can tackle the issue of ASP round linkability.

The fundamental problem here is the ASP gives a secret commitment used to lock the round output while expecting the user to use this same commitment in their forfeit. This is solely for the user's benefit. If the input vtxo's hashlock was instead just a signature check for the ASP's public key, it would break the atomicity of the scheme and allow the ASP to spend without revealing the participant spend information.

The question we have to ask ourselves now is, "Can a user to lock their forfeit transaction such that an ASP reveals the information needed to spend the round output when claiming the forfeit, without learning about the user?"


-----

 This would at least alleviate the issue of the ASP knowing which input and forfeit is associated with a round output in the optimistic case.

The approach here is to design a system where the forfeit secret is only unlocked after the action of a user exit. My approach to this was to treat the user's partial exit signature as another secret that hides the adaptor secret used to spend the exit into the forfeit. We thus make the privkey value needed for the exit signature to be `exit_signature + adaptor_secret`.

Now during our forfeit, we prove to the ASP that the exit transaction output can be taken with knowledge of the preimage of `exit_signature_commitment + adaptor_secret_commitment`.

Unfortunately this still doesn't solve the problem of linkability if during the forfeit stage we have to use the unique `exit_signature_commitment` to prove the vtxo was forfeited to pubkey `exit_signature_commitment + adaptor_secret_commitment`. We simply moved the problem of being able to link a forfeit to a round output using the `adaptor_secret_commitment` to being able to link a forfeit to a round output using the `exit_signature_commitment + adaptor_secret_commitment`.



----
 *the user* to participate in the selection of the secret commitment the ASP will use to lock our output. We derive this commitment such that the ASP can verify it is a function of their own secret commitment, and therefore of their original secret.

We can generate this secret commitment by making our own secret and commitment, `a` and `A` respectively, and adding it onto `T` to make `U = T + A`. This still isn't secret because we have to give  `A`. The trick to not giving `A` lies within the construction of ring signatures.

If we received the `T[i]` used for every participant, we could make a ring signature with the pubkeys `U - T[i]` for all `i`. We then prove we know the discrete log for one of them by providing a real signature for `U - T[i]` where `U - T[i] = A`, and make forged signatures for the rest of the `U - T[i]`s that don't equal `A`.

The ASP will then have proof that `U` is the sum of its own `T[i]` and a value `A` chosen by us, where we know `a`. The ASP doesn't need to care about which value `U - T[i]` is the right one for that participant since all they care about is the `U` chosen still can't be unlocked without `t` (meaning ASP must claim forfeit).

## Giving `s2` and `d` privately

As part of output registration, we still have to prove we know the preimage `s` for signature commitment `S`, and that the *same* preimage `s` is used in `d = r + s`.