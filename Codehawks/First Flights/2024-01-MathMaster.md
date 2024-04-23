# First Flight #8: Math Master - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings

  - ### [H-01. Increment will break the calculation in `MathMasters::mulWadUp`](#H-01)
  - ### [H-02. `MathMasters::mulWadUp` overflow check never works](#H-02)
  - ### [H-03. `MathMasters::mulWad` and `MathMasters::mulWadUp` will revert even when division result is within `uint256` range](#H-03)

- ## Low Risk Findings
  - ### [L-01. Incorrect error selector and memory offset in `MathMasters::mulWad` and `MathMasters::mulWadUp`](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #8

### Dates: Jan 25th, 2024 - Feb 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrp8xvh70001dq1os4gaqbv5)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 0
- Low: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Increment will break the calculation in `MathMasters::mulWadUp`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L56

## Description

`MathMasters::mulWalUp` contains the block of code below.

```javascript
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            ...
@>            if iszero(sub(div(add(z, x), y), 1)) {
@>                x := add(x, 1)
@>            }
            ...
        }
    }
```

Since `z` is always 0 at this step, it has the following behavior:

- If $(\frac{x}{y} - 1) == 0$, increment x by 1.
  Because of the rounded-down division of the EVM, every time $x \geq y > x/2$, the condition will be true.
  This increment will break the calculation because it makes no sense to increment in order to find the result of a multiplication and a division.

## Impact

**Likelihood:** High

- Every time $x \geq y > x/2$

**Impact:** High

- Function will return a wrong answer, leading to unexpected behavior on all contracts using this library.

## Proof of Concept

<details>

<summary>Foundry PoC added in MathMasters.t.sol</summary>

```javascript
import {Math} from "lib/openzeppelin-contracts/contracts/utils/math/Math.sol";
...
    function testMulWadUpFail() public {
        assertEq(
            MathMasters.mulWadUp(
                56023067736567210695140920363957329,
                // (above_number/2) + 1
                28011533868283605347570460181978665
            ),
            Math.mulDiv(
                56023067736567210695140920363957329,
                28011533868283605347570460181978665,
                uint256(1e18),
                Math.Rounding.Ceil
            )
        );
    }
```

</details>

## Recommended Mitigation

Remove all the parts incrementing `x`.

```diff
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            ...
-            if iszero(sub(div(add(z, x), y), 1)) {
-                x := add(x, 1)
-            }
            ...
        }
    }
```

## <a id='H-02'></a>H-02. `MathMasters::mulWadUp` overflow check never works

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L52

## Description

The overflow check in the `mulWadUp` function of the MathMasters library is flawed due to the `or` operation. If `x` contains only one bit different from `div(not(0),y)`, the result of the `or` operation will be greater than `x`. Even if `div(not(0),y) == x`, as `x` is not strictly superior to `x`, the condition won't be true.

```javascript
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
@>            if mul(y, gt(x, or(div(not(0), y), x))) {
                ...
            }
            ...
        }
    }
```

## Impact

**Likelihood:** High

- Every call

**Impact:** High

- The function will overflow if $x*y > type(uint256).max$. This behavior can be exploited by an attacker attempting to manipulate any contract using this library.

## Proof of Concept

<details>

<summary>Foundry PoC added in MathMasters.t.sol</summary>

```javascript
function testMulWadUpOverflow() public {
    vm.expectRevert(MathMasters.MathMasters__MulWadFailed.selector);
    // will overflow
    console2.log(MathMasters.mulWadUp(1e68, 1e68));
}
```

</details>

## Recommended Mitigation

Use the same check in `MathMasters::mulWad`.

```diff
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.

-            if mul(y, gt(x, or(div(not(0), y), x))) {
+            if mul(y, gt(x, div(not(0), y))) {
                ...
            }
            ...
        }
    }
```

## <a id='H-03'></a>H-03. `MathMasters::mulWad` and `MathMasters::mulWadUp` will revert even when division result is within `uint256` range

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L35-L59

## Description

`MathMasters::mulWad` and `MathMasters::mulWadUp` revert when the multiplication overflows without considering the subsequent division. However, the result of the multiplication and division can be within the `uint256` range in this case: $ x \cdot y > \text{type(uint256).max} > \frac{x \cdot y}{WAD} $

## Impact

Likelihood: Medium

- Occurs whenever : $ x \cdot y > \text{type(uint256).max} > \frac{x \cdot y}{WAD}$ which is equivalent to : $1e18 \cdot \text{type(uint256).max} > x \cdot y > \text{type(uint256).max}$

Impact: Medium/Low

- If users are unaware of this behavior, the program will revert, leading to unexpected behavior in contracts using this library.
- Other libraries, such as `Math` by OpenZeppelin, manage this case, potentially causing confusion for users.

## Proof of Concept

<details>

<summary>Foundry PoC added in MathMasters.t.sol</summary>

```javascript
    function testMulWadRevertEvenIfTheResultOfTheDivisionIsNotOverflowed()
        public
    {
        assertEq(
            // Will fail even if the result is 1e60 which is lower than type(uint256).max
            MathMasters.mulWad(1e60, 1e18),
            Math.mulDiv(1e60, 1e18, 1e18, Math.Rounding.Floor)
        );
        assertEq(
            // Will fail even if the result is 1e60 which is lower than type(uint256).max
            MathMasters.mulWadUp(1e60, 1e18),
            Math.mulDiv(1e60, 1e18, 1e18, Math.Rounding.Ceil)
        );
    }
```

</details>

## Recommended Mitigation

- Use a well-known and tested library like `Math` by OpenZeppelin.
- Alternatively, implement a mechanism inspired by existing libraries to handle large numbers when the result can be a valid number after division.

# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect error selector and memory offset in `MathMasters::mulWad` and `MathMasters::mulWadUp`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L40-L41

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L53-L54

## Description

Two errors are present in the code which will prevent to return the right error:

1. The `revert` statement uses the wrong memory offset. It should use the same offset as `mstore`, plus 28 bytes, because the selector is placed at the end of the 32 bytes in memory and has a length of 4. Here, it should be `0x5c`.

2. The selector `0xbac65e5b` does not correspond to `MathMasters__MulWadFailed`. The correct selector for this error is `0xa56044f7`.

```javascript
    function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
        // @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, div(not(0), y))) {
@>                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
@>                revert(0x1c, 0x04)
            }
            z := div(mul(x, y), WAD)
        }
    }
```

```javascript
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, or(div(not(0), y), x))) {
@>                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
@>                revert(0x1c, 0x04)
            }
            ...
        }
    }
```

## Impact

**Likelihood:** Medium

- Will occur every the multiplication overflows.

**Impact:** Low

- The program will revert with no error, leading to unexpected behavior in any contract using this library.

## Proof of Concept

<details>

<summary>Foundry PoC</summary>

```javascript
    function testMulWadWrongError() public {
        vm.expectRevert(MathMasters.MathMasters__MulWadFailed.selector);
        MathMasters.mulWad(1e68, 1e68);
    }
```

</details>

## Recommended Mitigation

Replace the values as follows:

```diff
    function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
        // @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, div(not(0), y))) {
-                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
-                revert(0x1c, 0x04)
+                mstore(0x40, 0xa56044f7) // `MathMasters__MulWadFailed()`.
+                revert(0x5c, 0x04)
            }
            z := div(mul(x, y), WAD)
        }
    }
```
