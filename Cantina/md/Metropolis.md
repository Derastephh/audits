[Contest details](https://cantina.xyz/code/076935b1-2706-48c6-bf0a-b3656aa24194/overview)

# [Info] Misplaced Recipient Allows Withdrawal of Another User's Funds

## Summary

A user can queue withdrawals using a differnt recipient address. This allows the recipient to withdraw funds that originated from a different user. This violates the principle of withdrawal ownership and could lead to confusion, loss of funds or fund mismanagement.

## Vulnerability Details

In the `BaseVault::queueWithdrawal` function, withdrawals are tracked using a recipient address (as seen below), rather than binding it to the original share owner (msg.sender):

queuedWithdrawals.userWithdrawals[recipient] += shares;
As a result, a user can queue withdrawals to an unintended recipient address, and the full amount can then be redeemed by that address, regardless of who originally deposited the shares.

During redemption, the vault allows either the recipient or the factory to call BaseVault::redeemQueuedWithdrawalNative, which is a good check but however this check is missing from BaseVault::queueWithdrawal.

Relevant check:

`if (user != msg.sender && msg.sender != address(_factory)) revert BaseVault__Unauthorized();`
Since the user is inferred from the recipient, this allows the recipient to claim withdrawals that he did'nt queue.

## Proof of Concept

Paste the following test into OracleVault.t.sol test file.
```solidity
function test_MisplacedRecipient() external {
    linkVaultToStrategy(vault, strategy);
    
    // Alice makes first deposits into the vault
    depositNativeToVault(vault, alice, 1e18, 1e18);

    // Bob deposits separately
    depositNativeToVault(vault, bob, 2e18, 1e18);

    // Bob queues withdrawal but uses address(20) as recipient
    uint256 shares = IERC20Upgradeable(vault).balanceOf(bob);
    vm.prank(bob);
    IBaseVault(vault).queueWithdrawal(shares, address(20));

    // Rebalance to finalize the queued withdrawal round
    vm.prank(owner);
    IStrategy(strategy).rebalance(0, 0, 0, 0, 0, 0, new bytes(0));

    // Bob can no longer redeem for themselves
    vm.prank(bob);
    vm.expectRevert();
    IBaseVault(vault).redeemQueuedWithdrawalNative(0, bob);

    // But address(20) can redeem everything
    vm.prank(address(20));
    IBaseVault(vault).redeemQueuedWithdrawalNative(0, address(20));

    console.log("Address 20 balance later: ", address(20).balance);
}
```

## Impact

This behavior breaks expected ownership ideology — a recipient can claim all queued shares sent to their address, regardless of who originally deposited them. It opens the door for:

Users queues withdrawals to an address mistakenly and can thereby lose their funds.
The recipient has sole authority to redeem funds, even if they weren’t the one who queued them.
Users can be tricked into using a malicious recipient address, causing their funds to be claimable by an attacker.

## Tools Used

Foundry
## Recommendations

Enforce that the recipient of a queued withdrawal must be msg.sender or a verified delegate.
