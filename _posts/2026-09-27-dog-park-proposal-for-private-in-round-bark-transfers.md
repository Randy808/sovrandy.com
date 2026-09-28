# Dog Pack: A Proposal for Private In-Round Bark Transfers

<video controls preload="metadata" style="width: 100%; height: auto;">
  <source src="{{ '/video/Dog-Pack-Private-In-Round-Bark-Transfers-1080p.mp4' | relative_url }}" type="video/mp4">
  <a href="{{ '/video/Dog-Pack-Private-In-Round-Bark-Transfers-1080p.mp4' | relative_url }}">Watch the video</a>
</video>

This post describes a proposed algorithm to unlink a forfeited vtxo from its in-round output.

The Bark SDK currently only supports in-round transfers between addresses of the same wallet as part of vtxo refreshes. The process is modeled in the following diagram:

```text
user -> server: Attests to input and declares desired destination pubkey
server -> user: Gives hash and proposed round
user -> server: Cosigns branches of vtxo tree for proposed round
server: Broadcasts transaction
server -> user: Gives finalized round
user -> server: Gives forfeit signatures spending old vtxo exit to ASP hashlock
server -> user: Gives secret and new vtxo
user: Can now exit new vtxo with secret, or spend vtxo
```

# Overview of in-round privacy issues

When senders refresh their vtxos, they can't establish privacy across rounds from on-chain observers or the ASP.

On-chain, a stubborn ASP refusing to give the secret for a round output can force a user to exit their old vtxo, and if the ASP refuses to honor the spend of the new vtxo, they can cause that one to exit as well. This non-cooperation creates an on-chain link between the 2 transactions through their shared hash.

Off-chain, the ASP always sees the linkage from the hashlock binding the forfeit transaction to the new round outputs.

# Addressing the privacy issues

To formalize the problem, we would like to avoid the ASP learning the relationships between input vtxos and round outputs, and find a way to maintain that privacy even when an ASP claims forfeited funds.

The first step I see that violates our 2 goals of on-chain privacy and privacy against ASP round linking is the attestation and output declaration step.

This step involves a user initiating an in-round transfer/refresh by submitting an attestation for an input and declaring an output public key. Since both pieces of information are given in one message, it trivially leaks the input/output relationship. Thus, our first goal is to separate these steps into 2.

The first step will take care of vtxo attestation, and the second step will take care of output declaration.

## Vtxo attestation

In the vtxo attestation step, the user will no longer submit an output public key. A slot is an authorization to receive one output of a specific denomination in the round. The user chooses a fresh random `slot_secret` and gives the ASP `slot_pubkey = slot_secret * G` for a requested denomination of their input vtxo, where `G` is the curve's generator.

Multiple denominations can be requested by providing different `slot_pubkey[i]` values, with each public key acting as an authorization bound to its registered denomination. The vtxo owner must sign these slot public keys and their denominations as part of their attestation to create a stronger binding between the two that will be helpful later on.

In terms of validation for this step, the ASP checks that the input is valid, that it hasn't already been registered, and that the sum of all token denominations equals the input's value after the agreed fees.

At this step, the ASP also establishes a MuSig2 signing session for the forfeit claim transaction and commits to the partial signature it will use. This is the transaction that lets the ASP collect from the forfeit output, not the transaction that creates that output. We call the partial signature scalar `claim_secret` and its public commitment `claim_point = claim_secret * G`. This scalar is specific to the fixed claim signing session. It is not the ASP's private key. The claim transaction, keys, and both parties' nonces are fixed here, so the signing challenge is fixed too. The participant calculates `claim_point` from the ASP's public nonces and public key contribution using that challenge, then checks the point the ASP provides. We'll put this point to use in the output declaration step.

## Output declaration

For the output declaration step, we want to prioritize 2 things:

1. The ability for an ASP to verify that an output can only be spent with knowledge of a forfeit claim secret

2. A way for us to hide the forfeit secret used in our output construction

The existing implementation already meets the first requirement since the output is locked by the same hashlock needed to spend the forfeit output.

The second condition was a bit harder to meet. The challenge, under the existing implementation, is to create a construction such that the condition requiring the preimage of a hash could be obfuscated.

To simplify the task of obfuscation, we'll first drop the hashlock from the forfeit output and introduce a new secret to lock our round output. Instead of the hashlock, we can use the ASP's partial signature for the forfeit claim transaction. Going forward, we'll call this secret `claim_secret[i]`, where `i` represents the ith vtxo submitted for the round.

With this, we can bind the reveal of secret information to the action of the ASP's forfeit claim. The invariant upheld is that when an ASP claims a forfeit, the participant learns a secret allowing them to spend the round output.

Now how can we use this secret to lock our output?

### Locking our output

Similar to the original model, the participant creates a new keypair to use in their round output. Let's call the private key in this keypair `offset` and the public key `offset_pubkey = offset * G`.

Instead of making `offset_pubkey` the final output public key we attach to our round output, we define the output public key as `output_pubkey = claim_point[i] + offset * G`, where `claim_point[i]` is the partial signature commitment associated with the participant's input.

Once the participant learns `claim_secret[i]`, they can calculate the private key `output_secret = claim_secret[i] + offset` needed to spend that output.

The participant also chooses a fresh, separate keypair to sign their branches of the vtxo tree. They already know this private key, unlike `output_secret`, which they can't calculate until they learn `claim_secret[i]`. The tree-signing pubkey is submitted with the anonymous output request, and the participant checks that the proposed tree uses it for their output.

### Hiding the claim point used

Hiding which claim point is used is not a straightforward task, so I leveraged one of the few cryptographic primitives known to me: ring signatures.

