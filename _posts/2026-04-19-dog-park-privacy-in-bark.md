# Dog Park - Privacy in Bark

Coming off the heels of '[Park - Privacy in Ark](https://uncensoredtech.substack.com/p/park),' I wanted to introduce a more comprehensive review of the privacy in Ark. For those wondering what 'more comprehensive' means, I'll be expanding the privacy analysis to protocol primitives and exploring what a privacy-conscious ASP (Ark Service Provider) implementation _has_ to reveal vs. what they choose to reveal. To spice things up, I decided to use the Second implementation, Bark, as a reference for this analysis, but will include occasional notes on what generalizes to other Ark implementations.

## Ark

Before I get into the nitty gritty of the privacy properties of each flow, I'd like to give a basic overview of what the flows are. The 5 flows I'll be covering are:

- onboards
- transfers
- balance checking
- offboards
- unilateral exits

As we go through the flows, an easy way to build intuition about what's public or not is to note the distinction of on-chain vs. off-chain activity. When something is broadcast on-chain, everyone can see it. When something is computed off-chain, that information is known only to the involved counterparties until explicitly revealed to others. Now, let's get started.

## Onboards

### Overview

Like all good layer 2 protocols, onboarding in Ark starts with a multisig transaction between the user and some counterparty. This multisig transaction acts like an escrow for which the ASP can claim a user's funds once they've made a payment on the user's behalf. In the general case, a user will send to their own Ark address when onboarding so they have layer 2 funds available for whenever they need it.

Looking deeper into this escrow transaction, note that the multisig is accomplished using a taproot output with a musig aggregated internal key gating the key path spend. In addition to this musig path, a script path is also included to allow one of the parties to claim their funds after a timeout. The party that gets to claim the timeout can differ between Ark implementations, but since I'm using Bark as my reference, it'll be the **server** claiming the funds from this transaction if the time runs out.

To avoid having a bad ASP stealing our money by refusing to honor refunds from already-broadcast onboard transactions, we request a refund transaction from the ASP **before** the onboard transaction goes on-chain. We do this by asking the ASP to cosign a new refund transaction, using the unsigned board transaction's output as input, that spends to a new script containing the refund path. I say 'containing' the refund path, because it also contains a musig and timeout path as well. This signed off-chain transaction will allow us to claw back our funds and is an example of a **vtxo** (virtual unspent transaction output), which is a utxo held off-chain that can be broadcast at any time.

### Privacy

To start thinking about the privacy of onboards, let's work through what an observer can learn from the on-chain transaction itself.

Assuming the board transaction is made with the Bark sdk, it'll be a multi-input transaction with 1-2 outputs depending on whether we have change or not. The first output will always contain the amount we would like to board and is locked using a musig internal key along with a script that allows the ASP to take the funds after a timeout.

Since taproot hides both the internal key and the script attached to the output, an outside observer is none the wiser on whether the transaction is even an Ark transaction or even whether it has a script attached to it. Meanwhile, the ASP knows your inputs to the boarding transaction and the vtxo public key the refund spends to.

Now, let's look at what a transfer looks like once we've onboarded a vtxo.

## Transfers

### Overview

Transfers can get tricky because there are 2 different types of transfers -- in-round and out-of-round. When either kind is made, the sending user has notify the recipient via the ASP so the recipient can verify their new balance. This will be covered more extensively in the balance checking section. In this section, I'll be covering more of the mechanics of the transfer itself.

#### In-round transfers

After onboarding and getting an initial vtxo, that vtxo is eligible for a transfer of either kind. When you make an **in-round** transaction, you are performing a swap between your vtxo and a new one that's anchored to a different on-chain transaction. The way it works is you tell the ASP you want to participate in a round, they create the round transaction alongside another series of transactions that allows you to spend from that round, and assign you an output that's locked by a hash.

You are then required to forfeit your existing vtxo in an offchain transaction, aptly called the 'forfeit transaction,' that's different from the round you're registering for. The output on this forfeit transaction is locked by a 'sweep' path requiring the ASP's public key and that hash's secret, along with a 'refund' path a user can use to reclaim the vtxo after a timeout. The idea is that the ASP would give you the preimage to spend your new vtxo and use that same preimage to sweep your **old** vtxo.

This hArk, or hashlock-based, approach is what superseded the connector-based approach featured in the original Ark protocol. It's simpler to implement than connector chains but it does come with some privacy tradeoffs as we'll see a little later when I touch on blinded output registrations.

Each round can have multiple senders and recipients, and is funded with the ASP's own money. The series of transactions mentioned before that allow a recipient to put their vtxos on-chain use this round transaction as input to the first presigned transaction in the chain. Each transaction spends from one another and represent smaller round transactions with subsets of the original recipient group in their outputs. Each group can be broken down until there's only 1 recipient, and the last transaction in this chain is what's referred to as an **exit** transaction. Each recipient has access to the series of transactions that lead to their exit, and they can broadcast these at any time. It's this exit transaction, which is initially held off chain like the rest of the transactions, that is signed over to the aforementioned hash in a forfeit transaction so it can be swapped for a vtxo in a new round.

#### Out-of-round transfers

Out of round transfers are those that don't need an on-chain transaction to be made. Instead of new vtxos being anchored to on-chain transactions, they become anchored to the sender's **off-chain** exit transaction. When a sender spends their exit to forfeit their vtxo, a new **checkpoint** transaction is made using the old transaction's exit as its input. This can be conceptualized as an off-chain 'round' of sorts, except with one input and potentially multiple outputs that can be spent to an exit transaction for the associated recipient.

