# First Flight #7: Horse Store - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. It is possible to feed a horse with an invalid token ID, i.e., an ID that does not correspond to any minted horse NFT](#h-01)
    - ### [H-02. Checking if a horse is happy after being fed in the past 24 hours returns incorrect result, violating the invariant](#h-02)
    - ### [H-03. Feeding horses at certain block timestamps will fail, violating the invariant that horses must be able to be fed at all times](#h-03)
    - ### [H-04. The total supply value used as the token ID is never updated, causing every subsequent mint to fail after the first successful one](#h-04)
    - ### [H-05. The total supply value used as the token ID is not loaded properly in the mint process, causing every subsequent mint to fail after the first successful one](#h-05)

- ## Low Risk Findings
    - ### [L-01. The Huff version does not match the Solidity version regarding the `fallback` function](#l-01)
    - ### [L-02. Call to `HorseStore.sol::isHappyHorse()` at block timestamp less than `86400` will revert due to the arithmetic underflow](#l-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #7

### Dates: Jan 11th, 2024 - Jan 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/clr6s75ut00013qg9z8bpkalo)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 0
   - Low: 2


# High Risk Findings

## <a id='h-01'></a>H-01. It is possible to feed a horse with an invalid token ID, i.e., an ID that does not correspond to any minted horse NFT            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.sol#L32

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L80

## Summary

It is possible to feed a horse whose token has not been minted yet or is burned.

## Vulnerability Details

It is possible to feed a horse with an ID that has not yet been minted and subsequently mint a horse with the same ID. This can lead to incorrect results when verifying the horse's happiness, as the verification process may indicate that the horse is happy even when it shouldn't be.

## Impact

Feeding a non-minted horse may cause false happiness verification results post-minting.

## Proof of Concept (PoC)

Add the next test in `Base_Test.t.sol`.

```javascript
function test_FeedingNotMintedHorseIsAllowed(uint256 horseId, uint256 feedAt) public {
    vm.warp(feedAt);
    horseStore.feedHorse(horseId); // feeding a horse that is not minted is allowed

    assertEq(horseStore.horseIdToFedTimeStamp(horseId), feedAt);
}
```

Run a test with `forge test --mt test_FeedingNotMintedHorseIsAllowed`.

## Tools Used

- Foundry

## Recommendations

To properly feed a horse, it's important to verify the validity of the token ID.

Recommended changes in `HorseStore.sol::feedHorse()` function:

```diff
function feedHorse(uint256 horseId) external {
+    _requireOwned(horseId); // reverts if the token with `horseId` has not been minted yet, or if it has been burned
    horseIdToFedTimeStamp[horseId] = block.timestamp;
}
```

Add the next test to `HorseStoreSolidity.t.sol`.

```javascript
import {IERC721Errors} from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";

function test_FeedingHorseBeforeMintingItIsNotAllowed(address randomOwner) public {
    vm.assume(randomOwner != address(0));
    vm.assume(!_isContract(randomOwner));

    uint256 horseId = horseStore.totalSupply();
    vm.expectRevert(abi.encodeWithSelector(IERC721Errors.ERC721NonexistentToken.selector, horseId));
    horseStore.feedHorse(horseId);
}
```

Run a test with `forge test --mt test_FeedingHorseBeforeMintingItIsNotAllowed`.

Recommended changes to `HorseStore.huff::FEED_HORSE()` function:

```diff
#define macro FEED_HORSE() = takes (0) returns (0) {
    timestamp               // [timestamp]
    0x04 calldataload       // [horseId, timestamp]
+    dup1                                           // [horseId, horseId, timestamp]
+    // revert if owner is zero address/not minted
+    [OWNER_LOCATION] LOAD_ELEMENT_FROM_KEYS(0x00)  // [owner, horseId, timestamp]
+    dup1 continue jumpi
+    NOT_MINTED(0x00)
+    continue:
+    dup3 dup3                                      // [horseId, timestamp, owner, horseId, timestamp]
+    STORE_ELEMENT(0x00)                            // [owner, horseId, timestamp]

    // End execution 
    0x11 timestamp mod      
    endFeed jumpi                
    revert 
    endFeed:
    stop
}
```

Add the next test to `HorseStoreHuff.t.sol`.

```javascript
function test_FeedingHorseBeforeMintingItIsNotAllowed(address randomOwner) public {
    vm.assume(randomOwner != address(0));
    vm.assume(!_isContract(randomOwner));

    uint256 horseId = horseStore.totalSupply();
    vm.expectRevert("NOT_MINTED");
    horseStore.feedHorse(horseId);
}
```

