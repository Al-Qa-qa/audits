# PoolTogether
PoolTogether contest || Vaults, ERC4626	 || 4 March 2024 to 11 March 2024 on [code4rena](https://code4rena.com/audits/2024-03-pooltogether#top)

## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[H-01](#h-01-prizevaultclaimyieldfeeshares-subtracts-all-fees-whatever-the-value-passed-to-it)|`PrizeVault::claimYieldFeeShares()` subtracts all fees whatever the value passed to it|HIGH|
||||
|[M-01](#m-01-the-winner-can-steal-claimer-receipent-and-force-him-to-pay-for-the-gas)|The winner can steal claimer receipent, and force him to pay for the gas|MEDIUM|
|[M-02](#m-02-unchecking-for-the-actual-assets-withdrawn-from-the-vault-can-prevent-users-from-withdrawing)|Unchecking for the actual assets withdrawn from the vault can prevent users from withdrawing|MEDIUM|
||||
|[L-01](#l-01-claiming-yields-fees-can-be-done-in-recovery-mode-breaks-protocol-invariants)|Claiming Yields Fees can be done in `recovery mode` breaks protocol invariants|LOW|
|[L-02](#l-02-checking-previous-approval-before-permit-is-done-with-strict-equality)|Checking previous approval before `permit()` is done with strict equality|LOW|
|[L-03](#l-03-no-checking-for-breefy-vault-strategies-state-paused-or-not)|No checking for Breefy Vault strategies state (paused or not)|LOW|
||||
|[NC-01](#nc-01-assets-must-precede-the-shares-according-to-the-erc4626-standard)|Assets must precede the shares according to the ERC4626 standard|INFO|

---

## [H-01] `PrizeVault::claimYieldFeeShares()` subtracts all fees whatever the value passed to it.

### Impact

Fees are increased when the liquidator claims them and contributes to the prize, and the `yieldFeeReceipent` can claim them anytime he wants to.

The fees are claimed by letting `yieldFeeReceipent` mint them.

The function subtracts all fees from `yieldFeeBalance` (i.e. resetting it to zero), but mints to the `yieldFeeReceipent` the number of shares passed as a parameter only.

[src/PrizeVault.sol#L611-L622](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L611-L622)
```solidity
    function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
        if (_shares == 0) revert MintZeroShares();

        uint256 _yieldFeeBalance = yieldFeeBalance; // @audit getting all yeilds
        if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

        yieldFeeBalance -= _yieldFeeBalance; // @audit subtracting all yeilds

        _mint(msg.sender, _shares); // @audit minting only the _shares value passed by the `yieldFeeRecipent`

        emit ClaimYieldFeeShares(msg.sender, _shares);
    }
```

So the `yieldFeeRecipient` will lose its fees if he decides to claim a part from his yields.


### Further problems that will occur

This will make the `prizeVault` earning yield calculations goes incorrect.

Since `totalDept` is determined by adding `totalSupply` to the `yieldFeeBalance`.

[PrizeVault.sol#L790-L792C6](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L790-L792C6)
```solidity
    function _totalDebt(uint256 _totalSupply) internal view returns (uint256) {
        return _totalSupply + yieldFeeBalance;
    }
```

So `totalDept` will get decreased by a value greater than the value that `totalAssets` will increase (if the `yieldFeeRecipent` claimed part of his prize).

which will make `prizeVault` think it earns yields from `yieldVault` but it is from the fee recipient's remaining amount in reality. And this amount can be then claimed by the `LiquidationPair`.

The function is restricted to the `yieldFeeRecipent`, and decreasing `totalDept` by a value more than `totalAssets` will not cause any critical issues (DoS or others), it will just make `prizeVault` earn yields not from `yieldVault` (similar to donations state), so no further impacts will occur.


### Prood of Concept

Let's take a scenario to illustrate the point:

- The vault is receiving yields.
- Liquidators are claiming these yields and participate in the prize pool.
- `yieldFeeBalance` is increasing.
- `yieldFeeBalance` reaches 1000.
- The `yieldFeeRecipent` decided to claim 500.
- `yieldFeeRecipent` fires `claimYieldFeeShares()`, and passed 500 shares.
- `yieldFeeRecipent` mints 500 successfully.
- All `yieldFeeBalance` dropped to zero instead of being 500 (1000 - 500).

### Tools Used
Manual Review

### Recommended Mitigations

1. we can subtract the value of shares from `yieldFeeBalance`:

> PrizeVault::claimYieldFeeShares
```diff
    function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
        if (_shares == 0) revert MintZeroShares();

        uint256 _yieldFeeBalance = yieldFeeBalance;
        if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

-       yieldFeeBalance -= _yieldFeeBalance;
+       yieldFeeBalance -= _shares;

        _mint(msg.sender, _shares);

        emit ClaimYieldFeeShares(msg.sender, _shares);
    }
```

2. Or We can disallow the `yieldFeeRecipent` from claiming part of the prize and mint all fees to him:

> PrizeVault::claimYieldFeeShares
```solidity
    // The function after modification
    function claimYieldFeeShares() external onlyYieldFeeRecipient {
        if (yieldFeeBalance == 0) revert MintZeroShares();

        uint256 _yieldFeeBalance = yieldFeeBalance;

        yieldFeeBalance = 0;

        _mint(msg.sender, _yieldFeeBalance);

        emit ClaimYieldFeeShares(msg.sender, _yieldFeeBalance);
    }
```


---

## [M-01] The winner can steal claimer receipent, and force him to pay for the gas

### Impact
When the winner earns his reward he can either claim it himself, or he can let a claimer contract withdraw it on his behaf, and he will pay part of his fees for that. This is as the user will not pay for the gas fees, instead the claimer contract will pay it instead.

The problem here is that the winner can make the claimer pay for the gas of the transaction, without paying the fees that the claimer contract take.

Claimer contracts are allowed for anyone to use them, transfer prizes to winners and claim some fees. where the one who fired the transaction is the one who will pay for the fees, so he deserved that fees.

[pt-v5-claimer/Claimer.sol#L120-L150](https://github.com/GenerationSoftware/pt-v5-claimer/blob/main/src/Claimer.sol#L120-L150)
```solidity
  // @audit and one can call the function
  function claimPrizes( ... ) external returns (uint256 totalFees) {
    ...

    if (!feeRecipientZeroAddress) {
      ...
    }

    return feePerClaim * _claim(_vault, _tier, _winners, _prizeIndices, _feeRecipient, feePerClaim);
  }
```


As in the function, the function takes winners, and he called set his fees (but it should not exceeds the maxFees which is initialzed in constructor).

Now We know that anyone can transfer winners prizes and claim some fees.

---

Before the prizes are claimed, the winner can initialze a hook before calling the `PoolPrize::claimPrize`, and this is used if the winner want to initialze another address as the receiver of the reward.

The hook parameter is passed by parameters that are used to determine the correct winner (winner address, tier, prizeIndex).

[abstract/Claimable.sol#L85-L95](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/abstract/Claimable.sol#L85-L95)
```solidity
    uint24 public constant HOOK_GAS = 150_000;

    ...

    function claimPrize( ... ) external onlyClaimer returns (uint256) {
        address recipient;

        if (_hooks[_winner].useBeforeClaimPrize) {
            recipient = _hooks[_winner].implementation.beforeClaimPrize{ gas: HOOK_GAS }(
                _winner,
                _tier,
                _prizeIndex,
                _reward,
                _rewardRecipient
            );
        } else {
            recipient = _winner;
        }

        if (recipient == address(0)) revert ClaimRecipientZeroAddress();

        uint256 prizeTotal = prizePool.claimPrize( ... );
      
        ...
    }

```

But to prevent OOG the gas is limited to 150K.

---

Now What can the user do to make the claimer pay for the transaction, and do not pay any fees is:

- he will make a `beforeClaimPrize` hook
- In this function, the user will simply claim his reward `Claimer::claimPrizes(...params)` but with settings no fees, and only passing his winning prize parameters (we got them from the hook).
- The winner (attacker) will not do any further interaction to not make the tx go `OOG` (remember we have only 150k).
- After the user claims his reward, he will simply return his address (the winner's address).
- The Claimer contract will go to claim this winner's rewards, but it will return 0 as it is already claimed.
- The Claimer will complete his process (claiming other prizes on behalf of winners).
- The winner (attacker) will end up claiming his reward without paying for the transaction gas fees.

_Note: The Claimer claiming function will not revert, as if the prize was already claimed the function will just emit an event and will not revert_

[Claimer.sol#L194-L198](https://github.com/GenerationSoftware/pt-v5-claimer/blob/main/src/Claimer.sol#L194-L198)
```solidity
  function _claim( ... ) internal returns (uint256) {
    ...

        try
          _vault.claimPrize(_winners[w], _tier, _prizeIndices[w][p], _feePerClaim, _feeRecipient)
        returns (uint256 prizeSize) {
          if (0 != prizeSize) {
            actualClaimCount++;
          } else {
            // @audit Emit an event if the prize already claimed
            emit AlreadyClaimed(_winners[w], _tier, _prizeIndices[w][p]);
          }
        } catch (bytes memory reason) {
          emit ClaimError(_vault, _tier, _winners[w], _prizeIndices[w][p], reason);
        }

    ...
  }
```

The only Check that can prevent this attack is the gas cost of calling `beforeClaimPrize` hook.

We will call one function `Claimer::claimPrizes()` by only passing one winner, and without fees. We calculated the gas that can be used by installing protocol contracts (Claimer and PrizePool), then grap a test function that first the function we need, and we got these results:

- Calling `Claimer::claimPrize()` costs `5292 gas` if it did not claimed anything.
- Calling `PrizePool::claimePrize()` costs `118124 gas`.

So the total gas that can be used is $118,124 + 5292 = 123,416$. which is smaller than `HOOK_GAS` by more than `25K`, so the function will not revert because of OOG error, and the reentrancy will occur.

_Another thing that may lead to mis-understanding is that the Judger may say ok if this happens the function will go to `beforeClaimPrize` hook again leading to infinite loop and the transaction will go `OOG`. But making the transaction `beforeClaimPrize` be fired to make a result and when called again do another logic is an easy task that can be made by implementing a counter or something. However, we did not implement this counter in our test. We just wanted to point out how the attack will work in our POC, but in real interactions, there should be some edge cases to take care of and further configurations to take care off._

### Proof of Concept
We made a simulation of how the function will occur. We found that the testing environment made by the devs is abstracted a little bit compared to the real flow of transactions in the production mainnet, so I made Mock contracts, and simulated the attack with them. Please go for the testing script step by step, and it will work as intended.

1. Add the following Imports and scripts in [`test/Claimable.t.sol::L8`](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/test/unit/Claimable.t.sol#L8)

<details>

<summary>Imports and Contracts</summary>

```solidity
import { console2 } from "forge-std/console2.sol";
import { PrizePoolMock } from "../contracts/mock/PrizePoolMock.sol";

contract Auditor_MockPrizeToken {
    mapping(address user => uint256 balance) public balanceOf;

    function mint(address user, uint256 amount) public {
        balanceOf[user] += amount;
    }

    function burn(address user, uint256 amount) public {
        balanceOf[user] -= amount;
    }
}

contract Auditor_PrizePoolMock {
    Auditor_MockPrizeToken public immutable prizeToken;

    constructor(address _prizeToken) {
        prizeToken = Auditor_MockPrizeToken(_prizeToken);
    }

    // The reward is fixed to 100 tokens
    function claimPrize(
        address winner,
        uint8 /* _tier */,
        uint32 /* _prizeIndex */,
        address /* recipient */,
        uint96 reward,
        address rewardRecipient
    ) public returns (uint256) {
        // Distribute rewards if the PrizePool earns a reward
        if (prizeToken.balanceOf(address(this)) >= 100e18) {
            prizeToken.mint(winner, 100e18 - uint256(reward)); // Transfer reward tokens to the winner
            // Transfer fees to the claimer Receipent.
            // Instead of adding balance to the PrizePool contract and then the claimerRecipent
            // Can withdraw it, we will transfer it to the claimerRecipent directly in our simulation
            prizeToken.mint(rewardRecipient, reward);
             // Simulating Token transfereing by minting and burning
            prizeToken.burn(address(this), 100e18);
        } else {
            return 0;
        }

        return uint256(100e18);
    }
}

contract Auditor_Claimer {
    ClaimableWrapper public immutable prizeVault;

    constructor(address _prizeVault) {
        prizeVault = ClaimableWrapper(_prizeVault);
    }

    function claimPrizes(
        address[] calldata _winners,
        uint8 _tier,
        uint256 _claimerFees,
        address _feeRecipient
    ) external {
        for (uint i = 0; i < _winners.length; i++) {
            prizeVault.claimPrize(_winners[i], _tier, 0, uint96(_claimerFees), _feeRecipient);
        }
    }
}
```
</details>

2. Add the following functions in [`test/Claimable.t.sol::L132`](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/test/unit/Claimable.t.sol#L132)

<details>
<summary>Testing Functions</summary>

```solidity
  Auditor_Claimer __claimer;

  function testAuditor_winnerStealClaimerFees() public {
      console2.log("Winner reward is 100 tokens");
      console2.log("Fees are 10% (10 tokens)");
      console2.log("=============");
      console2.log("Simulating the normal Operation (No stealing)");
      auditor_complete_claim_proccess(false);
      console2.log("=============");
      console2.log("Simulating winner steal recipent fees");
      auditor_complete_claim_proccess(true);
  }

  function auditor_complete_claim_proccess(bool willSteal) internal {
      // If tier is 1 we will take the claimer fees and if 0 we will do nothing
      uint8 tier = willSteal ? 1 : 0;

      Auditor_MockPrizeToken __prizeToken = new Auditor_MockPrizeToken();
      Auditor_PrizePoolMock __prizePool = new Auditor_PrizePoolMock(address(__prizeToken));

      address __winner = makeAddr("winner");
      address __claimerRecipent = makeAddr("claimerRecipent");

      // This will be like the `PrizeVault` that has the winner
      ClaimableWrapper __claimable = new ClaimableWrapper(
          PrizePool(address(__prizePool)),
          address(1)
      );

      // Claimer contract, that can transfer winners rewards
      __claimer = new Auditor_Claimer(address(__claimable));
      // Set new Claimer
      __claimable.setClaimer(address(__claimer));

      VaultHooks memory beforeHookOnly = VaultHooks(true, false, hooks);

      vm.startPrank(__winner);
      __claimable.setHooks(beforeHookOnly);
      vm.stopPrank();

      // PrizePool earns 100 tokens from yields, and we picked the winner
      __prizeToken.mint(address(__prizePool), 100e18);

      address[] memory __winners = new address[](1);
      __winners[0] = __winner;

      // Claim Prizes by providing `__claimerRecipent`
      __claimer.claimPrizes(__winners, tier, 10e18, __claimerRecipent);

      console2.log("Winner PrizeTokens:", __prizeToken.balanceOf(__winner) / 1e18, "token");
      console2.log(
          "ClaimerRecipent PrizeTokens:",
          __prizeToken.balanceOf(__claimerRecipent) / 1e18,
          "token"
      );
  }
```
</details>

3. Change [`beforeClaimPrize`](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/test/unit/Claimable.t.sol#L254-L274) hook function, and replace it with the following.
```solidity
    function beforeClaimPrize(
        address winner,
        uint8 tier,
        uint32 prizeIndex,
        uint96 reward,
        address rewardRecipient
    ) external returns (address) {
        address[] memory __winners = new address[](1);
        __winners[0] = winner;

        if (tier == 1) {
            __claimer.claimPrizes(__winners, 0, 0, rewardRecipient);
        }

        return winner;
    }
```

4. Check that everything is correct and run

```powershell
forge test --mt testAuditor_winnerStealClaimerFees -vv
```

**Output:**
```powershell
  Winner reward is 100 tokens
  Fees are 10% (10 tokens)
  =============
  Simulating the normal Operation (No stealing)
  Winner PrizeTokens: 90 token
  ClaimerRecipent PrizeTokens: 10 token
  =============
  Simulating winner steal recipient fees
  Winner PrizeTokens: 100 token
  ClaimerRecipent PrizeTokens: 0 token
```

In this test, we first made a reward and withdraw it from our Claimer contract normally (no attack happened), and then we made another prize reward but by making the attack when withdrawing it, which can be seen in the Logs.

### Tools Used
Manual Review + Foundry

### Recommended Mitigation Steps
We can check the prize state before and after the hook and if it changed from unclaimed to claimed, we can revert the transaction.

> Claimable.sol
```diff
    function claimPrize( ... ) external onlyClaimer returns (uint256) {
        address recipient;

        if (_hooks[_winner].useBeforeClaimPrize) {
+           bool isClaimedBefore = prizePool.wasClaimed(address(this), _winner, _tier, _prizeIndex);
            recipient = _hooks[_winner].implementation.beforeClaimPrize{ gas: HOOK_GAS }( ... );
+           bool isClaimedAfter = prizePool.wasClaimed(address(this), _winner, _tier, _prizeIndex);

+           if (isClaimedBefore == false && isClaimedAfter == true) {
+               revert("The Attack Occuared");
+           }
        } else { ... }
        ...
    }

```

_NOTE: We wrote this issue 30min before ending of the contest so we did not checked for the grammar quality nor the words, and the mitigation review may not be the best, or may not work (we did not tested it), Devs should keep this in mind when mitigating this issue._

---

## [M-02] Unchecking for the actual assets withdrawn from the vault can prevent users from withdrawing

When withdrawing assets from `PrizeVault`, the contract assumes that the requested assets for redeeming are the actual assets the contract (PrizeVault) received.

[PrizeVault.sol#L936](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L936)
```solidity
    function _withdraw(address _receiver, uint256 _assets) internal {
        ...
        if (_assets > _latentAssets) {
            ...
            // @audit no checking for the returned value (assets)
            yieldVault.redeem(_yieldVaultShares, address(this), address(this));
        }
        if (_receiver != address(this)) {
            _asset.transfer(_receiver, _assets);
        }
    }
```

Since the calculations are done by rounding Up asset value when getting shares in `previewWithdraw`, and then rounding Down shares when getting assets, there should not be any 1 wei rounding error.

This will not be the case for all `yieldVaults` supported by `PrizeVault`, where the amount can decrease if there is no enough balance in the vault for example, as that of `Beefy Vaults`.

[BIFI/vaults/BeefyWrapper.sol#L156-L158](https://github.com/beefyfinance/beefy-contracts/blob/master/contracts/BIFI/vaults/BeefyWrapper.sol#L156-L158)
```solidity
    function _withdraw( ... ) internal virtual override {
        ...

        uint balance = IERC20Upgradeable(asset()).balanceOf(address(this));
        if (assets > balance) {
            // @audit the assets requested for withdrawal will decrease
            assets = balance;
        }

        IERC20Upgradeable(asset()).safeTransfer(receiver, assets);

        emit Withdraw(caller, receiver, owner, assets, shares);
    }
```

And ofc if there is a fee on withdrawals, the amount will decrease but this is OOS.

## Recommended Mitigations

Check the returned value of assets transferred after withdrawing, and take suitable action if the value differs from the value requested. And take suitable action like emitting events to let Devs know what problem occurred when this user withdrew.

---

## [L-01] Claiming Yields Fees can be done in `recovery mode` breaks protocol invariants

### Impact

When claiming yieldsFee by the `yieldFeeRecipent`, the function did not check if the vault is `normal state (winning)` or in the `recovery mode (losing)`.

[PrizeVault.sol#L611-L622](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L611-L622)
```solidity
    function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
        if (_shares == 0) revert MintZeroShares();

        uint256 _yieldFeeBalance = yieldFeeBalance;
        if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

        yieldFeeBalance -= _yieldFeeBalance;

        // @audit No checking if we are in recovery mode or not before minting
        _mint(msg.sender, _shares);

        emit ClaimYieldFeeShares(msg.sender, _shares);
    }
```

So the yieldFeeReceipent can claim his yields, minting new shares to him even if the protocol is in `recovery mode`.

According to protocol invariants, `no new deposits or mints allowed` in the recovery mode

> README
>> `Yield Vault has Loss of Funds (recovery mode)`
>> - no new deposits or mints allowed

So if the protocol is in recovery mode new mints can occur, which is an action the protocol should not perform.

After investigating the case, I found that the other protocol invariants will not affected. Where the `totalDept is determined by totalSupply + yieldFeeBalance`.

[PrizeVault.sol#L790-L792C6](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L790-L792C6)
```solidity
    function _totalDebt(uint256 _totalSupply) internal view returns (uint256) {
        return _totalSupply + yieldFeeBalance;
    }
```

So this will not change the calculations of `totalAssets` to the `totalDept` ratio, just invariant breaks, which made me label this issue as MEDIUM not HIGH.

## Proof of Concept

Here is a scenario where this could happen.

- People deposit their money in the `PrizeVault`.
- The protocol `yieldVault` earns yields.
- `LiquidationPair` did liquidation (claim yields), and participated in the prize pool.
- The process occurs again and again and the accumulated fees increase.
- `yieldVault` losses and the `prizeVault` recovery mode reached (totalAssets < totalDept).
- The `yieldFeeRecipent` claimed his fees, minting new shares to him in the recovery mode.
- Protocol invariants broke.

### Tools Used
Manual Review

### Recommended Mitigation Steps
Prevent claiming fees if the protocol is in the `recovery mode`.

```diff
    function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
        if (_shares == 0) revert MintZeroShares();

        uint256 _yieldFeeBalance = yieldFeeBalance;
        if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

+       require(totalAssets() >= totalDebt(), "Can't mint shares in recovery mode");
        yieldFeeBalance -= _yieldFeeBalance;

        _mint(msg.sender, _shares);

        emit ClaimYieldFeeShares(msg.sender, _shares);
    }
``` 

---

## [L-02] Checking previous approval before `permit()` is done with strict equality

In `PrizeVault::depositWithPermit`, the check that is implemented by the protocol to overcome griefing users by frontrunning permit signing, is to check if the owner allowance equals or does not equal the assets being transferred.

[PrizeVault.sol#L539-L541](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L539-L541)
```solidity
    function depositWithPermit(... ) external returns (uint256) {
        ...

        // @audit strict check (do not check if the allowance exceeds required)
        if (_asset.allowance(_owner, address(this)) != _assets) {
            IERC20Permit(address(_asset)).permit(_owner, address(this), _assets, _deadline, _v, _r, _s);
        }

        ...
    }
``` 

So if the user approval exceeds the amount he wanted to do, the permit still will occur.

- Let's say Bob approved 1000 tokens.
- Bob wants to transfer 500 tokens.
- Bob first the function `depositWithPermit()`.
- (1000 != 500), and Bob will have an allowance of 1500 tokens when he only needs 500.

## Recommended Mitigations
Allow permit if the allowance is Smaller than the required assets

```diff
    function depositWithPermit(... ) external returns (uint256) {
        ...

-       if (_asset.allowance(_owner, address(this)) != _assets) {
+       if (_asset.allowance(_owner, address(this)) < _assets) {
            IERC20Permit(address(_asset)).permit(_owner, address(this), _assets, _deadline, _v, _r, _s);
        }

        ...
    }
```
---

## [L-03] No checking for Breefy Vault strategies state (paused or not)

When depositing or minting into a vault, the check that is done by the `ERC4626` is checking the `maxDeposit` parameter.

This function `maxDeposit()` will return 0 if the Vault is in pause state like AAVE-v3 [link](https://github.com/timeless-fi/yield-daddy/blob/main/src/aave-v3/AaveV3ERC4626.sol#L161-L164).

But In case of breefy, when depositing The function go to the internal deposie function first

1. [BIFI/vaults/BeefyWrapper.sol#L117-L130](https://github.com/beefyfinance/beefy-contracts/blob/master/contracts/BIFI/vaults/BeefyWrapper.sol#L117-L130)
```solidity
    // BeefyWrapper.sol
    function _deposit( ... ) internal virtual override {
        ...
        // @audit Step 1
        IVault(vault).deposit(assets);

        ...
    }
```
2. [/BIFI/vaults/BeefyVaultV7.sol#L100-L115](https://github.com/beefyfinance/beefy-contracts/blob/master/contracts/BIFI/vaults/BeefyVaultV7.sol#L100-L115)
```solidity
    // BeefyVaultV7.sol
    function deposit(uint _amount) public nonReentrant {
        // @audit Step 2
        strategy.beforeDeposit();

        ...
    }
```
3. [BIFI/strategies/Aave/StrategyAaveSupplyOnlyOptimism.sol#L104-L109](https://github.com/beefyfinance/beefy-contracts/blob/master/contracts/BIFI/strategies/Aave/StrategyAaveSupplyOnlyOptimism.sol#L104-L109)
```solidity
    // @audit Step 3
    // StrategyAaveSupplyOnlyOptimism.sol (We took AAVE Optmism as an example)
    function beforeDeposit() external override {
        if (harvestOnDeposit) {
            require(msg.sender == vault, "!vault");
            _harvest(tx.origin);
        }
    }
```
4. [BIFI/strategies/Aave/StrategyAaveSupplyOnlyOptimism.sol#L124-L139](https://github.com/beefyfinance/beefy-contracts/blob/master/contracts/BIFI/strategies/Aave/StrategyAaveSupplyOnlyOptimism.sol#L124-L139)
```solidity
    // @audit Step4
                                         /* ︾ */  
    function _harvest( ... ) internal whenNotPaused gasThrottle {
        ...
    }
```

So if the Breedy strategy vault is paused, the depositing will revert, and can not occur.

## Recommended Mitigations
Checking if the vault is paused or not is not directly supported, we need to check the strategy contract itself. However since the beefy wrapper supports more than one Strategy, checking the state may differ from one Strategy to another, but if all Strategies have the same interface, the check can be implemented before depositing.

---
---

## [NC-01] Assets must precede the shares according to the ERC4626 standard

In the implementation of `PrizeVault::_burnAndWithdraw()`, the function takes shares then it takes the assets parameter at the end. And this is not the way assets and shares are provided to the ERC4626 functions.

[PrizeVault.sol#L887-L893](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L887-L893)
```solidity
    function _burnAndWithdraw(
        address _caller,
        address _receiver,
        address _owner,
        uint256 _shares, // @audit Order should be (_assets, _shares) as that of ERC4626
        uint256 _assets
    ) internal {
        ...
    }
```

All functions depositing/minting/withdrawing/redeeming in ERC4626 make the assets parameter first then shares.

### Recommended Mitigations
Make the assets first, then the shares at the last in `PrizeVault::_burnAndWithdraw()`.

_NOTE: You will have to change all the calling of this function in PrizeVault + testing scripts files_.
