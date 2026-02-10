# [M-01] Swap1InchHook Strips Function Selector and Breaks Slippage Limit Causing DoS and Price Risk

## Summary

When usePrevHookAmount = true, the Swap1InchHook contract rebuilds the swap calldata incorrectly. It forgets to include the 1inch function selector, which causes all 1inch swaps to fail. On top of that, it adjusts the amount to swap but doesn’t update the minimum return value (minReturn), silently breaking slippage protection. This leads to a denial of service today, and dangerous slippage risk in the future.

## Finding Description

When the hook is supposed to reuse the amount from a previous hook (usePrevHookAmount = true), it rebuilds the calldata like this:
```solidity
bytes calldata txData_ = data[73:]; // original calldata, with function selector

// This reprocesses the calldata based on the selector
bytes memory updated = _validateTxData(..., txData_);
Inside _validateTxData, it checks the function selector:

bytes4 selector = bytes4(txData_[0:4]);
if (selector == I1InchAggregationRouterV6.unoswapTo.selector) {
    updatedTxData = _validateUnoswap(txData_[4:], ...); // skips the selector
}
In _validateUnoswap, it updates the amount from the previous hook:

if (usePrevHookAmount) {
    amount = ISuperHookResult(prevHook).outAmount(); // amount gets updated
}

// Re-encode calldata, but forgets to add the function selector back
updatedTxData = abi.encode(to, token, amount, minReturn, dex);
```
So now, the rebuilt calldata looks like regular parameters, not a function call, because it’s missing the selector. When the final executor tries to call 1inch:

`(bool success, ) = target.call{value: value}(callData);`
It fails immediately. The function selector is gone, so the 1inch contract doesn’t know what to do, and the transaction reverts.

It also silently weakens slippage limits. The amount to swap is increased, but ``minReturn` stays the same. This means your slippage buffer is bigger than you think, which could let trades go through at worse prices than intended.

## Impact Explanation

Right now, this breaks every swap that uses usePrevHookAmount = true with Swap1InchHook. That means:

Multi-step actions like “withdraw → swap → deposit” will fail completely.
Gas is wasted and any earlier steps (like token transfers) may already have occurred.
In the future, once the function selector bug is fixed:

minReturn (which protects against bad pricing) will still be too low compared to the new amount.
Users can get filled at bad prices or be front-run by sandwich bots.
The slippage protection users think they have is no longer reliable.

## Likelihood Explanation

Very likely:

This affects any uses of 1inch swaps and pipes the result from a previous hook.
The broken calldata causes every such transaction to fail immediately.
The minReturn issue introduces invisible price risk even after the revert bug is fixed.
This affects any user who chains hooks in a recipe involving 1inch swaps with usePrevHookAmount = true.

## Proof of Concept

Add the below test to `test/unit/SuperExecutor/SuperExecutor_sameChainFlow.t.sol` test file and run with `forge test --mt test_POCPresent_Swap1InchHook_StripsSelector`
```solidity
function test_POCPresent_Swap1InchHook_StripsSelector() public {
        uint256 amount = 1 ether;

        // Deploy a fresh Swap1InchHook that points to a dummy router
        Swap1InchHook vulnHook = new Swap1InchHook(address(0xDEAD));

        // Mock previous hook so `usePrevHookAmount` branch is taken
        MockHook prevHook = new MockHook(ISuperHook.HookType.INFLOW, underlying);
        prevHook.setOutAmount(amount); // value that the vuln-hook will read

        // Dummy dex pair required by the helper that builds unoswap calldata
        MockDex dexPair = new MockDex(underlying, underlying);

        // Build hook-data with `usePrevHookAmount = true`
        bytes memory hookData = _create1InchUnoswapToHookData(
            account, // dstReceiver
            underlying,
            Address.wrap(uint256(uint160(account))),
            Address.wrap(uint256(uint160(underlying))),
            amount,
            amount,
            Address.wrap(uint256(uint160(address(dexPair)))),
            true // <- triggers vulnerable code-path
        );

        // Let the hook produce its Execution array
        Execution[] memory execs = vulnHook.build(address(prevHook), account, hookData);
        assertEq(execs.length, 1, "expected exactly one execution");

        bytes memory callData = execs[0].callData;

        // Extract first 4 bytes of produced calldata
        bytes4 actualSelector;
        assembly {
            actualSelector := shr(224, mload(add(callData, 32)))
        }

        // Correct selector that should be present if hook were safe
        bytes4 expectedSelector = I1InchAggregationRouterV6.unoswapTo.selector;

        // In the vulnerable implementation the selector is MISSING
        assertTrue(
            actualSelector != expectedSelector,
            "Selector was not stripped. vulnerability no longer present"
        );
    }