Run a test with `forge test --mt test_FeedingHorseBeforeMintingItIsNotAllowed`.
## <a id='h-02'></a>H-02. Checking if a horse is happy after being fed in the past 24 hours returns incorrect result, violating the invariant            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L100

## Summary

A horse is marked as not happy even if it has been fed within the past 24 hours.

## Vulnerability Details

The current implementation of the `HorseStore.huff::IS_HAPPY_HORSE()` function won't provide an accurate result as it returns `false` even if the horse has been fed within the past 24 hours.

## Impact

One of the contract's invariants has been violated because the check for the horse's happiness is resulting in a false negative.

## Proof of Concept (PoC)

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_HorseIsNotHappyIfFedWithinPast24Hours(uint256 horseId, uint256 checkAt) public {
    uint256 fedAt = horseStore.HORSE_HAPPY_IF_FED_WITHIN();
    checkAt = bound(checkAt, fedAt + 1 seconds, fedAt + horseStore.HORSE_HAPPY_IF_FED_WITHIN() - 1 seconds); // it has been less than 24 hours since the last feeding time
    vm.warp(fedAt);
    horseStore.feedHorse(horseId);
    
    vm.warp(checkAt);
    assertEq(horseStore.isHappyHorse(horseId), false); // horse seems unhappy despite being fed less than 24 hours ago
}
```

Run a test with `forge test --mt test_HorseIsNotHappyIfFedWithinPast24Hours`.

## Tools Used

- Foundry

## Recommendations

Recommended changes to `HorseStore.huff::IS_HAPPY_HORSE()` function:

```diff
#define macro IS_HAPPY_HORSE() = takes (0) returns (0) {
    0x04 calldataload                   // [horseId]
    LOAD_ELEMENT(0x00)                  // [horseFedTimestamp]
    timestamp                           // [timestamp, horseFedTimestamp]
    dup2 dup2                           // [timestamp, horseFedTimestamp, timestamp, horseFedTimestamp]
    sub                                 // [timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    [HORSE_HAPPY_IF_FED_WITHIN_CONST]   // [HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
-    lt                                  // [HORSE_HAPPY_IF_FED_WITHIN < timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
+    gt                                  // [HORSE_HAPPY_IF_FED_WITHIN > timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    start_return_true jumpi             // [timestamp, horseFedTimestamp]
    eq                                  // [timestamp == horseFedTimestamp]
    start_return 
    jump
    
    start_return_true:
    0x01

    start_return:
    // Store value in memory.
    0x00 mstore

    // Return value
    0x20 0x00 return
}
```

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_HorseIsHappyIfFedWithinPast24Hours(uint256 horseId, uint256 checkAt) public {
    uint256 fedAt = horseStore.HORSE_HAPPY_IF_FED_WITHIN();
    checkAt = bound(checkAt, fedAt + 1 seconds, fedAt + horseStore.HORSE_HAPPY_IF_FED_WITHIN() - 1 seconds); // it has been less than 24 hours since the last feeding time
    vm.warp(fedAt);
    horseStore.feedHorse(horseId);
    
    vm.warp(checkAt);
    assertEq(horseStore.isHappyHorse(horseId), true);
}
```

Run a test with `forge test --mt test_HorseIsHappyIfFedWithinPast24Hours`.
## <a id='h-03'></a>H-03. Feeding horses at certain block timestamps will fail, violating the invariant that horses must be able to be fed at all times            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L86

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L88

## Summary

Feeding a horse at specific block timestamps is prevented by the implementation of the `HorseStore.huff::FEED_HORSE()` function, leading to Denial-of-Service (DoS).

## Vulnerability Details

The `HorseStore.huff::FEED_HORSE()` function contains a code segment that reverts the transaction if the remainder of the current block timestamp and number `17` (`0x11`) is equal to `0`, which is expressed as:

```
0x11 timestamp mod
endFeed jumpi
revert
endFeed:
stop
```

## Impact

Horses are not able to be fed at all times.

## Proof of Concept (PoC)

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_FeedingHorseRevertsAtSpecificTimestamps(uint256 horseId, uint256 feedAt) public {
    vm.assume(feedAt % 0x11 == 0); // simulate the timestamp of the block at which a transaction will fail
    vm.warp(feedAt);

    vm.expectRevert();
    horseStore.feedHorse(horseId);
}
```

Run a test with `forge test --mt test_FeedingHorseRevertsAtSpecificTimestamps`.

## Tools Used

- Foundry

## Recommendations

It is recommended to remove the code that reverts at certain block timestamps. 

Recommended changes to `HorseStore.huff::FEED_HORSE()` function:

```diff
#define macro FEED_HORSE() = takes (0) returns (0) {
    timestamp               // [timestamp]
    0x04 calldataload       // [horseId, timestamp]
    STORE_ELEMENT(0x00)     // []

    // End execution 
