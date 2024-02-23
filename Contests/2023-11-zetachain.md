# ZetaChain
ZetaChain contest || Layer1, Cross Chain, Omnichain || 20 Nov 2023 to 18 Dec 2023 on [code4rena](https://code4rena.com/audits/2023-11-zetachain)

## My Findings Summary

|ID|Title|Severity|
|:-:|-----|:------:|
|[H-01](#h-01-zeta-observer-nodes-are-not-listening-to-internal-txs-which-makes-smart-contract-wallets-users-funds-locked-when-making-omnichain-calls)|Zeta Observer nodes are not listening to `internal TXs`, which makes Smart Contract Wallets users' funds locked when making `Omnichain calls`|HIGH|
||||
|[L-01](#l-01-zetachain-allows-cctx-from-the-chain-to-itself)|ZetaChain allows CCTX from the chain to itself|LOW|
|[L-02](#l-02-unchecking-for-cctx-messaging-txs-onzetarevert-leads-to-loss-of-users-funds-in-reverting)|Unchecking for CCTX messaging TXs `onZetaRevert` leads to loss of users' funds in reverting|LOW|
|[L-03](#l-03-go-grpc-queries-version-incompatibility-with-gogoproto-may-cause-grpc-queries-to-panic)|`go-gRPC` queries version incompatibility with `gogoproto` may cause gRPC queries to panic|LOW|
|[L-04](#l-04-firing-depositzrc20andcallcontract-after-doing-the-oncrosschaincall-can-lead-to-read-only-reentrancy-for-application-listening-to-the-blockchain-events)|Firing `DepositZRC20AndCallContract` after doing the `onCrossChainCall` can lead to read-only reentrancy for application listening to the Blockchain events|LOW|
|[L-05](#l-05-unchecking-for-address0-when-creating-addresses-in-evmtoolsimmutablecreate2factorysafecreate2internal)|Unchecking for address(0) when creating addresses in `evm/tools/ImmutableCreate2Factory::safeCreate2Internal`|LOW|
|[L-06](#l-06-hardcoding-the-dexs-deadline-instead-of-making-it-controllable-by-users-is-not-a-good-practice)|Hardcoding the DEXs `deadline` instead of making it controllable by users is not a good practice|LOW|
|[L-07](#l-07-possible-event-reordering-via-reentrancy-in-zetatokenconsumeruniv2getethfromzeta)|Possible Event Reordering via Reentrancy in `ZetaTokenConsumerUniV2::getEthFromZeta()`|LOW|
|[L-08](#l-08-decreasing-the-zrc20-allowance-even-in-case-typeuint256max-in-zevmzrc20transferfrom)|Decreasing the `ZRC20` allowance even in case type(uint256).max in `zevm/ZRC20::transferFrom()`|LOW|
|[L-09](#l-09-zrc20transferfrom-transfers-tokens-before-spending-allowance-so-it-doesnt-follow-the-cei-pattern)|`ZRC20/transferFrom()` transfers tokens before spending allowance, so it doesn't follow the CEI pattern|LOW|
|[L-10](#l-10-burning-zrc20-functionality-is-available-to-anyone)|Burning ZRC20 functionality is available to anyone|LOW|
|[L-11](#l-11-using-transfer-to-send-native-coins-eth-instead-of-call-in-zevmwzetawithdraw)|using `transfer()` to send native coins (ETH) instead of `call` in `zevm/WZETA::withdraw()`|LOW|
||||
|[NC&#8209;01](#nc-1-completing-the-incompleted-parts-of-the-code-after-the-context-makes-some-parts-not-audited-when-adding)|Completing the incompleted parts of the code after the context makes some parts not audited when adding|INFO|
|[NC-02](#nc-2-contracts-that-will-be-inherited-and-will-not-be-used-alone-should-be-marked-as-abstract)|Contracts that will be inherited, and will not be used alone should be marked as abstract|INFO|
|[NC-03](#nc-3-doubling-events-triggering-to-represent-the-same-state-change)|Doubling events triggering to represent the same state change|INFO|
|[NC-04](#nc-4-completing-the-incompleted-parts-of-the-code-after-the-context-makes-some-parts-not-audited-when-adding)|Completing the incompleted parts of the code after the context makes some parts not audited when adding|INFO|
|[NC-05](#nc-5-repetable-checks-should-be-implemented-as-a-modifier)|Repetable checks should be implemented as a modifier|INFO|
|[NC-06](#nc-6-some-if-conditions-are-repeated-giving-the-same-result-on-pass)|Some if conditions are repeated giving the same result on pass|INFO|
|[NC-07](#nc-7-the-name-of-the-isystem-interface-does-not-express-its-functionality)|The name of the `ISystem` interface does not express its functionality|INFO|
|[NC-08](#nc-8-unchanged-variables-should-be-marked-as-constant)|Unchanged variables should be marked as `constant`|INFO|
|[NC-09](#nc-9-missing-readable-message-error-in-zevmwzetasol)|Missing readable message error in `zevm/WZETA.sol`|INFO|
|[NC-10](#nc-10-legacy-version-of-solidity-and-floating-pragma)|Legacy version of solidity and floating pragma|INFO|

---

## [H-01] Zeta Observer nodes are not listening to `internal TXs`, which makes Smart Contract Wallets users' funds locked when making `Omnichain calls`

### Lines of code

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_client.go#L952

### Vulnerability details


#### Impact

Zetachain allows omnichain TXs (sending funds from external chains to Zeta EVM chain), using different methods.
- If it's just sending main blockchain coin funds between addresses: You deposit funds directly to the `TSS_ADDRESS` and it will send money to the destination address as ZRC-20 tokens on Zeta EVM chain.
- If you are sending ERC20: You need to use `ERC20Custody::deposit()`.

The problem occurs in the first sending method. When the user sends funds (native chain coins) to the `TSS_ADDRESS`.

To define the receiver address you have two things to do:
- provide it in the data field of the TX in bytes format (+ any additional message if needed), and Observer nodes take out the rest of the job to decode.
- Not providing data field in the TX, and in that case, Observers use the caller itself as the receiver address on the Zeta blockchain.

The problem affects Omnichain (Inbound TXs, external EVM => Zeta EVM) which is made by smart contract wallet users.

When smart contract wallets make a sending request, it doesn't make an RPC call. They are making low-level `call` to transfer funds. And here is where the problem occurs.

Observers will not notice this transaction, funds will be sent to the `TSS_ADDRESS` and the Observers will not know that there is an Omnichain call has happened. So user funds will be locked in the `TSS_ADDRESS`.

**Why Smart Contract Wallets are important?**
We should keep in mind that Smart Contract wallets are heavily adopted, and The problem does not affect a small group of users. Mult-sig wallets are increasing, and Account Abstraction is coming, Users will deal with Smart Contract Wallets in the near future instead of EOA.

This is dangerous for users, as most of the users are using their wallets using a UI, and some popular wallets like MetaMask will support account abstraction soon, which will make users' wallets a Smart Contract Wallet as we illustrated before. So users will not be able to interact with ZetaChain and will lose their funds without knowing what is going wrong.


#### Proof of Concept

> In our auditing process of the protocol, we made a lot of things (installing deps, making contracts, writing deployments and interact scripts, etc...), So It will be hard for the judger to set up the development environment.
> We worked on the second part `protocol-contract`, and used zeta_testnet for our testing purposes.
> Here is the Dropbox link to download the `protocol-contract` we worked on, you will find `setup.md` to help you install deps, and run POC scripts easily without any problems.

Downloading link: https://www.dropbox.com/scl/fo/b078exeuugkyb72tcs7a0/h?rlkey=mzi4pjja4cbbnyc58yz6e3fns&dl=0

To not make the POC page too long, we will mention the main points we made to prove the problem. And In the Dropbox folder, everything including (testing script, contracts, etc...) exists with comments and a lot of logs to be easily understood.

After installing the `protocol-contracts` from the Dropbox, you can simply write this command in the console:
```
yarn run audit-H1:internal-to-address
```

What this script does, is what we are going to illustrate below.

At first, we made a simple contract that represented a smart contract wallet, sending and receiving coins.

<details>
  <summary>Contract</summary>
  
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.7;

/// @title Smart Contract Waller
/// @author Code4rena Warden
/// @notice This is not a complete wallet, it is just for demonistrating the vulnerability
contract InternalWallet {
   // to be able to receive ETH
   receive() external payable {}

   // Sending funds to `TSS_ADDRESS` to receive it on zetachain account
   // NOTE: this function should fail, and the funds will be locked in the `TSS_ADDRESS` when sending
   function transferOmnichain(uint256 amount, address tssAddress, bytes memory to) public payable returns (bool) {
       require(amount <= address(this).balance, "InternalWallet: not enough suffiecent");
       (bool success, ) = tssAddress.call{value: amount}(to);
       require(success, "InternalWallet: failed to make Omnichain call");
       return true;
   }

   // Withdraw funds after completeing testing
   function withdraw(address to) public returns (bool) {
       (bool success, ) = to.call{value: address(this).balance}("");
       require(success, "InternalWallet: failed to withdraw funds");
       return true;
   }
}
```
</details>

Lastly, we fired `transferOmnichain` with the following parameters:
-  `amount`: 0.1 tMATIC.
-  `tssAddress`: TSS_ADDRESS on mumbai_testnet (0x8531a5aB847ff5B22D855633C25ED1DA3255247e).
-  `to`: Our EOA on ZetaChain (0x0a485F234D49F28b688495d071D164E7dB0cBd9A).

![image](https://res.cloudinary.com/availablecoder/image/upload/v1702549728/code4rena/zetachain/rvkdxj8h8galiwgskmzj.png)

ZetaClient Observers should listen to the transaction, then send ZRC-20 tMATIC to the `to` address. But this did not occur.

This problem happened because ZetaClient Observers handles sending to `TSS_ADDRESS` if it is the `to` parameter of the PRC call, and if the RPC calls contain an internal transactions array, it did not check them.

- Affected Function: [node/zetaclient/evm_client.go#L774-L993C2](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_client.go#L774-L993C2)
- Affected line: [node/zetaclient/evm_client.go#L952](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_client.go#L952)
```go
func (ob *EVMChainClient) observeInTX() error {
   ...

   //task 1:  Query evm chain for zeta sent logs
   func() { ... }()

   // task 2: Query evm chain for deposited logs
   func() { ... }()

   // task 3: query the incoming tx to TSS address ==============
   func() {
       tssAddress := ob.Tss.EVMAddress() // after keygen, ob.Tss.pubkey will be updated
       if tssAddress == (ethcommon.Address{}) {
           ob.logger.ExternalChainWatcher.Warn().Msgf("observeInTx: TSS address not set")
           return
       }

       // query incoming gas asset
       for bn := startBlock; bn <= toBlock; bn++ {
           ...

           for _, tx := range block.Transactions() {
               ...

               // Checking for tx.to only not including internal transactions for each transaction
               if *tx.To() == tssAddress { ... }
           }
       }
   }()
...
}
```

Since smart contract wallets can't trigger the transfer directly, all sending from Smart Contract wallets (Multi-sig, Account Abstraction, ...) will fail.

(EOA => Smart Contract Wallet address => TSS_ADDRESS). The `to` will be the contract wallet address, not the `TSS_ADDRESS`.

And since the Observers didn't even listen to the TX (the transaction sent by the smart contract wallet), they will not revert, they will simply do nothing. like there is no one sending money. This will lead to the loss of all funds sent to the `TSS_ADDRESS` from Smart Contract Wallets in external chains when making Omnichain InBound TXs calls.

Here is one of the TXs we made:
https://mumbai.polygonscan.com/tx/0x1d919d43b255fd8a2645dfd6159153518b7f1e58798037ea91a1e921b0b7d69d

We can see that the transaction `to` parameter is the Smart Contract Wallet, So the Observers did not catch this TX.
![image](https://res.cloudinary.com/availablecoder/image/upload/v1702551048/code4rena/zetachain/uyx1f7oua9cycxdcisok.png)

## Tools Used
Manual Review, Hardhat, Polygon Mumbai explorer, and ZetaChain explorer

## Recommended Mitigation Steps

We should track the internal TXs for each RPC transaction. and if the `to` of one of the internal TXs equals the `TSS_ADDRESS` Observers should handle this transaction too (Observers handle it as an InBound Omnichain call).

Although the solution may be simple two things will be challenging when implementing this.

1. How can we decode data if it is an internal Tx?
2. Listening to each TX internal TXs may slow down monitoring TXs as the observer nodes will be performing a lot of computation.

### How can we decode data if it is an internal Tx?

In normal TXs to the `TSS_ADDRESS` you are providing the address in the first 20 bytes of the message, and if you want to make external logic on the ZetaChain you are providing them after the address in bytes.

Output message: (20 bytes: receiver address in ZetaChain)(bytes: the message that will be pathed to `conCrossChainCall`).

But In the case of internal TXs things differ, as the function call we fired on our smart contract wallet + message passed in the low-level call to the `TSS_ADDRESS` both exists in the output message.

Here is the output message of the TX I made in testing:
![image](https://res.cloudinary.com/availablecoder/image/upload/v1702555598/code4rena/zetachain/fgvkerspgcgmbo3t2qye.png)

We can write `cast pretty-calldata <OUTPUT_MESSAGE>` to decode the message, and know what it contained.

```
Method: 4696616c // transferOmnichain(uint256,address,bytes)
------------
[000]: 000000000000000000000000000000000000000000000000016345785d8a0000 // amount sent
[020]: 0000000000000000000000008531a5ab847ff5b22d855633c25ed1da3255247e // tssAddress
[040]: 0000000000000000000000000000000000000000000000000000000000000060 // The location of the bytes array
[060]: 0000000000000000000000000000000000000000000000000000000000000014 // bytes array length hex(14) = 20 bytes
[080]: 0a485f234d49f28b688495d071d164e7db0cbd9a000000000000000000000000 // The bytes array data (receiver address on ZetaChain)
```

If we go and see how Geth is logging for this transaction, we will find it has an array of `calls` that has one call that represents the internal transaction information.

https://mumbai.polygonscan.com/vmtrace?txhash=0x1d919d43b255fd8a2645dfd6159153518b7f1e58798037ea91a1e921b0b7d69d&type=gethtrace2#raw

![image](https://res.cloudinary.com/availablecoder/image/upload/v1702555971/code4rena/zetachain/nn2t6gzbnmgakrcmpvag.png)

So it will be easy for Observer nodes to extract the data sent to the `TSS_ADDRESS` regarding the data that represents overall transaction output data.

### Listening to each TX internal TXs may slow down monitoring TXs as the observer nodes will be performing a lot of computation.

Listening to all transactions, checking for internal transactions for every transaction, and scanning them may cause a lack of network speed (Observers nodes). We are aware of this problem, but we don't exactly know how much performance and speed we will lose when implementing listening to the internal TXs.

The protocol devs are the only one who can determine how much speed the network lose when implementing the internal TXs listening, but In the case of a significant loss in speed, I will mention some things that will help solve the problem if these problems arise after implementing the Mitigation.

1. The type of the transaction is `CALL`, when we do it directly (RPC), or indirectly (Smart Contract Call). We can only pick the internal TXs that go to the `to` address of type `CALL`.

2. We can separate the Observers' jobs, instead of Querying the three types of In TXs that should be tracked by each Observe, each Observer node is responsible for tracking one of the following EVM state changes:
  1. Query evm chain for zeta sent logs
  2. Query evm chain for deposited logs
  3. Query the incoming tx to TSS address

So we are separating the work into three, but this may be very hard to implement especially for the `emission` module, how can it distribute rewards?

The last solution may be useless, and unable to be implemented in real. However, I preferred to mention everything that can help in solving the problem of lack of network speed when implementing listening to the internal TXs.

### Assessed type

ETH-Transfer

---
---
---

## [L-01] ZetaChain allows CCTX from the chain to itself
The implementation of the CCTX logic doesn't check if the destination chain is the same as the source chain. This is not a good practice, as it is unnecessary and users will pay a lot of gas in case of wrong TXs.

[ZetaConnector.eth.sol#L31-L45](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ZetaConnector.eth.sol#L31-L45)
```solidity
function send(ZetaInterfaces.SendInput calldata input) external override whenNotPaused {
    bool success = IERC20(zetaToken).transferFrom(msg.sender, address(this), input.zetaValueAndGas);
    if (!success) revert ZetaTransferError();

    // No Check if (block.chainid == input.destinationChainId)
    emit ZetaSent( ... );
}
```

[ZetaConnector.non-eth.sol#L40-L53](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ZetaConnector.non-eth.sol#L40-L53)
```solidity
function send(ZetaInterfaces.SendInput calldata input) external override whenNotPaused {
    ZetaNonEthInterface(zetaToken).burnFrom(msg.sender, input.zetaValueAndGas);

    // No Check if (block.chainid == input.destinationChainId)   
    emit ZetaSent( ... );
}
```

When making the destination chain equal to the source chain, the nodes will handle it as a normal CCTX. and there is no check for the destination chain not to be the same as the source chain in the Node logic.

### Recommendation
I recommend providing a check in the connectors contracts (ZetaConnector.non-eth.sol && ZetaConnector.eth.sol) to revert the transaction if the `chainId` is the same as the destination chain.

---

## [L-02] Unchecking for CCTX messaging TXs `onZetaRevert` leads to loss of users' funds in reverting

In cross-chain messaging tx (sending a message from one chain to another). If the message parameter > 0, Zetaclient fires `onZetaMessage` in the destination address (on the destination chain). And if this message reverts, the nodes fire `onZetaRevert` in the sender contract (on the source chain). But if the `onZetaRevert` reverts on the sender contract, Zeta nodes don't do anything, which leads to the loss of funds (sender).

[ZetaConnector.eth.sol#L52-L70](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ZetaConnector.eth.sol#L52-L70)
```solidity
function onRevert( ... ) external override whenNotPaused onlyTssAddress {
    bool success = IERC20(zetaToken).transfer(zetaTxSenderAddress, remainingZetaValue);
    if (!success) revert ZetaTransferError();

    if (message.length > 0) {
        // If this gets reverted, the caller will not gain his remaining ZETA
        ZetaReceiver(zetaTxSenderAddress).onZetaRevert(
            ZetaInterfaces.ZetaRevert( ... )
        );
    }

    emit ZetaReverted( ... );
}
```

[ZetaConnector.non-eth.sol#L87-L122](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ZetaConnector.non-eth.sol#L87-L122)
```solidity
function onRevert( ... ) external override whenNotPaused onlyTssAddress {
    if (remainingZetaValue + ZetaNonEthInterface(zetaToken).totalSupply() > maxSupply)
        revert ExceedsMaxSupply(maxSupply);
    ZetaNonEthInterface(zetaToken).mint(zetaTxSenderAddress, remainingZetaValue, internalSendHash);

    if (message.length > 0) {
        // If this gets reverted, the caller will not gain his remaining ZETA
        ZetaReceiver(zetaTxSenderAddress).onZetaRevert(
            ZetaInterfaces.ZetaRevert( ...  )
        );
    }

    emit ZetaReverted( ... );
}
```

When the transaction gets reverted, we are minting/transferring (transferring in Ethereum only and minting in other EVMs, this is the devs' design decision) the remaining ZETA paid by the user (after taking gas fees and protocol fees) goes to the sender address on the source chain. But if for some reason the `onZetaRevert` gets reverted like out of gas, not implementing the interface, or anything else. The sender will lose his funds (remaining ZETA).

### POC
- Sending CCTX (Polygon => BSC) with 5 ZETA -> Total fees are 1 ZETA -> destination address will receive this 4 ZETA on BSC (on destination chain).
- Sending CCTX (Polygon => BSC) with 5 ZETA -> Total fees are 1 ZETA -> there is a message -> firing `onZetaMessage` -> TX reverted -> firing `onZetaRevert` -> sender address receives 4 ZETA (on source chain).
- Sending CCTX (Polygon => BSC) with 5 ZETA -> Total fees are 1 ZETA -> there is a message -> firing `onZetaMessage` -> TX reverted -> firing `onZetaRevert` -> TX reverted -> the sender loses his 4 ZETA (the remaining of ZETA after making the TX).

When the `onZetaRevert` function gets reverted, the money sent to the sender (the remaining ZETA) will get reverted too. and the caller will not gain his remaining ZETA.

### Recommendation
We recommend checking for the `onZetaRevert` using try/catch, and if it gets reverted we can simply do nothing or not emit `ZetaReverted`.

This will make the caller receive his funds even in case of a revert of `onZetaRevert`, as the refunding transaction will not get reverted.

---

## [L-03] `go-gRPC` queries version incompatibility with `gogoproto` may cause gRPC queries to panic.

ZetaChain uses cosmos SDK v4.6

> eeshenggoh: What's the cosmos sdk version that's to be used?
>
> Lucas | ZetaChain: 0.46

https://discord.com/channels/810916927919620096/1167486056283779172/1177523976508022814

In cosmos v0.46 docs, an issue was introduced in `go-grpc` queries that caused gRPC queries panic. And they provide that the `go-grpc` is **highly recommended** to be downgraded to v1.33.2.

The issue starts from grpc >= v1.33.2, according to what the docs say.

In new cosmos versions (v0.50), the issue seems to be solved and they are using the latest version of the `go-grpc`.
https://github.com/cosmos/cosmos-sdk/blob/main/go.mod#L61

Although, there is no currently a visible problem occurring from the version of grpc ZetaChain is using. We must be sure to provide the correct version according to the docs.

The version ZetaChain is using:
```go
// FILE: node/go.mod
25:     google.golang.org/grpc v1.55.0
```

### POC

1. Visit: https://docs.cosmos.network/v0.46/basics/
2. click on Anatomy of a Cosmos SDK Application
3. On the right navbar click Dependencies and Makefile.

The documentation describes the incompatibility issue between `gogoproto` and `go-grpc`.

And here are the GitHub discussions about the issue.
- https://github.com/cosmos/cosmos-sdk/issues/8426
- https://github.com/cosmos/cosmos-sdk/issues/8392
- https://github.com/cosmos/cosmos-sdk/issues/8426#issuecomment-783280919

### Recommendations

The better solution is to be up to date with the cosmos SDK version (0.50). But since it will take a lot of time and work, we can simply downgrade the version of the `go-grpc` to prevent any incompatibility issues.

```diff
// FILE: repos/node/go.mod
- google.golang.org/grpc v1.55.0
+ google.golang.org/grpc v1.33.2
```

---

## [L-04] Firing `DepositZRC20AndCallContract` after doing the `onCrossChainCall` can lead to read-only reentrancy for application listening to the Blockchain events

When there is a deposit function (Omnichain call) is observed, `crosschain:keeper:fungile_keeper:ZRC20DepositAndCallContract()` [[1](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/evm_deposit.go#L59)] is fired by the nodes if the caller is a contract. Then, it calls `fungile:keeper:DepositZRC20AndCallContract()` [[2](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/fungible/keeper/deposits.go#L75)], which calls `SystemContract::depositAndCall()` [[3](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L67-L79)].

Then, `onCrossChainCall` is called on the target contract.

Now if the `onCrossChainCall` made a withdrawal request (withdrawing ZRC-20 tokens). This is necessary for making some applications like swaps, and cross-chain NFTs, and ZetaChain used this behavior in making a swap example in the docs.

After the nodes handles this transaction on ZetaChain, they fire `crosschain:keeper:ProcessLogs()` [[4](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/evm_deposit.go#L88)], which checks for emitting events on the ZetaChain.[[5](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/evm_hooks.go#L93-L106)].

If there is a `Withdrawal` event, it will be processed by the nodes `crosschain:keeper:ProcessZRC20WithdrawalEvent()`, and it will emit the Withdrawal event on the blockchain level (ZetaChain) [[6](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/evm_hooks.go#L168)].

Lastly, in the last of the function, Zetanodes fires `DepositZRC20AndCallContract` event [[7](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/evm_deposit.go#L93-L101)]

So If some applications built either Web3 or Web2, which depends on the state of the accounts changing in ZetaChain using events, will read the `EmitZRCWithdrawCreated` event before emiting `DepositZRC20AndCallContract`.


### Recommendations
I recommend firing the Deposit event before processing Logs, so if there is a withdrawal event the events will be read that the user depositing first then withdrew, instead of reading events as Withdrawal then deposited.

---

## [L-05] Unchecking for address(0) when creating addresses in `evm/tools/ImmutableCreate2Factory::safeCreate2Internal`

When we deploy the addresses using `create2`, It is better to make sure that the address we get is not the ZERO_ADDRESS. This check helps ensure that the contract creation was successful and that the deployed contract has a valid address. And implementing the best practices is always better.

[evm/tools/ImmutableCreate2Factory.sol#L52-L64](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/tools/ImmutableCreate2Factory.sol#L52-L64)

```solidity
FILE: evm/tools/ImmutableCreate2Factory::safeCreate2Internal > assembly
52:    // using inline assembly: load data and length of data, then call CREATE2.
53:    assembly {
54:        // solhint-disable-line
55:        let encoded_data := add(0x20, initCode) // load initialization code.
56:        let encoded_size := mload(initCode) // load the init code's length.
57:        deploymentAddress := create2(
58:            // call CREATE2 with 4 arguments.
59:            callvalue, // forward any attached value.
60:            encoded_data, // pass in initialization code.
61:            encoded_size, // pass in init code's length.
62:            salt // pass in the salt value.
63:        )
64:    }
```

### Recommendations
Adding check for address(0) result, and revert if the created address is the zero address.

```diff
assembly {
// solhint-disable-line
let encoded_data := add(0x20, initCode) // load initialization code.
let encoded_size := mload(initCode) // load the init code's length.
deploymentAddress := create2(
    // call CREATE2 with 4 arguments.
    callvalue, // forward any attached value.
    encoded_data, // pass in initialization code.
    encoded_size, // pass in init code's length.
    salt // pass in the salt value.
)

+   if iszero(extcodesize(deploymentAddress)) {
+       revert(0, 0)
+   }

}
```

---

## [L-06] Hardcoding the DEXs `deadline` instead of making it controllable by users is not a good practice

In DEX, `deadline` is used to revert the TX if it has been waiting for a long time. However, in the consumer contract, which is used to swap between ZETA and tokens/coins, we hardcoded the deadline to be `block.timestamp + 200`.

Since the transaction will be in the same block, which means the block.timestamp will be the same, the check will always pass by the DEXs. We are preventing users from setting the maximum time of latency their swap should not exceed.

[evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L19](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L19)
[evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L49](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L49)

```solidity
uint256 internal constant MAX_DEADLINE = 200;
...

function getZetaFromEth( ... ) external payable override returns (uint256) {
    ...

    uint256[] memory amounts = uniswapV2Router.swapExactETHForTokens{value: msg.value}(
        minAmountOut,
        path,
        destinationAddress,
        block.timestamp + MAX_DEADLINE
    );
```

This affects all swap functions in all the consumer contracts (there are 4 consumer contracts, and each consumer has 4 swapping functions) i.e. 16 total instances.

### Recommendations
I recommend adding a parameter for the `deadline`, where users will have the freedom to set the maximum delay that the transaction can bear.

Besides this, we can add a placeholder (default value) if this parameter is not set by users.

---

## [L-07] Possible Event Reordering via Reentrancy in `ZetaTokenConsumerUniV2::getEthFromZeta()`

In past reports, there is an issue that explains that the order of the event `ZetaExchangedForEth` can be manipulated. This could disrupt any application relying on the consistent order of events (in `evm/tools/ZetaTokenConsumerUniV3.strategy.sol`).

> Category of Vulnerability - Verdise July 2022 - Deficient Smart Contract
>
> Possible Event Reordering via Reentrancy in getEthFromZeta
>
> Relevant Contracts: ZetaTokenConsumerUniV3.strategy.sol

[https://github.com/code-423n4/alpha-zetachain/blob/main/BlockChomper-0xladboy-reentrant/Collection_%26_Classes_of_Vulnerabilitiesc.md](https://github.com/code-423n4/alpha-zetachain/blob/main/BlockChomper-0xladboy-reentrant/Collection_%26_Classes_of_Vulnerabilitiesc.md)

We are aware that the past issues are not acceptable, but the problem mentioned in this report says it affects UniV3 consumers only, and the Zeta dev team solved it. However, this problem can also happen in UniV2 consumer but the report did not mention it, and it is not solved. So we mentioned it here.

When I examined ZetaClient, I found that the ZetaClient is not listening to the `ZetaExchangedForEth` event, so I couldn't point out how this was a problem in the past report. So I preferred to mention it as LOW, as I couldn't find any issue that will occur when doing such a thing.

### Issue Description

In UniswapRouterV2, when exchanging native ETH, for an amount of tokens. the transfer of ether occurs inside the router itself, so the re-entrance problem could occur since the low-level `call` is done before emitting.

[evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L95-L122](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L95-L122)

```solidity
function getEthFromZeta( ... ) external override returns (uint256) {
    ...

    uint256[] memory amounts = uniswapV2Router.swapExactTokensForETH( ... );

    uint256 amountOut = amounts[path.length - 1];

    emit ZetaExchangedForEth(zetaTokenAmount, amountOut);
    return amountOut;
}
```

### POC
1. `getEthFromZeta()` fires `UniRouterV2.swapExactTokensForETH()`[[1](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L284-L300)]
2. Which fires `_swap()`, and swap our tokens (ZETA => WETH)
3. Then, we are sending native ETH coin to the swapper instead of ETH using `TransferHelper.safeTransferETH` [[2](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L299)]
4. ETH is sent to the swapper [[3](https://github.com/Uniswap/solidity-lib/blob/master/contracts/libraries/TransferHelper.sol#L47-L50)]
5. The re-entrance can occur at this point since we are making low-level `call`

### Recommendations
The best solution in my opinion is to make a `nonReentrant` modifier, this will make sure that the swapper can not re-enter `ZetaTokenConsumerUniV2::getEthFromZeta()` again.

---

## [L-08] Decreasing the `ZRC20` allowance even in case type(uint256).max in `zevm/ZRC20::transferFrom()`

In `zevm/ZRC20::transferFrom()`, we are decreasing the allowance for all values of the approved coins the spender can use. This is not ideal as the popular ERC20 libs like [OpenZepplein](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol#L305-L315) and [Solmate](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol#L97) don't decrease the allowance in the case of `type(uint256).max` approval.

[contracts/zevm/ZRC20.sol#L157-L166](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L157-L166)
```solidity
function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
    _transfer(sender, recipient, amount);

    uint256 currentAllowance = _allowances[sender][_msgSender()];
    if (currentAllowance < amount) revert LowAllowance();

    // spending allowance for all values of allowance even in case of `type(uint256).max`
    _approve(sender, _msgSender(), currentAllowance - amount);

    return true;
}
```

ZRC-20 is like ERC-20 in Zeta EVM, and we should make sure that they implement the same logic, for the common functions like `transferFrom` and `approve`.

### Recommendations
Adding a check to not spend the tokens if the allowance is set to `type(uint256).max`.

```diff
// FILE: zevm/ZRC20::transferFrom()
function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
_transfer(sender, recipient, amount);

uint256 currentAllowance = _allowances[sender][_msgSender()];
if (currentAllowance < amount) revert LowAllowance();

-        _approve(sender, _msgSender(), currentAllowance - amount);
+        if (currentAllowance != type(uint256).max) _approve(sender, _msgSender(), currentAllowance - amount);

return true;
}
```

---

## [L-09] `ZRC20/transferFrom()` transfers tokens before spending allowance, so it doesn't follow the CEI pattern

In `ZRC20/transferFrom()`, we are sending tokens from one account to another like any ERC20 `transferFrom()` function. But In the current implementation of the function, we are:

1. transferring the funds (Interaction)
2. Check the allowance (Check)
3. Spend the allowance (Effect)

[zevm/ZRC20.sol#L157-L166](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L157-L166)
```solidity
function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
    _transfer(sender, recipient, amount); // transferring in first

    uint256 currentAllowance = _allowances[sender][_msgSender()];
    if (currentAllowance < amount) revert LowAllowance(); // checking for allowance in second

    _approve(sender, _msgSender(), currentAllowance - amount); // spending allowance in the last step

    return true;
}
```

It is better to follow the CEI pattern when writing smart contracts, and this is the implementation that is written in popular ERC20 libs like [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol#L154-L159) and [solmate](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol#L90-L110).

In addition to this, users will pay more gas in the transaction when failing because of the insufficient allowance, as they are making the transfer logic first which consumes gas.

### Recommendation

Following the CEI pattern, by checking allowance, spending allowance, and transferring in the last step.

```diff
function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
-        _transfer(sender, recipient, amount);

    uint256 currentAllowance = _allowances[sender][_msgSender()];
    if (currentAllowance < amount) revert LowAllowance();

    _approve(sender, _msgSender(), currentAllowance - amount);

+        _transfer(sender, recipient, amount);

    return true;
}
```

---

## [L-10] Burning ZRC20 functionality is available to anyone
When making Omnichain (outBound TXs) i.e. (ZETA => External EVM). users burn their ZRC20 tokens like ETH ZRC20, and the ZetaClient will send them the native ETH in the Ethereum blockchain.

This is why users can burn their tokens. this functionality is implemented in the `ZRC20::withdraw()` function.

[zevm/ZRC20.sol#L256-L264](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L256-L264)
```solidity
function withdraw(bytes memory to, uint256 amount) external override returns (bool) {
    ...

    _burn(msg.sender, amount); // burning ZRC20 tokens, before sending them to the user on external chain
    emit Withdrawal(msg.sender, to, amount, gasFee, PROTOCOL_FLAT_FEE);
    return true;
}
```

There is another public `burn` function that is available for anyone to use, which burns tokens without doing anything else. This function is used by `FUNGIBLE_MODULE_ADDRESS` and other Authorized addresses of the ZetaClient to manage tokens supply, liquidity, and other things.

[zevm/ZRC20.sol#L173-L176](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L173-L176)
```solidity
// Anyone can burn his token
function burn(uint256 amount) external returns (bool) {
    _burn(msg.sender, amount);
    return true;
}
```

This function is available for anyone to burn his tokens, which is not a good use case, nor it is implemented in the ERC20 token standard.

We are aware that some ERC20 tokens need the burning functionality, like `WZETA` in non-Ethereum-EVM chains, which burns the amount in the source chain when sending them to the Zeta EVM chain or an external chain. However, allowing anyone to burn any amount of ZRC20 tokens he has without reason is not needed.

### Recommendations
Restrict the `burn` function to be called by the Authorized addresses of the ZetaClients only or the Authorized addresses that will use this function.

---

## [L-11] using `transfer()` to send native coins (ETH) instead of `call` in `zevm/WZETA::withdraw()`

_We know that `WZETA.sol` is the same as the WETH code. but I want to illustrate that WETH code has been written for a long time when solidity was still in v0.4. So relying on the same code even in new projects is not a good choice, as it is better to use the best practices and modern coding techniques in solidity language that are changing each year._

In `zevm/WZETA::withdraw()`, we are using the solidity `transfer()` function to send ZETA coin to the user and burn his WZETA token. using `transfer()` comes with its drawbacks like limited gas forwarded (2300 gas), and unclear error message when reverting. So it's better to use the low-level `call()` function instead.

[zevm/WZETA.sol#L25-L30C6](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/WZETA.sol#L25-L30C6)
```solidity
function withdraw(uint wad) public {
    require(balanceOf[msg.sender] >= wad);
    balanceOf[msg.sender] -= wad;
    msg.sender.transfer(wad); // using transfer instead of call
    Withdrawal(msg.sender, wad);
}
```

### Recommendations
Replace `transfer` with `call`, and check for any error that occurs in `call`

```diff
function withdraw(uint wad) public {
    require(balanceOf[msg.sender] >= wad);
    balanceOf[msg.sender] -= wad;
-        msg.sender.transfer(wad);
+        (bool success,) msg.sender.call{value: wad}("");
+        require(success, "WZETA: Withdrawing failed");
    Withdrawal(msg.sender, wad);
}
```

---
---
---

## [NC-1] Completing the incompleted parts of the code after the context makes some parts not audited when adding

In the `node` folder, there are a lot of things that are not implemented, and the developers mark them as `TODO`, or `FIXME`.

These parts will be added after the Auditing context, making some parts in the code not audited.

Here are some of the things that are to be implemented: ( [1](https://github.com/search?q=repo%3Acode-423n4%2F2023-11-zetachain%20TODO%3A&type=code), [2](https://github.com/search?q=repo%3Acode-423n4%2F2023-11-zetachain+FIXME%3A&type=code) )

### Recommendations
Since the context starts before making these things, it is better to make a final auditing process before launching, or at least for the parts that have been modified.

---

## [NC-2] Contracts that will be inherited, and will not be used alone should be marked as abstract

In `evm/ZetaConnectorBase.sol`, this contract represents an interface for all Connector contracts, and All Connector contracts will inherit from it [link](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ZetaConnector.base.sol#L15).

The contracts that will not be used alone, and will be used by other contracts should be marked as `abstract`. This is exactly like class and abstract class in Java.

### Recommendations
All contracts that will not be deployed on their own like `ZetaConnectorBase` better to be marked as abstract. This will also make an error message if the devs try to deploy it on its own by mistake.

---

## [NC-3] Doubling events triggering to represent the same state change

In `evm/Zeta.eth.sol`, there are two functions to mint and burn tokens (`ZetaNonEth::mint()`, `ZetaNonEth::burnFrom()`). These two functions mint/burn tokens, and then they fire events Minted/Burnt.

These events are not necessary, as the ERC20 OpenZeppelin lib is triggering a Transfer event that represents minting and burning.

[evm/Zeta.non-eth.sol#L62-L71C6](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/Zeta.non-eth.sol#L62-L71C6)
```solidity
function mint( ... ) external override {
    ...

    _mint(mintee, value); // This emits Transfer(address(0), account, amount);

    emit Minted(mintee, value, internalSendHash);
}
```
[evm/Zeta.non-eth.sol#L73-L82](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/Zeta.non-eth.sol#L73-L82)
```solidity
function burnFrom( ... ) public override(ZetaNonEthInterface, ERC20Burnable) {
    ...

    ERC20Burnable.burnFrom(account, amount); // This emits Transfer(account, address(0), amount);

    emit Burnt(account, amount);
}
```

- Minting: `Transfer(address(0), account, amount);` [[here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.8.0/contracts/token/ERC20/ERC20.sol#L269)]
- Burning: `Transfer(account, address(0), amount);` [[here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.8.0/contracts/token/ERC20/ERC20.sol#L298)]

Minting is transferring from address(0), and burning is transferring to address(0), so emitting another event for expressing this state change is not necessary.

### Recommendations
I suggest depending on the ERC20 Transfer event and removing the Minted and Burned events

---

## [NC-4] Completing the incompleted parts of the code after the context makes some parts not audited when adding

This is the same as the NC-1 issue, we separate it as it lies on the `protocol-contracts` folder i.e. smart contract level, not at the node level. It is just an organizational act to make it easier for the judge and the sponsor to track problems.

In `evm/tools/ZetaTokenConsumerTrident`, `hasZetaLiquidity()` function logic is not implemented. So this part will not be audited when being implemented after the end of the context [link](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerTrident.strategy.sol#L203-L206).

### Recommendations
Since the context starts before making these things, it is better to make a final auditing process before launching, or at least for the parts that have been modified.

---

## [NC-5] Repetable checks should be implemented as a modifier

In `zevm/SystemContract.sol`, some function implements a restricted role of the caller to be `FUNGIBLE_MODULE_ADDRESS`.

Here are the functions that implement this check ( [1](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L74) [2](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L124) [3](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L135) [4](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L146) [5](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L157) [6](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L168) )

Another checking that is implemented more that one time is the zero address check ( [1](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L158) [2](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L169) )

Repeating the same check more than one time is not good for code readability, and it is better to make the reusability pattern as perfect as we can in the codebase.

### Recommendations
I suggest making two modifiers one for the fungible address `onlyFungible`, and the other for the zero address `nonZeroAddress`. And implement these modifiers to the function that implements these checks.

---

## [NC-6] Some if conditions are repeated giving the same result on pass

In `zevm/ZRC20.sol`, `_transfer()`, and `_approve()` functions, we are checking if the sender/owner parameter is the zero address or not. Then, check if the recipient/spender is the zero address or not ( [1](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L179-L180) [2](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L212-L213) ).

Since the result in case of passing will be the same error message (`ZeroAddress()`), there is no need to check each of them separately.

### Recommendations
Group the two if conditions for these two functions to be one if condition.

```diff
// zevm/ZRC20.sol > _transfer
function _transfer( ... ) internal virtual {
-        if (sender == address(0)) revert ZeroAddress();
-        if (recipient == address(0)) revert ZeroAddress();

+        if (sender == address(0) || recipient == address(0)) revert ZeroAddress();

    ...
}
```
```diff
// zevm/ZRC20.sol > _transfer
function _approve( ... ) internal virtual {
-        if (owner == address(0)) revert ZeroAddress();
-        if (spender == address(0)) revert ZeroAddress();

+        if (owner == address(0) || spender == address(0)) revert ZeroAddress();

    ...
}
```

---

## [NC-7] The name of the `ISystem` interface does not express its functionality

`ISystem` interface implements the variables and mappings that are used in both `SystemContract.sol` and `ZRC20.sol` [link](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/Interfaces.sol#L4-L7). However, the name suggests that it belongs to the SystemContract only.

### Recommendations
I suggest making the interface name `ISystemAndZRC20` instead of `ISystem`.

---

## [NC-8] Unchanged variables should be marked as `constant`

In `WZETA.sol`, `symbol` and `decimals` variables are declared, and their value cannot be changed. It is better to make these variables `constant`.

```solidity
// zevm/WZETA.sol
string public symbol = "WZETA";
uint8 public decimals = 18;
```

_We know that `WZETA.sol` is the same as the WETH code. but I want to illustrate that WETH code has been written for a long time when solidity was still in v0.4. So relying on the same code even in new projects is not a good choice, as it is better to use the best practices and modern coding techniques in solidity language that are changing each year._

Making these variables constant will not only make the codebase more robust and maintainable, but also it will be less costly when deploying the contract.

### Recommendations
I suggest Adding the `constant` modifier before the variable name (symbol and decimals).

```diff
// zevm/WZETA.sol
-    string public symbol = "WZETA";
-    uint8 public decimals = 18;
+    string public constant symbol = "WZETA";
+    uint8 public constant decimals = 18;
```

---

## [NC-9] Missing readable message error in `zevm/WZETA.sol`

In `WZETA.sol`, error messages `require` statments don't provide a readable reason when reverting or throwing an error when makign the transaction ( [1](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/WZETA.sol#L26) [2](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/WZETA.sol#L47) [3](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/WZETA.sol#L50) ).

_We know that `WZETA.sol` is the same as the WETH code. but I want to illustrate that the WETH code has been written for a long time when solidity was still in v0.4. So relying on the same code even in new projects is not a good choice, as it is better to use the best practices and modern coding techniques in solidity language that are changing each year._

### Recommendations
Adding a readable message to express the reason when the TX gets reverted.

---

## [NC-10] Legacy version of solidity and floating pragma

All contracts that are created have version 0.8.7, Now (from the time of writing this report) solidity is at v0.8.23.

We recommend having the last stable version of the solidity for the contracts, to support all features that are introduced in the new versions.

And some contracts use legacy solidity versions:

- `zevm/WETH9.sol`: solidity v0.4.18 and it implements a floating pragma version
- `evm/tools/ImmutableCreate2Factory`: solidity v0.5.10

### Recommendations
Making the solidity version the last stable version is better to support all new features in the new versions, and to prevent any bugs that are introduced in the legacy versions like v0.5 and v0.4.
