---
eip: TBD
title: Varying Gas Limit for Shards
author: Eli Hanover <eli.hanover@emory.edu>
type: Standards Track
category: Core
status: Draft
created: 2018-05-03
---


# Simple Summary



# Abstract
This draft EIP proposes and discusses the details of implementing varying gas limits in the Validator Manager Contract (VMC) in Phase 1 of Ethereum's sharding implementation.  By allowing collators time to submit collation headers proportional to the gas limit of their shard, this would raise the gas_limit cap on transactions while maintaining the same period length and without sacrificing decentralization.


# Motivation (work on this some more)
By allowing for variable gas limits between shards, we can permit higher gas transactions without altering the period length.  Doing so allows for more expensive state changes on some shards that are independent of lower gas_limit shards.



# Specification and Rationale
#### Initial Concerns and How to Ameliorate Them
If we were to simply specify higher gas limits for a select few  shards in the VMC, this would require proportionally higher computational resources on the collators of these higher gas limit shards.  This has multiple issues.  First, if we do not change the collator's reward proportional to the increase in resources needed to validate this collation, there is less incentive to solve for the collation header, potentially increasing dropout in specific shards.  This could ultimately lead to shards not being solved.  Additionally, the added computational resources needed could cause collators who did want to solve the collation header to nonetheless fail to because of the time restraint.

These concerns suggest a few necessary changes to other parts of the VMC, including:
1. Change the condition that collators must publish their collation header within their allocated period to one that allows them the ability to publish in an extended range of periods proportional to the gas limit of the shard they are validating.
2. Change the collators' reward to an amount proportional to the gas limit of the shard being validated in order to keep collators' incentives equal across shards.


#### Required Changes to the VMC
1. VMC must store the gas limit of each shard.
2. Change collator's reward to a function of the gas limit of the respective shard.
3. Change condition that collators must publish collation headers within their one allocated period to grant them publishing ability in the next N shards, in which N is proportional to the gas limit of the shard they are validating.

#### Shard Gas Limit Data Structure
Instead of gas_limit being a constant, we must store the gas_limit for each shard. i.e, replace
```
def get_collation_gas_limit() -> int128:
    return 10000000
```
with
```
gas_limits: int128[self.SHARD_COUNT]
```
in which the values would be set manually (although potential for a voting mechanic exists, that can be discussed elsewhere).
```
def get_collation_gas_limit(shard_id):
    return gas_limits[shard_id]
```
where gas_limits is initialized as a global array with the indexes corresponding to the shard_id and the value representing the gas limit of that shard.

#### Collator Reward Function Implementation


#### Proportional Period Publishing
In ```add_header(...)``` function of VMC, change condition that the collator is sending ```add_header(...)``` transaction in it's one allocated period to allow for submission up to floor(get_collation_gas_limit(shard_id)/10000000)-1 periods ahead.  In other words,
```
# expected_period_number == this period number
assert expected_period_number == floor(block.number /self.PERIOD_LENGTH)
```
becomes
```
# expected_period_number <= this period number <= expected_period_number + additional periods permitted
assert expected_period_number >= floor(block.number/self.PERIOD_LENGTH)
assert floor(block.number/self.PERIOD_LENGTH) <= expected_period_number + floor(get_collation_gas_limit/10000000)
```
Note that the number of additional periods is roughly proportional to the ```gas_limit``` of the respective collation.


#### Routing Transactions into These Differing Shards
Does this just take care of itself?  Validators of specific shard will accept if they have enough gas...but this makes me wonder..I changed the gas limit on the VMC, but does that do enough?  Hmm...well there's only a cap on the gas, so I guess it should work out?  How to avoid collator processing less, or do the transaction gas prices just take care of it?  I guess these larger gas transactions will bounce around until a higher gas_limit shard "picks it up", so it should work out in theory.

# Rationale


# Backward Compatibility
There should not be any backwards compatibility issues.  This is merely a change to a few VMC parameter values and conditions.

# Test Cases


# Implementations


# Copyright Waiver
