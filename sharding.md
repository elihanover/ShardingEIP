# Sharding 101 Notebook
---
## Ethereum's Scaling Problem
- Current implementation of Ethereum designed to scale at O(c), where c = resources of an average node

- More specifically, ***every validating node processes every transaction and stores state of the main chain***
  - Offers resistance from 51% attacks

- Thus, ***security of network scales with the network's resources, but not throughput***
  - Avoids centralization of block publishing to a few super-nodes at the cost of scalability
  - But leaves us currently at *7-15 tx/s* as opposed to Visa's *24,000 tx/s*

---
## Sharding's Approach to This Problem:
- Parallelize chain into s ***shards*** (100 planned at the moment)
- Each shard is its own chain of accounts and transactions
- Main just only used to store "block" headers*
- Parallelizing blocks allows an improvement to the factor of ~s
- *O(sc)*
- Often called *quadratic sharding* because:
  - O(c) resources per shard
  - s = O(c) shards
  - O(sc) ≤ O(c^2)

![Sharding Diagram](https://cdn-images-1.medium.com/max/1600/1*Utj_kmEEENsOEyHPVJ9Ndw.png)

---
## Sharding 101: Components

### *Collation*
```python
[
    header: [
        shard_id: int128,
        period: int128,
        chunk_root: bytes32, # pointer to collation body
        proposer_address: address
    ],
    body: CollationBody # blob (binary large object) of transactions
]
```

### *Proposer*
- Connects to running ethereum node and collects transactions into collations (again, just a shard block)
- Gathers ~1Mb of transactions
- Does NOT execute them, just collects
- Submits collation header to the main chain, specifically the SMC
- Gets paid in transaction fees (although payment isn't part of phase 1 and imaginably some of this could be used for the executors)

### *SMC (Sharding Manager Contract)*
- Smart contract (in first implementation) that manages shards
- Collects collation headers from the proposers so that notaries can vote on them

### *Notaries*
- Vote on the collation proposed collation headers in each shard
- Randomly sampled from the validator pool
    - Phase 1: stake ether into the contract to be selected
    - After PoS: use existing (main chain) validator pool

>#### Notary Sampling and Periods:
>  - Randomness is *essential* to avoid 1% attacks
>    - Notaries could collude to validate incorrect collations on a specified shard
>    - With random sampling, the likelihood of this happening even with 33% malicious validators to sample from is very, *very* low.
>  - Each ***period*** (every 100 blocks) sampling occurs
>  - 135 notaries assigned to each shard for **one** period
>  - Can only vote in that period
>  - 90 total votes needed, not weighted like in Casper (as of now)
>  - *At most one collation per shard per period*
![](https://cdn-images-1.medium.com/max/2000/1*Zv2eD1bzLcspLYpJ2k0L9g.png)

### *Overview*
<!--[Proposer{bg:wheat}]fetch txs-.->[TXPool], [TXPool]-.->[Proposer{bg:wheat}], [Proposer{bg:wheat}]-package txs>[Collation|header|body], [Collation|header|body]-submit header>[Sharding Manager Contract], [Notary{bg:wheat}]downloads collation availability and votes-.->[Sharding Manager Contract]-->
![](https://yuml.me/69cbd7da.png)
---


---
## Implementation
### Proposer Client
*Connects to ethereum node and collects transactions, packages them into collations, and submits them to the SMC so that notaries can download them and vote*

---
### Notary Client
*Connects to ethereum node in order to interface with the SMC, check whether they have been sampled to be a notary, download collations, and vote on them*

---
### Web of Clients
<!--[Transaction Generator]generate test txs->[Shard TXPool],[Geth Node]-deploys>[Sharding Manager Contract{bg:wheat}], [Shard TXPool]<fetch pending txs-.->[Proposer Client], [Proposer Client]-propose collation>[Sharding Manager Contract],[Notary Client]download availability and vote->[Sharding Manager Contract{bg:wheat}]-->
![system functioning](https://yuml.me/6c2f90a5.png)

---
### SMC Implementation (Phase 1)
###### Registering Functions (notary)
``` python
# User deposits ether to join notary pool
register_notary() returns bool
```
``` python
# Update notary pool to deregister msg.sender as notary
# Can't withdraw for a few periods
deregister_notary() returns bool
```
``` python
# Once deregistration period over, release deposit
release_notary() returns bool
```

###### Proposing Function (proposer)
``` python
# Called be proposer to add_collation header
# Ensures shard exists and in the right period
add_header(int128 shard_id, int128 period, bytes32 chunk_root) returns bool
```

###### Voting Functions (notary)
``` python
# Randomly sample notary for a given shard
get_member_of_committee(shard_id: int128, index: int128) -> address
```
``` python
# Sampled notaries call this to vote
submit_vote(int128 shard_id, int128 period, bytes32 chunk_root, int128 index) -> bool
```



---
## Beyond Phase 1
#### *Cross Shard Transactions*
- If each shard has its own account space and transaction history independent of others, then I can't transfer to you if your account is on another shard...
  - **In theory, without cross-shard transactions, you've created multiple versions of ether (eth0...eth99) that could be worth dramatically more or less**
- Need some way to transact across different shards
  - Doomed otherwise IMO
- In the works, idea is to have receipts stored on the main chain that record destruction of ether on one chain and would permit creation of ether in another chain
  - But still super early and there's plenty of attacks to consider  
  - Not to mention implementation itself

#### *Super-quadratic sharding*
- Why not create more shards to scale the system even more?
- The problem is that you want any node to be able to process the top-level blocks (containing just shard headers)
  - This would mean they can't run the main chain
- **Solution**: *shards within shards*

#### *Tight coupling*
- If blockchain contains invalid collation header in any shard, the whole blockchain is invalid
- i.e. sharding not just a smart contract, but a part of the main chain itself
  - In smart contract implementation, in theory you could have a committee that votes for an invalid collation, but that invalid shard will still have executed correctly as a smart contract
---
## Discussion
> #### Heterogenous Sharding
>*Why not allow some larger blocks?*  
>  
> Wouldn't affect frequency of blocks or periods, but could allow higher gas transactions


> #### Governance
> Who gets to decide sharding parameters?  
> (n_shards, homogeneous vs. heterogenous, collation size, etc.)  
>  
> In research phase these have been discussed and loosely agreed upon, but IMO it's a major discussion to be had
---
