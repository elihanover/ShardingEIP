# Sharding 101 Notebook
---
### Ethereum's Scaling Problem
---
- Current implementation of Ethereum designed to scale at O(c), where c = resources of an average node

- More specifically, ***every validating node processes every transaction and stores state of the main chain***
  - Offers resistance from 51% attacks

- Thus, ***security of network scales with the network's resources, but not throughput***
  - Avoids centralization of block publishing to a few super-nodes at the cost of scalability
  - But leaves us currently at *7-15 tx/s* as opposed to Visa's *24,000 tx/s*

---
### Sharding's Approach to This Problem:
---
- Parallelize chain into n ***shards*** (100 planned at the moment)
- Each shard is its own chain of accounts and transactions
- Main just only used to store "block" headers*
- Parallelizing blocks allows an improvement to the factor of ~n
- *O(nc)*
- Often called *quadratic sharding* because...¿?

---
### Sharding 101
___
![Sharding Diagram](flowchart.jpg)

#### *Collation*
![Collation]()

#### *Proposer*
- Connects to running ethereum node and collects transactions into collations (again, just a shard block)
- Gathers ~1Mb of transactions
- Does NOT execute them, just collects
- Submits collation header to the main chain, specifically the SMC

#### *SMC (Sharding Manager Contract)*
- Smart contract (in first implementation) that manages shards
- Collects collation headers from the proposers so that notaries can vote on them

#### *Notaries*
- Vote on the collation proposed collation headers in each shard
- Randomly sampled from the validator pool
    - Phase 1: stake ether into the contract to be selected
    - After PoS: use existing (main chain) validator pool
    ##### Notary Sampling and Periods:
    - Randomness is essential to avoid 1% attacks
      - Notaries could collude to validate incorrect collations on a specified shard
      - With random sampling, the likelihood of this happening even with 33% malicious validators to sample from is very, *very* low.
    - Each ***period*** (every 100 blocks) sampling occurs
    - 135 notaries assigned to each shard for **one** period
    - Can only vote in that period
    - 90 total votes needed, not weighted like in Casper (as of now)
    - *At most one collation per shard per period*

---
### SMC Implementation
---

---
### Proposer Client
---

---
### Notary Client
---

---
### Beyond Phase 1
---

---
### Discussion
###### Heterogenous Sharding
###### Governance
---
