# Mystic Finance - Findings Report

Prepared by: Nwebor Stephen

Auditors:

- [Stephen](https://github.com/Derastephh)
- [Greg](https://x.com/0xitsgreg)

# Table of contents
<details>

<summary>See table</summary>

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Missing Collateral Tracking in `updateLeverageBundle` Can Cause Accounting Drift, Locking User Funds or Break Close Position](#H-01)
    - ### [H-02. Anyone Can Call `stPlumeMinter::restake` to Continously Restake and Block Withdrawals](#H-02)
    - ### [H-03. Multiple Counting in currentWithheldETH allows Deletion of User Fund, Without User Receiving Value, or Just Straight Reverts](#H-03)
    - ### [H-04. Volatile Price-Based Keys Lock Leveraged Positions Permanently](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Missing Slippage Protection in Morpho/Mystic LeverageBundler Allows Collateral Theft on Position Close](#M-01)

</details>
</br>

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Mystic Finance

### Dates: May 13th, 2025 - May 18th, 2025

[See more contest details here](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings?created_by=greg,derastephh&status=duplicate,disputed,rejected,confirmed,acknowledged,fixed)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 4
- Medium: 1
- Low: 0

</br>


# High Risk Findings

## <a id='H-01'></a>H-01. Missing Collateral Tracking in `updateLeverageBundle` Can Cause Accounting Drift, Locking User Funds or Break Close Position

### Summary
The `MysticLeverageBundler::updateLeverageBundle` function forgets to update internal collateral tracking when leverage is adjusted. Over time, this causes a mismatch between what the contract thinks a user has and what is actually onchain. This leads to broken position closes or stuck collateral that the user cannot recover.

### Finding Description
When a user increases or decreases leverage, the function calls `MysticLeverageBundler::updatePositionTracking` to update borrow and collateral values. However, it always passes 0 for the collateral value:
```solidity
// MysticLeverageBundler::updatePositionTracking
if (borrowDelta > 0) {
    // user is increasing leverage
    bundler.multicall(mainBundle); // ← supplies extra collateral on-chain
    updatePositionTracking(pairKey, additionalBorrowAmount, 0, msg.sender, true);
                      // ------------------- collateral is not updated!
} else if (borrowDelta < 0) {
    // user is decreasing leverage
    bundler.multicall(mainBundle); // ← withdraws collateral on-chain
    updatePositionTracking(pairKey, repayAmount, 0, msg.sender, false);
                      // ------------------- collateral is not updated!
}
```

`MysticLeverageBundler::updatePositionTracking(bytes32, uint256 borrow, uint256 collateral, …)` is the sole entry-point that mutates the bookkeeping mappings:
```solidity
totalCollaterals[pairKey]           ±= collateral;
totalCollateralsPerUser[pairKey][u] ±= collateral;
But inside the mainBundle, actual collateral does move onchain. For example:
```

In leverage increase:
```solidity
_createMaverickSwapCall(...)                 // swaps borrow → collateral
_createMysticSupplyCall(collateralAsset,...) // supplies collateral to Mystic
In leverage decrease:

_createMysticWithdrawCall(...)               // withdraws collateral from Mystic
    _createMaverickSwapCall(...)                 // swaps collateral → borrow
```

So collateral is changing on-chain, but the contract doesn't record those changes internally.

In contrast, when a position is opened or closed, both borrow and collateral are tracked correctly:
```solidity
// Opening
updatePositionTracking(pairKey, borrowAmount, totalCollateral, msg.sender, true);

// Closing
updatePositionTracking(pairKey, debtToCover, collateralForRepayment, msg.sender, false);
```

This makes `MysticLeverageBundler::updateLeverageBundle` the only path where internal collateral tracking is skipped.

### Impact Explanation

Because collateral values aren't updated in `MysticLeverageBundler::updateLeverageBundle`, the internal records drift over time:

- After repeated leverage increase, the internal record shows too little collateral.
- After repeated leverage decrease, it shows too much.

This breaks close position logic:
```solidity
// MysticLeverageBundler.sol
collateralForRepayment = 
    totalCollateralsPerUser[pairKey][user] * debtToClose 
  / totalBorrowsPerUser[pairKey][user];
```

If `totalCollateralsPerUser` is too high, this formula overestimates how much collateral is available to swap. The contract tries to swap more than it actually has, leading to a flashloan repayment failure and a revert.

If `totalCollateralsPerUser` is too low, the user ends up recovering less than they actually deposited, with the rest stuck in Mystic forever.

### Likelihood Explanation
This bug is guaranteed to happen if users interact with `MysticLeverageBundler::updateLeverageBundle`:

- It happens on the first leverage adjustment, and worsens each time.
- Any user or vault adjusting leverage will eventually trigger the error.
- There is no function to resync internal records with actual values on Mystic.

### Proof of Concept:

<details>
<summary>Code</summary>

Paste the following test code in `bundler3/test/AaveLeverageBundlerTest.sol` file and run the test with `forge test --fork-url https://rpc.plume.org --mt testDecreaseLeverage -vv` and `forge test --fork-url https://rpc.plume.org --mt testIncreaseLeverage -vv`.

```solidity
function testDecreaseLeverage() public {
        // Get initial position data
        PositionData memory before = getPositionData(USER, address(borrowToken), address(collateralToken));
        uint256 initialUserBalance = collateralToken.balanceOf(USER);
        
        // Execute the open leverage function
        vm.startPrank(USER);
        Call[] memory bundleCalls = leverageBundler.createOpenLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            address(collateralToken),
            INITIAL_COLLATERAL,
            LEVERAGE_2X,
            DEFAULT_SLIPPAGE
        );
        
        // Get position data afterVal operation
        PositionData memory afterVal = getPositionData(USER, address(borrowToken), address(collateralToken));

        // Call updateLeverageBundle() with decreasing leverages repeatedly
        Call[] memory bundless = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            LEVERAGE_1_5X,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bundlesss = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            20000,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bundlessss = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            15000,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bundl = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            10001,
            DEFAULT_SLIPPAGE
        );

        // Attempt to close fails
        vm.expectRevert();
        Call[] memory bundles = leverageBundler.createCloseLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            type(uint256).max
        );
        vm.stopPrank();
}

function testIncreaseLeverage() public {
        PositionData memory before = getPositionData(USER, address(borrowToken), address(collateralToken));
        uint256 initialUserCollateralBalance = collateralToken.balanceOf(USER);
        uint256 initialUserBorrowBalance = borrowToken.balanceOf(USER);
        console.log("initialUserCollateralBalance: ", initialUserCollateralBalance);
        console.log("initialUserBorrowBalance: ", initialUserBorrowBalance);
        
        // Execute the open leverage function
        vm.startPrank(USER);
        Call[] memory bundleCalls = leverageBundler.createOpenLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            address(collateralToken),
            INITIAL_COLLATERAL,
            LEVERAGE_2X,
            DEFAULT_SLIPPAGE
        );
        
        // Get position data afterVal operation
        PositionData memory afterVal = getPositionData(USER, address(borrowToken), address(collateralToken));

        // Call updateLeverageBundle() with increasing leverages repeatedly
        Call[] memory bundless = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            40000,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bundlesss = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            50000,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bundlessss = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            60000,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bundl = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            70000,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bund = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            150000,
            DEFAULT_SLIPPAGE
        );

        Call[] memory bundles = leverageBundler.createCloseLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            type(uint256).max
        );

        uint256 currentUserCollateralBalance = collateralToken.balanceOf(USER);
        uint256 currentUserBorrowBalance = borrowToken.balanceOf(USER);
        console.log("currentUserCollateralBalance: ", currentUserCollateralBalance);
        console.log("currentUserBorrowBalance: ", currentUserBorrowBalance);
        vm.stopPrank();
    }
```
</details>

### Recommendation

- Fix the bug by correctly tracking collateral movements in `MysticLeverageBundler::updateLeverageBundle`:
```solidity
if (borrowDelta > 0) {
    bundler.multicall(mainBundle);
    updatePositionTracking(pairKey, additionalBorrowAmount, addedCollateral, msg.sender, true);
} else if (borrowDelta < 0) {
    bundler.multicall(mainBundle);
    updatePositionTracking(pairKey, repayAmount, removedCollateral, msg.sender, false);
}
```

</br>



## <a id='H-02'></a>H-02. Anyone Can Call `stPlumeMinter::restake` to Continously Restake and Block Withdrawals

### Summary

The `stPlumeMinter::restake` function can be called by anyone. This lets any external user unstake any ETH meant for withdrawal with any validator ID they choose, as long as this is the existing validator that users use for deposit, stake, etc. It also allows them to mint `frxETHToken` to the contract without checks. This can lead to unchecked minting of `frxETHToken`, blocked withdrawals, and abuse of the staking system.

### Finding Description

The `stPlumeMinter::restake` function is missing access control. It can be called by anyone and is not limited to protocol roles. Here is the function:
```solidity
function restake(uint16 validatorId) external nonReentrant returns (uint256 amountRestaked) {
    _rebalance(); // Calls internal reward/ETH-handling logic
    (StakeInfo memory info) = plumeStaking.stakeInfo(address(this));
    amountRestaked = plumeStaking.restake(
        validatorId,
        info.cooled + info.parked
    );
    emit Restaked(address(this), validatorId, amountRestaked);
}
```
When called, it performs three things after first calling `stPlumeMinter::_rebalance`:

Claims ETH rewards through `stPlumeMinter::_claim`, even if it’s not a real claim:
```solidity
uint256 amount = _claim();
Mints frxETH using the claimed amount:
frxETHToken.minter_mint(address(this), amount);
Restakes all ETH from cooled and parked balances:
plumeStaking.restake(validatorId, info.cooled + info.parked);
```
It also allows ETH meant for user withdrawals to be restaked without any check. There are no restrictions on the validator chosen. Validator ID just has to be valid. So, if a user tries to withdraw later, it will revert due to restaked balances. This results in the user not getting their ETH they previously unstaked, supposedly ready for withdrawals.

Exploit scenario:

- user deposits and stake eth.
- user unstakes and waits for cooldown.
- attacker watches and calls restake, restaking available ETH in a validator ID.
- user withdraw reverts.

### Impact Explanation

This allows attackers to:

Restake user funds meant for withdrawals, thereby blocking withdrawals.
Mint unbacked `frxETHToken`, possibly breaking the 1:1 ETH peg.
Continously lock up user funds and break trust in the system.
This is a major privilege escalation and puts user funds at risk of being continously frozen.

### Likelihood Explanation

This is very likely to be exploited. The function is publicly exposed, has no access checks, and can be triggered by any user. A motivated attacker could easily script repeated abuse.

### Proof of Concept

<details>
<summary>Code</summary>

Paste the following test in `Liquid-Staking/test/fork/stPlumeMinter.t.sol` file and run the test with `forge test --fork-url https://rpc.plume.org --mt testRestake -vv`

```solidity
function testRestake() public {
        address attacker = makeAddr("attacker");
        // user2 and user1 deposits, stakes and unstakes
        vm.startPrank(user2);
        minter.submit{value: 5 ether}();

        frxETHToken.approve(address(minter), 2 ether);
        minter.unstake(1 ether);
        uint256 previousUser2EthBalance = user2.balance;
        
        vm.stopPrank();

        vm.startPrank(user1);
        minter.submit{value: 6 ether}();

        frxETHToken.approve(address(minter), 3 ether);
        minter.unstake(2 ether);
        uint256 previousUser1EthBalance = user1.balance;
        user2.balance;
        
        vm.stopPrank();

        vm.warp(block.timestamp + 3 days);

        // attacker calls restake() befor user1 and user2 tries to withdraw
        vm.prank(attacker);
        minter.restake(1);

        // withdrawal fails for both users
        vm.prank(user2);
        vm.expectRevert();
        minter.withdraw(user2);

        vm.prank(user1);
        vm.expectRevert();
        minter.withdraw(user1);

        uint256 currentUser2EthBalance = user2.balance;
        uint256 currentUser1EthBalance = user1.balance;

        assertEq(currentUser1EthBalance, previousUser1EthBalance);
        assertEq(currentUser2EthBalance, previousUser2EthBalance);
    }
```
</details>

### Recommendation

- Add proper access control to the `stPlumeMinter::restake` function, only trusted roles should be able to call it.
- Or add a check requiring that users calling `stPlumeMinter::restake` are users who have existing non-zero `request.amount` and it should only stake the available user `request.amount` and not every user unstaked ETH balance for that validator ID.

</br>






## <a id='H-03'></a>H-03. Multiple Counting in currentWithheldETH allows Deletion of User Fund, Without User Receiving Value, or Just Straight Reverts

### Summary

The contract adds the same contract ETH balance to `currentWithheldETH` multiple times, during any functions that call `_stPlumeMinter::_rebalance`. This causes the internal ETH tracking to become inflated. When a user tries to withdraw, the contract believes there’s enough ETH and proceeds but if the actual ETH balance is lower, the call to send ETH fails silently. The user balance is still cleared, resulting in permanent loss of funds or even if the .call has a require success check, the withdraw function will still always reverts, and user cannot receive funds, leading to lock of funds.

### Finding Description

When `_stPlumeMinter::unstake` is called, the `_stPlumeMinter::_rebalance` is first called which later calls `_stPlumeMinter::depositEther`:

```solidity
function unstake(uint256 amount) external nonReentrant returns (uint256 amountUnstaked) {
        _rebalance();
        ....
}
 // then calls internal depositEther()
function _rebalance() internal {
        uint256 amount = _claim();
        frxETHToken.minter_mint(address(this), amount);
        frxETHToken.transfer(address(sfrxETHToken), amount);
        depositEther(address(this).balance);
    }
```

Multiple calls to `_stPlumeMinter::rebalance` adds the entire `address(this).balance` to `currentWithheldETH` again and again if it’s below the stake threshold `(0.1 ether)`:
```solidity
function depositEther(uint256 _amount) internal returns (uint256 depositedAmount) {
    .....
if (_amount < plumeStaking.getMinStakeAmount()) {
    currentWithheldETH += _amount; // increases with each call
    return 0;
}
.....
}
```
This leads to `currentWithheldETH` tracking more ETH than the contract actually holds.

Withdrawals then use this inflated value:
```solidity
function withdraw(address recipient) external nonReentrant returns (uint256 amount) {
 ......
amount = request.amount;
uint256 withdrawn;
request.amount = 0; // clears user withdraw request
request.timestamp = 0; // // clears user withdraw timestamp
.....
if (amount <= currentWithheldETH) {
    currentWithheldETH -= amount;
    ....
    recipient.call{value: amount}(""); // fails silently if balance too low
}
}
```
So if `currentWithheldETH` says there is enough ETH but `address(this).balance` is actually lower, the withdrawal proceeds, internal records are cleared, but no ETH is received by the user. Even if the `.call` checks for success, the withdraw function will still always reverts, and user cannot receive funds, leading to lock of funds.

### Impact Explanation

This vulnerability silently wipes out user funds. A user might trigger a withdrawal thinking they are getting ETH, but receive nothing if the internal accounting was inflated. Even worse, their ETH claim is still cleared from the system. There’s no error, no revert, and no way to recover the lost ETH. Over time, as more counting happens, this can affect more users and leave the protocol in a broken state where nobody can safely withdraw.

Even if the `.call` checks for success, the withdraw function will still always reverts, and user cannot receive funds, leading to lock of funds.

### Likelihood Explanation

This is very likely to happen as it can be triggered through regular use (e.g., unstake, withdraw, etc). A user does not have to do anything malicious. But a malicious user can intentionally cause it by triggering repeated counts and then withdrawing, blocking other users chances of withdrawal. It's easy to trigger and has no protections in place.

### Proof of Concept

<details>
<summary>Code</summary>

Paste the following test in `Liquid-Staking/test/fork/stPlumeMinter.t.sol` file and run the test with `forge test --fork-url https://rpc.plume.org --mt test_Fakewithdraw -vv`

```solidity
    function test_Fakewithdraw() public {
        // First submit ETH
        vm.prank(user1);
        minter.submit{value: 5 ether}();

        vm.startPrank(user1);
        frxETHToken.approve(address(minter), 1 ether);
        minter.unstake(1);

        vm.warp(block.timestamp + 1 days);

        minter.withdraw(user1);
        console.log("user1 eth balance before withdraw: ", user1.balance); // 95.000000000000000001
        minter.unstake(0.002 ether);

        vm.warp(block.timestamp + 1 days);

        minter.withdraw(user1);
        // user receives no ETH value
        console.log("user1 eth balance after withdraw: ", user1.balance); // 95.000000000000000001
        vm.stopPrank();
    }
```
</details>

### Recommendation

- Never count ETH in `currentWithheldETH`. Only increase this variable when actual new ETH enters.
- Add a balance check before withdrawals.

</br>









## <a id='H-04'></a>H-04. Volatile Price-Based Keys Lock Leveraged Positions Permanently

### Summary

Leveraged positions in `MysticLeverageBundler.sol` are tracked using a pairkey that includes the live price ratio between the borrowToken and `collateralToken`. Since prices keep changing, the `pairkey` used to open a position is different from the `pairkey` calculated when trying to close or update it later. This makes the contract lose track of the user’s position, preventing them from managing or closing it. Sometimes, the funds get stuck permanently.

### Finding Description

Position key depends on current price ratio, the contract calculates a position key like this:
```solidity
// MysticLeverageBundler.sol
function getPairKey(address borrowToken, address collateralToken) public view returns (bytes32) {
    uint256 borrowPrice = mysticAdapter.getAssetPrice(borrowToken);
    uint256 collateralPrice = mysticAdapter.getAssetPrice(collateralToken);

    uint256 ratio = (collateralPrice * SLIPPAGE_SCALE) / borrowPrice; // current price ratio

    require(
        ratio <= SLIPPAGE_SCALE + 500 && ratio >= SLIPPAGE_SCALE - 500, // ±5% allowed
        "Price deviation too high for safe leverage"
    );

    return keccak256(abi.encodePacked(borrowToken, collateralToken, ratio));
}
```
Because ratio uses live prices, it changes as prices move.

When opening a position, the system saves the user’s data with that `pairkey`:

```solidity
// MysticLeverageBundler::_createOpenLeverageBundleWithFlashloan
bytes32 pairKey = getPairKey(asset, collateralAsset); // uses price at time t₀
......
updatePositionTracking(pairKey, borrowAmount, collateralAmount, user, true);
```
Later, when the user tries to close or change the position, the key is calculated again with current prices:
```solidity
// MysticLeverageBundler::createCloseLeverageBundle
bytes32 pairKey = getPairKey(asset, collateralAsset); // uses price at time t₁ (t₁ ≠ t₀)

uint256 debtToClose = totalBorrowsPerUser[pairKey][msg.sender];
require(debtToClose > 0, "no debt found");  // This fails if key changed
```

Since prices have changed, the new key points to empty storage. The contract thinks the user has no debt and reverts.

Also in `MysticLeverageBundler::getPairKey`, if price moves more than 5%, the key function reverts directly:
```solidity
// MysticLeverageBundler::getPairKey
require(ratio within ±5%, "Price deviation too high for safe leverage");
```
If the price difference is too large, the key function refuses to return a key, making all position management functions unusable. This exact issue also exists in `MorphoLeverageBundler` with the same logic.

### Impact Explanation

- Normal price fluctuations cause the key to change.
- Positions become unreachable because the system looks up a different key than the one used to save them.
- Users cannot close, update, or manage their positions.
- Funds can get stuck and locked forever.
- When price changes exceed 5%, users are completely locked out.

### Likelihood Explanation

Price changes are frequent and expected in any market, especially crypto. So this bug is very likely to affect users anytime prices move.

### Proof of Concept

<details>
<summary>Code</summary>

For the PoC, i created minimized mock contracts to be able to simulate price changes. Create a `POC.t.sol` file under `/bundler3/test/poc` folder and paste the below code in the file. Run test with `forge test --mt testPairKeyChangeWithPrice -vv`

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.0;

import {ErrorsLib} from "../../src/libraries/ErrorsLib.sol";
import {MysticLeverageBundler} from "../../src/calls/MysticLeverageBundler.sol";
import {IMysticAdapter} from "../../src/interfaces/IMysticAdapter.sol";
import {IMaverickV2Pool} from "../../src/interfaces/IMaverickV2Pool.sol";
import {IMaverickV2Factory} from "../../src/interfaces/IMaverickV2Factory.sol";
import {IMaverickV2Quoter} from "../../src/interfaces/IMaverickV2Quoter.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IBundler3, Call} from "../../src/interfaces/IBundler3.sol";
import {MysticAdapter} from "../../src/adapters/MysticAdapter.sol";
import {Bundler3, Call} from "../../src/Bundler3.sol";
import "../../lib/forge-std/src/Test.sol";
import "../helpers/mocks/ERC20Mock.sol";
import {ICreditDelegationToken} from "../../src/interfaces/ICreditDelegationToken.sol";
import {MaverickSwapAdapter} from "../../src/adapters/MaverickAdapter.sol";
import {IMysticV3 as IPool, ReserveDataMap as ReserveData} from "../../src/interfaces/IMysticV3.sol";

contract MockMysticAdapter {
    mapping(address => uint256) public prices;
    uint256 public constant SLIPPAGE_SCALE = 10000;

    constructor() {}

    function setPrice(address asset, uint256 price) external {
        prices[asset] = price;
    }

    function getAssetPrice(address asset) external view returns (uint256) {
        return prices[asset];
    }

    function getPairKey(address borrowToken, address collateralToken) external view returns (bytes32) {
        uint256 borrowPrice = prices[borrowToken];
        uint256 collateralPrice = prices[collateralToken];
        require(borrowPrice > 0 && collateralPrice > 0, "Missing price");
        
        uint256 ratio = (collateralPrice * SLIPPAGE_SCALE) / borrowPrice;

        require(
            ratio <= SLIPPAGE_SCALE + 500 && ratio >= SLIPPAGE_SCALE - 500,
            "Price deviation too high for safe leverage"
        );

        return keccak256(abi.encodePacked(borrowToken, collateralToken, ratio));
    }
}


contract LeverageTest is Test {
    address borrowToken = address(0xB0);
    address collateralToken = address(0xC0);

    MockMysticAdapter mysticAdapter;

    function setUp() public {
        mysticAdapter = new MockMysticAdapter();

        mysticAdapter.setPrice(borrowToken, 1000e6);
        mysticAdapter.setPrice(collateralToken, 1040e6);

    }

    function testPairKeyChangeWithPrice() public {
        // pairKey with price t1
        bytes32 keyBefore = mysticAdapter.getPairKey(borrowToken, collateralToken);

        // simulate price changes within the 5% range
        mysticAdapter.setPrice(borrowToken, 1000e6);
        mysticAdapter.setPrice(collateralToken, 1030e6);

        // pairKey with price t2 (when price changes)
        bytes32 keyAfter = mysticAdapter.getPairKey(borrowToken, collateralToken);

        // assert that the pairkey are different
        assertTrue(keyBefore != keyAfter, "Pair key should change with price update");

        console.logBytes32(keyBefore);
        console.logBytes32(keyAfter);

    }
}
```
</details>

### Recommendation

- Don’t use live price-based keys to track positions.
- Allow users to update or close positions using stable keys regardless of price changes.
- Handle price deviations in a way that doesn’t block basic position management.

</br>














# Medium Risk Findings

## <a id='M-01'></a>M-01. Missing Slippage Protection in Morpho/Mystic LeverageBundler Allows Collateral Theft on Position Close

### Summary

When a user closes a leveraged position using `MorphoLeverageBundler` or `MysticLeverageBundler`, one of the swaps in `MysticLeverageBundler::_createCloseLeverageBundleWithFlashloan` has no slippage protection. This opens the door for attackers to manipulate the price in the chosen Maverick pool and steal nearly all of the user collateral during the close.

### Finding Description

In both `MorphoLeverageBundler` and `MysticLeverageBundler`, the helper function `MysticLeverageBundler::_createCloseLeverageBundleWithFlashloan` builds a bundle that includes two swaps.

The second swap uses `amountOutMin = 0` and `slippage = 0` as seen below:
```solidity
// ❶ after the flash-loan
mainBundle[1] = _createERC20TransferCall(borrowAsset,
                                         address(maverickAdapter),
                                         type(uint256).max);

/* ❷ critical swap – ZERO slippage protection */
mainBundle[2] = _createMaverickSwapCall(
    borrowAsset,             // tokenIn
    collateralAsset,         // tokenOut
    type(uint256).max,       // amountIn: “use full balance”
    0,                       // amountOutMin: **zero**
    0,                       // slippage:     **zero**
    false
);
```
These values are passed to the `MaverickSwapAdapter::swapExactTokensForTokens` function which recomputes `amountOutMin` whenever `amountIn == type(uint256).max`:

```solidity
if (amountIn == type(uint256).max || amountIn > balance) {
    amountIn     = balance;
    amountOutMin = amountIn * slippage / SLIPPAGE_SCALE;   // = 0 because slippage = 0
}

require(amountOutReceived >= amountOutMin, "Insufficient output amount");
```
Here’s what happens:

The MaverickAdapter sees `slippage = 0` and `amountIn = max`, and calculates `amountOutMin = 0`.
As a result, the swap will succeed no matter how little the user receives, even `1 wei`.
The system selects the Maverick pool to swap through based only on the largest total reserves, without considering actual value or price.
This setup gives attackers everything they need:

- They can create and fund a fake pool to make sure it’s selected.
- Then they manipulate the pool price before and after the user's bundle, draining the collateral from the second swap.

### Impact Explanation

The user may lose almost all of their remaining collateral when they close a leveraged position. This attack works during normal closes, and since the transaction doesn’t revert, the user may not even notice right away. The impact scales with the size of the position, so larger users are more heavily affected.

### Likelihood Explanation

- It only requires pool seeding and basic frontrunning.
- The slippage bypass makes the swap highly predictable and abusable.
- The naive pool selection makes it easy for attackers to control which pool gets picked.

### Proof of Concept

<details>
<summary>Code</summary>

A minimalist poc tp prove `amountOutMin = 0`. Paste the following test code in `bundler3/test/AaveLeverageBundlerTest.sol` file and run the test with `forge test --fork-url https://rpc.plume.org --mt testSlippage -vv` and confirm `amountOutMin = 0`

```solidity
// _swapExactTokensForTokens() taken from MaverickAdapter.sol

function _swapExactTokensForTokens(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOutMin,
        uint256 slippage,
        address to,
        int32 tickRange
    ) internal returns (uint256 amountOut) {
        uint256 SLIPPAGE_SCALE = 10000;
        // Get the best pool for this pair
        // IMaverickV2Pool pool = getBestPool(tokenIn, tokenOut);
        uint256 balance = IERC20(tokenIn).balanceOf(address(this));

        if (amountIn == type(uint256).max || amountIn > balance) {
            amountIn = balance;
            amountOutMin = amountIn * slippage / SLIPPAGE_SCALE;
            console.log("amountOutMin: ", amountOutMin); // confirm amountOutMin = 0
        }
    }

    function testSlippage() public {
        uint256 amountOut = _swapExactTokensForTokens(
            address(borrowToken),
            address(collateralToken),
            type(uint256).max,
            0,
            0,
            USER,
            0
        );
    }
```
</details>

Exploit PoC:

- The attacker lowers the price of the borrowed token (borrowToken) from, say, 100 to 50 (or raises the `collateralToken` price). This makes the flashloan repayment need more collateral than normal, that is, the victim now needs to pay 200 tokens instead of 100.
- The victim’s close position transaction runs and repays the flashloan with these inflated amounts. The leftover borrowed tokens (the surplus) are sent to the next swap step.
- Right before the second swap happens, the attacker quickly raises the borrowed token price back from 50 to 100, making the swap extremely unfavorable. Since the swap has no minimum output or slippage protection, it will accept any output, even as low as 1 token instead of the expected 100.
- Finally, the attacker resets the price back to normal, hiding the manipulation and capturing the large difference. The victim’s transaction finishes without errors, but almost all their collateral is stolen.

Because the transaction never reverts, the victim’s position closes successfully, but almost all of their remaining collateral is siphoned off by the attacker through this price manipulation attack.

### Recommendation

- Require a non-zero slippage and set a meaningful `amountOutMin`.
- Improve the pool selection logic to account for price impact, not just reserve size.