-    0x11 timestamp mod      
-    endFeed jumpi                
-    revert 
-    endFeed:
    stop
}
```

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_FeedingHorseIsPossibleAtAnyTimestamp(uint256 horseId, uint256 feedAt) public {
    vm.warp(feedAt);

    horseStore.feedHorse(horseId);
}
```

Run a test with `forge test --mt test_FeedingHorseIsPossibleAtAnyTimestamp`.
## <a id='h-04'></a>H-04. The total supply value used as the token ID is never updated, causing every subsequent mint to fail after the first successful one            



## Summary

Implementation of the `HorseStore.huff::MINT_HORSE()` function does not increment the total supply used to determine the token ID, causing Denial-of-Service (DoS) as it allows only one NFT to be minted.

## Vulnerability Details

When minting a horse NFT, assigning a unique token ID to each minted NFT is crucial. In the case of the `HorseStore.huff::MINT_HORSE()` function, the total supply value is used to determine the token ID. However, the total supply value is never updated. As a result, after the first successful mint, any attempts to mint a new token will fail with an error message `ALREADY_MINTED` indicating that the token has already been minted.

## Impact

Only one NFT can ever be minted.

## Proof of Concept (PoC)

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_MintingHorseRevertsAfterFirstSuccessfulMint(address randomOwner) public {
    vm.assume(randomOwner != address(0));
    vm.assume(!_isContract(randomOwner));

    uint256 horseId = horseStore.totalSupply();
    vm.prank(randomOwner);
    horseStore.mintHorse(); // first successful mint
    assertEq(horseStore.ownerOf(horseId), randomOwner);

    vm.expectRevert("ALREADY_MINTED"); // any mint transactions made after the first mint will be reverted
    vm.prank(randomOwner);
    horseStore.mintHorse();
}
```

Run a test with `forge test --mt test_MintingHorseRevertsAfterFirstSuccessfulMint`.

## Tools Used

- Foundry

## Recommendations

NOTE: The mitigation recommended for the current finding includes the mitigation recommended for the finding related to the total supply value not being loaded properly.

The total supply value should be incremented on every successful mint. 

Recommended changes to `HorseStore.huff::MINT_HORSE()` function:

```diff
#define macro MINT_HORSE() = takes (0) returns (0) {
    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
+    sload          // [totalSupply] mitigation recommended for the finding related to the total supply value not being loaded properly
+    dup1           // [totalSupply, totalSupply] duplicate total supply value
-    caller         // [msg.sender, totalSupply]
+    caller         // [msg.sender, totalSupply, totalSupply] update comment with new stack item
    _MINT()        // [totalSupply] 
+    // Update total supply.
+    0x1 add        // [totalSupply + 1] increment total supply
+    [TOTAL_SUPPLY] // [TOTAL_SUPPLY, totalSupply + 1] push total supply storage slot to the stack
+    sstore         // [] store new total supply
    stop           // []
}
```

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_MintMultipleHorses(uint256 amount) public {
    amount = bound(amount, 2, 20);
    for (uint256 i = 0; i < amount; ++i) {
        uint256 horseId = horseStore.totalSupply();
        vm.prank(user);
        horseStore.mintHorse();
        assertEq(horseStore.ownerOf(horseId), user);
    }
    uint256 totalSupply = horseStore.totalSupply();
    assertEq(totalSupply, amount);
}
```

Run a test with `forge test --mt test_MintMultipleHorses`.
## <a id='h-05'></a>H-05. The total supply value used as the token ID is not loaded properly in the mint process, causing every subsequent mint to fail after the first successful one            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L74

## Summary

Implementation of the `HorseStore.huff::MINT_HORSE()` function does not load properly the total supply value used to determine the token ID, causing Denial-of-Service (DoS) as it allows only one NFT ever to be minted.

## Vulnerability Details

When minting a horse NFT, assigning a unique token ID to each minted NFT is crucial. In the case of the `HorseStore.huff::MINT_HORSE()` function, the total supply value is used to determine the token ID. However, there is an issue with the function that causes it not to load the total supply value correctly. As a result, the function assigns the same token ID of `0` to every minted NFT, which causes the transaction to fail with the error message `ALREADY_MINTED`. This issue prevents the successful minting of subsequent NFTs after the first one.

