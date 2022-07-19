# A quick tour on some consensus mechanisms

## introduction

### On previous talks:
- scaling the bottlenecks:
    - compute power
    - memory
    - network
- instead of scaling vertically, scale horizontally
- obviously, but how do the pieces interact with each other?

### With scaling comes great ~~responsibility~~ coordination

It's not as easy as setting a network and everyone screams what to do. You need to either have a **coordinator** or a way for peers to be **coordinated**

### Consensus Protocols

- By consensus we mean that a general agreement has been reached.
- Consensus Protocols allow distributed systems to work together and stay secure.

![](https://i.imgur.com/6v2FT7E.png)

### Some Examples

Consensus algorithms:
- Paxos <img src="https://i.imgur.com/j7WpCT1.png" width=110px height=70px />
- Multi Paxos
- Raft <img src="https://i.imgur.com/ck2rs4N.png" width=100px height=85px />
(:arrow_double_down: Node permission mechanisms) 
- **PoW** <img src="https://i.imgur.com/KVwIxfK.png" width=110px height=70px />
- **PoS** <img src="https://i.imgur.com/Ieze7ok.png" width=110px height=70px />

Byzantine Fault Tolerance
![](https://i.imgur.com/yyktjs6.png)

### Proof of Work & Proof of Stake

- The consensus we need to reach is **what is the next block to add to a chain of Blocks**.
- Necesities
    - resist Sybil attacks as much as possible
    - scalability
- Both make use of economic incentives.
    - **direct** vs **indirect**(pay with electricity)
- In both cases, the idea is to make **potential attacks not economically worth it even if successful**.

### Today's Menu

1. Algorand's $BF\star$ 
2. DiemBFT (v4.0)

<img src="https://i.imgur.com/3fkcDcg.png" width=18%/>

## DiemBFT

### Introduction

- Strongly Based on the HotStuff protocol
- Clearly described transaction finality
- State Machine Replication (SMR) paradigm

### Protocol

- The goal of the DiemBFT protocol is to commit blocks in sequence.
- The protocol operates in a pipeline of rounds. 
- The commitment of a block is delegated to many leaders by splitting the commitment process into 3 rounds.
- In round $r$, the leader will make decisions regarding the blocks from round $r-2$, $r-1$ and $r$

### Round steps

- A leader proposes a new block. 
- Validators send their votes to the leader of the next round. 
- The round could continue with the following situations:
    - If quorum is reached, the leader forms a quorum certificate (QC) and embeds it in its proposal.
    - If Validators give up on a round by timeout, they enter the next round with a timeout certificate (TC) of the round.
- The leader of the next round faces a choice as to how to extend the current block-tree it knows.

### Finality 

- The head of the “2-chain”, consisting of the two consecutive rounds that have formed a QC, becomes committed. 
- The safety rules of Diem guarantees that committed branches can never be discarded.
- All created forks are discarded.

<img src="https://i.imgur.com/HM0OGr9.png" width=50% />

### Assumptions

- 2/3 of the total nodes must be honest.
- A period of synchronism happens after some unknown finite time where message time to deliver is bounded.

## Algorand

### Introduction

- Algorand is a cryptocurrency.
- It uses a Byzantine agreement protocol called BA⋆
    - Scales to many users.
    - Reach consensus on a new block with low latency and without the possibility of forks.
    - Uses verifiable random functions (VRFs) to randomly select users in a private and non-interactive way.

### Verifiable Random Function

- The VRF takes a secret key and a value and produces a pseudorandom output, with a proof that anyone can use to verify the result.
- Used to choose leaders to propose a block and committee members to vote on a block.
- The more Algos in an account, the better chance the account has of winning – it’s as if every Algo in an account gets its own lottery number.

### Desired Properties

- With overwhelming probability, **all users agree on the same transactions**. 
- This holds even for isolated users that are disconnected from the network.
- **Makes progress** under additional assumptions about network reachability. 
- Aims to reach **consensus** on a new set of transactions **within roughly one minute**.

### Protocol
<img src="https://i.imgur.com/LGlOriS.png" width=60%/>

- Block proposal
- Soft vote
- Certify votes

## Block proposal

- Accounts are selected to propose new blocks to the network. 
- Once an account is selected by the VRF, the node propagates the proposed block along with the VRF output, which proves that the account is a valid proposer. 
- We move from the propose step to the soft vote step.

## Soft vote

- Filter the number of proposals down to one, guaranteeing that only one Block gets certified. 
- The node will only propagate the block proposal with the lowest VRF hash.
- Each node run the VRF for every participating account to see if they participate in the soft vote committee.
- A quorum of votes is needed to move to the next step.

### Certified vote

- A new committee checks the block proposal that was voted in the Soft Vote stage.
- If valid, the new committee votes again to certify the block. 
- All votes are collected and validated by each node until a quorum is reached.
- The round ends and the node creates a certificate for the block and write it to the ledger.
- If a quorum is not reached in a certifying committee vote then the network will enter recovery mode.

### Assumptions

- Honest users run bug-free software
- 2/3 of the total money is held by honest users
- An adversary can corrupt targeted users but not too many.
- 95% of honest users can send messages and they will be received by most other honest users within a known time bound (strong synchrony).
- In every period of time there must be a strongly synchronous period of minor length (weak synchrony).

## Challenges

- Sybil attacks.

- Scalability.

- Resiliency over Denial of Service (DOS).

## How does Algorand face this challenges?

- Weighted users $\rightarrow$ Sybil Attacks.

- Consensus by committee $\rightarrow$ Scalability.

- Cryptographic sortition $\rightarrow$ DOS.

- Participant replacement $\rightarrow$ DOS.

### Facing Sybil Attacks
#### Weighted users

- Users have a weight based on the money in their account. 
- As long as more than some fraction (over $\frac{2}{3}$) of the money is owned by honest users, Algorand can avoid forks and double-spending.

### Facing Scalability
#### Consensus by committee

- BA⋆ achieves scalability by randomly choosing a committee among all users based on the users’ weights. 
- Allows Algorand to ensure that a sufficient fraction of committee members are honest.

### Facing DOS
#### Cryptographic Sortition

- The committee selection is non-interactive. 
- Every user can independently determine if they are chosen to be on the committee by computing a Verifiable Random Function (VRF). 
- An adversary does not know which user to target until that user starts participating in BA⋆ (and only sends one message as a committee member).

### How does DiemBFT face this challenges?

- Node Permission $\rightarrow$ Sybil Attack

- Three Phase Commit (PBFT + HotStuff) $\rightarrow$ Scalability

- Leader Reputation System $\rightarrow$ DOS 

### Facing Sybil Attacks
#### Node Permission

- Validator Nodes in DiemBFT will be run by members of the Association third-party operators approved by Diem Networks US.
$\implies$ It is a permissioned protocol

### Facing Scalability
#### Three Phase Commit

- DiemBFT is inspired by the linear three-phase HotStuff but gets rid of the three-step latency cost. 
- Preserves the communication linearity of HotStuff when the leader is alive.
- Allows for a quadratic cost during the view-change protocol to regain the ability to commit in two steps.

### Facing DOS
#### Leader Reputation System

- DiemBFT designs a novel leader election mechanism that achieves a better leader utilization. 
- The number of times a crashed leader is elected as leader is bounded. 
- The leader election mechanism exploits the last committed state to implement a reputation scheme that tracks active validators.

## Summary

| Property | DiemBFT  | Algorand | 
| -------- | -------- | -------- | 
| Permissioned?     | :heavy_check_mark: | :x: | 
| Node Selection | VRF| VRF | 
| Forks? | :heavy_check_mark:  | :heavy_check_mark:(unlikely)  |
| block responsibility | 3 leaders | 1 leader |

# Sources

- [Consensus Mechanisms](https://ethereum.org/en/developers/docs/consensus-mechanisms/).
- [Proof of Stake](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/).
- [Casper](https://arxiv.org/pdf/1710.09437.pdf).
- [HotStuff](https://arxiv.org/pdf/1803.05069.pdf).
- [PBFT](https://pmg.csail.mit.edu/papers/osdi99.pdf).
- [DiemBFT](https://developers.diem.com/papers/diem-consensus-state-machine-replication-in-the-diem-blockchain/2021-08-17.pdf).
- [Byzantine Agreement](https://lamport.azurewebsites.net/pubs/reaching.pdf)
- [Algorand (short)](https://algorandcom.cdn.prismic.io/algorandcom%2Fa26acb80-b80c-46ff-a1ab-a8121f74f3a3_p51-gilad.pdf).
- [Algorand (long)](https://arxiv.org/pdf/1607.01341.pdf).
- [Byzantine Generals Problem](https://dl.acm.org/doi/pdf/10.1145/357172.357176).
