# ZetaChain
ZetaChain contest || Layer1, Cross Chain, Omnichain || 19 Aug 2023 to 04 Sept 2024 on [cantina](https://cantina.xyz/competitions/80a33cf0-ad69-4163-a269-d27756aacb5e/leaderboard)

## My Findings Summary

|ID|Title|Severity|
|:-:|-----|:------:|
|[H&#8209;01](#h-01-all-zetatoken-transfers-from-evm---zeta-cant-be-recovered)|All ZetaToken Transfers from `EVM -> Zeta` can't be recovered|HIGH|
|[H-02](#h-02-revertcontext-struct-implementation-uses-uint64-for-assets-to-recover-which-is-too-small)|`RevertContext` struct implementation uses `uint64` for assets to recover which is too small|HIGH|
||||
|[M&#8209;01](#m-01-universal-apps-cant-verify-the-original-message-source-in-case-of-recovery)|Universal Apps can't verify the original message source in case of recovery|MEDIUM|
|[M-02](#m-02-forcing-gas_limit-in-native-token-transfers-can-make-smart-contracts-receiving-go-oog)|Forcing `GAS_LIMIT` in native token transfers can make Smart Contracts receiving go OOG|MEDIUM|
|[M-03](#m-03-malicious-users-can-manipulate-revertable-contracts-onrevert-as-a-normal-message-on-evm-chains)|Malicious users can manipulate `Revertable` Contracts `onRevert()` as a normal message on `EVM` chains|MEDIUM|
|[M-04](#m-04-message-transfereing-with-onrevert-cant-get-recovered)|Message transfereing with `onRevert()` can't get recovered|MEDIUM|
|[M-05](#m-05-handling-reverting-with-onrevert-support-for-zeta---evm-will-always-revert-if-the-token-is-zetatoken)|Handling Reverting with `onRevert()` support for `Zeta -> EVM` will always revert if the token is ZetaToken|MEDIUM|
|[M-06](#m-06-a-malicious-user-can-cause-a-dos-zetachain-clients-by-spamming-evm-inbound-calls)|All ZetaToken Transfers from `EVM -> Zeta` can't be recovered.|MEDIUM|


---

## [H-01] All ZetaToken Transfers from `EVM -> Zeta` can't be recovered.

### Description
When depositing tokens from `EVM -> ZetaChain` we are either transfering Native tokens / ERC20 tokens or the Zetatoken in that source chain (also in ERC20).

[GatewayEVM.sol#L248-L250](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/evm/GatewayEVM.sol#L248-L250)
```solidity
    function deposit( ... ) ... {
        if (amount == 0) revert InsufficientERC20Amount();
        if (receiver == address(0)) revert ZeroAddress();

@>      transferFromToAssetHandler(msg.sender, asset, amount);

        emit Deposited(msg.sender, receiver, amount, asset, "", revertOptions);
    }
    ...
    // @audit If the tokens is ZetaToken ERC20 we are transferring it to Connector, else deposit into ERC20Custody
    function transferFromToAssetHandler(address from, address token, uint256 amount) private {
@>      if (token == zetaToken) { ... }
        else { ... }
    }
```


Native tokens like `ether` in Ethereum, and `matic` in Polygon are represented as `ZRC20` on ZetaChain, and zeta token has the same implementation as `WETH9`.

If we checked how we are recovering deposits with messages from `EVM -> Zeta`, which are transfers using `depositAndCall()` on EVM Gateways. we will find there are two function implementations for `depositAndCall()` in ZEVM Gateway.

[GatewayZEVM.sol#L310-L327](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L310-L327) | [GatewayZEVM.sol#L334-L350](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L334-L350)
```solidity
    // deposit and call with ZRC20 tokens
    function depositAndCall( ... ) ... {
        ...

@>      if (!IZRC20(zrc20).deposit(target, amount)) revert ZRC20DepositFailed();
        UniversalContract(target).onCrossChainCall(context, zrc20, amount, message);
    }
    ...
    // deposit and call with ZetaToken
    function depositAndCall( ... ) ... {
        ...

@>      _transferZETA(amount, target);
        UniversalContract(target).onCrossChainCall(context, zetaToken, amount, message);
    }
```

For `ZRC20` tokens we are calling `deposit()`, but if the token is ZetaToken we are not calling `deposit()`, as ZetaToken in ZEVM is like wrapped ETH (don't implement that function), so there is another way to recover Zetatoken transfers from Source chain, by transferring them to the recipient blockchain tokens.

The issue is that there is no function in `GatewayZEVM` that allows recovering transfers of ZetaToken from the source chain without a message, there is only one instance of `deposit()` that is used for `ZRC20` tokens, but there is no another one for depositing `Zeta Token`.

This will result in the inability to recover ZetaToken deposits from EVM Chains with no messages, leading to loss of user funds. 

### Recommendations
Implement a `deposit()` function that recovers the native Zeta deposits without messages.

```diff
diff --git a/v2/contracts/zevm/GatewayZEVM.sol b/v2/contracts/zevm/GatewayZEVM.sol
index d51ec41..13786a1 100644
--- a/v2/contracts/zevm/GatewayZEVM.sol
+++ b/v2/contracts/zevm/GatewayZEVM.sol
@@ -276,7 +276,11 @@ contract GatewayZEVM is
 
         if (target == FUNGIBLE_MODULE_ADDRESS || target == address(this)) revert InvalidTarget();
 
-        if (!IZRC20(zrc20).deposit(target, amount)) revert ZRC20DepositFailed();
+        if (zrc20 == zetaToken) {
+            _transferZETA(amount, target);
+        } else {
+            if (!IZRC20(zrc20).deposit(target, amount)) revert ZRC20DepositFailed();
+        }
     }
 
     /// @notice Execute a user-specified contract on ZEVM.
```

---

## [H-02] `RevertContext` struct implementation uses `uint64` for assets to recover which is too small

### Description

When transferring tokens `Zeta <-> EVM`, we are providing `RevertOptions`, where we can either call `onRevert()` hook or not depending on the option we provide.

[Revert.sol#L10-L16](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/Revert.sol#L10-L16)
```solidity
struct RevertOptions {
    address revertAddress;
    bool callOnRevert; // << either to call `onRevert()` on destination
    address abortAddress;
    bytes revertMessage;
    uint256 onRevertGasLimit;
}
```

If we are going to call `onRevert()` the `TSS_ROLE` recovers assets by transferring them to the receiver, it provides `RevertContext` Struct object which is passed as a parameter when calling `onRevert()` to make the receiver recover the token back, but the problem is that the number of tokens is expressed as `uint64` not `uint256`.

[Revert.sol#L22-L26](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/Revert.sol#L22-L26)
```solidity
struct RevertContext {
    address asset;
>>  uint64 amount;
    bytes revertMessage;
}
```

`uint64` is so small amount to express an ERC20 token, we will not be able to acctually recover most of normal deposits.

- `type(uint64).max` = 18_446_744_073_709_551_615 ~= `18.45e18`.

Since the common decimal is `18`, this means that if the amount of tokens to recover exceeds `18.45` we will not be able to recover them, as this will result in overflow.

If we say that the token amount worth `1$`, this means that we will not be able to recover the transfer what worth `19$` or more.

A side Note here, is that ZetaToken is worth `0.4476$`, from the time of writing this report. so we will not be able to recover even `9$` worth of Zeta transfers in case of reverting with `onRevert()` hook.

So this will result in the inability to recover most of ERC20 transfers because of extremely low data size for assets.

### Recommendations
Change the type of assets to `uint256` instead of `uint64`.

```diff
diff --git a/v2/contracts/Revert.sol b/v2/contracts/Revert.sol
index 7675f0c..3d0d483 100644
--- a/v2/contracts/Revert.sol
+++ b/v2/contracts/Revert.sol
@@ -21,7 +21,7 @@ struct RevertOptions {
 /// @param revertMessage Arbitrary data sent back in onRevert.
 struct RevertContext {
     address asset;
-    uint64 amount;
+    uint256 amount;
     bytes revertMessage;
 }
```

---
---
---

## [M-01] Universal Apps can't verify the original message source in case of recovery

### Description

> We will talk about Universal Apps in `ZetaChain` (ZEVM), but the issue existed also in EVM (Revertable Contracts).

Universal Apps are the apps are the contracts deployed on `ZEVM`, that will be used to integrate with all `EVMs` connected to Zetachain, besides Bitcoin, and Solana.

These Apps should implement 2 important functions.
1. `onCrossChainCall()`: this will get fired when a message comes from `EVM` to `ZEVM`.
2. `onRevert()`: this will get executed when making a call from that `Universal App` on `ZEVM` to another destination `EVM` and it gets reverted.

[zevm/interfaces/UniversalContract.sol#L24-L34](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/interfaces/UniversalContract.sol#L24-L34)
```solidity
interface UniversalContract {
    function onCrossChainCall( ... ) external;

    function onRevert(RevertContext calldata revertContext) external;
}
```

`RevertContext` is the struct Object that is passed to the `Universal App` on `ZEVM` when calling `onRevert()` in case of reverting.

[Revert.sol#L22-L26](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/Revert.sol#L22-L26)
```solidity
struct RevertContext {
    address asset;
    uint64 amount;
    bytes revertMessage;
}
```

- `asset`: is the `ZRC20` token address (native coin is address_zero) we are transferring (in case of token transfer).
- `amount`: is the amount of tokens we are transferring.
- `revertMessage`: is used so that the Universal App can recover assets, etc...

When the user makes a transfer `ZEVM <-> EVM`, he passes `RevertOptions`, which includes the address ZetaClients will call in the case of revert of Omnichain tx.

[Revert.sol#L10-L16](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/Revert.sol#L10-L16) 
```solidity
struct RevertOptions {
>>  address revertAddress;
    bool callOnRevert;
    address abortAddress;
    bytes revertMessage;
    uint256 onRevertGasLimit;
}
```

The users have the freedom to set the `revertAddress`, which is the address that we will call `onRevert()` on it, as any value, when transferring from `ZEVM` to any `EVM`.

[zevm/GatewayZEVM.sol#L172](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L172)
```solidity
    function withdrawAndCall(
        bytes memory receiver,
        uint256 amount,
        address zrc20,
        bytes calldata message,
        uint256 gasLimit,
>>      RevertOptions calldata revertOptions
    ) ... { ... }
```

We know that calling `onRevert()` is only called in case of a transfer from `ZEVM` to `EVM` gets reverted, so Universal App restricted it to be only callable by `GatewayZEVM`, but the problem here is how can the Univeral App authorize that this calling of `onRevert()` is for a failed transfer from his side?

We will illustrate what is the problem in the POC.

### Proof of Concept
- We have two universal Apps `UA1` and `UA2`.
- each of these apps implements the `onCrossChainCall` and `onRevert`, and can be used to interact between `ZEVM` and other supported `EVMs` in Zetachain.
- `onRevert()` is used to recover user assets back, in case the transfer reverts.
- Attacker is using `UA1` (UA1 is a malicious Universal App made by the Attack).
- Attacker made a transfer from `UA1` to another `EVM` chain.
- Attacker provided the RevertOptions with revertAddress equal to `UA2` (since he controls `UA1`).
- Message reverted.
- ZetaClients will recover the failed transfer.
- They will call `onRevert()` on the revertAddress provided, which is `UA2`.
- `UA2` will make the recovery process, thinking it was an original transfer from his side.
- Now, let's examine what happens:
  - MessageCalling/Transfering of tokens occurring from `UA1`
  - The transfer revert.
  - Recovery occurs at `UA2`.

The issue is that `UA2` will not be able to authorize whether this calling of `onRevert()` is because a reverted transfer (Omnichain transfer) happened on his side or not. as calling `onRevert()` is not providing the sender/caller for that withdrawal, which is `UA1`.

There is no way for Universal App to authorize that the calling of `onRevert()` is because of a revert of a transfer (Omnichain transfer) from their contract or another contract, as we are not passing the sender.

### Recommendations

Pass the sender of the message in the `RevertContext` Object, so that `Universal Apps` can authorize that they were the original sender of that message to be recovered.

```diff
diff --git a/v2/contracts/Revert.sol b/v2/contracts/Revert.sol
index 7675f0c..ad5a775 100644
--- a/v2/contracts/Revert.sol
+++ b/v2/contracts/Revert.sol
@@ -20,6 +20,7 @@ struct RevertOptions {
 /// @param amount Amount specified with the transaction.
 /// @param revertMessage Arbitrary data sent back in onRevert.
 struct RevertContext {
+    bytes sender;
     address asset;
     uint64 amount;
     bytes revertMessage;
```

- Since `UA1` is a Universal App and interacts with EVMs supported in ZetaChain, it can use `GatewayZEVM`.
- When making a call, we are emitting the sender (msg.sender) that calls `GatewayZEVM`, which is `UA1`.
- ZetaClient knows who was the sender by listening to the event.
- ZetaClients can pass that sender in the RevertContext object.
- Universal Apps can authorize there was the original sender of that message.

_NOTE: we have to make the `sender` in bytes to support `onRevert()` in non EVM chains_

---

## [M-02] Forcing `GAS_LIMIT` in native token transfers can make Smart Contracts receiving go OOG

### Description
When withdrawing ZRC20 tokens, we are passing the `GAS_LIMIT` as a constant value.

[zevm/GatewayZEVM.sol#L144](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L144) | [zevm/GatewayZEVM.sol#L91-L94](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L91-L94) 
```solidity
    function withdraw( ... ) ... {
        if (receiver.length == 0) revert ZeroAddress();
        if (amount == 0) revert InsufficientZRC20Amount();

>>      uint256 gasFee = _withdrawZRC20(amount, zrc20);
    }
    ...
    function _withdrawZRC20(uint256 amount, address zrc20) internal returns (uint256) {
        // Use gas limit from zrc20
>>      return _withdrawZRC20WithGasLimit(amount, zrc20, IZRC20(zrc20).GAS_LIMIT());
    }
```

ZRC20 tokens are not just representing ERC20 on differnet EVM chains, but it also include native EVM chain coin. i.e it can include ethereum.

Forcing the `GAS_LIMIT` to be a constant value is Ok for most of the tokens, as in most cases transfering ERC20 tokens gas cost is constant for all tokens, it can differ but gas used will always be the same when ever you transfer, and who ever the receiver, so Making the value is constant is OK.

The problem is that Native coins gas used is not constant, and varies according to the receiver.
- If the receiver is EOA
- If the receiver is Smart Contract Wallet (Safe wallet consums more gas for example)
- If the receiver is Account Abstraction wallet
- If the receiver is a Smart Contract (with fallback handler)

Smart Contracts can have custome logic when they receive ether, this is the case for Safe Smart Contract Wallets for example. so forcing the value of gas is not good as If we made it the amount for EOA addresses, then Smart Contract Wallets transfer function can revert. and if we made it with large value we are making users pay more than they should.

This occuar for only withdrawing process, in case of withdrawAndCall we are providing the value ourselves.

### Recommendation
Make the `GAS_LIMIT` customizable, by either making it passed by the user or adding it to the Limit `GAS_LIMIT + user additional gas passed`.

---

## [M-03] Malicious users can manipulate `Revertable` Contracts `onRevert()` as a normal message on `EVM` chains

### Description

> Universal Apps are apps that support `onCrossChainCall()` and `onRevert()` on `ZetaBlockchain`, and Revertable contracts are contracts that support `onRevert()` on `EVM` chains, we will use `Universal Apps` expression to express both og them in this report.

When sending transactions from `Zeta` to `EVM` we are doing an arbitrary call to the destination contract. If there are tokens to transfer, we approve the destination contract to do what he wants with tokens then we reset the Approval to zero.


[evm/GatewayEVM.sol#L78-L83](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/evm/GatewayEVM.sol#L78-L83) | [evm/GatewayEVM.sol#L163-L169](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/evm/GatewayEVM.sol#L163-L169)
```solidity
    function executeWithERC20( ... ) ... {
        ...
        if (!resetApproval(token, to)) revert ApprovalFailed();
1:      if (!IERC20(token).approve(to, amount)) revert ApprovalFailed();
        // Execute the call on the target contract
2:      _execute(to, data);

        // Reset approval
        if (!resetApproval(token, to)) revert ApprovalFailed();
        ...
    }
    ...
    function _execute(address destination, bytes calldata data) internal returns (bytes memory) {
3:      (bool success, bytes memory result) = destination.call{ value: msg.value }(data);
        if (!success) revert ExecutionFailed();

        return result;
    }

```

This is the flow of a successful message from `Zeta` to `EVM`, now let's say how users interact when making transactions from `EVM` to `Zeta`.

Now lets discuss the flow for transactions from `EVM` to `Zeta`. If we transfered a transaction from `EVM` to `Zeta` and the transaction gets reverted, we are sending assets to the user back on source chain `EVM` chain and call `onRevert()` if it is a Revertable contract (supports `onRevert()`).

Now this is the problem, Revertable contracts `onRevert()` hook should be designed in a way to recover assets back after it received them, ZetaClients calls it by providing `RevertContext` Object, which contains the ERC20 address, amount received, and a custom message.

[Revert.sol#L22-L26](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/Revert.sol#L22-L26)
```solidity
struct RevertContext {
    address asset;
    uint64 amount;
    bytes revertMessage;
}
```

The way of recovering Failed `EVM -> Zeta` txs is by calling `revertWithERC20()`, in case of ERC20 token transfers, which sends money to the Revertable contract then call `onRevert()` hook, to recover assets back.


[evm/GatewayEVM.sol#L187-L206C6](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/evm/GatewayEVM.sol#L187-L206C6)
```solidity
    function revertWithERC20( ... ) ... {
        ...

1:      IERC20(token).safeTransfer(address(to), amount);
2:      Revertable(to).onRevert(revertContext);

        emit Reverted(to, token, amount, data, revertContext);
    }
```


Know These Universal apps `onRevert()` takes the `asset` and `amount` with a custom message to handle tokens, and since Gateway is a trusted entity, Universal Apps, which are apps built to interact between different BlockChains, trusts this calling `onRevert()` to recover tokens etc...

The idea here is that The ERC20 `address` and `amount` was transferred to the Universal App actually, if the Universal app receives a call `onRevert()`, the parameters including ERC20 address, assets amount are handled to be already received by the Universal App. 

Universal App will only make `GatewayEVM` has the authority to call this function, as the assets and amount passed are the assets that the app just received from the Gateway.

### Proof of Concept

The issue here lies in the arbitrary call we can make when doing `Zeta -> EVM`.
1. Make an Omnichain call `Zeta -> EVM`.
2. We made the receiver one of the `Universal Apps` on the destination `EVM` chain that supports `onRevert()`.
3. Call a function selector `onRevert()`, and add `RevertContext` as parameter input.
4. ZetaClients process the message.
5. They made that arbitrary call to the `Univeral App`.
6. The Universal app `onRevert()` function gets fired with the ERC20 address, and amount in the `RevertContext` parameter received from the `GatewayEVM`.
7. The Universal app now will try to recover these assets back (since the caller is GatewayEVM App, it thinks it is a recovery method)
8. Taking any ERC20 tokens in the `Universal App` easily using this flow, by recovering tokens transfer assets that is not yet transfered.

_NOTE: the issue here is not because of `Universal App` implementation, no. the idea here is that `onRevert()` is designed to get called to recover assets that was transferred by the `GatewayEVM`, so `Universal Apps` implementing it will implement it that way, i.e recovering a failed transfer of assets we send from the `App` to `Zeta`, we received the tokens back and want to recover them. So `onRevert()` should be called for a reverted message only and not called independently._

_Since the caller is always the `GatewayEVM` Universal Apps will have no way to distinguish between `onRevert()` which is a real recovery process, of funds they receiver, or a normal Omnichain message comes from Zeta. Since GatewayEVM can call arbitrary calls, which makes the issue in the design implementation of ZetaChain itself, not in the `Universal App`_

### Recommendations

Prevent the execution of `onRevert(RevertContext)` function when doing the arbitrary call. This can be done by preventing calling this function selector.

```diff
diff --git a/v2/contracts/evm/GatewayEVM.sol b/v2/contracts/evm/GatewayEVM.sol
index 18a1766..b9f137d 100644
--- a/v2/contracts/evm/GatewayEVM.sol
+++ b/v2/contracts/evm/GatewayEVM.sol
@@ -76,6 +76,7 @@ contract GatewayEVM is
     /// @param data Calldata to pass to the call.
     /// @return The result of the call.
     function _execute(address destination, bytes calldata data) internal returns (bytes memory) {
+        if (bytes4(data) == Revertable.onRevert.selector) revert("InvalidSelector()");
         (bool success, bytes memory result) = destination.call{ value: msg.value }(data);
         if (!success) revert ExecutionFailed();
```

---

## [M-04] Message transfereing with `onRevert()` can't get recovered

### Description
When passing messages either with token transfer or not `EVM <-> Zeta` if the destination is ZetaChain then we are not paying gas for a destination, and if the destination is `EVM` then we pay for the gas.

If the transfer/message sending failed we can have a mechanism to recover this by calling `onRevert()` in the revertAddress on the source chain either `EVM` or `Zeta`.

[Revert.sol#L10-L16](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/Revert.sol#L10-L16)
```solidity
struct RevertOptions {
>>  address revertAddress;
    bool callOnRevert;
    address abortAddress;
    bytes revertMessage;
>>  uint256 onRevertGasLimit;
}
```

Users provide the gas limit that will be used to execute their `onRevert()`. since users don't pay for `onRevertGasLimit`, ZetaChain takes an amount of the assets transferred to cover the gas that will be used for that gasLimit (this happens at the protocol level).

We are aware that if the amount of assets transferred value is less than the value of `onRevertGasLimit`, ZetaClients will not process the recovery process as assets transferred can't pay for `onRevertGasLimit`, but we don't know actually how this case is handled and we think it is a design choice after all.

The problem that we want to point out in this report is that there are type of message calls that don't even pass assets (message transferring). As we can see in the GatewayEVM, we call  `call()` function that is used as message protocol transfering only, no assets are transferred.

[evm/GatewayEVM.sol#L306-L317](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/evm/GatewayEVM.sol#L306-L317)
```solidity
    function call(
        address receiver,
        bytes calldata payload,
>>      RevertOptions calldata revertOptions
    ) ... {
        if (receiver == address(0)) revert ZeroAddress();
        emit Called(msg.sender, receiver, payload, revertOptions);
    }
```

As we can see, this function is used to transfer a message from `EVM` to `Zeta`, since ZetaClients pay the gas on Zeta, we are not talking gasFees for destination chain execution, since it is ZetaChain, but if the execution on the destination failed, ZetaClients will try to process recovery process, by calling revertAddress in `RevertOptions` on the source `EVM` chain.

The problem here is that ZetaClients are designed to take an amount of assets transfered to cover for the `onRevertGasLimit`, but as this is just a message transfereing not assets transferring, ZetaClients will not be able to do anything. as no tokens are paid by that user.

The function accepts `RevertOptions`, which is used in recovery in case of failed of the message/token transfer. In case of token transfer, we are taking some tokens, but for message transfers, there are no tokens. it is impossible to call `onRevert()` since there are no funds paid by the sender to recover `onRevertGasLimit`.

This will end up being unable to recover reverted messages since no assets are transferred to it.

This issue affects message transfering with no assets in `GatewayZEVM` too.

### Recommendations
Either take the amount `onRevertGasLimit` in case of supporting Reverting with `onRevert()` mechanism. or disallow recovering for messages only (this will be easier).

_NOTE: there is a side issue we mentioned in this issue report which is the small amount of token transferred, that is worth cents, can be unrecoverable because the `onRevertGasLimit` can exceed all transferred oken value, especially in the Ethereum network that has High tx cost. But as we said earlier we think this is a design choice. Please if this side issue is judged a real issue, consider duplicating it with its group_ 

---


## [M-05] Handling Reverting with `onRevert()` support for `Zeta -> EVM` will always revert if the token is ZetaToken

### Description
When depositing tokens from `Zeta -> EVM` we either transfer Zeta token (native) or ZRC20 tokens.

[zevm/GatewayZEVM.sol#L131-L136](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L131-L136) | [zevm/GatewayZEVM.sol#L200-L205](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L200-L205)
```solidity
    // @audit withdrawing ZRC20 tokens from ZetaChain to a destination EVM chain
    function withdraw( ... ) ... {
        ...

>>      uint256 gasFee = _withdrawZRC20(amount, zrc20);
        emit Withdrawn( ... );
    }
    ...
    // @audit withdrawing native ZetaToken from ZetaChain to destination EVM chain
    function withdraw( ... ) ... {
        ...

>>      _transferZETA(amount, FUNGIBLE_MODULE_ADDRESS);
        emit Withdrawn( ... );
    }
```


When transferring `ZRC20` from `Zeta` to `EVM`, we are burning tokens on ZetaChain, but if the token is ZetaToken itself native coin in ZetaChain, we are not burning we are transferring it to the `Fungible address` using `_transferZETA()`.

If this transfer process reverts (from Zeta to EVM), we should be able to recover them by sending them back to the sender on the source chain `ZetaChain`. and in case the sender is a Universal App (supports  `onRevert()`), we should transfer the tokens back to him then call `onRevert()` directly.

The problem is that the function(s) used to handle reverting in `GatewayZEVM` are `executeRevert()` and `depositAndRevert()`.

1. `depositAndRevert()` is used for all `ZRC20` tokens, but it can't be called in case of Native tokens, as it do not transfer Native, and it calls `ZRC20::deposit(address,uint256)` interface, which only existed in ZRC20 tokens, not ZetaWrapped token (wrapped Zeta is similar to WETH9).

[zevm/GatewayZEVM.sol#L366-L371](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L366-L371)
```solidity
    function depositAndRevert( ... ) ... {
        if (zrc20 == address(0) || target == address(0)) revert ZeroAddress();
        if (amount == 0) revert InsufficientZRC20Amount();
        if (target == FUNGIBLE_MODULE_ADDRESS || target == address(this)) revert InvalidTarget();

>>      if (!IZRC20(zrc20).deposit(target, amount)) revert ZRC20DepositFailed();
        UniversalContract(target).onRevert(revertContext);
    }
```

2. And in the case of `executeRevert()` we are not allowing sending any native coins (zeta), so we can't recover Zeta by sending Native.

[zevm/GatewayZEVM.sol#L355-L359](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/zevm/GatewayZEVM.sol#L355-L359)

```solidity
    function executeRevert(address target, RevertContext calldata revertContext) external onlyFungible whenNotPaused {
        if (target == address(0)) revert ZeroAddress();

        UniversalContract(target).onRevert(revertContext);
    }
```

This is not the case in `GatewayEVM`, where recovering native coins can occuar as `executeRevert()` supports taking native coin.

[evm/GatewayEVM.sol#L111-L113](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/evm/GatewayEVM.sol#L111-L113)
```solidity
    function executeRevert( ... ) ... {
        if (destination == address(0)) revert ZeroAddress();
1:      (bool success,) = destination.call{ value: msg.value }("");
        if (!success) revert ExecutionFailed();
2:      Revertable(destination).onRevert(revertContext);

        emit Reverted(destination, address(0), msg.value, data, revertContext);
    }
```

So this will end up being unable to recover the Zeta token in case of a revert occur when transferring it from `Zeta` to `EVM` supporting `onRevert()` hook.

Now, ZetaClient it has nothing to do, because of two reasons.
1. Client can't simply send native tokens to recipient and call `onRevert()` as this is not an atomic transaction.
2. The `onRevert()` hook is Universal Contracts is not accessable to anyone. These hooks should only be callable by the `    GatewayZEVM`, for authorization. and GatewayZEVM will not be able to handle that case.

### Proof of Concept
- UserA has a Universal app on ZetaChain and Wants to send some ZetaTokens to another connected `EVM`.
- He fired the transaction using wrappedZeta token.
- GatewayZEVM gets called with `withdraw()` Zeta, and we sends native Zetacoin to the `Fungible Address`.
- The tx event is noticed by ZetaClients and they process `depositAndCall` on `EVM` chain.
- the function reverted.
- Preparing calling `depositAndRevert()` on the source chain `ZEVM`.
- The function will revert as ZetaToken is not implementing `deposit(address,uint256)` interface.
- No way to recover wrapped Zeta or Sending native Zetacoin, then calling `onRevert()`  in one transaction.
- Loss of funds to that User

### Recommendations
Supporting recovering wrapped Zeta by accepting native coin (Zeta) as Payable function, wrap them, then transfer to the recipient before executing the revert.

```diff
diff --git a/v2/contracts/zevm/GatewayZEVM.sol b/v2/contracts/zevm/GatewayZEVM.sol
index d51ec41..6ef1ff1 100644
--- a/v2/contracts/zevm/GatewayZEVM.sol
+++ b/v2/contracts/zevm/GatewayZEVM.sol
@@ -352,9 +352,14 @@ contract GatewayZEVM is
     /// @notice Revert a user-specified contract on ZEVM.
     /// @param target The target contract to call.
     /// @param revertContext Revert context to pass to onRevert.
-    function executeRevert(address target, RevertContext calldata revertContext) external onlyFungible whenNotPaused {
+    function executeRevert(address target, RevertContext calldata revertContext) external payable onlyFungible whenNotPaused {
         if (target == address(0)) revert ZeroAddress();
 
+        if (msg.value > 0) {
+            IWETH9(zetaToken).deposit{value: msg.value}();
+            IWETH9(zetaToken).transfer(to, msg.value);
+        }
+
         UniversalContract(target).onRevert(revertContext);
     }
```

_NOTE: in the case of `GatewayEVM` depositing native is done by depositing native coin, so the recover process is by sending native to the recipien. But in case of `GatewayZEVM` users don't deposit native, they are depositing wrapped Zeta, and ZEVM handles wrapping process, so We found it better to make the mitigation by recovering the funds by sending wrapped Zeta instead of Native token, as the Universal App can't pay wrapped Zeta and receive Native in recovery_

---

## [M-06] A malicious user can cause A `DoS` ZetaChain clients by spamming `EVM` inbound calls

### Description
In the current design, users do not pay gas for ZetaChain Outbound tx when they make `EVM -> Zeta` transfer.

The problem is that ZetaClients listen to all this requests, and know this if `Called()` event is fired on that source blockchain so that it can process it and make the outbound tx required.

When sending a message we are using `GatewayEVM::call()`, and it actually do nothing rather than emitting an event.

[GatewayEVM.sol#L306-L317C6](https://github.com/zeta-chain/protocol-contracts/blob/main/v2/contracts/evm/GatewayEVM.sol#L306-L317C6)
```solidity
    function call( ... ) ... {
        if (receiver == address(0)) revert ZeroAddress();
        emit Called(msg.sender, receiver, payload, revertOptions);
    }
```

when we calculated the gas it consumes it results in `18480`. which is too low amount, not a lot.

We will use Polygon, which is the cheapest chain to calculate how much it costs in USD to make a `call()`.

Cost (USD) = gas used × Average gas Price × Native token USD price × 10<sup>-9</sup>

We will put Average gas Price = `30 gwei`, and matic price is `0.6$`

Cost (USD) = 18480 × 30 × 0.6 × 10<sup>-9</sup> = `0.00033264$`

We only need to spend `0.00033$` to make one inbound `Polygon -> Zeta` message. and ZetaClients will process it.

Now what if the user spammed txs and made like `10_000` call, this will result in ZetaClients processing `10K` transactions in the queue, to be able to complete which will affect network speed.

We should keep in mind that whatever the speed ZetaChain is processing transactions, it is like `many to one` relation. Where all EVM chains in addition to Solana and BTC make inbound and it should handle it, besides outbound tx (Zeta -> EVM). So it does alot of work by default. and having such congestion with `10K` message to be processed and getting called will make the network speed go down.

These `10K` calls will only cost the attacker `3.3$` dollars in Polygon for it to take place. Now imagine the Attackr sends `1 Million` call.

### Attack analysis
Since Polygon is a Blockchain with a block gas limit, The attacker can't simply process `1 Million` call in a second.

let's say the Gas for each call is `20K` instead of `18480`, So the maximum number of calls will be:

30 Million (Block gas limit) / 20K (gas for single call) = `1500` call/block.

The user can call the Gateway `1500` times in a single block, and since the block is created every `2` seconds in Polygon, we will end up with these results. _Note: single call costs `0.00033$`_

|Number of Calls|Num of Blocks|Time to complete (seconds)|Cost (USD)|
|:--------------|:------------|:-------------------------|:---------|
|1,500|1|2|0.495$|
|10,000|7|14|3.3$|
|100,000|67|134 (~2 min)|33$|
|1,000,000|667| 1334 (~22 min)|330$|
|10,000,000|6667|13334 (~3.7 hour)|3,300$|

- The Attacker will be able to force ZetaChain to process `100,000` request in only 2 minutes paying only `33$`.

- The Attacker can make `10` million requests within `3.7` hours, i.e the ZetaClient should be able to process `2.7` million request in an Hour to not get DoS'ed in the case of the last scenario.

This will result in DoS of other users using the network as Zetachain as real transactions are in the queue waiting. and just spam message requests are the current that is getting handled by the network.

### Recommendations
Take a certain fee when making `EVM -> Zeta` calls, even if a small amount, this will make the attack cost a lot for the attacker.
