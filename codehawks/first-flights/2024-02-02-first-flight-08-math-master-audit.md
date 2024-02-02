# First Flight #8: Math Master - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Faulty implementation of `MathMasters.sol::mulWadUp()` function produces incorrect result for certain input values](#h-01)
    - ### [H-02. Incorrect overflow detection in `MathMasters.sol::mulWadUp()` function leads to inaccurate results](#h-02)

- ## Low Risk Findings
    - ### [L-01. Version compatibility issue prevents use of library for contracts using version `0.8.3` of Solidity](#l-01)
    - ### [L-02. `MathMasters.sol::mulWad()` and `MathMasters.sol::mulWadUp()` functions revert with the non-intended custom error selector](#l-02)
    - ### [L-03. Due to an incorrect memory address handling in `MathMasters.sol::mulWad()` and `MathMasters.sol::mulWadUp()` functions, a failed function call does not revert with the intended error](#l-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #8

### Dates: Jan 25th, 2024 - Feb 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrp8xvh70001dq1os4gaqbv5)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 3


# High Risk Findings

## <a id='h-01'></a>H-01. Faulty implementation of `MathMasters.sol::mulWadUp()` function produces incorrect result for certain input values            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L56

## Summary

The `MathMasters.sol::mulWadUp()` function may produce incorrect results for certain input values.

## Vulnerability Details

The `MathMasters.sol::mulWadUp()` function contains a code segment that, if certain conditions are satisfied, modifies one of the input values (`x`) used in later calculations, which could result in an incorrect outcome.

The code is as follows:

```javascript
if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
```

## Impact

The `MathMasters.sol::mulWadUp()` function will produce incorrect results for certain input values.

## Proof of Concept (PoC)

We can verify this behavior by running the existing test `testMulWadUpFuzz` in `MathMasters.t.sol`.

### Fuzz testing using Foundry

By increasing the number of fuzz runs for each fuzz test case, we can potentially obtain counterexamples for which this test will fail.

The `foundry.toml` configuration file can be modified to increase the number of fuzz runs for each fuzz test case:

```diff
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

[fuzz]
-runs = 100
+runs = 1000000
# runs = 1000000 # Use this one for prod
max_test_rejects = 65536
seed = '0x1'
dictionary_weight = 40
include_storage = true
include_push_bytes = true
extra_output = ["storageLayout", "metadata"]

# See more config options https://github.com/foundry-rs/foundry/blob/master/crates/config/README.md#all-options
```

Run a test with `forge test --mt testMulWadUpFuzz`.

We should see an output similar to this in the terminal:

```bash
Running 1 test for test/MathMasters.t.sol:MathMastersTest
[FAIL. Reason: assertion failed; counterexample: calldata=0xfb6ea94f00000000000000000000000000000000000000000002bfc66f73219d55eb3274000000000000000000000000000000000000000000015fe337b990ceaaf5993b args=[3323484123583475243233908 [3.323e24], 1661742061791737621616955 [1.661e24]]] testMulWadUpFuzz(uint256,uint256) (runs: 3979, μ: 944, ~: 1189)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 184.17ms
```

### Formal verification testing using Halmos

We can also use Halmos, a symbolic testing tool, to execute the same test:

`halmos --function testMulWadUpFuzz --solver-timeout-assertion 0`

Halmos should find counterexamples for which this test fails. We should see an output similar to this in the terminal:

```bash
Running 1 tests for test/MathMasters.t.sol:MathMastersTest
Counterexample:
    p_x_uint256 = 0x0000000000000000000000000000000100000003ae000000383e000000800214 (340282367212473342719134338885200380436)
    p_y_uint256 = 0x00000000000000000000000000000000d01034837fffffffbd0a10977a23ed2d (276563564976801819313845769479676751149)
```

## Tools Used

- Manual review
- Foundry
- Halmos

## Recommendations

It is recommended to remove the code segment that modifies the `x` parameter.

Recommended changes to `MathMasters.sol::mulWadUp()` function:

```diff
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        if mul(y, gt(x, or(div(not(0), y), x))) {
            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
            revert(0x1c, 0x04)
        }
-        if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
        z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
    }
}
```

We can create a tests to check inputs that both Foundry and Halmos identified as counterexamples while running the `testMulWadUpFuzz` test in `MathMasters.t.sol`.

Add the following tests in `MathMasters.t.sol`:

```javascript
function test_MulWadUpCounterexampleFuzz() public {
    uint256 x = 3323484123583475243233908;
    uint256 y = 1661742061791737621616955;

    uint256 result = MathMasters.mulWadUp(x, y);
    uint256 expected = x * y == 0 ? 0 : (x * y - 1) / 1e18 + 1;
    assertEq(result, expected);
}
```

```javascript
function test_MulWadUpCounterexampleHalmos() public {
    uint256 x = 340282367212473342719134338885200380436;
    uint256 y = 276563564976801819313845769479676751149;

    uint256 result = MathMasters.mulWadUp(x, y);
    uint256 expected = x * y == 0 ? 0 : (x * y - 1) / 1e18 + 1;
    assertEq(result, expected);
}
```