## Impact

Only one NFT can ever be minted.

## Proof of Concept (PoC)

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_MintingHorseRevertsAfterFirstSuccessfulMint(address randomOwner) public {
    vm.assume(randomOwner != address(0));
    vm.assume(!_isContract(randomOwner));

    uint256 horseId = horseStore.totalSupply();
    vm.prank(randomOwner);
    horseStore.mintHorse(); // first successful mint
    assertEq(horseStore.ownerOf(horseId), randomOwner);

    vm.expectRevert("ALREADY_MINTED"); // any mint transactions made after the first mint will be reverted
    vm.prank(randomOwner);
    horseStore.mintHorse();
}
```

Run a test with `forge test --mt test_MintingHorseRevertsAfterFirstSuccessfulMint`.

## Tools Used

- Foundry

## Recommendations

The value stored at the `TOTAL_SUPPLY` storage slot must be loaded properly. 

Recommended changes to `HorseStore.huff::MINT_HORSE()` function:

```diff
#define macro MINT_HORSE() = takes (0) returns (0) {
    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
+    sload        // [totalSupply] retrieve the value stored in the `TOTAL_SUPPLY` slot and push it onto the stack
-    caller       // [msg.sender, TOTAL_SUPPLY]
+    caller       // [msg.sender, totalSupply] update comment with new stack item
    _MINT()        // [] 
    stop           // []
}
```

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_MintMultipleHorses(uint256 amount) public {
    amount = bound(amount, 2, 20);
    for (uint256 i = 0; i < amount; ++i) {
        uint256 horseId = horseStore.totalSupply();
        vm.prank(user);
        horseStore.mintHorse();
        assertEq(horseStore.ownerOf(horseId), user);
    }
    uint256 totalSupply = horseStore.totalSupply();
    assertEq(totalSupply, amount);
}
```

Run a test with `forge test --mt test_MintMultipleHorses`.
		


# Low Risk Findings

## <a id='l-01'></a>L-01. The Huff version does not match the Solidity version regarding the `fallback` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L171

## Summary

The functionality of the `fallback` function in the Huff version differs from that of the `fallback` function in the Solidity version.

## Vulnerability Details

There is no fallback function in the Solidity version, while in the Huff version, the `totalSupply` function is a fallback function.

## Impact

It is important to note that third-party contracts may depend on the success of the execution of the `fallback` function or the values returned by it, which could lead to unexpected results if the implementation of the `fallback` function differs between the Solidity and Huff versions. Therefore, it is necessary to ensure that the `fallback` function is implemented consistently across all versions to prevent any potential issues.

## Proof of Concept (PoC)

Add the next test in `HorseStoreSolidity.t.sol`.

```javascript
function test_Fallback() public {
    (bool success, ) = address(horseStore).call("");
    assertFalse(success); // there is no fallback function
}
```

Run a test with `forge test --mt test_Fallback`.

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_Fallback() public {
    // mint 10 NFTs
    for (uint256 i = 1; i <= 10; ++i) {
        vm.prank(user);
        horseStore.mintHorse(); // mint NFT
        uint256 totalSupply = horseStore.totalSupply(); // retrieve total supply
        assertEq(totalSupply, i);
        (bool success, bytes memory data) = address(horseStore).call(""); // execute fallback
        assertTrue(success); // there is fallback function
        uint256 fallbackData = abi.decode(data, (uint256));
        assertEq(fallbackData, totalSupply); // returned value from the fallback execution is the same as the total supply
    }
}
```

Run a test with `forge test --mt test_Fallback`.

## Tools Used

- Foundry

## Recommendations

Recommended changes to `HorseStore.huff::MAIN()` function to be reverted if no valid function selector is found:

```diff
#define macro MAIN() = takes (0) returns (0) {
    // Identify which function is being called.
    0x00 calldataload 0xE0 shr

    dup1 __FUNC_SIG(totalSupply) eq totalSupply jumpi
    dup1 __FUNC_SIG(feedHorse) eq feedHorse jumpi
    dup1 __FUNC_SIG(isHappyHorse) eq isHappyHorse jumpi
    dup1 __FUNC_SIG(horseIdToFedTimeStamp) eq horseIdToFedTimeStamp jumpi
    dup1 __FUNC_SIG(mintHorse) eq mintHorse jumpi
    dup1 __FUNC_SIG(HORSE_HAPPY_IF_FED_WITHIN) eq horseHappyIfFedWithin jumpi

    dup1 __FUNC_SIG(approve) eq approve jumpi
    dup1 __FUNC_SIG(setApprovalForAll) eq setApprovalForAll jumpi
    dup1 __FUNC_SIG(transferFrom) eq transferFrom jumpi

    dup1 __FUNC_SIG(name) eq name jumpi
    dup1 __FUNC_SIG(symbol) eq symbol jumpi
    dup1 __FUNC_SIG(tokenURI) eq tokenURI jumpi
    dup1 __FUNC_SIG(supportsInterface)eq supportsInterface jumpi

    dup1 __FUNC_SIG(getApproved) eq getApproved jumpi
    dup1 __FUNC_SIG(isApprovedForAll) eq isApprovedForAll jumpi

    dup1 __FUNC_SIG(balanceOf) eq balanceOf jumpi
    dup1 __FUNC_SIG(ownerOf)eq ownerOf jumpi
    