```

## Recommendation

- Re-attach the function selector when rebuilding calldata:
```solidity
bytes memory encoded = abi.encodeWithSelector(selector, to, token, amount, minReturn, dex);
```
- Update minReturn proportionally if the amount is changed, or warn users that minReturn will be interpreted differently.







# [L-01] Manager Role Change Breaks Pending Proposals in SuperLedgerConfiguration

## Summary

If a manager creates a proposal and then hands over their role to someone else, the proposal can become stuck. The new manager can’t accept it, and the old one isn’t allowed to act on it anymore. As a result, no new proposals can be made for that yield source, permanently freezing governance for that configuration.

## Finding Description

When a new configuration is proposed in the SuperLedgerConfiguration contract, the proposal stores the current manager’s address at that moment:
```solidity
yieldSourceOracleConfigProposals[id] = YieldSourceOracleConfig({
    yieldSourceOracle: config.yieldSourceOracle,
    feePercent:       config.feePercent,
    feeRecipient:     config.feeRecipient,
    manager:          existingConfig.manager,  // ← snapshot of current manager
    ledger:           config.ledger
});
```
Later, if the manager role is transferred using the standard two-step process:
```solidity
function transferManagerRole(bytes4 id, address newMgr) external {
    if (yieldSourceOracleConfig[id].manager != msg.sender) revert NOT_MANAGER();
    pendingManager[id] = newMgr;
}

function acceptManagerRole(bytes4 id) external {
    if (pendingManager[id] != msg.sender) revert NOT_PENDING_MANAGER();
    yieldSourceOracleConfig[id].manager = msg.sender; // live config updated
    delete pendingManager[id];
}
```
Only the live configuration is updated. The stored proposal still points to the old manager’s address.

Now, if the new manager tries to accept the proposal:
```solidity
function acceptYieldSourceOracleConfigProposal(bytes4[] calldata ids) external {
    ...
    if (proposal.manager != msg.sender) revert NOT_MANAGER();
}
```
They hit a revert because they’re not the one who originally created the proposal. The old manager is the one saved in the proposal, but they no longer have the role and can’t proceed either. That means the proposal is stuck, permanently.

Worse still, because this stuck proposal never gets cleared, the contract won't allow any new proposals for that id:
```solidity
if (yieldSourceOracleConfigProposalExpirationTime[id] > 0)
    revert CHANGE_ALREADY_PROPOSED();
```
There’s no function to cancel, overwrite, or expire this dead proposal. So once a handoff happens during an active proposal, that configuration is frozen forever.

## Impact Explanation

Any yield source with a stuck proposal becomes ungovernable. That means:

You can’t change fee recipients (even if they’re malicious).
You can’t update fee rates.
You can’t replace or fix broken oracles.
If the protocol relies heavily on a frozen yieldSourceOracleId, it may need to move everything to a new ID. That means updating contracts and migrating users, a messy and costly process.

## Likelihood Explanation

This can easily happen during regular operations. If a manager hands off their role during the 7 day proposal window, something that’s totally allowed, the system breaks. Since role transfers are common in real world governance, this issue is very likely to happen over time.

## Proof of Concept

Add the below test to `test/unit/accounting/LedgerTests.t.sol` test file and run with `forge test --mt test_ProposalOrphanAfterManagerTransfer`
```solidity
function test_ProposalOrphanAfterManagerTransfer() public {

        // initial configuration by Manager (this)
        bytes4 oracleId = bytes4(keccak256("ORPHAN_POC"));
        address oracle = address(mockOracle);
        uint256 feePercent = 100; // 1%
        address feeRecipient = address(this);
        address ledgerAddr = address(mockBaseLedger);

        ISuperLedgerConfiguration.YieldSourceOracleConfigArgs[]
            memory configs = new ISuperLedgerConfiguration.YieldSourceOracleConfigArgs[](1);

        configs[0] = ISuperLedgerConfiguration.YieldSourceOracleConfigArgs({
            yieldSourceOracleId : oracleId,
            yieldSourceOracle : oracle,
            feePercent : feePercent,
            feeRecipient : feeRecipient,
            ledger : ledgerAddr
        });

        // manager sets the initial config
        config.setYieldSourceOracles(configs);

        // same manager proposes an update
        configs[0].yieldSourceOracle = address(0xDEAD);
        config.proposeYieldSourceOracleConfig(configs); // open proposal owned by current manager (address(this))

        // Transfer the manager role to newManager
        address newManager = address(0xBEEF);
        config.transferManagerRole(oracleId, newManager);

        // newManager accepts the role
        vm.prank(newManager);
        config.acceptManagerRole(oracleId); // manager stored in live config is now newManager

        // fast-forward timelock so proposal can theoretically be accepted
        vm.warp(block.timestamp + 1 weeks + 1);

        // New manager tries to accept the still pending proposal => Reverts => proposal is “orphaned”
        bytes4[] memory ids = new bytes4[](1);
        ids[0] = oracleId;

        vm.prank(newManager);
        vm.expectRevert(ISuperLedgerConfiguration.NOT_MANAGER.selector);
        config.acceptYieldSourceOracleConfigProposal(ids);

        // New manager tries to create a fresh proposal but is blocked because the orphaned one still exists =>  governance dead-lock proven
        configs[0].yieldSourceOracle = address(0xABCD); // any different value
        vm.prank(newManager);
        vm.expectRevert(ISuperLedgerConfiguration.CHANGE_ALREADY_PROPOSED.selector);
        config.proposeYieldSourceOracleConfig(configs);
    }
```
## Recommendation

- Tie proposals to the manager role, not to the specific address that held the role at proposal time.
- Alternatively, add an auto-expiry feature for stale proposals that haven’t been accepted in time.