Run both tests with `forge test --mt test_MulWadUpCounterexample`.
## <a id='h-02'></a>H-02. Incorrect overflow detection in `MathMasters.sol::mulWadUp()` function leads to inaccurate results            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L52

## Summary

The `MathMasters.sol::mulWadUp()` function has an incorrectly implemented overflow risk check.

## Vulnerability Details

The `MathMasters.sol::mulWadUp()` function implements an overflow check, which condition will never be evaluated to be `true`. This means that if the input values to the function cause an overflow, the function will still produce a result, but it will be incorrect. As a consequence, calls to this function with such input values will not be reverted as expected, and the output will be unreliable.

## Impact

The function call will not revert if the input overflows and will return an incorrect result.

## Proof of Concept (PoC)

Add the following test in `MathMasters.t.sol`:

<a id='test'></a>
```javascript
function test_MulWadUpShouldRevertOnOverflow(uint256 x, uint256 y) public {
    vm.assume(y != 0 && x > type(uint256).max / y); // Input values that will result in overflow

    vm.expectRevert(); // Expected to revert due to overflow
    MathMasters.mulWadUp(x, y);
}
```

Run a test with `forge test --mt test_MulWadUpShouldRevertOnOverflow`.

We should see an output similar to this in the terminal:

```bash
Running 1 test for test/MathMasters.t.sol:MathMastersTest
[FAIL. Reason: call did not revert as expected; counterexample: calldata=0x00a981f80000000000000000000000000000000000000000000000000000000000000002800000000000000000000000000000000000ffffffffffffffffffffffffffff args=[2, 57896044618658097711785492504343953926634997525117140554556420534452894040063 [5.789e76]]] test_MulWadUpShouldRevertOnOverflow(uint256,uint256) (runs: 0, μ: 0, ~: 0)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 9.17ms
```

The test has failed because the function call did not revert as expected.

## Tools Used

- Manual review
- Foundry

## Recommendations

It is necessary to correct the implementation of the overflow condition check.

Recommended changes to `MathMasters.sol::mulWadUp()` function:

```diff
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
-        if mul(y, gt(x, or(div(not(0), y), x))) {
+        if mul(y, gt(x, div(not(0), y))) {
            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
            revert(0x1c, 0x04)
        }
        if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
        z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
    }
}
```

