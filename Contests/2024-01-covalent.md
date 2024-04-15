# Covelant
Covelant contest || Staking, Nodes Block Producers || 22 Jan 2024 to 16 Jan 2024 on [sherlock](https://audits.sherlock.xyz/contests/127)

## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[M&#8209;01](#h-01-reentrancy-in-nextgencoremint-can-allow-users-to-mint-tokens-more-than-the-max-allowance)|Validators can stake greater than `validatorMaxStake`|MEDIUM|

---

## [M-01] Validators can stake greater than `validatorMaxStake`

### Summary
Validators can change their addresses via `OS::setValidatorAddress()`, making their `stake` amount exceeds `validatorMaxStake`.

### Vulnerability Detail
Validators should not stake with a value greater than `validatorMaxStake`, to distribute control over the network and BSP, and to prevent centralization control over the protocol by rich people.

In `OS::setValidatorAddress()`, the validator can change his address by sending staked tokens and shares to a new address.

```solidity
// FILE: OperationalStaking.sol#L697
  v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
  v.stakings[newAddress].staked += v.stakings[msg.sender].staked;
```

If the new address, is already a staker (delegator), it will increase the staked amount of the new validator (old staked + delegator staked).

By taking validatorMaxStake = `350_000e18` and maxCapMultiplier = `10`.

If the validator staked the maximum available tokens i.e.`350_000e18`, and delegated to another address that already has stakes, the validator will have stakes greater than `validatorMaxStake` parameter.

The more the validator stakes, the more delegators can stake their tokens with him.

So if the validator staked `350_000e18` (max), used another address and staked `350_000e18`, then changed the address to the address he staked with, the validator will end up with `700_000e18` staked tokens.

This will not only make the validator stake more than the limit, but also The `ValidatorMaxCap` will rise from: `3_500_000e18` to `7_000_000e18`, allowing the validator to accept more delegates.

[OperationalStaking.sol#L431-L435](https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L431-L435)
```solidity
    // cannot stake more than validator delegation max cap
    uint128 delegationMaxCap = v.stakings[v._address].staked * maxCapMultiplier;
    uint128 newDelegated = v.delegated + amount;
    require(newDelegated <= delegationMaxCap, "Validator max delegation exceeded");
    v.delegated = newDelegated;
```

$ValidatorMaxCap(max) = validatorMaxStake x maxCapMultiplier$

$ValidatorMaxCap(max) = 350,000e18 x 10 = 3,500,000e18$

---

$ValidatorMaxCap(now) = validator.stake x maxCapMultiplier$

$ValidatorMaxCap(now) = 700,000e18 x 10 = 7,000,000e18$

And if the validator wants to stake more, he can simply use another address, stake with it, and then change his address to him (again and again without stopping). Making the validator be able to break the `validatorMaxStake` and `maxCapMultiplier` checks.

### Impact
Validators can break `validatorMaxStake` and `maxCapMultiplier` checks.

### Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L697

### Tool used
Manual Review + Foundry

### Recommendation
Check the new validator staked balance when changing his address.

```diff
// OperationalStaking::setValidatorAddress‎() L696-L700
    v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
    v.stakings[newAddress].staked += v.stakings[msg.sender].staked;

+   require(v.stakings[newAddress].staked <= validatorMaxStake, "exceeds `validatorMaxStake` parameter");
    
    delete v.stakings[msg.sender];
```





