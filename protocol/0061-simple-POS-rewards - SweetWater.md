# Validator and Staking POS Rewards
This describes the SweetWater requirements for calculation and distribution of rewards to delegators and validators. For more information on the overall approach, please see the relevant research document.

## Calculation

At the end of an [epoch](./0050-epochs.md), payments are calculated. This is done per active validator:

* First, `score_val(stake_val)` calculates the relative weight of the validator given the stake it represents.
* For each delegator that delegated to that validator, `score_del` is computed: `score_del(stake_del, stake_val)` where `stake_del` is the stake of that delegator, delegated to the validator, and `stake_val` is the stake that validator represents.
* The fraction of the total available reward a validator gets is then `score_val(stake_val) / total_score` where `total_score` is the sum of all scores achieved by the validators. The fraction a delegator gets is calculated accordingly.
* Finally, the total reward for a validator is computed, and their delegator fee subtracted and divided among the delegators.
* If the validator (or, the associated key) does not have sufficient stake (at least the network parameter min_own_take), 
  then the reward is set to zero. The corresponding money is kept by the network, not distributed among the other validators.


Variables used:

- `min_val`: minimum validators we need (for now, 5). This is a network parameter that can be changed through a governance vote (post-sweetwater).
- `compLevel`: competitition level we want between validators (default 1.1). This is a Network parameter that can be changed through a governance vote. Valid values are in the range 1 to infinity i.e. (including 1 but excluding infinity) i.e. `1 <= compLevel < infinity`.
- `num_val`: actual number of active validators. The value is derived from the environment.
- `a`: The scaling factor; which will be `max(min_val, num_val/compLevel)`. So with `min_val` being 5, if we have 6 validators, `a` will be `max(5, 5.4545...)` or `5.4545...`. This is computed from the parameters/staking data.
- `delegator_share`: propotion of the validator reward that goes to the delegators. The initial value is 0.883. This is a network parameter that can be changed through a governance vote. Valid values are in the range 0 to 1 (inclusive) i.e. `0 <= delegator_share <= 1`.

Functions:

- `score_val(stake_val)`: `min(stake_val, 1/a), where totalstake is the total number of staked tokens. This function assumes that the stake is normalized, i.e., the sum of stake_val for all validators equals 1. If this is not the case, 
  stake_val needs to be replaced by stake_val/total_stake, where total_stake is the sum of stake_val over all validators.
- `score_del(stake_del, stake_val)`: for now, this will just return `stake_del`, but will be replaced with a more complex formula later on, which deserves independent testing.
- `delegator_reward(stake_val)`: `stake_val * delegator_share`. Long term, there will be bonuses in addition to the reward.



## Distribution of Rewards
We assume a function total_payment() which computes the total payment for a given epoch, as well as some ressource pool 
from which the resources are taken; if the total_payment for a given epoch exceeds the size of the pool, then the 
entire pool is paid out. 


The total payment will then be distributed among validators and delegators following above formulas.


## Maximal Delegatable Stake
The maximal delegatable amount of stake is supposed to prevent delegators from delegating too much to an individual validator, and is an additional measure to the economic incentive.
For this value to be meaningful, it needs to be based on the total number of delegated tokens, not on the total number of existing tokens, which can be substantially higher.

To this end, at the beginning of each epoch, we need to compute the total number of tokens (it is sufficient to make an approximation that can be slightly higher. To simplify things, this can be done by simple adding all delegations and substracting all undelegations to the current amount of delegated tokens, ignoring that some delegations might fail.

The value at which delegation is stopped is then computed similar to the reward function:
```
a := max(s.minVal, s.numVal/s.compLevel)
max_delegatable_tokens = total_delegated_tokens / a
```

Comments:

With the new reward function, this is not critical, as validators do not lose money if they
get too much delegation
We can recursively compute the exact amount (compute the approximation as above, see if any delegations would fail with this threshold, substract, those, repeat). This is not necessary at this point though
We can use the existing parameter to add a stretch-factor to allow every validator go get a little bit too much. This is also not needed for SW though.


