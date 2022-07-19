# Algorand Notes

## Introduction

Algorand is a new cryptocurrency designed to confirm transactions on the order of one minute. The core of Algorand uses a Byzantine agreement protocol
called BA⋆ that scales to many users, which allows Algorand to reach consensus on a new block with low latency and without the possibility of forks. A key technique that makes BA⋆ suitable for Algorand is the use of verifiable random functions (VRFs) to randomly select users in a private and non-interactive way.

Algorand faces three challenges. 
- Algorand must avoid Sybil attacks, where an adversary creates many pseudonyms to influence the Byzantine agreement protocol.
- BA⋆ must scale to millions of users, which is far higher than the scale at which state-of-the-art Byzantine agreement protocols operate.
- Algorand must be resilient to denial-of-service attacks, and continue to operate even if an adversary disconnects some of the users.

To face this, Algorand uses the following techniques:

**Weighted users:** To prevent Sybil attacks, Algorand assigns a weight to each user. In Algorand, the users have a weigh based on the money in their account. Thus, as long as more than some fraction (over 2/3) of the money is owned by honest users, Algorand can avoid forks and double-spending.

**Consensus by committee:** BA⋆ achieves scalability by
choosing a committee (a small set of representatives randomly selected from the total set of users) to run each step of its protocol. BA⋆ chooses committee members randomly among all users based on the users’ weights. This allows Algorand to ensure that a sufficient fraction of committee members are honest.

**Cryptographic sortition:** To prevent an adversary from targeting committee members, BA⋆ selects committee members in a private and non-interactive way. This means that every user in the system can independently determine if they are chosen to be on the committee, by computing a function of their private key and public information from the blockchain. If the function indicates that the user is chosen, it proves to other users that is a committee member. Since membership selection is non-interactive, an adversary does not know which user to target until that user starts participating in BA⋆.

**Participant replacement:** Finally, an adversary may target a committee member once that member sends a message in BA⋆. BA⋆ mitigates this attack by requiring committee members to speak just once. Thus, once a committee member sends his message (exposing his identity to an adversary), the committee member becomes irrelevant to BA⋆. BA⋆ achieves this property by avoiding any private state (except for the user’s private key), which makes all users equally capable of participating, and by electing new committee members for each step of the Byzantine agreement protocol.

## Related Work

**Byzantine Consensus**

Most Byzantine consensus protocols require more than
2/3 of servers to be honest, and Algorand’s BA⋆ inherits
this limitation (in the form of 2/3 of the money being held by honest users). Fixed servers are also problematic in terms of targeted attacks that either compromise the servers or disconnect them from the network. Algorand achieves better performance (confirming transactions in about a minute, reaching similar throughput) without having to choose a fixed set of servers ahead of time.

Bitcoin-NG suggests using the Nakamoto consensus to elect a leader, and then have that leader publish blocks of transactions, resulting in an order of magnitude of improvement in latency of confirming transactions over Bitcoin. Hybrid consensus refines the approach of using
the Nakamoto consensus to periodically select a group of
participants and runs a Byzantine agreement between selected participants to confirm transactions until new servers are selected. Although Hybrid consensus makes the set of Byzantine servers dynamic, it opens up the possibility of forks, due to the use of proof-of-work consensus to agree on the set of servers; this problem cannot arise in Algorand.

Pass and Shi’s paper acknowledges that the Hybrid
consensus design is secure only with respect to a “mildly
adaptive” adversary that cannot compromise the selected
servers within a day. Algorand’s BA⋆ explicitly addresses this open problem by immediately replacing any chosen committee members. As a result, Algorand is not susceptible to either targeted compromises or targeted DoS attacks.

Stellar takes an alternative approach to using Byzantine consensus in a cryptocurrency, where each user can trust
quorums of other users, forming a trust hierarchy. Consistency is ensured as long as all transactions share at least one transitively trusted quorum of users, and sufficiently many of these users are honest. Algorand avoids this assumption, which means that users do not have to make complex trust decisions when configuring their client software.

**Proof of Stake**

There is a key difference, between Algorand using monetary value as weights and many proof-of-stake cryptocurrencies. In many proof-of-stake cryptocurrencies, a malicious leader (who assembles a new block) can create a fork in the network, but if caught (e.g., since two versions of the new block are signed with his key), the leader loses his money. The weights in Algorand, however, are only to ensure that the attacker cannot amplify his power by using pseudonyms; as long as the attacker controls less than 1/3 of the monetary value, Algorand can guarantee that the probability for forks is negligible.

