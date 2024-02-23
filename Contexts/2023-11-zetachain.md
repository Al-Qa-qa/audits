# ZetaChain
ZetaChain context | 20 Nov 2023 to 18 Dec 2023 on code4rena | [Context page](https://code4rena.com/audits/2023-11-zetachain)

## My Findings Summary

|ID|Title|Severity|
|:-:|-----|:------:|
|H-01|Zeta Observer nodes are not listening to `internal TXs`, which makes Smart Contract Wallets users' funds locked when making `Omnichain calls`|HIGH|
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