If we attempt to execute the same [test](#test) that was used in the PoC, we can see that it now passes as the function call reverted as expected:

```bash
Running 1 test for test/MathMasters.t.sol:MathMastersTest
[PASS] test_MulWadUpShouldRevertOnOverflow(uint256,uint256) (runs: 100, μ: 3665, ~: 3665)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.60ms
```
		


# Low Risk Findings

## <a id='l-01'></a>L-01. Version compatibility issue prevents use of library for contracts using version `0.8.3` of Solidity            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L3

## Summary

There are compatibility issues between the library and smart contracts using version `0.8.3` of Solidity.

## Vulnerability Details

Custom errors were introduced in Solidity version `0.8.4`. This prevents smart contracts using version `0.8.3` from using this library.

References: [Solidity 0.8.4 Release Announcement](https://soliditylang.org/blog/2021/04/21/solidity-0.8.4-release-announcement)

## Impact

Smart contracts using version `0.8.3` of Solidity cannot use this library.

## Proof of Concept (PoC)

If we attempt to compile a smart contract that uses version `0.8.3` of Solidity and includes the library, a compilation error will occur.

Let's create a new smart contract in `src/MathMastersExposed.sol` with a version`0.8.3` of Solidity that will include the library:

```javascript
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity 0.8.3;

import {MathMasters} from "src/MathMasters.sol";

contract MathMastersExposed {
    using MathMasters for uint256;
}
```

If we attempt to compile this smart contract using `forge build`, we will encounter a compilation error:

```bash
Compiler run failed:
Error (2314): Expected ';' but got '('
 --> src/CustomErrors.sol:5:36:
  |
5 |     error MathMasters__MulWadFailed();
  |                                    ^

Error (2314): Expected ';' but got '('
  --> src/MathMasters.sol:14:41:
   |
14 |     error MathMasters__FactorialOverflow();
   |
```

## Tools Used

- Manual review
- Foundry

## Recommendations

The library's pragma should not include version `0.8.3`.

Recommended changes to the `MathMasters.sol` library:

```diff
- pragma solidity ^0.8.3;
+ pragma solidity ^0.8.4;
```

If we change the pragma from `0.8.3` to `0.8.4` in our previously created smart contract, we can now compile it successfully.
## <a id='l-02'></a>L-02. `MathMasters.sol::mulWad()` and `MathMasters.sol::mulWadUp()` functions revert with the non-intended custom error selector            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L40

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L53

## Summary

The function selector, functions are reverting with, is not the intended one of the error `MathMasters__MulWadFailed()`.

## Vulnerability Details

Both functions are indicating that in case of overflow, they will revert with the function selector of the error `MathMasters__MulWadFailed()`, but instead of that they revert with the function selector of the `MulWadFailed()` error located in the Solady library (https://github.com/vectorized/solady/blob/main/src/utils/FixedPointMathLib.sol#L69).

## Impact

Business logic in third-party contracts may depend on matching the reverted reason with the function selector of the error `MathMasters__MulWadFailed()`, which could lead to unexpected results if the error is not successfully caught and properly handled.

## Proof of Concept (PoC)

With the command `cast sig`, we can find the function selector of any function or custom error.

Executing `cast sig "MathMasters__MulWadFailed()"` shows that the output is `0xa56044f7`, a function selector different from the one used in the codebase (`0xbac65e5b`).

## Tools Used

- Manual review
- Foundry
  - cast

## Recommendations

We should replace the function selector `0xbac65e5b` with the function selector of the error `MathMasters__MulWadFailed()`, which is `0xa56044f7`.

Recommended changes to the `MathMasters.sol::mulWad()` function:

```diff
function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
    // @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        if mul(y, gt(x, div(not(0), y))) {
-            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
+            mstore(0x40, 0xa56044f7) // `MathMasters__MulWadFailed()`.
            revert(0x1c, 0x04)
        }
        z := div(mul(x, y), WAD)
    }
}
```

Recommended changes to the `MathMasters.sol::mulWadUp()` function:

```diff
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        if mul(y, gt(x, or(div(not(0), y), x))) {
-            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
+            mstore(0x40, 0xa56044f7) // `MathMasters__MulWadFailed()`.
            revert(0x1c, 0x04)
        }
        if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
        z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
    }
}
```

## <a id='l-03'></a>L-03. Due to an incorrect memory address handling in `MathMasters.sol::mulWad()` and `MathMasters.sol::mulWadUp()` functions, a failed function call does not revert with the intended error            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L40

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L53

## Summary

When a function call fails due to overflow, it does not revert with the intended error. Instead, it reverts without providing a reason.

## Vulnerability Details

The information regarding the reverted error is stored at the memory location of the free memory pointer (`0x40`), but the functions revert with the data stored at the memory location `0x1c`. 

It's important to note that there is no need to override the free memory pointer in this case, and it is not being updated in a secure manner. This overriding will result in very expensive memory expansion, and If the function wouldn't revert, there would be a possibility of encountering "Out of Gas" error.

## Impact

When an overflow occurs, the function reverts without providing an intended error.

## Proof of Concept (PoC)

We can check whether the function call will revert with the intended error.

Add the following test in `MathMasters.t.sol`:

<a id='test'></a>

```javascript
function test_MulWadShouldRevertWithIntendedError(uint256 x, uint256 y) public {
    vm.assume(y != 0 && x > type(uint256).max / y); // Input values that will result in overflow

    vm.expectRevert(0xbac65e5b); // intended revert error
    MathMasters.mulWad(x, y);
}
```

Run a test with `forge test --mt test_MulWadShouldRevertWithIntendedError`.

We should see an output similar to this in the terminal:

```bash
Running 1 test for test/MathMasters.t.sol:MathMastersTest
[FAIL. Reason: Error != expected error:  != 0xbac65e5b; counterexample: calldata=0xc680b3700000000000000000000000000000000000000000000000000000000000000002800000000000000000000000000000000001ffffffffffffffffffffffffffff args=[2, 57896044618658097711785492504343953926635002717413999089384049064949223260159 [5.789e76]]] test_MulWadShouldRevertWithIntendedError(uint256,uint256) (runs: 0, μ: 0, ~: 0)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 13.95ms
```

This test has failed because the function call did revert, but not with an error that was expected in the test.

## Tools Used

- Manual review
- Foundry

## Recommendations

The reason for reverting should be stored in memory in the same location that `revert` uses to locate the data it's supposed to revert with.

`revert` returns four bytes (`0x04`) starting from memory location `0x1c`. A four-byte error, when stored in memory, will be padded to 32 bytes. According to this, we can easily calculate the location where the error should be stored in memory:

`(0x1c + 0x04) - 0x20 = 0x0`, where `0x20` is the hexadecimal representation of 32.

The error should be stored at the memory location `0x00`, which is known as scratch space ([Layout in Memory](https://docs.soliditylang.org/en/v0.8.14/internals/layout_in_memory.html#layout-in-memory)).

Recommended changes to the `MathMasters.sol::mulWad()` function:

```diff
function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
    // @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        if mul(y, gt(x, div(not(0), y))) {
-            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
+            mstore(0x00, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
            revert(0x1c, 0x04)
        }
        z := div(mul(x, y), WAD)
    }
}
```

Recommended changes to the `MathMasters.sol::mulWadUp()` function:

```diff
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        if mul(y, gt(x, or(div(not(0), y), x))) {
-            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
+            mstore(0x00, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
            revert(0x1c, 0x04)
        }
        if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
        z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
    }
}
```

If we attempt to execute the same test that was used in the PoC, we can see that it now passes as the function call reverted with the expected error.