Ouroboros is a recent proposal for realizing proof-of-stake. For security, Ouroboros assumes that honest users can communicate within some bounded delay. Furthermore, it selects some users to participate in a joint-coin-flipping protocol and assumes that most of them are incorruptible by the adversary for a significant epoch (such as a day). In contrast Algorand assumes that the adversary may temporarily fully control the network and immediately corrupt users in targeted attacks.

**Trees and DAGs instead of chains**

GHOST, SPECTRE, and Meshcash are recent proposals for increasing Bitcoin’s throughput by replacing the underlying chainstructured ledger with a tree or directed acyclic graph (DAG) structures, and resolving conflicts in the forks of these data structures. These protocols rely on the Nakamoto consensus using proof-of-work. In contrast, Algorand is focused on eliminating forks but it would be interesting in the future.

## Goals and Assumptions

Algorand achieve two main goals allowing users to agree on an ordered log of transacions:
- Safety goal: With overwhelming probability, all users agree on the same transactions. More precisely, if one honest user accepts transaction A, then any future transactions accepted by other honest users will appear in a log that already contains A. This holds even for isolated users that are disconnected from the network.
- Liveness goal: In addition to safety, Algorand also makes progress (i.e., allows new transactions to be added to the log) under additional assumptions about network reachability. Algorand aims to reach consensus on a new set of transactions within roughly one minute.

**Assumptions**

- Honest users run bug-free software.
- The fraction of money held by honest users is above some threshold h (a constant greater than 2/3), but that an adversary can participate in Algorand and own some money. 
- An adversary can corrupt targeted users, but that an adversary cannot corrupt a large number of users that hold a significant fraction of the money.
- Algorand makes a “strong synchrony” assumption that most honest users (e.g., 95%) can send messages that will be received by most other honest users (e.g., 95%) within a known time bound.
- Algorand makes a “weak synchrony” assumption: in every period of length b (think of b as a day or a week), there must be a strongly synchronous period of length s < b (an s of a few hours suffices).
- Loosely synchronized clocks across all users in order to recover liveness after weak synchrony. Specifically, the clocks must be close enough in order for most honest users to kick off the recovery protocol at approximately the same time. If the clocks are out of sync, the recovery protocol does not succeed.

# Overview

Algorand requires each user to have a public key. Algorand maintains a log of transactions, called a blockchain. Each transaction is a payment signed by one user’s public key transferring money to another user’s public key. Algorand grows the blockchain in asynchronous rounds, similar to Bitcoin. In every round, a new block, containing a set of transactions and a pointer to the previous block, is appended to the blockchain.

Algorand users communicate through a gossip protocol.
The gossip protocol is used by users to submit new transactions. Each user collects a block of pending transactions that they hear about, in case they are chosen to propose the next block. Algorand uses BA⋆ to reach consensus on one of these pending blocks-

BA⋆ executes in steps, communicates over the same gossip protocol, and produces a new agreed-upon block. BA⋆
can produce two kinds of consensus: final consensus and
tentative consensus. If one user reaches final consensus,
this means that any other user that reaches final or tentative consensus in the same round must agree on the same block value. Thus, Algorand confirms a transaction when the transaction’s block (or any successor block) reaches final consensus. On the other hand, tentative consensus means that other users may have reached tentative consensus on a different block (as long as no user reached final consensus). A user will confirm a transaction from a tentative block only if and when a successor block reaches final consensus. The tentative consensus can be produced by an adversary if the network is strongly synchronous, in this case Algorand will reach final consensus on a successor block a few rounds later. The second case is that network was only weakly synchronous. In this case, BA⋆ can reach tentative consensus on two different blocks, forming multiple forks.  To recover liveness, Algorand periodically
invokes BA⋆ to reach consensus on which fork should be used going forward.

**How Algorands component fit together**

**Gossip protocol** Algorand implements a gossip network
(similar to Bitcoin) where each user selects a small random set of peers to gossip messages to. Every message is signed by the private key of its original sender. To avoid forwarding loops, users do not relay the same message twice. Algorand implements gossip over TCP and weighs peer selection based on how much money they have, so as to mitigate pollution attacks.

**Block proposal** All Algorand users execute cryptographic sortition to determine if they are selected to propose a block in a given round. Sortition ensures that a small fraction of users are selected at random, weighed by their account balance, and provides each selected user with a priority, which can be compared between users, and a proof of the chosen user’s priority. Since sortition is random, there may be multiple users selected to propose a block, and the priority determines which block everyone should adopt. Selected users distribute their block of pending transactions through the gossip protocol, together with their priority and proof. To ensure that users converge on one block with high probability, block proposals are prioritized based on the proposing user’s priority, and users wait for a certain amount of time to receive the block.

