# Intuition
Intuition contest || Account Abstraction, Vaults || 21 Jun 2024 to 05 Jul 2024 on [hats.finance](https://app.hats.finance/audit-competitions/intuition-0x538dbadc50cc87b281cd655f1edbc6ebda02a66a/leaderboard)

## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[L&#8209;01](#l-01-changing-atomwarden-will-result-in-losing-atomwalletinitialdepositamount-for-created-and-not-deployed-atoms)|Changing `atomWarden` will result in losing `atomWalletInitialDepositAmount` for Created and not Deployed Atoms|LOW|
|[L-02](#l-02-unchecking-passed-value-in-setatomdepositfractionfortriple-to-feedenominator)|Unchecking passed value in `setAtomDepositFractionForTriple()` to feeDenominator|LOW|

---

## [L-01] Changing `atomWarden` will result in losing `atomWalletInitialDepositAmount` for Created and not Deployed Atoms

### Description

When creating new Atom wallets, there are two processes. First, is the creation of the atom vault. Second, is deploying the wallet.

When creating atom, `atomWalletInitialDepositAmount` goes to the atom wallet address that will be deployed using the current ID.

[EthMultiVault.sol#L481-L488](https://github.com/hats-finance/Intuition-0x538dbadc50cc87b281cd655f1edbc6ebda02a66a/blob/main/src/EthMultiVault.sol#L481-L488)
```solidity
        address atomWallet = computeAtomWalletAddr(id);

        // deposit atomWalletInitialDepositAmount amount of assets and mint the shares for the atom wallet
        _depositOnVaultCreation(
            id,
            atomWallet, // receiver
            atomConfig.atomWalletInitialDepositAmount
        );
```

When creating `atomWallet` address that will receive the initialDeposit, it is calculating using the current args, and `atomWarden` is one of the args.

[EthMultiVault.sol#L1421-L1423](https://github.com/hats-finance/Intuition-0x538dbadc50cc87b281cd655f1edbc6ebda02a66a/blob/main/src/EthMultiVault.sol#L1421-L1423)
```solidity
        bytes memory initData = abi.encodeWithSelector(
@>          AtomWallet.init.selector, IEntryPoint(walletConfig.entryPoint), walletConfig.atomWarden, address(this)
        );
```

But in case of deploying, we recompute this address again.

[EthMultiVault.sol#L366](https://github.com/hats-finance/Intuition-0x538dbadc50cc87b281cd655f1edbc6ebda02a66a/blob/main/src/EthMultiVault.sol#L366)
```solidity
    function deployAtomWallet(uint256 atomId) external whenNotPaused returns (address) {
        if (atomId == 0 || atomId > count) {
            revert Errors.MultiVault_VaultDoesNotExist();
        }

        // compute salt for create2
        bytes32 salt = bytes32(atomId);

        // get contract deployment data
@>      bytes memory data = _getDeploymentData();
        ...
        assembly {
            atomWallet := create2(0, add(data, 0x20), mload(data), salt)
        }
        ...
    }
```

So all AtomVaults that did not deployed there Wallets, will not be able to claim their initialAmount, if the `atomWarden` changed.

**Senario**
- UserA Created atom wallet
- After a while, the team changed `atomWarden` using `setAtomWarden`
- UserA deployed his AtomWallet using its vault ID, but the address is totally different from the one the received `InitialDepositAmount`

### Recommendations
In case of changing AtomWarden, you need to check that all created atoms gets deployed. this can either be done on-chain, or off-chain.

---

## [L-02] Unchecking passed value in `setAtomDepositFractionForTriple()` to feeDenominator

### Description
There are two types of fees that can be taken when deploying, depositing or redeeming from vaults.

- Static Fees
- % of fees relative to the deposited/redeemed amount

When setting the % of fees, the team checks that they do not exceeds `feeDenominator`. this can be seen in `setEntryFee()`, `setExitFee()`, and `setProtocolFee()`.

The problem lies in `setAtomDepositFractionForTriple()`, where there is no check for this value relative to the `feeDenominator`.

[EthMultiVault.sol#L270-L272](https://github.com/hats-finance/Intuition-0x538dbadc50cc87b281cd655f1edbc6ebda02a66a/blob/main/src/EthMultiVault.sol#L270-L272)
```solidity
    function setAtomDepositFractionForTriple(uint256 atomDepositFractionForTriple) external onlyAdmin {
        tripleConfig.atomDepositFractionForTriple = atomDepositFractionForTriple;
    }
```
This fees is a percentage fees taken from the amount deposited to TripleVaults, and goes to the Atom Vaults this TripleVault refers too.

[EthMultiVault.sol#L1210-L1213](https://github.com/hats-finance/Intuition-0x538dbadc50cc87b281cd655f1edbc6ebda02a66a/blob/main/src/EthMultiVault.sol#L1210-L1213)
```solidity
    function atomDepositFractionAmount(uint256 assets, uint256 id) public view returns (uint256) {
        uint256 feeAmount = isTripleId(id) ? _feeOnRaw(assets, tripleConfig.atomDepositFractionForTriple) : 0;
        return feeAmount;
    }
```

As we can see it is a % with respect to the total deposited. The problem will occur if there is no prevention from setting `atomDepositFractionForTriple` with value >= feeDenominator.

This will make the process revert because of underflow error, as fees is expected to be smaller than the real value.

### Recommendations

Check that the value do not exceeds to equal `feeDenominator`. And it can get capped to a given value like `feeDenominator / 2` to something.

> EthMultiVault::setAtomDepositFractionForTriple()

```diff
    function setAtomDepositFractionForTriple(uint256 atomDepositFractionForTriple) external onlyAdmin {
+       require(atomDepositFractionForTriple < generalConfig.feeDenominator, "EthMultiVault: exceeds Denominator");
        tripleConfig.atomDepositFractionForTriple = atomDepositFractionForTriple;
    }
```
