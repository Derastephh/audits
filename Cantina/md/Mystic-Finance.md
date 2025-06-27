# [H1] Missing Collateral Tracking in `updateLeverageBundle` Can Cause Accounting Drift, Locking User Funds or Break Close Position

## Summary
The `MysticLeverageBundler::updateLeverageBundle` function forgets to update internal collateral tracking when leverage is adjusted. Over time, this causes a mismatch between what the contract thinks a user has and what is actually onchain. This leads to broken position closes or stuck collateral that the user cannot recover.

## Finding Description
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

## Impact Explanation

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

## Likelihood Explanation
This bug is guaranteed to happen if users interact with `MysticLeverageBundler::updateLeverageBundle`:

- It happens on the first leverage adjustment, and worsens each time.
- Any user or vault adjusting leverage will eventually trigger the error.
- There is no function to resync internal records with actual values on Mystic.

## Proof of Concept:

<details>
<summary>Code</summary>
Paste the following test code in bundler3/test/AaveLeverageBundlerTest.sol file and run the test with `forge test --fork-url https://rpc.plume.org --mt testDecreaseLeverage -vv` and `forge test --fork-url https://rpc.plume.org --mt testIncreaseLeverage -vv`.

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

## Recommendation

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