**Agreement using BA⋆**. To reach consensus on a single block, Algorand uses BA⋆. Each user initializes BA⋆ with the highest-priority block that they received. BA⋆ executes in repeated steps. Each step begins with sortition, where all users check whether they have been selected as committee members in that step. Committee members then broadcast a message which includes their proof of selection. These steps repeat until, in some step of BA⋆, enough users in the committee reach consensus. (Steps are not synchronized across
users; each user checks for selection as soon as he observes the previous step had ended)

**Efficiency** When the network is strongly synchronous,
BA⋆ guarantees that if all honest users start with the same initial block, BA⋆ establishes final consensus over that block and terminates precisely in 4 steps. Under the same network conditions, and in the worst case of a particularly lucky adversary, all honest users reach consensus on the next block within expected 13 steps.

## Cryptographic Sortition

Cryptographic sortition is an algorithm for choosing a random subset of users according to per-user weights. The randomness in the sortition algorithm comes from a publicly known random seed. To allow a user to prove that they were chosen, sortition requires each user to have a public/private key pair (pki ,ski).
Sortition is implemented using verifiable random functions (VRFs). Informally, on any input string x, _VRFsk(x)_ returns two values: a hash and a proof. The proof π enables anyone that knows pk to check that the hash indeed corresponds to x, without having to know sk. For security, we require that the VRF provides these properties even if pk and sk are chosen by an attacker.


Sortition provides two important properties.
- Given a random seed, the VRF outputs a pseudo-random hash value, which is essentially uniformly distributed between 0 and $2^{hashlen} −1$. As a result, users are selected at random based on their weights.
- An adversary that does not know $sk_i$ cannot guess how many times user i is chosen, or if i was chosen at all (more precisely, the adversary cannot guess any better than just by randomly guessing based on the weights).

In each round of Algorand a new seed is published. The
seed published at Algorand’s round r is determined using
VRFs with the seed of the previous round r −1. This seed (and the corresponding VRF proof π) is included in every proposed block, so that once Algorand reaches agreement on the block for round r −1, everyone knows seedr at the start of round r. If the block does not contain a valid seed (e.g., because the block was proposed by a malicious user and included invalid transactions), users treat the entire proposed block as if it were empty.
The selection seed is refreshed once every R rounds.

## Block Proposal

To ensure that some block is proposed in each round, Algorand sets more than one proposer. 

**Minimizing unnecessary block transmissions** One
risk of choosing several proposers is that each will gossip their own proposed block. For a large block (say, 1 MByte), this can incur a significant communication cost. To reduce this cost, the sortition hash is used to prioritize block proposals. Algorand users discard messages about blocks that do not have the highest priority seen by that user so far. Algorand
also gossips two kinds of messages: one contains just the priorities and proofs of the chosen block proposers (from sortition), and the other contains the entire block.

**Waiting for block proposals** Each user must wait a certain amount of time to receive block proposals via the gossip protocol. Choosing this time interval does not impact Algorand’s safety guarantees but is important for performance. To determine the appropriate amount of time to wait for block proposals, we consider the plausible scenarios that a user might find themselves in. If the user is one of the first users to reach consensus in round r −1, the user must somehow wait for others to finish the last step of BA⋆ from round r −1. At this point, some proposer in round r that happens to have the highest priority will gossip their priority and proof message, and the user must somehow wait to receive that message. Then, the user can simply wait until they receive the block corresponding to the highest priority proof (with a timeout $λ_{block}$, on the order of a minute, after which the user will fall back to the empty block).

It is impossible for a user to wait exactly the correct
amount for the first two steps of the above scenario. Thus, Algorand estimates these quantities and waits for the estimated time value to identify the highest priority. Experimentally shows that these parameters are, conservatively, 5 seconds each.

**Malicious proposers** Even if some block proposers are
malicious, the worst-case scenario is that they trick different Algorand users into initializing BA⋆ with different blocks. This could in turn cause Algorand to reach consensus on an empty block. However, it turns out that this scenario is relatively unlikely. In particular, if the adversary is not the highest priority proposer in a round, then the highest priority proposer
will gossip a consistent version of their block to all users. If the adversary is the highest priority proposer in a round, they can propose the empty block, and thus prevent any real transactions from being confirmed. However, this happens with probability of at most 1−h, by Algorand’s assumption that at least h > 2/3 of the weighted user are honest.

## BA⋆