### Privacy

For in-round transfers, the only on-chain/public action that's produced is the round transaction. This round transaction, like other taproot transactions, has its internal key and locking script obfuscated from the world until spend time. Therefore, outside the participating parties, only the ASP knows about vtxo transfers. Unilateral exits can break that shared output with transactions that progressively dwindle the number of recipient groups, and reveal scripts in the process, but I'll cover more of that in the unilateral exit section.

## Balance checking

### Overview

Every holder of Bitcoin on Bark needs to have a way to query the ASP for any vtxos sent to their Ark address. To prevent random observers from requesting the vtxos of an arbitrary address, an authorization system was built in the form of the Bark mailbox.

The Bark mailbox is a notification system where the ASP stores all notifications for a Bark wallet under a 'mailbox identifier' for that wallet. This mailbox identifier needs to be a public key for which the sdk makes a dedicated keypair for.

As mentioned before, a sender needs to give additional information to the ASP to push the notification to the receiver's mailbox. This additional information can be found in the Ark address generated by the receiver's wallet. At first, one might think this could be a huge loss for privacy if different addresses could be linked to the same mailbox public key, but Bark has preventative measures against this.

Instead of the raw mailbox public key being included in the address, a blinded/tweaked public key is provided instead. This blinded public key only needs to be decoded by 2 parties, the wallet and the server. So, the most straightforward way to do this is by tweaking the mailbox public key with a shared secret that can be computed using a Diffie-Hellman exchange. The input to this process for the receiving user is the private key belonging to the Ark address we're generating the tweak for, and the server's public key. Then when the sender tells the ASP they sent a vtxo to that address, the server can recompute the secret using its private key and the address the vtxo is assigned to.

### Privacy

Because of this mailbox mechanism, an outside observer cannot learn about a Bark wallet's balance unless given the proper authorization. There's also no on-chain transaction that would allow an observer to learn much about the wallet's mailbox public key (much like an on-chain transaction doesn't reveal anything about the xpub of an on-chain wallet). The ASP of course knows the vtxos that belong to your wallet and the IP addresses you're making requests from, similar to other consumer wallets.

## Offboards

### Overview

Offboards represent a collaborative exit path for vtxos in Bark. It involves telling the server which vtxos you want to offboard, getting an unsigned offboard transaction in response, signing it along with providing vtxos forfeit signatures, and receiving and broadcasting the final offboard transaction signed by both parties.

Unlike transfers, the forfeit transactions in offboards don't use hashlocks to prove a transaction was made on their behalf, they use connectors. **Connectors** are transaction outputs first constructed from unsigned unbroadcast transactions, that are used as inputs to transactions that should only be valid if that unsigned transaction is successfully signed and broadcast. The Ark protocol was designed to use this mechanism for both transfers and offboards, but Bark switched to a hashlock exchange to minimize the on-chain footprint.

### Privacy

Given that an offboard transaction can be funded by any of the ASP's utxos, there is no direct link between the on-chain offboard transaction and the offboarding user's vtxos. That means an outside observer does not learn which wallet requested the offboard.

## Unilateral Exits

### Overview

I already touched upon unilateral exits in the transfer section, but it refers to a way that a user can turn their vtxo into an on-chain utxo they can spend **without** any interaction from the server. When you perform a unilateral exit, a series of transactions are published on chain with the last one being freely spendable to an address of the user's choosing. There are timelocks to allow an ASP to stop the exit of a vtxo that was forfeited to them, AND there's a separate timelock that will let an ASP claim _any_ exited vtxo after a period in a sweep, but once the user's timelock ends (which is guaranteed to be before the sweep timelock ends) the user can spend it to wherever they want.

### Privacy

From an outside perspective, still nothing much can be learned from this process. Ignoring generic transaction-graph leakage, the _important_ privacy-relevant information is the revealed locking script and public key of the user who is exiting. If the unilateral exit was done on a sender's already-spent vtxo, no information can be learned about the recipient that the vtxo was spent to. If the recipient were to perform a unilateral exit _after_ the sender tried to exit their already-spent vtxo, an observer would then be able to link the sender and recipient by matching the hash revealed on both sides.

In summary, exiting a vtxo will create a new utxo that has an amount equaling the vtxo, but it still wouldn't leak information on the wallet that created it. The sender of that vtxo can also identify the exit as having belonged to that recipient since it was linked to the hash they needed for their forfeit.

## Blinded output registration

Another idea that has been floating around since Ark was conceptualized is this idea for it to operate like a coinjoin in making the ASP oblivious to which outputs a particular vtxo is spent to. In the connector-based design for Ark transfers, this worked, but in the hArk approach, things aren't as simple.

When using connectors, a forfeit transaction's spending condition is dependent on whether a **transaction** with a particular output set is broadcast on-chain. With the hArk approach, the forfeit transaction's spending condition is dependent on an **off-chain** **output** being included in an **already made** transaction. A hArk forfeit itself is one side of an atomic swap with another vtxo.

This means there's no ambiguity with respect to the output a forfeit transaction is attached to since both are linked by the hash in the new vtxo's spending condition.

## Final thoughts

I think the privacy afforded by Bark maintains a nice balance between privacy and usability. When used within the normal operating model, the ASP has no more insight into your transactions than the average wallet enabling on-chain transactions (so excluding wallets connected to one's node). It's also not too different from most consumer lightning wallets that use an LSP as the network gateway, although receivers can still have better privacy in lightning. All in all I think Bark seems to pay attention to privacy while not letting it bog down the user experience and I'm hopeful they'll continue to uphold this standard.
