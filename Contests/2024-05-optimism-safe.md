# OP Labs | safe-extensions

Optimism-safe || MultiSig, Safe Wallet || 6 May 2024 to 10 May 2024 on [cantina](https://cantina.xyz/leaderboard/d47f8096-8858-437d-a9f5-2fe85ac9b95e)

## Summary

|ID|Title|
|:-:|-----|
|[M-01](#m-01-council-safe-owners-thresholds-invariants-are-not-verified-in-normal-executions)|`Council SAFE` Owners' thresholds Invariants are not verified in normal executions|
|[L-01](#l-01-foundation-wallet-can-make-council-wallet-do-calls-to-arbitrary-addresses)|Foundation Wallet can make Council Wallet do calls to arbitrary addresses|
|[I-01](#i-01-duplicate-importing-of-the-same-contract-interface)|Duplicate Importing of the Same Contract Interface|

---

## [M-01] `Council SAFE` Owners' thresholds Invariants are not verified in normal executions 

### Relevant Context
[LivenessGuard.sol#L125-L149
](https://cantina.xyz/code/d47f8096-8858-437d-a9f5-2fe85ac9b95e/packages%2Fcontracts-bedrock%2Fsrc%2FSafe%2FLivenessGuard.sol?scope=in_scope#L125-L149)

### Finding Description
Council `SAFE` wallet must have a threshold of at least `75%` of the owners, this threshold is being configured when one of the owners is removed using `LivenessModule.sol`.

[LivenessModule.sol#L240-L244](https://cantina.xyz/code/d47f8096-8858-437d-a9f5-2fe85ac9b95e/packages%2Fcontracts-bedrock%2Fsrc%2FSafe%2FLivenessModule.sol?scope=in_scope#L240-L244)
```solidity
    function _verifyFinalState() internal view {
        ...

        // @audit verify that the new threshold is `75%`, after removing owners
         uint256 threshold = SAFE.getThreshold();
        require(
            threshold == getRequiredThreshold(numOwners),
            "LivenessModule: Insufficient threshold for the number of owners"
        );
    }
```

The owner will be removed if he did not show his liveness in `LivenessGuard`, and if `LIVENESS_PERIOD` passes, it will be able to get removed by anyone using `LivenessModule`.

The problem here is that the protocol uses this check to confirm that all `SAFE` owners have access to their Private keys and no one lost his key. However, what about the case of stealing the key?

If one of `SAFE` owners noticed that his Private Key got stolen, the council should remove that owner, but if this wallet owner showed his Liveness recently, they should remove it by doing a tx using `SAFE::execTransaction()` (Sign Multisig tx to remove that owner), as using `LivenessModule` will not allow them to remove him in this case.

And this is the problem, when removing the owner whose key got stolen, the `threshold` value is not checked if it is still >= `75%` or not, so if the value of the `threshold` changes wrongly the Wallet will be set with wrong threshold, as the value is passed as a parameter, and is not verified.

This is an `ADMIN` issue after all, But as the team designed the MultiSig invariants to be onchain, having such a thing not checked onchain makes centralization issues, especially when dealing with Big Protocols like Layer2(s).

And there are other invariant which is `MIN_OWNERS`, but since removing owners below that number will give the accessibility for anyone to remove others it is not that big thing.

_**In Breif**: Invariants are only checked if removing/adding owners is done by `LivenessModule`, but not if it is done using the wallet itself. And we pointed out why the Wallet will need to do such a tx._

## Impact Explanation
Council `SAFE` can be left with threshold smaller than required, or getting set wrongly. And if it is set with smaller threshold than required, then a small group can combine together to kick off the rest and replace there wallets with another ones. taking the control of the Council which is a critical wallet.

### Likelihood Explanation

Likelihood: LOW, as it is an ADMIN mistake. 

Severity: HIGH
- a partial group of the council can control the wallet, and kick off others.
- Breaking threshold invariant.


### Proof of Concept

- One of the Wallet Owners noticed that his key got stoled (and it may be 2 or 3)
- This person Notified The Council about the problem
- The Council wanted to make a batch transaction to remove the Old Owner(s) (whose private key(s) got stolen) and add new one(s).
- The threshold is set wrongly by mistake.
- The Wallet will be in a state of the wrong threshold, which is not a desired state.

NOTE: setting the number wrongly can be a mistake, or some of the council may try to deceive others into signing such a transaction, and there is no onchain check that will prevent that tx.

1. Making threshold value less value (firing the tx as it is getting signed)
2. The Malicious Owners fired another tx to kick off the others and replace their keys with the keys they control.

This scenario is a bit hard and may be unrealistic, but I wanted to point out all the cases that can occur.


### Recommendation
Check the threshold is not set wrongly in `LivenessGuard::checkAfter()`.

```diff
    function checkAfterExecution(bytes32, bool) external {
        ...

+       uint256 THRESHOLD_PERCENTAGE = 75;
+       uint256 requiredThreshold = (ownersAfter.length * THRESHOLD_PERCENTAGE + 99) / 100;
+       require(SAFE.getThreshold() == requiredThreshold, "LivenessGuard: Threshold was set incorrectly");
    }
```

This will not only prevent changing owners wrongly, or the malicious actions between council members. but it will also verify all council `SAFE` TXs including changing threshold tx itself, and preserves >= `75%` invariant.

And for the `MIN_OWNERS` check, I pointed out that anyone can use `LivenessModule` to transfer the ownership to `FALLBACK_OWNER`, if the numbers became less than `MIN_OWNERS (8)`.

---

## [L-01] Foundation Wallet can make Council Wallet do calls to arbitrary addresses

### Description
In `DeputyGuarduanModule::blacklistDisputeGame()` and `DeputyGuarduanModule::setRespectedGameType()`, the Portal address is passed by the Foundartion wallet (DeputyGuardian). this should be one of the OP_Portal contracts, but what if the address was not one of them?

Since the encoding happens according to `blacklistDisputeGame()` and `setRespectedGameType()` functions, the function signature is restricted only to two values, so this should not make Council `SAFE` affected by these calls.

However, since the council wallet is a critical wallet having such a thing is not ideal. We do not know what are the contracts the council `SAFE` has the authority of it, and there may be a signature collision in one of them that allows the Foundation Wallet to force the Council wallet for unintended behavior.

### Recommendation
Don't allow the call to any address, and make it restricted to only a number of addresses. This can be done by implementing `EnumerableSet` DS from `OpenZeppelin` for the available addresses.

Another solution (partial mitigation configured by OP team) is to make another wallet (Guardian Wallet) as the wallet that will have the Desputy Guardian module. but this will not prevent Foundation from letting Guardian Wallet do arbitrary calls, but it is OK as the wallet should not be that important.

---

## [I-01] Duplicate Importing of the Same Contract Interface

### Description
In `LivenessModule` contract, `OwnerManager` interface is imported twice as seen in the codesniped.

### Recommendations
Import it only one time