The execution consists of two phases:
1. In the first phase, BA⋆ reduces the problem of agreeing on a block to agreement on one of two options. This phase always takes two steps.
2. In the second phase, BA⋆ reaches agreement on one of these options: 
    - either agreeing on a proposed block, or 
    - agreeing on an empty block.

    This phase takes between two an eleven steps depending on if a malicious highest-priority proposer colluding with a large fraction of committee participants at every step (second case).
    
In each step, every committee member casts a vote for some value, and all users count the votes. Users that receive more than a threshold of votes for some value will vote for that value in the next step (if selected as a committee member). If the users do not receive enough votes for any value, they time out, and their choice of vote for the next step is determined by the step number.

In the common case, when the network is strongly syn- chronous and the highest-priority block proposer was hon- est, BA⋆ reaches final consensus by using its final step to confirm that there cannot be any other agreed-upon block in the same round. Otherwise, BA⋆ may declare tentative consensus if it cannot confirm the absence of other blocks due to possible network asynchrony.

### Main procedure

![](https://i.imgur.com/5SsPZ4D.png)

### Voting

![](https://i.imgur.com/o6Pkt6V.png)

### Counting Votes

![](https://i.imgur.com/dtSJ4iw.png)
![](https://i.imgur.com/MuxtPEd.png)

### Reduction

![](https://i.imgur.com/epnVEbW.png)

### Binary Agreement

Reaches consensus on one of two values: either the hash passed to BinaryBA⋆() or the hash of the empty block. BinaryBA⋆() relies on Reduction() to ensure that at most one non-empty block hash is passed to BinaryBA⋆() by all honest users.

![](https://i.imgur.com/YfeJBYK.png)

## Algorand

**Block Format**

Once a user receives a block from the highest-priority proposer, the user validates the block contents before passing it on to BA⋆. In particular:
- All transactions are valid
- The seed is valid
- The previous block hash is correct 
- The block round number is correct
- The timestamp is greater than that of the previous block and also approximately current (say, within an hour). 

If any of them are incorrect, the user passes an empty block to BA⋆.

**Safety and liveness**

If the network is not strongly synchronous, BA⋆ may create forks. Forks do impact liveness: users on different forks will have different "last_block" values, which means they will not count each others’ votes. As a result, at least one of the forks (and possibly all of the forks) will not have enough participants to cross the vote threshold, and BA⋆ will not be able to reach consensus on any more blocks on that fork.

To resolve these forks, Algorand periodically proposes a fork that all users should agree on, and uses BA⋆ to reach consensus on whether all users should, indeed, switch to this fork. To determine the set of possible forks, Algorand users passively monitor all BA⋆ votes and keep track of all forks. Users then use loosely synchronized clocks to stop regular block processing and kick off the recovery protocol at every time interval (e.g., every hour), which will propose one of these forks as the fork that everyone should agree on.

**Bootstraping**

- Bootstrapping the system: 
To deploy Algorand, a common genesis block must be provided to all users, along with the initial cryptographic sortition seed.

- Bootstrapping new users. 
Users that join the system need to learn the current state of the system. To help users catch up, Algorand generates a certificate for every block that was agreed upon by BA⋆ (including empty blocks). The certificate is an aggregate of the votes from the last step of BinaryBA⋆() (not including the final step) that would be sufficient to allow any user to reach the same conclusion by processing these votes. 
One potential risk created by the use of certificates is that an adversary can provide a certificate that appears to show that BA⋆ completed after some large number of steps. This gives the adversary a chance to find a BA⋆ step number in which the adversary controls more than a threshold of the selected committee members. Algorand set the committee size to be sufficiently large to ensure the attacker has negligible probability of finding such a step number. 

**Communication**

Algorand’s blocks are larger than the maximum
packet size, so it is inevitable that some packets from a chosen block proposer will be sent before others. A particularly fast adversary could take advantage of this to immediately DoS any user that starts sending multiple packets, on the presumption that the user is a block proposer.

Formally, this means that Algorand’s liveness guarantees
are slightly different in practice: instead of providing liveness in the face of immediate targeted DoS attacks, Algorand ensures liveness as long as an adversary cannot mount a targeted DoS attack within the time it takes for the victim to send a block over a TCP connection (a few seconds).

---

- [Algorand short explanation](https://algorandcom.cdn.prismic.io/algorandcom%2Fa26acb80-b80c-46ff-a1ab-a8121f74f3a3_p51-gilad.pdf)
- [Algorand long paper](https://algorandcom.cdn.prismic.io/algorandcom%2Fece77f38-75b3-44de-bc7f-805f0e53a8d9_theoretical.pdf)