To hide which `claim_point[i]` is used when using `slot_pubkey[i]`, the participant constructs a ring containing every branch pubkey registered for the requested denomination. For each candidate slot `j` in that denomination ring, we include 2 public keys `slot_pubkey[j]` and `output_pubkey[i] - claim_point[j]`. This turns the ring signature into a multi-key ring proof, reminiscent of Monero's MLSAG (Multi-layered Linkable Spontaneous Anonymous Group).

Suppose Alice, Bob, and Carol each register a slot for the same denomination. Alice's output proof includes all 3 slots, but only Alice knows the `slot_secret` and offset for the slot she uses. The ASP can verify the proof without learning which slot that was.

Implicit in the requirements for this mechanism is the ASP publishing each denomination pubkey collected to all participants. Each denomination pubkey will also have been signed by its vtxo owner in the vtxo attestation step, so it can't be replaced by an ASP-derived value in this step.

In the ring, for one hidden slot `i`, the combined proof establishes knowledge of `slot_secret` and `offset` (since `offset_pubkey = output_pubkey[i] - claim_point[i]`). Slots from the same input share a `claim_point`, so the same offset works for those slots, but we still need the `slot_secret` for whichever slot we use. This approach is similar to the one used in the confidential transaction scheme described in [Confidential Transactions For Dummies, Part 2 - Borromean Range Proofs]({% post_url 2025-12-21-confidential-transactions-for-dummies-pt-2 %}).

The message we'll sign in the ring signature is:

```
  "Bark private output declaration v1" ||
  round_id ||
  denomination
```

`"Bark private output declaration v1"` acts as a label for domain separation, the round ID marks it for that round and attempt, and the denomination makes it particular to that ring. Here `round_id` includes both Bark's `round_seq` and `attempt_seq`, since a round can have more than one attempt.

We also include a nullifier, akin to a 'key image', in this construction to ensure the same real branch isn't used twice. The ASP rejects an output registration if its nullifier has already been used for an accepted output in that round.

The combined proof thus contains:

```text
slot_pubkey[i] = slot_secret * G
nullifier = slot_secret * nullifier_base
output_pubkey - claim_point[i] = offset * G
```

`nullifier_base` is a public curve point calculated with hash-to-curve from a domain-specific label, the round identifier, and the denomination (`"Bark slot nullifier v1" || round_id || denomination`).

`nullifier_base` can be shared by every candidate in a denomination ring. The proof contains a single `nullifier = slot_secret * nullifier_base`, which is used in every branch of the ring signature. Only in the real branch does the signer know the `slot_secret` relating both the slot public key and the nullifier to their respective bases. The other branches are simulated.

The challenge hashes the message, the ring's full ordered list of public keys, the nullifier, and the branch's computed nonce points for the slot, nullifier, and offset checks. The public keys are separate inputs to the challenge hash, not part of the message. Since they include `output_pubkey - claim_point[j]`, this also binds the proof to the requested output public key.

The participant submits `output_pubkey[i]` (by itself), the tree-signing pubkey, the denomination, and the combined proof.

This request should also come back anonymously, so it must avoid reusing connections or identifiers that link it to input registration.

Once all participants have their exit information and the new round's funding transaction has the required confirmations, the participant forfeits their old vtxo to the ASP in exchange for the `claim_secret` that will allow them to calculate `output_secret`. These confirmations are for the anchor funding the new round outputs, not the already-confirmed anchor behind the old input vtxo.

The ASP knows which input was forfeited and which `claim_secret` it released. The earlier anonymous output declaration does not tell it which output used that secret.

If the ASP withholds `claim_secret[i]`, the participant will be forced to exit if they don't want to lose money. If the ASP still wants to move forward, it will claim the forfeited exit, allowing the participant to extract `claim_secret[i]` from the completed claim signature and calculate `output_secret`.

This works because the combined MuSig2 signature scalar is the sum of the participant's and ASP's respective partial signature scalars, along with the tweak-derived value made from the script tree. Since the participant knows their own partial signature, they can calculate:

```
claim_secret = completed_signature_scalar - user_partial_signature - challenge*derive_sign(tweaked_agg_pubkey_parity_bit)*tweak
```

The participant also checks that the extracted `claim_secret` gives the registered `claim_point`.

If the exited vtxo is never claimed with a forfeit exit, the user is then able to claim their exit as described in the original model.

## Privacy after a claim

The ASP already knows every `claim_secret`, but it doesn't know which offset the participant chose. Spending an output reveals a signature, not `output_secret`.

The participant never reveals their `slot_secret` or offset, including during an exit. The combined proof must hide which slot they used.

## Drawbacks

As with coinjoins, outputs can be linked to vtxo inputs when there is insufficient liquidity. If there's one large-value vtxo and many small ones, heuristics may be able to link outputs.

The ASP can also impact privacy by providing fake liquidity with its own funds, reducing the anonymity set and making it easier to link real participant inputs and outputs.

The privacy afforded by this scheme also depends on other participants requesting the same denominations. A denomination with only one remaining input provides no input/output privacy. The choice of denominations therefore matters, even if every proof works as intended.

On the topic of denominations, I'll also mention the problem of bundle fingerprinting. Although I ask that each participant submit their outputs over a fresh connection, timing analysis might bundle outputs submitted around the same time. This is bad because the sum of this subset may uniquely identify an input vtxo. Batching, randomized declaration timing, and avoiding uniquely identifying denomination decompositions can help with this, but the details of doing these things are out of scope for this post.

Another drawback is that the ring proof grows with the number of candidate slots, and splitting an input into smaller denominations creates more outputs and proofs. If those outputs later need to be exited, having more outputs can also mean more on-chain fees.

The ASP can also use its control over secret release to try to learn the relationship through timing analysis. For example, it could release one input's `claim_secret` and watch for an output that immediately becomes active or is spent.
