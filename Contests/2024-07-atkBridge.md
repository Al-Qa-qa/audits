# Ark Bridge
Ark Bridge contest || NFT Bridge, Starknet || 31 Jul 2024 to 28 Aug 2024 on [Codehawks](https://codehawks.cyfrin.io/c/2024-07-ark-project/results?lt=contest&sc=reward&sj=reward&page=1&t=leaderboard)

## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[H&#8209;01](#h-01-permanent-l1---l2-dos-because-of-whitelisting-linked-list-logic)|Permanent (L1 -> L2) DoS Because of whitelisting Linked List Logic.|HIGH|
|[H&#8209;02](#h-02-the-bridging-process-will-revert-if-the-collection-is-matched-on-the-destination-chain-and-not-matched-on-the-source-chain)|The Bridging Process will revert if the Collection is matched on the destination chain and not matched on the source chain|HIGH|
|[H&#8209;03](#h-03-unable-to-remove-whitelist-collection-because-of-incorrect-linked-list-element-removing-logic)|Unable to remove whitelist collection because of incorrect Linked List element removing logic|HIGH|
|[H&#8209;04](#h-04-there-is-no-way-to-upgrade-nfts-collections-on-l1-that-is-mapped-to-original-collections-on-l2)|There is no way to upgrade NFTs collections on `L1` that is mapped to Original Collections on `L2`|HIGH|
|[H&#8209;05](#h-05-l2bridge-is-incompatible-with-erc721-that-returns-felt252-for-strings)|`L2Bridge` is incompatible with ERC721 that returns `felt252` for strings|HIGH|
||||
|[M&#8209;01](#m-01-deposites-from-l2-to-l1-will-be-unwithdrawable-from-l1-when-activating-use_withdraw_auto)|Deposites from L2 to L1 will be unwithdrawable from L1 when activating `use_withdraw_auto`|MEDIUM|
|[M&#8209;02](#m-02-possibility-of-reverting-when-withdrawing-tokens-from-l1-if-the-receiver-is-an-aa-wallet)|Possibility of reverting when withdrawing Tokens from L1, if the receiver is an AA wallet|MEDIUM|
|[M&#8209;03](#m-03-a-malicious-user-can-bridge-an-nft-on-l2-and-destroy-it-on-l1)|A Malicious user can bridge an NFT on `L2` and destroy it on `L1`|MEDIUM|
||||
|[L&#8209;01](#l-01-escrowed-nfts-are-not-burned-when-withdrawing-them-from-l2bridge)|Escrowed NFTs are not burned when withdrawing them from `L2Bridge`|LOW|
|[L&#8209;02](#l-02-tokenutil_callbaseuri-will-always-fail-because-of-different-reasons)|`Tokenutil::_callBaseUri` will always fail because of different reasons|LOW|
|[L&#8209;03](#l-03-implementation-contract-left-uninitialized-and-can-be-initialized)|Implementation contract left uninitialized and can be initialized|LOW|
|[L&#8209;04](#l-04-checking-baseuri-value-will-succeed-even-if-the-returned-string-is-empty)|Checking `baseURI` value will succeed even if the returned string is empty|LOW|
|[L&#8209;05](#l-05-existed-collections-are-not-whitelisted-when-bridging)|Existed collections are not whitelisted when Bridging|LOW|

---

## [H-01] Permanent (L1 -> L2) DoS Because of whitelisting Linked List Logic.

### Vulnerability Details
When doing a Bridged transaction (L2 -> L1), the sequencer calls `L2Bridge::withdraw_auto_from_l1`.

When calling this function in the last of it we call `ensure_erc721_deployment` and in the last of this function, we call `_white_list_collection` if the collection is not whitelisted we add it to the whitelisted Linked List.

[bridge.cairo#L471-L478](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L471-L478)
```cairo
        // update whitelist if needed
        let (already_white_listed, _) = self.white_listed_list.read(l2_addr_from_deploy);
        if already_white_listed != true {
❌️          _white_list_collection(ref self, l2_addr_from_deploy, true);
            self.emit(CollectionWhiteListUpdated {
                collection: l2_addr_from_deploy,
                enabled: true,
            });
        }
```

If we check the logic of the addition in `_white_list_collection` we will find that it adds the new collection in the last of that Linked List, and we are using single Linked List DS, not Double. So we are iterating over all the elements to put this NFT collection in the Linked List.

[bridge.cairo#L502-L514](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L502-L514)
```cairo
      // find last element
      loop {
          let (_, next) = self.white_listed_list.read(prev);
          if next.is_zero() {
              break;
          }
          let (active, _) = self.white_listed_list.read(next);
          if !active {
              break;
          }
          prev = next;
      };
❌️    self.white_listed_list.write(prev, (true, collection));
```

As we can see we are iterating from the `head` to the next element to the next one, and so on till we reach the end of the linked list (the next element is zero), then we link our collection to it.

Whitelisting a linked list is only a part of the logic done when calling `withdraw_auto_from_l1`, besides deploying the NFT collection, mint NFTs to the L2 recipient, and other things, so the tx gas cost will increase and may even reach the maximum.

Another thing we need to take into consideration is that this function is called by the sequencer, so reverting the tx is possible, and the sequencer may not accept the message from L1 from the beginning if the fees provided was small.

### Proof of Concept
> Normal Scenario
- WhiteListing is disabled in `L1Bridge`, so any NFT collection can be bridged.
- Bridge is so active and a lot of NFTs are bridged from L1 to L2.
- Once the collection is deployed, it is added to `whitelisted` collection Linked List.
- The number of whitelisted collections increases 10, 20, 50, 100, 200, 500, 1000, ...
- The cost for Bridging new NFT collection became too large as `withdraw_auto_from_l1` iterates `1000` time besides deploying a new contract and some checks.
- The transaction will get reverted when the sequencer calls it because of the huge TX cost, or it will not accept processing the message in the first place from L1.

> Attack Scenario
- WhiteListing is disabled in `L1Bridge`, so any NFT collection can be bridged.
- Bridge is so active and a lot of NFTs are bridged from L1 to L2.
- An Attacker spammed Bridging different NFTs from (L1 -> L2).
- The number of whitelisted collections increases 10, 20, 50, 100, 200, 500, 1000, ...
- The cost for Bridging new NFT collection became too large as `withdraw_auto_from_l1` iterates `1000` time besides deploying a new contract and some checks.
- The transaction will get reverted when the sequencer calls it because of the huge TX cost, or it will not accept processing the message in the first place from L1.

### Impact
Permanent DoS for Bridging new NFTs from L1 -> L2

### Tools Used
Manual Review

### Recommendations
You can use something like OpenZeppelin `EnumberableSets` in solidity, and if it is not found we can use `Head/Tail` Linked List, where we will store the First and the last element in the linked list, so when we add a new element we will do it at `θ(1)` not `θ(N)` Average time complexity.

---

## [H-02] The Bridging Process will revert if the Collection is matched on the destination chain and not matched on the source chain

### Vulnerability Details
When Bridging Collections `L1<->L2`, we are checking if that NFT collection has a pair on the destination chain or not. and if it has an address on the destination, then we use it instead of redeploying new one.

[CollectionManager.sol#L111-L149](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/CollectionManager.sol#L111-L149)
```solidity
    function _verifyRequestAddresses(address collectionL1Req, snaddress collectionL2Req) ... {
        address l1Req = collectionL1Req;
        uint256 l2Req = snaddress.unwrap(collectionL2Req);
        address l1Mapping = _l2ToL1Addresses[collectionL2Req];
        uint256 l2Mapping = snaddress.unwrap(_l1ToL2Addresses[l1Req]);

        // L2 address is present in the request and L1 address is not.
        if (l2Req > 0 && l1Req == address(0)) {
            if (l1Mapping == address(0)) {
                // It's the first token of the collection to be bridged.
                return address(0);
            } else {
                // It's not the first token of the collection to be bridged,
                // and the collection tokens were only bridged L2->L1.
                return l1Mapping;
            }
        }

        // L2 address is present, and L1 address too.
        if (l2Req > 0 && l1Req > address(0)) {
            if (l1Mapping != l1Req) {
                revert InvalidCollectionL1Address();
            } else if (l2Mapping != l2Req) {
                revert InvalidCollectionL2Address();
            } else {
                // All addresses match, we don't need to deploy anything.
                return l1Mapping;
            }
        }

        revert ErrorVerifyingAddressMapping();
    }
``` 

This function (`_verifyRequestAddresses`) is called whenever we withdraw tokens, where if the request came from `L2` Bridge has a valid `collectionL1` address (l1Req), we are doing checks that the matching of addresses is the same on both chains.

- `l2Req` is the NFT collection we withdrew from on `L2`, and it should be a valid NFT collection address
- `l1Req` is the `l2_to_l1_addresses` on `L2` where if the collection has matching on `L2` it will use that address when bridging tokens from `L2` to `L1`.

[bridge.cairo#L274](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L274)
```cairo
    let collection_l1 = self.l2_to_l1_addresses.read(collection_l2);
```

So if the NFT collection has an `L1<->L2` matching on `L2` we will do a check that ensures the NFT collection `L1` and `L2` addresses on `L2Bridge` are the same in `L1Bridge`.
```solidity
    if (l2Req > 0 && l1Req > address(0)) {
        if (l1Mapping != l1Req) {
            revert InvalidCollectionL1Address();
        } else if (l2Mapping != l2Req) {
            revert InvalidCollectionL2Address();
        } else {
            // All addresses match, we don't need to deploy anything.
            return l1Mapping;
        }
    }
```

The problem is that setting `l1<->l2` addresses on the `L2Bridge` doesn't mean that that value is always set on `L1Bridge`.

We are only setting `l1<->l2` on the chain we are withdrawing from, so when bridging from `L1` to `L2`. `L2Bridge` will set the matching between collections but `L1` will not set that matching. So if we tried to withdraw from `L2` to `L1` the withdrawing will revert as it will compare a collection address with the address zero.

## Scenario
- There is an NFT collection on `L1`, this collection has no matching on either `L1` or `L2`
- UserA bridged tokens from that collections to `L2`
- req.collectionL2 is `address(0)` as the is no `l1<->l2` matching on `L1Bridge`
```solidity
                revert InvalidCollectionL2Address();
            } else {
                // All addresses match, we don't need to deploy anything.
                return l1Mapping;
            }
        }

        revert ErrorVerifyingAddressMapping();
    }
```
- Since there is no matching in `L1`, `l1Mapping` and `l2Mapping` are `address(0)`, and will go for the check `l1Mapping != l1Req`, which will be true, ending up withdrawing from `L1` getting reverted.

### Proof of Concept
Add the following test function function in `apps/blockchain/ethereum/test/Bridge.t.sol`.

```solidity
    function test_auditor_collection_matching_one_chain() public {
        // alice deposit token 0 and 9 of collection erc721C1 to bridge
        test_depositTokenERC721();

        // Build the request and compute it's "would be" message hash.
        felt252 header = Protocol.requestHeaderV1(CollectionType.ERC721, false, false);

        // Build Request on L2
        Request memory req = buildRequestDeploy(header, 9, bob);
        req.collectionL1 = address(erc721C1);
        uint256[] memory reqSerialized = Protocol.requestSerialize(req);
        bytes32 msgHash = computeMessageHashFromL2(reqSerialized);

        // The message must be simulated to come from starknet verifier contract
        // on L1 and pushed to starknet core.
        uint256[] memory hashes = new uint256[](1);
        hashes[0] = uint256(msgHash);
        IStarknetMessagingLocal(snCore).addMessageHashesFromL2(hashes);

        // Withdrawing tokens will revert as There is no matching on L1
        address collection = IStarklane(bridge).withdrawTokens(reqSerialized);
    }
```

In the cmd write the following command.

```shell
forge test --mt test_auditor_collection_matching_one_chain -vv
```

> Output

The function will revert with an error message `InvalidCollectionL1Address()`
```powershell
Ran 1 test for test/Bridge.t.sol:BridgeTest
[FAIL. Reason: InvalidCollectionL1Address()] test_auditor_collection_matching_one_chain() (gas: 648188)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 5.71ms (1.77ms CPU time)
```

## Impacts
Lossing of NFTs Bridged from `L2` to `L1`

### Tools Used
Manual Review + Foundry

### Recommendations
Do not check the correct `l1<->l2` matching on `L1` and `L2` if the `L1` has no matching yet.

```diff
diff --git a/apps/blockchain/ethereum/src/token/CollectionManager.sol b/apps/blockchain/ethereum/src/token/CollectionManager.sol
index ec9429a..f790f70 100644
--- a/apps/blockchain/ethereum/src/token/CollectionManager.sol
+++ b/apps/blockchain/ethereum/src/token/CollectionManager.sol
@@ -113,7 +113,6 @@ contract CollectionManager {
         snaddress collectionL2Req
     )
         internal
-        view
         returns (address)
     {
         address l1Req = collectionL1Req;
@@ -133,6 +132,13 @@ contract CollectionManager {
             }
         }
 
+        // L2 is present, L1 address too, and there is no mapping
+        if (l2Req > 0 && l1Req > address(0) && l1Mapping == address(0) && l2Mapping == 0) {
+            _l1ToL2Addresses[l1Req] = collectionL2Req;
+            _l2ToL1Addresses[collectionL2Req] = l1Req;
+            return l1Req;
+        }
+
         // L2 address is present, and L1 address too.
         if (l2Req > 0 && l1Req > address(0)) {
             if (l1Mapping != l1Req) {
```

### Existence of the issue on the Starknet Side
We illustrated the issue when Bridging tokens from `L2` to `L1` and they have matchings on `L2` but not in `L1`, this issue existed also when Bridging from `L1` to `L2` when `L1` has a matching but `L2` has not. where the implementaion of the verification function is the same.

[collection_manager.cairo#L170-L200](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/token/collection_manager.cairo#L170-L200)
```cairo
fn verify_collection_address( ... ) -> ContractAddress {

    // L1 address must always be set as we receive the request from L1.
    if l1_req.is_zero() {
        panic!("L1 address cannot be 0");
    }

    // L1 address is present in the request and L2 address is not.
    if l2_req.is_zero() { ... } else {
        // L1 address is present, and L2 address too.
        if l2_bridge != l2_req {
❌️          panic!("Invalid collection L2 address");
        }

        if l1_bridge != l1_req {
            panic!("Invalid collection L1 address");
        }
    }

    l2_bridge
}
```

If `l1_req` and `l2_req` have values (there is a matching in `L1`) we will go for the `else` block, and since `l2_bridge` is `address_zero` (no matching on `L2`) withdrawing process will revert on `L2`.

The scenario is the same as we illustrated but with replacing `L1` by `L2` and `L2` by `L1`. So to not repeat ourselves we mentioned it here briefly.

To mitigate the issue on `L2` we will do the same as in `L1` where if there is no matching in `L2` but there is in `L1` we will just return the addresses.

---

## [H-03] Unable to remove whitelist collection because of incorrect Linked List element removing logic

### Summary
Because of the incorrect  implementation of removing an element from a Linked List, removing whitelisted NFTs will not be possible.

### Vulnerability Details

This issue is related to Linked List Data Structures.

The current Data Structure used for Storing `WhiteListed` NFTs on `Starknet` is Linked List. In Ethereum, we are using arrays, but the Protocol preferred to use LinkedList in Starknet.

When removing an element from a linked list, we need to start from the `Head` then go to the `Next`, then `Next` ... till we reach the element. But in the current removing logic, the protocol forgot to Move to the `Next` element when removing.

[bridge.cairo#L523-L537](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L523-L537)
```cairo
      // removed element from linked list
      loop {
          let (active, next) = self.white_listed_list.read(prev);
          if next.is_zero() {
              // end of list
              break;
          }
          if !active {
              break;
          }
          if next == collection {
              let (_, target) = self.white_listed_list.read(collection);
              self.white_listed_list.write(prev, (active, target));
              break;
          }
          // @audit where is `prev = next` step 
      };
```  

As we can see from the `loop` we are just reading the `prev` element, which is the `head` when beginning iteration, check for the next element if it is zero, or not activated. and check for the next element to be the collection we need to remove, But when we finish the iteration, we are not moving a step forward i.e `prev = next`, so this will make us loop in an infinite loop, leading to the tx reverted in the last because of the consumption of all gas.

This is not the case when adding new element, where the team handled it correctly.

[bridge.cairo#L512](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L512)
```cairo
      // find last element
      loop {
          ...
@>        prev = next;
      };
```

This will make removing any element in the list except `first` one in most cases unable to get removed.

### Proof of Concept

We made a simple Script to demonstrate how removing an element will not succeed, and the tx will revert when doing this.

Add this test in `apps/blockchain/starknet/src/tests/bridge_t.cairo`.

```cairo
    #[test]
    fn auditor_whitelist_collection_remove_collection_dos_issue() {
        let erc721b_contract_class = declare("erc721_bridgeable");

        let BRIDGE_ADMIN = starknet::contract_address_const::<'starklane'>();
        let BRIDGE_L1 = EthAddress { address: 'starklane_l1' };

        let bridge_address = deploy_starklane(BRIDGE_ADMIN, BRIDGE_L1, erc721b_contract_class.class_hash);
        let bridge = IStarklaneDispatcher { contract_address: bridge_address };
        
        start_prank(CheatTarget::One(bridge_address), BRIDGE_ADMIN);
        bridge.enable_white_list(true);
        stop_prank(CheatTarget::One(bridge_address));
        
        let collection1 = starknet::contract_address_const::<'collection1'>();
        let collection2 = starknet::contract_address_const::<'collection2'>();
        let collection3 = starknet::contract_address_const::<'collection3'>();
        let collection4 = starknet::contract_address_const::<'collection4'>();
        let collection5 = starknet::contract_address_const::<'collection5'>();

        start_prank(CheatTarget::One(bridge_address), BRIDGE_ADMIN);
        bridge.white_list_collection(collection1, true);
        bridge.white_list_collection(collection2, true);
        bridge.white_list_collection(collection3, true);
        bridge.white_list_collection(collection4, true);
        bridge.white_list_collection(collection5, true);
        stop_prank(CheatTarget::One(bridge_address));
        
        // Check that Collections has been added successfully
        let white_listed = bridge.get_white_listed_collections();
        assert_eq!(white_listed.len(), 5, "White list shall contain 5 elements");
        assert_eq!(*white_listed.at(0), collection1, "Wrong collection address in white list");
        assert_eq!(*white_listed.at(1), collection2, "Wrong collection address in white list");
        assert_eq!(*white_listed.at(2), collection3, "Wrong collection address in white list");
        assert_eq!(*white_listed.at(3), collection4, "Wrong collection address in white list");
        assert_eq!(*white_listed.at(4), collection5, "Wrong collection address in white list");
        assert!(bridge.is_white_listed(collection1), "Collection1 should be whitelisted");
        assert!(bridge.is_white_listed(collection2), "Collection2 should be whitelisted");
        assert!(bridge.is_white_listed(collection3), "Collection3 should be whitelisted");
        assert!(bridge.is_white_listed(collection4), "Collection4 should be whitelisted");
        assert!(bridge.is_white_listed(collection5), "Collection5 should be whitelisted");

        // This should Revert
        start_prank(CheatTarget::One(bridge_address), BRIDGE_ADMIN);
        bridge.white_list_collection(collection3, false);
        stop_prank(CheatTarget::One(bridge_address));
    }
```

To run it you can write this command on `apps/blockchain/starknet` path

```bash
snforge test auditor_whitelist_collection_remove_collection_dos_issue
```

> Output:
```powershell
[FAIL] starklane::tests::bridge_t::tests::auditor_whitelist_collection_remove_collection_dos_issue

Failure data:
    Got an exception while executing a hint: Hint Error: Error in the called contract (0x03b24bdfb3983f3361a7f81e871041cc45f3e1c21bfe3f1cbcaf7bec224627d5):
Error at pc=0:7454:
Could not reach the end of the program. RunResources has no remaining steps.
Cairo traceback (most recent call last):
...
```

### Impact
- Inability to remove whitelisted collections.
- If a specific whitelisted NFT collection is malicious, we cannot unwhitelist it.
- The only way to remove a specific NFT will be to start removing from the `head` till we reach that NFT collection to be `unwhitelisted` which will affect other NFT collections and is not an applicable solution in production.

### Tools Used
Manual Review + Starknet Foundry

### Recommendations
Move to the next element in the linked list once completing the iteration.

```diff
diff --git a/apps/blockchain/starknet/src/bridge.cairo b/apps/blockchain/starknet/src/bridge.cairo
index 23cbf8a..35e003b 100644
--- a/apps/blockchain/starknet/src/bridge.cairo
+++ b/apps/blockchain/starknet/src/bridge.cairo
@@ -535,6 +535,7 @@ mod bridge {
                         self.white_listed_list.write(prev, (active, target));
                         break;
                     }
+                    prev = next;
                 };
                 self.white_listed_list.write(collection, (false, no_value));
             }
```

---

## [H-04] There is no way to upgrade NFTs collections on `L1` that is mapped to Original Collections on `L2`

### Vulnerability Details
When we are bridging Tokens `L1<->L2` if the NFT collection in the source chain has no address on the destination chain, we are deploying new ERC721 NFT collection addresses and attaching them.

When deploying we are making the Collection upgradable, where we can change the implementation of that NFT collection if needed.

[Deployer.sol#L23-L38](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/Deployer.sol#L23-L38)
```solidity
    function deployERC721Bridgeable(... ) ... {
❌️      address impl = address(new ERC721Bridgeable());
        
        bytes memory dataInit = abi.encodeWithSelector(
            ERC721Bridgeable.initialize.selector,
            abi.encode(name, symbol)
        );

❌️      return address(new ERC1967Proxy(impl, dataInit));
    }
```

Since ERC1967 makes the Proxy admin is the sender, so the Bridge is the only address that can upgrade the Collection implementation.

The issue is that no method exists in our `L1Bridge` contract to upgrade the NFT collection implementation in `L1`. However, if we checked the `L2Bridge` we will find that upgrading an NFT collection is a supported thing and intended.

[bridge.cairo#L369-L373](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L369-L373)
```cairo
        fn collection_upgrade(ref self: ContractState, collection: ContractAddress, class_hash: ClassHash) {
            ensure_is_admin(@self);
            IUpgradeableDispatcher { contract_address: collection }
                .upgrade(class_hash);
        }
```

As we can see in `L2Bridge` there is a function `collection_upgrade` which takes the NFT collection address and calls `upgrade`, which will allow upgrading the created NFT collection implementation if needed.

This will result in the inability to upgrade NFT collections that was deployed on `L1` if needed.

### Proof of Concept
- Bridge is active
- Tokens are Bridge `L1<->L2`
- There are new collections created on `L1` and attached to original collections on `L2`
- There are new collections created on `L2` and attached to original collections on `L1`
- The admin decided to upgrade some collections created on `L1` and `L2`
- The admin will be able to upgrade `L2` collections but will not be able to upgrade `L1` collections

### Impact
Inability to upgrade Created NFT collections on `L1`

### Tools Used
Manual Review

### Recommendation
Implement a function to upgrade the NFT collection on `L1Bridge`, the same as that in `L2Bridge`.

```diff
diff --git a/apps/blockchain/ethereum/src/Bridge.sol b/apps/blockchain/ethereum/src/Bridge.sol
index e62c7ce..0df98c2 100644
--- a/apps/blockchain/ethereum/src/Bridge.sol
index e62c7ce..0df98c2 100644
--- a/apps/blockchain/ethereum/src/Bridge.sol
+++ b/apps/blockchain/ethereum/src/Bridge.sol
@@ -374,4 +374,16 @@ contract Starklane is IStarklaneEvent, UUPSOwnableProxied, StarklaneState, Stark
         emit L1L2CollectionMappingUpdated(collectionL1, snaddress.unwrap(collectionL2));
     }
 
+    function collectionUpgrade(
+        ERC721Bridgeable collectionAddress,
+        address newImplementation,
+        bytes memory initData
+    ) external onlyOwner {
+        if (init.length > 0) {
+            collectionAddress.upgradeToAndCall(newImplementation, initData);
+        } else {
+            collectionAddress.upgradeTo(newImplementation);
+        }
+    }
+
 }
```

_NOTE: This Mitigation is not tested, so it may be implemented correctly._

---

## [H-05] `L2Bridge` is incompatible with ERC721 that returns `felt252` for strings


### Vulnerability Details
When retrieving ERC721 collection metadata values (name, symbol, token_uri), we are retrieving their returned data to be `ByteArray`. Since these functions will return string values we need to understand what is the data type of string value in Cairo.

Cairo is a low-level programming language, it does not support strings, and strings are represented using either `felt252` or `ByteArray`.

If the string length is smaller than `31` length it can be represented in only one `felt252`.

First we need to understand why there are two types for retrieving strings.

The original type was normal `felt252`, but on March 2024 Starknet introduced `ByteArray` type.

This means that all NFT collections (ERC721 tokens) are created from the launching of the Starknet blockchain till the type of supporting ByteArray having an interface that returns the `name`, `symbol`, and `token_uri` as `felt252`.

OpenZepeplin starknet contract versions from the beginning to `v0.9.0` use `felt252` for their ERC721 tokens.

[OZ::0.9.0::erc721.cairo#L219-L221](https://github.com/OpenZeppelin/cairo-contracts/blob/release-v0.9.0/src/token/erc721/erc721.cairo#L219-L221)
```cairo
        fn name(self: @ComponentState<TContractState>) -> felt252 {
            self.ERC721_name.read()
        }
```

And from `v0.10.0` to `v0.15.0` they use `ByteArray`. The protocol uses v0.11.0, which assigns the returned value to `ByteArray`.

[OZ::0.11.0::erc721.cairo#L221-L223](https://github.com/OpenZeppelin/cairo-contracts/blob/release-v0.11.0/src/token/erc721/erc721.cairo#L221-L223)
```cairo
        fn name(self: @ComponentState<TContractState>) -> ByteArray {
            self.ERC721_name.read()
        }
```

The Starknet launched `November 2021` or `February 2022`, and `ByteArray` was supported on March 2024. so all ERC721 tokens created in these 2 years have an interface that returns `felt252` for `name`, `symbol`, and `token_uri`. 

The protocol is not handling such a case and deals with NFTs on layer2 as they return `ByteArray` when calling `name`, `symbol`, which is totally incorrect, as the majority of NFTs were created before supporting `ByteArray` so dealing with the returned value when calling `<collection>.name()` will only work for ERC721 tokens that was created after supporting `ByteArray`, and use it.

When making calls there are two types of calls in starknet:
1. low-level call: similar to `.call` in solidity.
2. High-level calls: interface calls.

In solidity the returned data is in Bytes whatever its type was, the same is in Starknet when doing low-level call the returned type is Span<felt252> what ever the type was.

We are handling retrieving token_uri correctly, where we are doing an internal call when getting them, which result in a return value as `Span<felt252>`

[collection_manager.cairo#L107-L123](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/token/collection_manager.cairo#L107-L123)
```cairo
    match starknet::call_contract_syscall(
        collection_address,
        token_uri_selector,
        calldata,
    ) {
❌️      Result::Ok(span) => span.try_into(),
        Result::Err(_e) => {
            match starknet::call_contract_syscall(
                collection_address, tokenURI_selector, calldata,
            ) {
❌️              Result::Ok(span) => span.try_into(),
                Result::Err(_e) => {
                    Option::None
                }
            }
        }
    }
```

Since the returned value is felt252, we will convert it into `ByteArray` without problems and there is a custom implementation for `try_into` than handles converting felt252 into ByteArray correctly in `byte_array_extra`.

The issue is that when getting token `name` and symbol we are retrieving them with High-level call.

[collection_manager.cairo#L69-L70](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/token/collection_manager.cairo#L69-L70)
```cairo
    #[derive(Drop)]
    struct ERC721Metadata {
        name: ByteArray,
        symbol: ByteArray,
        base_uri: ByteArray,
        uris: Span<ByteArray>,
    }
    ...
    Option::Some(
        ERC721Metadata {
❌️          name: erc721.name(),
❌️          symbol: erc721.symbol(),
            base_uri: "",
            uris
        }
    )
```

The interface we are defining `ERC721Metadata` declares string values as `ByteArray` we explained why `uris` will get handled correctly, and for `base_uri` it is written manually, but for `name()` and `symbol()` we are calling them directly and expect the returned value to be of type `ByteArray`.

`erc721` is the collection address we are going to Bridge the token from it from `L2` to `L1`, and since not all NFT collections on `L2`, but actually the majority of NFT collections on `L2` return `felt252`, making `erc721.name()` will result in an exception because of incorrect assigning. the execution will stop even we have an `Option` return value because this is an execution error that occurred before reaching the end of the function.

### Proof of Concept
Doing a POC for such an issue is complicated, but to make it achievable we separated it into two parts:

1. In the Starknet development environment make another `erc721_collection` that returns `name()`, `symbol()`, `token_uri()` as strings. Achieving this needs a lot of modifications to `erc721_collection.cairo` file so we made another file `erc721_collection_2.cairo` and make it as an NFT collection that returns `felt252`.
2. Add the following test script in the last of `apps/blockchain/starknet/src/tests/bridge_t.cairo`.
```cairo
    fn deploy_erc721b_old(
        erc721b_contract_class: ContractClass,
        name: felt252,
        symbol: felt252,
        bridge_addr: ContractAddress,
        collection_owner: ContractAddress,
    ) -> ContractAddress {
        let mut calldata: Array<felt252> = array![];
        let base_uri: felt252 = 'https://my.base.uri/';
        name.serialize(ref calldata);
        symbol.serialize(ref calldata);
        base_uri.serialize(ref calldata);
        calldata.append(bridge_addr.into());
        calldata.append(collection_owner.into());

        erc721b_contract_class.deploy(@calldata).unwrap()
    }

    #[test]
    fn deposit_token_auditor_old_erc721() {
        // Need to declare here to get the class hash before deploy anything.
        let erc721b_contract_class = declare("erc721_bridgeable");
        let erc721b_old_contract_class = declare("erc721_bridgeable_2");

        let BRIDGE_ADMIN = starknet::contract_address_const::<'starklane'>();
        let BRIDGE_L1 = EthAddress { address: 'starklane_l1' };
        let COLLECTION_OWNER = starknet::contract_address_const::<'collection owner'>();
        let OWNER_L1 = EthAddress { address: 'owner_l1' };

        let bridge_address = deploy_starklane(BRIDGE_ADMIN, BRIDGE_L1, erc721b_contract_class.class_hash);

        let erc721b_address = deploy_erc721b_old(
            erc721b_old_contract_class,
            'everai',
            'DUO',
            bridge_address,
            COLLECTION_OWNER
        );

        let erc721 = IERC721Dispatcher { contract_address: erc721b_address };

        mint_range(erc721b_address, COLLECTION_OWNER, COLLECTION_OWNER, 0, 10);

        let bridge = IStarklaneDispatcher { contract_address: bridge_address };

        start_prank(CheatTarget::One(erc721b_address), COLLECTION_OWNER);
        erc721.set_approval_for_all(bridge_address, true);
        stop_prank(CheatTarget::One(erc721b_address));

        start_prank(CheatTarget::One(bridge_address), COLLECTION_OWNER);
        println!("We will call bridge.deposit_tokens()...");
        // This should revert
        bridge.deposit_tokens(
            0x123,
            erc721b_address,
            OWNER_L1,
            array![0, 1].span(),
            false,
            false);
        println!("bridge.deposit_tokens() finished calling successfully");
        stop_prank(CheatTarget::One(bridge_address));


    }
```

Run the following command.

```shell
snforge test deposit_token_auditor_old_erc721
```

> Output
```shell
[FAIL] starklane::tests::bridge_t::tests::deposit_token_auditor_old_erc721

Failure data:
    Got an exception while executing a hint: Hint Error: 0x4661696c656420746f20646573657269616c697a6520706172616d202331 ('Failed to deserialize param #1')
```

_NOTE: To not make the report too large, we mentioned in point `1` what we need in order for the POC to work, all you have to do is to deploy an `erc721_collection` that returns `felt252` for `name()`, and we preferred to leave this to you as mention how to make this needs a lot of modifications. And for the POC, you need to make sure that you are deploying the correct address, our POC works when there are two contracts `erc721_collection` and `erc721_collection_2` and the `2` version is the one that returns `felt252`, you need to import these contract and make sure to add them in order to get them in artificats and not receive path errors. And ofc having the sponsor with you when setting it up is the best_

### Impact
- The majourity of NFTs on `L2`, which returns `felt252` when calling `name()` or `symbol()` can't get briged.

### Tools Used
Manual review + Starknet Foundry

### Recommendations
reterive `erc721.name()` and `erc721.symbol()` using low-level to get the result as `Span<felt252>` then convert them into `ByteArray`.

---
---
---

## [M-01] Deposites from L2 to L1 will be unwithdrawable from L1 when activating `use_withdraw_auto`

### Vulnerability Details
There was an issue related to Auto Withdrawals from L1 reported by `Cairo_Security_Clan`, and the team chose to remove this feature from the L1 side when withdrawing.

[Bridge.sol#L169-L173](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Bridge.sol#L169-L173)
```solidity
        if (Protocol.canUseWithdrawAuto(header)) {
            // 2024-03-19: disabled autoWithdraw after audit report
            // _consumeMessageAutoWithdraw(_starklaneL2Address, request);
❌️          revert NotSupportedYetError();
        } else { ... }
```

The problem is that this value is still taken as a parameter from the user in L2 Bridge when depositing Tokens. And no check guarantees that the value will be `false`, it is left to the user to set it.

[bridge.cairo#L249](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L249)
```cairo
        fn deposit_tokens(
            ref self: ContractState,
            salt: felt252,
            collection_l2: ContractAddress,
            owner_l1: EthAddress,
            token_ids: Span<u256>,
❌️          use_withdraw_auto: bool,
            use_deposit_burn_auto: bool,
        ) { ... }
```

So people Bridging From L2 still think that auto withdrawal is supported. But it is actually not. which will end up Locking for their NFT in the L2 Bridge and the inability to withdraw them from L1 as calling `L1Bridge::withdrawTokens()` will always revert.

### Proof of Concept
- UserA Wanted to Bridge one NFT from L2 to L1.
- UserA thinks `use_withdraw_auto` is allowed as it is to him to either set or not set it.
- UserA called `L2Bridge::deposit_tokens()` by setting `use_withdraw_auto` to true.
- UserA Waited till `Starknet` verified the tx and put his message as an approved `L2toL1` message.
- UserA tried to withdraw his NFTs on L1 by calling `L1Bridge::withdrawTokens()`.
- Transactions get reverted whenever he tries to withdraw his token on L1.

### Impact
Permanent Lock of NFTs on L2 Bridge Side and the inability to withdraw them from L1 side.

### Tools Used
Manual Review

### Recommended Mitigation
Ensure that `use_withdraw_auto` is set to false when withdrawing from L2 side.

```diff
diff --git a/apps/blockchain/starknet/src/bridge.cairo b/apps/blockchain/starknet/src/bridge.cairo
index 23cbf8a..5b82f1d 100644
--- a/apps/blockchain/starknet/src/bridge.cairo
+++ b/apps/blockchain/starknet/src/bridge.cairo
@@ -250,6 +250,7 @@ mod bridge {
         ) {
             ensure_is_enabled(@self);
             assert(!self.bridge_l1_address.read().is_zero(), 'Bridge is not open');
+            assert(use_withdraw_auto != true, "Auto Withdraw is Disappled");
 
             // TODO: we may have the "from" into the params, to allow an operator
             // to deposit token for a user.
```

---

## [M-02] Possibility of reverting when withdrawing Tokens from L1, if the receiver is an AA wallet

### Vulnerability Details

When Bridging tokens from `L2->L1`, we provide the receiver address in L1, and after we made the tx on L2, gets proved by Starknet Protocol, the user can call `L1Bridge::withdrawTokens()` to receive his tokens in L1.

We extract the tokens Bridged from `L2` to `L1`, and if that token was escrowed in `L1Bridge` we send it to the user from the `L1Bridge` to him, otherwise we mint it.

[Bridge.sol#L201](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Bridge.sol#L201)
```solidity
    function withdrawTokens( ... ) ... {
        ...
        for (uint256 i = 0; i < req.tokenIds.length; i++) {
            ...
❌️          bool wasEscrowed = _withdrawFromEscrow(ctype, collectionL1, req.ownerL1, id);
            ...
        }
        ...
    }
```

When we transfer ERC721 token to the user we are using `safeTransferFrom` function, which checks if the receiver is a contract it calls `onERC721Receiver()` function.

[Escrow.sol#L79](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Escrow.sol#L79)
```solidity
    function _withdrawFromEscrow(... ) ... {
        ...

        if (collectionType == CollectionType.ERC721) {
❌️          IERC721(collection).safeTransferFrom(from, to, id);
        } else { ... }
        ...
    }
```

The problem is that if the receiver is an Account Abstraction wallet, this function `onERC721Receiver()` is not a must to be presented. The implementation of AA Wallets didn't implement this interface nor it is implemented in the `ERC4337`. so if the receiver is an AA wallet the transaction will revert as it will not find that function and will revert.

This doesn't mean that the NFT will be lost. since AA wallets contains `execute` function that can call arbitraty calls, they can handle NFTs, so implementing that interface `onERC721Receiver()` is not a must.

### Proof of Concept
- UserA wants to Bridge NFTs from `L2` to `L1`.
- UserA wants to send his NFTs to his AA wallet on `L1`.
- UserA initiated the tx and called `L2Bridge::deposit_tokens()`.
- When UserA wanted to call the `L1Bridge::withdrawTokens()` to withdraw his tokens.
- tx reverted as the UserA `AA` wallet do not implement `onERC721Receive()`.
- NFTs will be locked in the Bridge, and lost forever.

### Impact
if the receiver on `L1` was an AA wallet NFTs may end up locked in the Bridge forever without having the apility to withdraw them.

### Recommendations
Use `transfer` instead of `safeTransfer` when transferring NFTs.

```diff
diff --git a/apps/blockchain/ethereum/src/Escrow.sol b/apps/blockchain/ethereum/src/Escrow.sol
index c58bce9..47de782 100644
--- a/apps/blockchain/ethereum/src/Escrow.sol
+++ b/apps/blockchain/ethereum/src/Escrow.sol
@@ -76,7 +76,7 @@ contract StarklaneEscrow is Context {
         address from = address(this);
 
         if (collectionType == CollectionType.ERC721) {
-            IERC721(collection).safeTransferFrom(from, to, id);
+            IERC721(collection).transferFrom(from, to, id);
         } else {
             // TODO:
             // Check here if the token supply is currently 0.
```

---

## [M-03] A Malicious user can bridge an NFT on `L2` and destroy it on `L1`


### Vulnerability Details

When withdrawing tokens, these tokens are either escrowed by the bridge or not. If the bridge holds the token, we transfer it from the bridge to the `ownerL1`, else this means that the NFT collection is a custom NFT collection deployed by the bridge and we call `ERC721Collection::mintFromBridge` to mint it.

[Bridge.sol#L201-L209](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Bridge.sol#L201-L209)
```solidity
        bool wasEscrowed = _withdrawFromEscrow(ctype, collectionL1, req.ownerL1, id);

        if (!wasEscrowed) {
            ...
❌️              IERC721Bridgeable(collectionL1).mintFromBridge(req.ownerL1, id);
        }
``` 

If the NFT is escrowed, we transfer it to the owner and burn it (setting escrow to `address_zero`).

[Escrow.sol#L78-L86](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Escrow.sol#L78-L86)
```solidity
    function _withdrawFromEscrow( ... ) ... {
        if (!_isEscrowed(collection, id)) {
            return false;
        }

        address from = address(this);

        if (collectionType == CollectionType.ERC721) {
1:          IERC721(collection).safeTransferFrom(from, to, id);
        } else { ... }

2:      _escrow[collection][id] = address(0x0);

        return true;
    }
```

As we can see we are transferring the NFT using `safeTransferFrom`. Then, burning it and this opens up a Reentrancy!

since `safeTransferFrom` calls `onERC721Received()`, the user can do whatever he wants before the NFT gets burned. now what if the user deposited the NFT to receive it on `L2` again via calling `L2Bridger::depositTokens()`?

When depositing we are escrowing the tokenId(s) we are depositing, where the sender sends the token(s) to the bridge then it is escrowed.

[Bridge.sol#L129](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Bridge.sol#L129)
```solidity
    function depositTokens( ... ) ... {
        ...
❌️      _depositIntoEscrow(ctype, collectionL1, ids);
    }
    ...
    function _depositIntoEscrow( ... ) ... {
        assert(ids.length > 0);

        for (uint256 i = 0; i < ids.length; i++) {
            uint256 id = ids[i];

            if (collectionType == CollectionType.ERC721) {
                IERC721(collection).transferFrom(msg.sender, address(this), id);
            } else { ... }

❌️          _escrow[collection][id] = msg.sender;
        }
    }
```

Now since we called this function in `onERC721Received()` when we return from it, we will return back to the point of withdrawing escrowed tokens, where we will set escrow to `address(0)`. Now this will result in resetting the escrow mapping for that token into `address(0)` although it is owned by the `L1Bridge`.

### Attack Scenario
We are withdrawing escrowed tokens when either Bridging from `L2` to `L1` and tokens are escrowed, or we are canceling Message Requests from `L1` to `L2` that the Starknet sequencer did not process, we will illustrate the two attacks.

**Canceling Message:**
1. UserA deposited an NFT token and provided less `msg.value` to the StarknetMessaging protocol, using a contract implementing `onERC721Received()` that calls `L1Bridge::depositTokens()` again.
2. The sequencer didn't process the message because of low gas paid.
3. UserA messaged the protocol to cancel his message.

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns(bytes4) {
        uint256[] memory ids = new uint256[](1);
        ids[0] = tokenId;
        IStarklane(bridge).depositTokens{value: 30000}(
            0x123,
            collection,
            ownerL2,
            ids,
            false
        );
        

        return this.onERC721Received.selector;
    }

}

interface IStarklaneEscrowed is IStarklane {
    function _escrow(address collection, uint256 tokenId) external view returns(address);
}

contract AuditorTest is BridgeTest {

    function test_auditor_deposit_before_burn() public {
        
        uint256 delay = 30;
        IStarknetMessagingLocal(snCore).setMessageCancellationDelay(delay);
        uint256[] memory ids = new uint256[](2);
        ids[0] = 0;
        ids[1] = 1;

        // deploy an attack contract that will deposite on receiving an NFT
        AttackerContract attackContract = new AttackerContract{value: 10 * 30000}(bridge, erc721C1);

        (uint256 nonce, uint256[] memory payload) = setupCancelRequest(address(attackContract), ids);
        assert(IERC721(erc721C1).ownerOf(ids[0]) != address(attackContract));
        assert(IERC721(erc721C1).ownerOf(ids[1]) != address(attackContract));

        Request memory req = Protocol.requestDeserialize(payload, 0);

        vm.expectEmit(true, false, false, false, bridge);
        emit CancelRequestStarted(req.hash, 42);
        IStarklane(bridge).startRequestCancellation(payload, nonce);

        skip(delay * 1 seconds);
        // - Cancel request will transfer NFTs to the receiver and call `onERC721Received`
        // - We will call bridge.depositTokens
        // - Tokens will get escrowed by the Bridge
        // - onERC721Received will return back
        // - Token will get Burned
        // - The recever will receive his NFT on L2
        IStarklane(bridge).cancelRequest(payload, nonce);

        assert(IERC721(erc721C1).ownerOf(ids[0]) == bridge);
        assert(IERC721(erc721C1).ownerOf(ids[1]) == bridge);
        assert(IStarklaneEscrowed(bridge)._escrow(erc721C1, 0) == address(0x00));
        assert(IStarklaneEscrowed(bridge)._escrow(erc721C1, 1) == address(0x00));


        console2.log("Token[0] owner:", IERC721(erc721C1).ownerOf(ids[0]));
        console2.log("Token[1] owner:", IERC721(erc721C1).ownerOf(ids[1]));
        console2.log("Token[0] escrowed:", IStarklaneEscrowed(bridge)._escrow(erc721C1, 0));
        console2.log("Token[1] escrowed:", IStarklaneEscrowed(bridge)._escrow(erc721C1, 1));
    }
}
```

Since `escrow` variable is private, you will need to make it a `public` visibility in order for the test to work, add the public visibility to it.

[Escrow.sol#L17](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Escrow.sol#L17)
```diff
-   mapping(address => mapping(uint256 => address)) _escrow;
+   mapping(address => mapping(uint256 => address)) public _escrow;
```

On the cmd write the following command.
```shell
forge test --mt test_auditor_deposit_before_burn -vv
```

> Output:
```powershell
[PASS] test_auditor_deposit_before_burn() (gas: 1176814)
Logs:
  Token[0] owner: 0xc7183455a4C133Ae270771860664b6B7ec320bB1
  Token[1] owner: 0xc7183455a4C133Ae270771860664b6B7ec320bB1
  Token[0] escrowed: 0x0000000000000000000000000000000000000000
  Token[1] escrowed: 0x0000000000000000000000000000000000000000
```

The POC checks that the Bridge will end up being the owner of the tokens, and they are not escrowed, which means anyone who tries to withdraw it back when bridging from `L2` to `L1` will end up trying minting it which will revert.

### Impact
An innocent user can lose his NFTs on `L2` when Bridging them back to `L1`. 

### Tools Used
Manual Review + Foundry

### Recommendations
Implement the CEI pattern by burning The token before transferring it.

```diff
diff --git a/apps/blockchain/ethereum/src/Escrow.sol b/apps/blockchain/ethereum/src/Escrow.sol
index c58bce9..b5d8ad1 100644
--- a/apps/blockchain/ethereum/src/Escrow.sol
+++ b/apps/blockchain/ethereum/src/Escrow.sol
@@ -74,6 +74,7 @@ contract StarklaneEscrow is Context {
         }
 
         address from = address(this);
+        _escrow[collection][id] = address(0x0);
 
         if (collectionType == CollectionType.ERC721) {
             IERC721(collection).safeTransferFrom(from, to, id);
@@ -83,7 +84,6 @@ contract StarklaneEscrow is Context {
             IERC1155(collection).safeTransferFrom(from, to, id, 1, "");
         }
 
-        _escrow[collection][id] = address(0x0);
 
         return true;
     }
```

---
---
---

## [L-01] Escrowed NFTs are not burned when withdrawing them from `L2Bridge`.

### Vulnerability Details

In `L1Bridge` when we are withdrawing NFTs, and these NFTs are held by the Bridge, we transfer them from the Bridge to the receiver, then we burn that NFT by setting escrow address to `zero`.

[Escrow.sol#L76-L86](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Escrow.sol#L76-L86)
```solidity
    function _withdrawFromEscrow( ... ) ... {
        ...

        address from = address(this);

        if (collectionType == CollectionType.ERC721) {
1:          IERC721(collection).safeTransferFrom(from, to, id);
        } else {
            // TODO:
            // Check here if the token supply is currently 0.
            IERC1155(collection).safeTransferFrom(from, to, id, 1, "");
        }

2:      _escrow[collection][id] = address(0x0);

        return true;
    }
```

But in `L2Bridge` escrowed NFT token is not burned when transferring, we are just transferring the NFT without resetting the escrow mapping to `address_zero`.

[bridge.cairo#L157-L162](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L157-L162)
```cairo
            let is_escrowed = !self.escrow.read((collection_l2, token_id)).is_zero();

            if is_escrowed {
❌️              IERC721Dispatcher { contract_address: collection_l2 }
                .transfer_from(from, to, token_id);
                // @audit NFT is not reset to zero?!!
            } else { ... }
```

So this will result in an NFT being escrowed by the `L2Bridge` (which is like being owned by him), But it is actually owned by another owner, which is the `ownerL2` that received the tokens when Bridging tokens from `L1` to `L2`.

### Proof of Concept
- UserA Bridged an NFT token from `L2` to `L1` via `L2Bridge::deposit_tokens()`.
- This NFT token is now escrowed by the `L2Bridge` and it holds it.
- UserA received the NFT on `L1` by his `L1Address`.
- Time passed, and this NFT is traded between addresses.
- This NFT is now getting Bridged from `L1` to `L2` via `L1Bridge::depositTokens()`
- This NFT token is now escrowed by the `L1Bridge` and it holds it.
- This token is getting withdrawn from `L2Bridge`.
- Token was transferred to the `ownerL2` but did not burn on `L2Bridge` (it is still escrowed).
- NFT token is escrowed by `L2Bridge`, but it doesn't hold it.
- NFT tokens is escrowed in both `L1Bridge` and `L2Bridge`.

### Impacts
- Incorrect State modification, where the NFT tokens will still be escrowed by `L2Bridge` but it doesn't hold them.
- The NFTs will be escrowed in both `L1Bridge` and `L2Bridge` which breaks NFTs bridging invariant, where the NFT will exist in both Bridged in the time it should only be on only one Bridge (should be escrowed one chain).

### Tools Used
Manual Review

### Recommendations
Burn the NFT by setting its escrow to `address_zero` in `L2Bridge`.

```diff
diff --git a/apps/blockchain/starknet/src/bridge.cairo b/apps/blockchain/starknet/src/bridge.cairo
index 23cbf8a..85ec4d9 100644
--- a/apps/blockchain/starknet/src/bridge.cairo
+++ b/apps/blockchain/starknet/src/bridge.cairo
@@ -159,6 +159,7 @@ mod bridge {
             if is_escrowed {
                 IERC721Dispatcher { contract_address: collection_l2 }
                 .transfer_from(from, to, token_id);
+            self.escrow.write((collection_l2, token_id), starknet::contract_address_const::<0>());
             } else {
                 if (req.uris.len() != 0) {
                     let token_uri = req.uris[i];
```

---

## [L-02] `Tokenutil::_callBaseUri` will always fail because of different reasons

### Vulnerability Details

When Bridging NFT tokens we need to handle their `URIs`, according to there `Base URI` if existed, or each NFT `tokenURI` separately if not.

[TokenUtil.sol#L89-L92](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/TokenUtil.sol#L89-L92)
```solidity
        // How the URI must be handled.
        // if a base URI is already present, we ignore individual URI
        // else, each token URI must be bridged and then the owner of the collection
        // can decide what to do
```

Now if we check how getting `baseURI` is done, we will find that the logic contains a log of mistakes.

[TokenUtil.sol#L150-L155](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/TokenUtil.sol#L150-L155)
```solidity
    function _callBaseUri(address collection) ... {
        ...
❌️      bytes[2] memory encodedSignatures = [abi.encodeWithSignature("_baseUri()"), abi.encodeWithSignature("baseUri()")];

        for (uint256 i = 0; i < 2; i++) {
            bytes memory encodedParams = encodedSignatures[i];
            assembly {
                success := staticcall(gas(), collection, add(encodedParams, 0x20), mload(encodedParams), 0x00, 0x20)

```

First, we are calling `_baseUri()`, then if the call fails we call `baseUri()` function to get the `baseURI`, but calling these two functions will always fail (`100%`) because of the following reasons.

#### 1. Calling internal functions will always fail

`_baseUri()` is not a public function, it is an internal function in `ERC721`, so calling this will always revert, i.e failed.

#### 2. Incorrect function naming will lead to calling different function signature
The signature for `base URI` is `_baseURI` by making the three letters UpperCase, but in the code we are making just the `U` is in upper case and `r`, `i` are small letters, this will result in a totally different signature.

#### 3. There is no function signature named `baseURI()` or `baseUri()` in NFT contract
The third thing is that `baseUri/baseURI()` is not even exist in `ERC721`, and if we checked the modified version for ERC721 made by the protocol [`ERC721Bridgeable`](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/ERC721Bridgeable.sol), we will find that it is also not implementing any public function to retrieve that baseURI separately.

So by collecting all these things together, we can conclude that retrieving the `baseURI` will always fail.

This mean we will always end up looping through all NFT items we want to Bridge, getting their URIs to Bridge a large number of tokens, even if the collection supports `baseURI`.

[TokenUtil.sol#L97-L103](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/TokenUtil.sol#L97-L103)
```solidity
        else {
            string[] memory URIs = new string[](tokenIds.length);
            for (uint256 i = 0; i < tokenIds.length; i++) {
                URIs[i] = c.tokenURI(tokenIds[i]);
            }
            return (c.name(), c.symbol(), "", URIs);
        }
```

### Impacts
- `100%` DoS for the function used to retrieve `baseURI`
- The cost for Bridging large tokens will increase significantly when Bridging Large NFTs amount as we will loop through all there URIs to get them.
- payload size will increase a lot without knowing, which can even make NFT end up locking in the Bridge Due to a payload size limit in Starknet (This is a known issue, but we are highlighting that the payload will increase without knowing. When Bridging `20` NFTs for example supports base URI, the user thinks that tokenURIs array is `0`, but it will actually be `20` which will make the payload size increase significantly)  

### Tools Used
Manual Review

### Recommendations
1. Remove calling `_baseUri()` as it is an internal function which will always fail.
2. Change the function name to call from `baseUri()` to `baseURI()`.
3. In `ERC721Bridgeable` make a public function named `baseURI()` that returns the base URI value.

_NOTE: We planned to group this bug into only one issue, although it contains three different root causes as we illustrated. But we think that all of them relate to the same thing. If the Judger decides to separate into different issues, Please consider this issue to be dup for all separated issues (as it mentioned all root causes)._

---

## [L-03] Implementation contract left uninitialized and can be initialized

### Vulnerability Details
The `Bridge` contract will be `ERC1967`, and the implementation will be the `Bridge`. But If we checked the Bridge implementation we will find that it is left initialized, and anyone can initialize it.

This will allow anyone to initialize the implementation contract and take its OwnerShip.

### Recommendations
Prevent initializing the contract the implementation contract. This can be done by initializing `address(0)`, this will prevent initializing the implementation contract in the init logic implemented by the team. 

```diff
diff --git a/apps/blockchain/ethereum/src/Bridge.sol b/apps/blockchain/ethereum/src/Bridge.sol
index e62c7ce..b4d5175 100644
--- a/apps/blockchain/ethereum/src/Bridge.sol
+++ b/apps/blockchain/ethereum/src/Bridge.sol
@@ -35,6 +35,9 @@ contract Starklane is IStarklaneEvent, UUPSOwnableProxied, StarklaneState, Stark
     bool _enabled;
     bool _whiteListEnabled;
 
+    constructor() {
+        _initializedImpls[address(0)] = true;
+    }
 
     /**
        @notice Initializes the implementation, only callable once.
```

---

## [L-04] Checking `baseURI` value will succeed even if the returned string is empty

### Vulnerability Details
When Bridging tokens we are getting their URIs. We first try to get the baseURI, and if it contains value we just return it and ignore getting each NFT `tokenURI` separately.

[TokenUtil.sol#L89-L92](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/TokenUtil.sol#L89-L92)
```solidity
        // How the URI must be handled.
        // if a base URI is already present, we ignore individual URI
        // else, each token URI must be bridged and then the owner of the collection
        // can decide what to do
```

If we check how the returned string value is compared we will find out that the `returnedValue` will always pass the check.

[TokenUtil.sol#L158](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/token/TokenUtil.sol#L158)
```solidity
    function _callBaseUri(address collection) ... {
        ...
        for (uint256 i = 0; i < 2; i++) {
            bytes memory encodedParams = encodedSignatures[i];
            assembly {
❌️              success := staticcall(gas(), collection, add(encodedParams, 0x20), mload(encodedParams), 0x00, 0x20)
                if success {
                    returnSize := returndatasize()
❌️                  returnValue := mload(0x00)
                    ...
                }
            }
❌️          if (success && returnSize >= 0x20 && returnValue > 0) {
                return (true, abi.decode(ret, (string)));
            }
        }
        return (false, "");
    }
```

- when we make the call we are storing the returned data at slot `0x00`(returnOffset) with length `0x20`(returnSize)
- Then, we store this value in `returnValue`
- Then, we compare it to be greater than `0`

Now the issue is that the first `32 bytes` in the returned data of a string variable is not its value nor its length it is the offset we are starting from.

To make it simpler if the string is < 32 bytes in length the returned bytes will be as follows:
```shell
 [0x00 => 0x20]: String Length Offset
 [0x20 => 0x40]: String Length value
 [0x40 => 0x60]: String value in Hexa
```

Here is the returned bytes when the returned value is `baseURI()`:
```shell
 [000]: 0000000000000000000000000000000000000000000000000000000000000020
 [020]: 0000000000000000000000000000000000000000000000000000000000000009
 [040]: 6261736555524928290000000000000000000000000000000000000000000000
```

So when copying the first 32 bytes from slot `0x00` we are storing the offset not the length, which will make the value always be > 0, leading to pass the check even if `baseUri()` returned an empty string.

We should keep in mind that an empty string base URI is the default value for the base URI according to `ERC721`, so returning an empty string means it is not set, and we should not treat it as a valid `baseURI`.

### Impact
Passing baseURI even if it returns an empty string (not present), and not getting each tokenURIs value separately, which is not how the function should work.

### Tools Used
Manual Review and Foundry

### Recommendations
Assign `returnValue` to be the string length value, not the offset loading the value from the offset.

This can be made by copying the first `0x40` bytes of the string, where the first `0x20 bytes` will contain the offset and the second `0x20 bytes`will contain the length (in normal Solidity encoding).

So we will store `0x40` bytes at memory slot `0x00` this is normal as `0x00` and `0x20` are not used, and then `mload(0x20)` to get the length.

```diff
diff --git a/apps/blockchain/ethereum/src/token/TokenUtil.sol b/apps/blockchain/ethereum/src/token/TokenUtil.sol
index 41cc17d..1c0a26c 100644
--- a/apps/blockchain/ethereum/src/token/TokenUtil.sol
+++ b/apps/blockchain/ethereum/src/token/TokenUtil.sol
@@ -152,10 +152,10 @@ library TokenUtil {
         for (uint256 i = 0; i < 2; i++) {
             bytes memory encodedParams = encodedSignatures[i];
             assembly {
-                success := staticcall(gas(), collection, add(encodedParams, 0x20), mload(encodedParams), 0x00, 0x20)
+                success := staticcall(gas(), collection, add(encodedParams, 0x20), mload(encodedParams), 0x00, 0x40)
                 if success {
                     returnSize := returndatasize()
-                    returnValue := mload(0x00)
+                    returnValue := mload(0x20)
                     ret := mload(0x40)
                     mstore(ret, returnSize)
                     returndatacopy(add(ret, 0x20), 0, returnSize)
```

---

## [L-05] Existed collections are not whitelisted when Bridging

### Vulnerability Details
When Bridging NFTs between `L1<->L2`, we are whitelisting the NFT collection in the destination chain if it is not whitelisted, and since we are checking that depositing NFTs from the source chain should be from a whitelisted NFT, it is important to integrability between L1 and L2.

The problem is that we are only whitelisting the collection on the destination chain if it doesn't have an attachment to it on L1, whereas after deploying it we are whitelisting it. But if the collection already existed we are not whitelisting it if needed.

[Bridge.sol#L183-L196](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Bridge.sol#L183-L196)
```solidity
        if (collectionL1 == address(0x0)) {
            if (ctype == CollectionType.ERC721) {
                collectionL1 = _deployERC721Bridgeable( ... );
                // update whitelist if needed
❌️              _whiteListCollection(collectionL1, true);
            } else {
                revert NotSupportedYetError();
            }
        }
```

As we can see, whitelisting occurs only if the NFT collection does not exist on `L1`, so whitelisting the collection will not occur if the collection already exists.

We will discuss the scenario where this can occur and introduce problems in the `Proof of Concept` section.

### Proof of Concept

- whitelisting is disabled, and any NFT can be Bridged.
- NFTs are Bridged from L1, To L2.
- NFT collections on L1 are not whitelisted collections now (whitelisting is disabled).
- On L2 we are deploying addresses that maps to Collections on L1 and whitelist them. 
- the protocol enables whiteListing.
- NFTs collections that were created on L2 are now whitelisted and can be Bridged to L1, but still, these Collection L1 addresses are not whitelisted.
- We can Bridge from L2 to L1 easily as these collections are whitelisted.
- We can withdraw the NFT from L1 easily (there is no check for whitelisting when withdrawing).
- since `L1<->L2` mapping is already set. NFTs Collections are still `not-whitelisted` on L1.
- The process will end up with the ability to Bridge NFTs from `L2 to L1` as they are whitelisted on L2, but the inability to Bridge them from `L1 to L2` as they are not whitelisted on `L1`.

### Tools Used
Manual Review

### Recommendations
Whitelist the collection if it is newly deployed or if it already exists and is not whitelisted.

```diff
diff --git a/apps/blockchain/ethereum/src/Bridge.sol b/apps/blockchain/ethereum/src/Bridge.sol
index e62c7ce..e069544 100644
--- a/apps/blockchain/ethereum/src/Bridge.sol
+++ b/apps/blockchain/ethereum/src/Bridge.sol
@@ -188,13 +188,14 @@ contract Starklane is IStarklaneEvent, UUPSOwnableProxied, StarklaneState, Stark
                     req.collectionL2,
                     req.hash
                 );
-                // update whitelist if needed
-                _whiteListCollection(collectionL1, true);
             } else {
                 revert NotSupportedYetError();
             }
         }
 
+        // update whitelist if needed
+        _whiteListCollection(collectionL1, true);
+
         for (uint256 i = 0; i < req.tokenIds.length; i++) {
             uint256 id = req.tokenIds[i];
```

### Existence of the issue on the Starknet Side
This issue explains the flow from Ethereum to Starknet. the problem also existed in L2 side, where we are only whitelisting if there is no address for that collection on `L2`. If the `collection_l2 == 0`, we are deploying the collection on L2 and whitelist it. But if it is already existed we are just returning the address without whitelisting it if existed, same as that in L1.

[bridge.cairo#L440-L442](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L440-L442)
```cairo
    fn ensure_erc721_deployment(ref self: ContractState, req: @Request) -> ContractAddress {
        ...
        let collection_l2 = verify_collection_address( ... );

        if !collection_l2.is_zero() {
❌️          return collection_l2;
        }
        ...
        // update whitelist if needed
        let (already_white_listed, _) = self.white_listed_list.read(l2_addr_from_deploy);
        if already_white_listed != true {
            _white_list_collection(ref self, l2_addr_from_deploy, true);
            ...
        }
        l2_addr_from_deploy
    }
```

So if we changed the order, the Collections can be whitelisted on `L1`, but not on `L2` (if the Proof of Concept was in the reverse order), Bridging these NFTs collections from L1 to L2 will be allowed, but we will not be able to Bridge them from `L2` to `L1`, as they are not whitelisted on `L2`.

**Recommendations**: the same as that in `L1`, we need to whitelist collections if they already exist.

```diff
diff --git a/apps/blockchain/starknet/src/bridge.cairo b/apps/blockchain/starknet/src/bridge.cairo
index 23cbf8a..1d3521e 100644
--- a/apps/blockchain/starknet/src/bridge.cairo
+++ b/apps/blockchain/starknet/src/bridge.cairo
@@ -438,6 +438,15 @@ mod bridge {
         );
 
         if !collection_l2.is_zero() {
+            // update whitelist if needed
+            let (already_white_listed, _) = self.white_listed_list.read(l2_addr_from_deploy);
+            if already_white_listed != true {
+                _white_list_collection(ref self, l2_addr_from_deploy, true);
+                self.emit(CollectionWhiteListUpdated {
+                    collection: l2_addr_from_deploy,
+                    enabled: true,
+                });
+            }
             return collection_l2;
         }
```

_NOTE: if the issue will be judged as two separate issues. Please, consider making this report as duplicate of both of them._
