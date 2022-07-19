# DiemBFT Notes

Diem Byzantine Fault Tolerance (DiemBFT) is the 4th version of the Diem consensus. It is responsible for forming agreement on ordering and finalizing transactions among a configurable set of validators.

It incorporates a leader reputation mechanism that provides leader utilization under crash faults while making sure that not all committed blocks are proposed by Byzantine leaders. Leader utilization guarantees that crashed leaders are not elected, preventing unnecessary latency delays.

## Introduction

### Permissioned & Open Network
The security of DiemBFT depends on the quality of validator node operators. Validator Nodes in DiemBFT will be run by members of the Association third-party operators approved by Diem Networks US to operate the validator node in their behalf. This means that this needs a whitelisted node operators to start working.

### Classical BFT


The main guarantee provided in this approach is resilience against Byzantine failures  preventing individual faults from contaminating the entire system. A second important guarantee this approach provides for DiemBFT is a clearly described transaction finality (when a participant sees confirmation of a transaction from a quorum of validators, they can be sure that the transaction has completed).

### Enhancements

DiemBFT changes leaders in two separate cases. 
- First, in normal operation DiemBFT rotates the leader-role among validators in order to provide fairness and symmetry, this is done with a mechanism that achieves leader utilization (number of times a crashed leader is elected as leader is bounded) and keeping a reputation scheme avoiding round-robin (random) election.
- Second, DiemBFT changes leaders when they are considered faulty. 

Finally, validators in DiemBFT can verify safety in an isolated secure safety module by only storing locally a small (constant) amount of information. This enables the modeling of three types of validators: honest, compromised, and Byzantine, such that an adversary controls a compromised validator but cannot access its safety module. DiemBFT guarantees safety for any number of compromised validators as long as the faction of Byzantine ones is less than 1/3.

## Problem Definition

The goal of DiemBFT is to maintain a database of programmable resources with fault tolerance.
To model failures we count with one adversary, we assume a secure trusted host and define three types of validators:
- Byzantine validator - all code is controlled by the adversary.
- Compromised validator - all code outside the secure trusted host is controlled by the adversary.
- Honest validator - nothing is controlled by the adversary

The Agreement property requires that honest validators never commit contradicting chain prefixes. Compromised
validators might locally commit anything since the committed information is stored outside the trusted
hardware module.

### System model

**Network:** The model for DiemBFT is called partial synchrony. When the network is under attack change to asynchronous model and remains synchronous for the rest of the time.

### Technical background

Byzantine Fault Tolerance (BFT) is the ability of a distributed system to provide safety and liveness guarantees in the presence of faulty, or “Byzantine,” validators below a certain threshold. 

**Round by Round BFT Solutions:** Diem Follows the HotStuff approach for BFT, a round by round manner. In each round, there is a fixed mapping that designates a leader for the round (e.g., by taking the round modulo n, the number of participants). The leader role is to populate the network with a unique proposal for the round. The leader is successful if it populates the network with its proposal before honest validators give up on the round and time out.

In the classical approach to BFT there are two phases per round:
- The first phase a quorum of validators certifies a unique proposal, forming a quorum certificate, or QC.
- In the second phase, a quorum of votes on a certified proposal drives a commit decision. 

The leaders of future rounds always wait for a quorum of validators to report about the highest QC they voted for. If a quorum of validators report that  they did not vote for any QC in a round r, then this proves that no proposal was committed at round r.

In the HotStuff approach we have an extra phase. The first two phases are similar but the result of the second phase is a certified certificate, or a QC-of-QC rather than a commit decision. The commit decision is reached in the third phase with a quorum of votes of the previous result or a QC-of-QC-of-QC.

**Chaining:** In the chaining approach, the phases for commitment are spread across rounds. Every phase is carried in a round and contains a new proposal.
- The leader of round k drives only a single phase of certification of its proposal. 
- In the next round, k + 1, a leader again drives a single phase of certification. The k + 1 leader sends its own k + 1 proposal. However, it also piggybacks the QC for the k proposal. In this way, certifying at round k + 1 generates a QC for k + 1, and a QC-of-QC for k. 
- As a result, in a 2-phase protocol, the k proposal can become committed, when the (k + 1) proposal obtains a QC.
![](https://i.imgur.com/FGX1x0s.png)

**Round synchronization:** For round synchronization, PBFT, doubles the duration of rounds until progress is observed. 
In DiemBFT, if a validator gives up on a certain round it broadcasts a timeout message carrying a certificate for entering the round. 
This brings all honest validators to the round within the transmission of delay bound. When timeout messages are collected from a quorum, they form a timeout certificate (TC). The TC also includes the signatures of the 2f + 1 nodes on the highest round QC they are aware of. This later serves as proof for the leader to safely extend the chain even if some parties are locked on a higher round than the one the newly elect leader.

## DiemBFT Protocol

The goal of the DiemBFT protocol is to commit blocks in sequence.
The protocol operates in a pipeline of
rounds. The round contemplates all this actions:
- A leader proposes a new block. Validators send their votes to the leader of the next round. 
- When a quorum of votes is collected, the leader of the next round forms a quorum certificate (QC) and embeds it in the next proposal. Validators may also give up on a round by timeout.
- If this is the situation it may cause transition to the next round without obtaining a QC, in which case validators enter the next round through a view-change mechanism by forming or observing a timeout certificate (TC) of the current round. In fact, entering round r + 1 requires observing either a QC or TC of round r. 
- The leader of the next round faces a choice as to how to extend the current block-tree it knows. In DiemBFT the leader always extends the highest certified leaf with a direct child.

When two uninterrupted rounds complete, the head of the “2-chain”, consisting of the two consecutive rounds that have formed a QC, becomes committed. The entire branch ending with the newly committed block becomes committed. Forking can happen for various reasons such as a malicious leader, message losses, and others. The safety rules of Diem guarantees that committed branches can never be discarded.

DiemBFT guarantees that only one fork becomes committed through a simple voting rule that consists of two ingredients: 
- First, validators vote in strictly increasing rounds.
- Second, each block has to include a QC or a TC from the previous round.

If the previous round results in a TC then the validators check that the new leader’s proposal is safe to extend. This check consists of looking at the TC which bears the 2f + 1 highest qc round (the highest QC included in a block the validator voted for) from distinct nodes. If the new proposal extends the highest of these highest qc round, this serves as proof that nothing from a round higher can even be committed (Otherwise at least one would have reported a higher highest qc round).

The implementation of the DiemBFT protocol is broken into the following modules:
- Main module that is the glue, dispatching messages and timer event handlers.
- Ledger module that stores a local, forkable speculative ledger state. It provides the interface for SMR service and to be connected to higher level logic (the execution).
- A Block-tree module that generates proposal blocks. It keeps track of a tree of blocks pending commitment with votes and QC’s on them.
- A Safety module that implements the core consensus safety rules. The Safety : Private part controls the private key and handles the generation and verification of signatures. It maintains minimal state and can be protected by a secure hardware component.
- A Pacemaker module that maintains the liveness and advances rounds. It provides a “heartbeat” to Safety.
- A MemPool module that provides transactions to the leader when generating proposals.
- Finally, a LeaderElection module that maps rounds to leaders and achieves optimal leader utilization under static crash faults while maintaining chain quality under Byzantine faults.

---

- [DiemBFT official paper](https://developers.diem.com/papers/diem-consensus-state-machine-replication-in-the-diem-blockchain/2021-08-17.pdf)
- [Consensus overview in Diem from the official repo](https://github.com/diem/diem/blob/main/consensus/README.md)
- [Diem Glossary](https://developers.diem.com/docs/reference/glossary/)

