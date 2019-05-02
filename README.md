---
eip: TBD
title: Varying Collation Body Size for Shards
author: Eli Hanover <eli.hanover@emory.edu>
type: Standards Track
category: Core
status: Draft
created: 2018-05-03
---


# Simple Summary
Heterogeneous Sharding allows for shards to have different properties from each other.  More specifically, this discusses the potential for differing collation body sizes between shards to allow for higher gas transactions without sacrificing block latency.


# Abstract
This draft EIP proposes and discusses the details of implementing varying collation body sizes into the Validator Manager Contract (VMC) in Phase 1 of Ethereum's sharding implementation.  By allowing collators time to submit collation headers proportional to the size of their shard's collation body, this would raise the gas_limit cap on transactions while maintaining the same period length and without sacrificing decentralization.


# Motivation
By allowing for variable collation body sizes between shards, we can permit higher gas transactions without altering the period length.  Doing so allows for more expensive state changes on some shards that are independent of smaller collation body shards.



# Specification and Rationale
#### Initial Concerns and How to Ameliorate Them
If we were to simply specify higher collation body sizes for a select few shards in the VMC, this would require proportionally higher computational resources of the executors of these higher gas limit shards.  In addition, notaries would be unable to download collation bodies within the notary burst overhead.  There are multiple issues here.  First, if we do not change the executors' reward proportional to the increase in resources needed to validate this collation, there is less incentive to execute the appropriate state transition function, potentially resulting in "abandoned" collations in specific shards.  Additionally, the added computational resources needed could cause executors who did want to solve the collation header to nonetheless fail because of the time restraint imposed by periods.  In essence,
the requirements and rewards of shards should be proportional to the size of the collation body of that shard.

These concerns suggest a few necessary changes to other parts of the VMC, including:
1. Set the notary burst overhead proportional to the collation body size of the specified shard to allow notaries sufficient time to download the collation body and vote.
2. Set the notaries' reward proportional to the collation body size of the respective shard.
3. Reward the executors proportional to the collation body size. (?)
4. Grant executors time proportional to the collation body size. (?)

##### In essence, set the proposal, notary, execution time line proportional the collation size of its shard.  Note that this means that notaries must be given access to submit collation headers in a range of periods.
Note that granting validators the status of notary for more than one period interferes with proposed sampling schemes, as we cannot sample from the full set of validators in the validator pool.  This raises the question of whether this makes coordinated attacks on the sampling mechanism possible. (NEXT: look into this)

#### Required Changes to the VMC
1. VMC must store the collation size of each shard.
2. Change notaries' and executors' reward to a function of the collation body size of the respective shard.
3. Change condition that notaries must publish collation headers within their one allocated period to grant them publishing ability in the next N shards, in which N is proportional to the collation body size of the shard they are validating.
4. Set the notary burst overhead proportional to the collation body size of the respective shard.

#### Shard Gas Limit Data Structure
Instead of gas_limit being a constant, we must store the gas_limit for each shard. i.e, replace
``` python
def get_collation_gas_limit() -> int128:
    return 10000000
```
with
``` python
def get_collation_gas_limit(shard_id):
    return self.GAS_LIMITS[shard_id]
```
where GAS_LIMITS is just added as a parameter to the contract constructor
``` python
_GAS_LIMITS: int128[self._SHARD_COUNT]
```
```python
self.GAS_LIMITS = _SHARD_COUNT
```

#### Collator Reward Function Implementation
Wherever the reward is applied....set the reward proportional to the gas limit of that shard.
``` python
reward = 0.001 * get_collation_gas_limit(shard_id) / 10000000
```


#### Proportional Period Publishing
In ```add_header(...)``` function of VMC, change condition that the collator is sending ```add_header(...)``` transaction in it's one allocated period to allow for submission up to floor(get_collation_gas_limit(shard_id)/10000000)-1 periods ahead.  In other words,
``` python
# expected_period_number == this period number
assert expected_period_number == floor(block.number /self.PERIOD_LENGTH)
```
becomes
``` python
# expected_period_number <= this period number <= expected_period_number + additional periods permitted
assert expected_period_number >= floor(block.number/self.PERIOD_LENGTH)
assert floor(block.number/self.PERIOD_LENGTH) <= expected_period_number + floor(get_collation_gas_limit/10000000)
```
Note that the number of additional periods is roughly proportional to the ```gas_limit``` of the respective collation.

Additionally, does the collator need to know their allocated period length?  In theory, they could just calculate it themselves, but if they should know, I'd assume that calling it from the VMC is safer.  But the question does still stand, do they need to know, or should they just solve it?  Does knowing the number of period help them at all?  I guess they do since if they are delayed then they should stop proposing the collation...but is this implemented now?

# Rationale
(baked into implementation section, should be fine)

# Backward Compatibility
There should not be any backwards compatibility issues.  This is merely a change to a few VMC parameter values and conditions.
