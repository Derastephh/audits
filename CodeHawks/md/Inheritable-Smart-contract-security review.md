# Inheritable Smart Contract Wallet - Findings Report

Prepared by: Nwebor Stephen
Lead Auditors: 

- [Nwebor-Stephen](https://github.com/Derastephh)

Assisting Auditors:

- None

# Table of contents
<details>

<summary>See table</summary>

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Unauthorized Ownership Takeover via `InheritanceManager::inherit` Function](#H-01)
    - ### [H-02. Business Logic Flaw in `inheritanceManager::buyOutEstateNFT`: leads to absence of Financial Settlement](#H-02)
    - ### [H-03. Funds Burned, and DOS on ERC20 tokens withdrawals Due to Improper Beneficiary Removal](#H-03)

- ## Low Risk Findings
    - ### [L-01. Rounding Precision Error in `inheritanceManager::buyOutEstateNFT` function leads to loss of funds](#L-01)
</details>
</br>

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #35

### Dates: Mar 6th, 2025 - Mar 13th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-03-inheritable-smart-contract-wallet)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 0
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Unauthorized Ownership Takeover via `InheritanceManager::inherit` Function            



## Summary

The `InheritanceManager::inherit` function lacks access control, allowing an attacker to take over ownership under specific conditions. If the owner has only one beneficiary and inheritance deadline is passed, the attacker can trigger `InheritanceManager::inherit` and claim ownership. Once they become the new owner, they can exploit functions restricted to the contract owner, such as `InheritanceManager::sendERC20`, to steal funds.

## Vulnerability Details

<https://github.com/CodeHawks-Contests/2025-03-inheritable-smart-contract-wallet/blob/9de6350f3b78be35a987e972a1362e26d8d5817d/src/InheritanceManager.sol#L217>

```javascript
function inherit() external {
        if (block.timestamp < getDeadline()) {
            revert InactivityPeriodNotLongEnough();
        }
        if (beneficiaries.length == 1) {
            owner = msg.sender;
            _setDeadline();
        } else if (beneficiaries.length > 1) {
            isInherited = true;
        } else {
            revert InvalidBeneficiaries();
        }
    }
```

### Exploitation Scenario

1. The contract has only one beneficiary.
2. The attacker waits for the inheritance deadline to pass.
3. The attacker calls `inheritanceManager::inherit()`, becoming the new owner.
4. As the new owner, the attacker calls `inheritanceManager::sendERC20` to drain ERC20 tokens.

## Proof of Concept

Paste the following test in `inheritanceManagerTest.t.sol`

```javascript
function test_sendERC20FromAttacker() public {
    usdc.mint(address(im), 10e18);
    weth.mint(address(im), 10e18);
    vm.startPrank(owner);
    im.sendERC20(address(weth), 1e18, user1);
    console.log("Previous owner balance before attack:", weth.balanceOf(address(im)));
    address user2 = makeAddr("user2");
    im.addBeneficiery(user2);
    vm.stopPrank();

    vm.warp(1 + 90 days);

    vm.startPrank(address(attacker));
    console.log("Attacker balance before attack:", weth.balanceOf(address(attacker)));
    uint256 fullInheritanceBalance = weth.balanceOf(address(im));
    im.inherit();
    im.sendERC20(address(weth), fullInheritanceBalance, address(attacker));
    console.log("Previous owner balance after attack:", weth.balanceOf(address(im)));
    console.log("Attacker balance before attack:", weth.balanceOf(address(attacker)));
    vm.stopPrank();
}
```

## Impact

* Allows unauthorized takeover of the contract.
* Drains contract balance by exploiting `inheritanceManager::sendERC20` and `inheritanceManager::sendETH`.

## Tools Used

* Foundry

## Recommendations

* Introduce access control such as adding the `onlyBeneficiaryWithIsInherited` Modifier on the `inheritanceManager::inherit` function as this ensures only valid beneficiaries can call `inheritanceManager::inherit`, and also if beneficiaries are not trusted, then remove the below IF statement.

```javascript
if (beneficiaries.length == 1) {
            owner = msg.sender;
            _setDeadline();
```

## <a id='H-02'></a>H-02. Business Logic Flaw in `inheritanceManager::buyOutEstateNFT`: leads to absence of Financial Settlement            



## Summary

The `inheritanceManager::buyOutEstateNFT` function contains a business logic flaw that allows a beneficiary to obtain the NFT without making payments to other beneficiaries. This occurs because the function exits early if the caller is a beneficiary, preventing fund distribution while still burning the NFT.

## Vulnerability Details

<https://github.com/CodeHawks-Contests/2025-03-inheritable-smart-contract-wallet/blob/9de6350f3b78be35a987e972a1362e26d8d5817d/src/InheritanceManager.sol#L269>

The issue occurs in the following loop:

```javascript
for (uint256 i = 0; i < beneficiaries.length; i++) {
    if (msg.sender == beneficiaries[i]) {
        return; // Exits early before payments are distributed
    } else {
        IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
    }
}
```

If msg.sender is a beneficiary, the function returns early, skipping the payment distribution. Despite this, `nft.burnEstate(_nftID)` is outside the loop and still executes. As a result, the NFT is burned, but other beneficiaries do not receive their rightful share of the buyout.

### Example Scenario

1. The estate NFT is valued at 1,000 USDC with 4 beneficiaries.
2. One of the beneficiaries calls `inheritanceManager::buyOutEstateNFT`.
3. The function returns early, skipping fund distribution.
4. The NFT is burned, and the beneficiaries receive no compensation.

## Impact

* Beneficiaries do not receive their fair share of the NFT’s value.
* The contract does not fulfill its intended purpose.

## Tools Used

* Manual code review

## Recommendations

* To fix this issue, the check if `(msg.sender == beneficiaries[i])` is unnecessary and should be removed like indicated below, ensuring payments are distributed before burning the NFT:

```javascript
for (uint256 i = 0; i < beneficiaries.length; i++) {
    IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
}
nft.burnEstate(_nftID);
```

## <a id='H-03'></a>H-03. Funds Burned, and DOS on ERC20 tokens withdrawals Due to Improper Beneficiary Removal            



## Summary

The contract fails to properly update the beneficiary list when a beneficiary is removed. As a result, the inheritance distribution calculation includes removed beneficiaries, causing a portion of the funds to be sent to `address(0)`, effectively burning them, and also and causing reverts to occur for ERC20 tokens withdrawal.

## Vulnerability Details

### Issue in `inheritanceManager::withdrawInheritedFunds` Function

```javascript
uint256 divisor = beneficiaries.length;
uint256 amountPerBeneficiary = ethAmountAvailable / divisor;
```

The function determines `amountPerBeneficiary` using `beneficiaries.length`, which still includes removed beneficiaries. Since `inheritanceManager::removeBeneficiary` only deletes an entry but does not reduce the array size, the `divisor` remains unchanged. The contract then attempts to send funds to removed beneficiaries, resulting in ETH being sent to `address(0)`, effectively burning it, and causing reverts to occur for ERC20 tokens withdrawal.

**Proof Of Concept**

Paste the following test in `inheritanceManagerTest.t.sol` file.

```javascript
    function testETHGetsBurned() public {
        vm.deal(address(im), 12e18);
        address user2 = makeAddr("user2");
        address user3 = makeAddr("user3");
        address user4 = makeAddr("user4");
        address user5 = makeAddr("user5");
        vm.startPrank(owner);
        im.addBeneficiery(user2);
        im.addBeneficiery(user3);
        im.addBeneficiery(user4);
        im.addBeneficiery(user5);
        im.removeBeneficiary(user2);
        im.removeBeneficiary(user3);
        vm.stopPrank();
        console.log("Beneficiaries length:", im.getBeneficiariesLength()); // still remains 4 despite removal of two beneficiaries

        vm.warp(1 + 90 days);
        console.log("Contract Balance before funds are inherited:", address(im).balance); // 12e18

        vm.startPrank(user4);
        im.inherit();
        im.withdrawInheritedFunds(address(0));
        console.log("user4 balance", user4.balance); // 3e18 instead of 6e18
        console.log("user5 balance", user5.balance); // 3e18 instead of 6e18
        console.log("Contract Balance after funds are inherited:", address(im).balance); // 0

       vm.expectRevert();
       im.withdrawInheritedFunds(address(usdc)); // reverts
       vm.stopPrank();
    }
```

**POC Explanation**

1. The contract originally holds 12 ETH.
2. Owner adds 4 beneficiaries, but later removed 2 beneficiaries.
3. Since two beneficiaries were removed but still counted in the divisor, the `inheritanceManager::withdrawInheritedFunds` function incorrectly assumes there are 4 recipients.
4. Only 6 ETH is properly distributed to active beneficiaries.
5. The remaining 6 ETH is sent to `address(0)`, effectively burning it.
6. ERC20 tokens reverts, thereby causing Denial of Service.

## Impact

* Some ETH meant for active beneficiaries is lost forever.
* Beneficiaries receive less than their rightful share.
* Unintended burning of funds that should have been inherited.
* ERC20 tokens reverts, thereby causing Denial of Service.

## Tools Used

* Foundry

## Recommendations

* Modify `inheritanceManager::removeBeneficiary` to correctly update the array:

```javascript
function removeBeneficiary(address _beneficiary) external onlyOwner {
    uint256 indexToRemove = _getBeneficiaryIndex(_beneficiary);
    beneficiaries[indexToRemove] = beneficiaries[beneficiaries.length - 1];
    beneficiaries.pop();
}
```

This ensures the array size properly reflects the number of active beneficiaries.

    


# Low Risk Findings

## <a id='L-01'></a>L-01. Rounding Precision Error in `inheritanceManager::buyOutEstateNFT` function leads to loss of funds            



## Summary

A rounding issue in the `inheritanceManager::buyOutEstateNFT` function causes incorrect fund transfer due to integer division errors.

## Vulnerability Details

**vulnerable code**

```javascript
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
    uint256 value = nftValue[_nftID];
    console.log(value);
    uint256 divisor = beneficiaries.length;
    uint256 multiplier = beneficiaries.length - 1;
    uint256 finalAmount = (value / divisor) * multiplier;

    IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
    for (uint256 i = 0; i < beneficiaries.length; i++) {
        if (msg.sender == beneficiaries[i]) {
            return;
        } else {
            IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
        }
    }
    nft.burnEstate(_nftID);
}
```

### More Explanation

* The calculation of `finalAmount` involves integer division (`value / divisor`), which causes truncation.
* Example: If `value = 300003` and `beneficiaries.length = 4`, the expected `finalAmount` should be **225002**, but due to truncation, it results in **225000**.

### Proof of Concept

**POC**

Paste the following test in the `inheritanceManagerTest.t.sol` file.

```javascript
function testRoundingIssue() public {
    address user2 = makeAddr("user2");
    address user3 = makeAddr("user3");
    address user4 = makeAddr("user4");
    uint256 value = 300003; // 3e5 + 3
    vm.warp(1);
    vm.startPrank(owner);
    im.addBeneficiery(user1);
    im.addBeneficiery(user2);
    im.addBeneficiery(user3);
    im.addBeneficiery(user4);
    im.createEstateNFT("our beach-house", value, address(usdc));
    vm.stopPrank();
    usdc.mint(user3, 4e6);
    vm.warp(1 + 90 days);
    vm.startPrank(user3);
    usdc.approve(address(im), 4e6);
    im.inherit();
    im.buyOutEstateNFT(1);
    vm.stopPrank();

    uint256 divisor = 4; // beneficiaries.length
    uint256 multiplier = 3; // beneficiaries.length - 1
    uint256 expectedFinalAmount = (value * multiplier) / divisor; // 225002
    uint256 incorrectFinalAmount = (value / divisor) * multiplier; // 225000
    console.log("Expected final amount:", expectedFinalAmount);
    console.log("Incorrect final amount:", incorrectFinalAmount);
}
```

## Impact

* Over time, miscalculations could accumulate into significant lost funds.

## Tools Used

* Foundry

## Recommendations

* Use multiplication before division to prevent truncation:

  ```javascript
  uint256 finalAmount = (value * multiplier) / divisor;
  ```
* Implement **precision handling** by using a scaling factor (e.g., `value * 1e18` and then dividing at the end).