+    // Revert if no match is found.
+    0x00 0x00 revert

    totalSupply:
        GET_TOTAL_SUPPLY()
    feedHorse:
        FEED_HORSE()
    isHappyHorse:
        IS_HAPPY_HORSE()
    mintHorse:
        MINT_HORSE()
    horseIdToFedTimeStamp:
        GET_HORSE_FED_TIMESTAMP()
    horseHappyIfFedWithin:
        HORSE_HAPPY_IF_FED_WITHIN()

    approve:
        APPROVE()
    setApprovalForAll:
        SET_APPROVAL_FOR_ALL()
    transferFrom:
        TRANSFER_FROM()
    name:
        NAME()
    symbol:
        SYMBOL()
    tokenURI:
        TOKEN_URI()
    supportsInterface:
        SUPPORTS_INTERFACE()
    getApproved:
        GET_APPROVED()
    isApprovedForAll:
        IS_APPROVED_FOR_ALL()
    balanceOf:
        BALANCE_OF()
    ownerOf:
        OWNER_OF()
    MINT_HORSE()
}
```

Add the next test in `HorseStoreHuff.t.sol`.

```javascript
function test_FallbackNotExist() public {
    (bool success, ) = address(horseStore).call("");
    assertFalse(success);
}
```

Run a test with `forge test --mt test_FallbackNotExist`.
## <a id='l-02'></a>L-02. Call to `HorseStore.sol::isHappyHorse()` at block timestamp less than `86400` will revert due to the arithmetic underflow            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.sol#L42

## Summary

Calculation logic to determine horse happiness will revert the transaction if executed before block timestamp `86400`.

## Vulnerability Details

There is an issue in the `HorseStore.sol::isHappyHorse()` function implementation where the calculation won't work when executed at a block timestamp earlier than `86400`. This is due to an arithmetic underflow problem, which causes the function to revert.

## Impact

It wouldn't be possible to obtain a result from the `HorseStore.sol::isHappyHorse()` function.

## Proof of Concept (PoC)

Add the next test in `HorseStoreSolidity.t.sol`.

```javascript
import {stdError} from "forge-std/StdError.sol";

function test_IsHappyHorseRevertsWhenExecutedAtBlockTimestampLessThan86400(uint256 horseId, uint256 checkAt) public {
    checkAt = bound(checkAt, 0, horseStore.HORSE_HAPPY_IF_FED_WITHIN() - 1 seconds); // timestamp less than 86400
    vm.warp(checkAt);
    vm.expectRevert(stdError.arithmeticError);
    horseStore.isHappyHorse(horseId);
}
```

Run a test with `forge test --mt test_IsHappyHorseRevertsWhenExecutedAtBlockTimestampLessThan86400`.

## Tools Used

- Foundry

## Recommendations

Recommended changes in `HorseStore.sol::isHappyHorse()` function:

```diff
function isHappyHorse(uint256 horseId) external view returns (bool) {
-    (horseIdToFedTimeStamp[horseId] <= block.timestamp - HORSE_HAPPY_IF_FED_WITHIN) {
-        return false;
-    }
-    return true;
+    return HORSE_HAPPY_IF_FED_WITHIN > block.timestamp - horseIdToFedTimeStamp[horseId];
}
```

Add the next test in `HorseStoreSolidity.t.sol`.

```javascript
function test_IsHappyHorseExecutedAtBlockTimestampLessThan86400(uint256 horseId, uint256 checkAt) public {
    checkAt = bound(checkAt, 0, horseStore.HORSE_HAPPY_IF_FED_WITHIN() - 1 seconds); // timestamp less than 86400
    vm.warp(checkAt);
    horseStore.isHappyHorse(horseId);
}
```

Run a test with `forge test --mt test_IsHappyHorseExecutedAtBlockTimestampLessThan86400`.
