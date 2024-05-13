# First Flight #8: Math Master - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect implementation of logic in If statement leads to Overflow in mulWadUp function.](#H-01)

- ## Low Risk Findings
    - ### [L-01. Free Memory Pointer Overwritten](#L-01)
    - ### [L-02. Incorrect Error Selector used in revert](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #8

### Dates: Jan 25th, 2024 - Feb 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrp8xvh70001dq1os4gaqbv5)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect implementation of logic in If statement leads to Overflow in mulWadUp function.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L52

## Summary
The mulWadUp function attempts to prevent integer overflow by implementing the following logical check : <br>
// Equivalent to `require(y == 0 || x <= type(uint256).max / y)`. <br>
if mul(y, gt(x, or(div(not(0), y), x))) { <br>
&nbsp;&nbsp; mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`. <br>
&nbsp;&nbsp; revert(0x1c, 0x04) <br>
} <br>

Lets take a look at the if statement inside the assembly block
- if mul(y, gt(x, or(div(not(0), y), x))) { ~ } <br>

The logic that we want to implement is require(y == 0 || x <= type(uint256).max / y). However, the or operation done between the division of type(uint256).max / y and x performs a Bitwise OR. The value X will never be greater than the bitwise OR of another number and itself. 

## Vulnerability Details
This vulnerability will allow an overflow to happen without any reverts.


    function testMulWadUpOverflow() public pure {

        uint256 result = MathMasters.mulWadUp(UINT256_MAX, UINT256_MAX);
        assert(result < UINT256_MAX);

        result = MathMasters.mulWadUp(UINT256_MAX, 2e18);
        assert(result < UINT256_MAX);
    }
## Impact
This allows the mulWadUp() function to overflow without reverting. 
## Tools Used
Foundry
## Recommendations
Remove the or operation in the if statement. 
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Free Memory Pointer Overwritten            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L40

https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L53

## Summary
The Free Memory Pointer is overwritten on lines 40 and 53, when 0xbac65e5b is stored into memory slot 0x40. 

## Vulnerability Details
        
    function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
        // @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, div(not(0), y))) {
                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
                revert(0x1c, 0x04)
            }
            z := div(mul(x, y), WAD)
        }
    }
We use mstore to overwrite the Free Memory Pointer which is stored at slot 0x40. 

## Impact
The program thinks the next available space in memory is 0xbac65e5b now. However, doesn't have a huge impact because we revert in the following line. 

## Tools Used
Foundry

## Recommendations
Use scratch space memory slot instead of the Free Memory Pointer slot. Make sure the revert is pointing to the correct location in memory as well. 

    function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
        // @solidity memory-safe-assembly
        bytes32 hash_of_error = keccak256("MathMasters__MulWadFailed()");
        bytes4 error_selector = bytes4(hash_of_error);

        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, div(not(0), y))) {
                mstore(0x00, error_selector) // `MathMasters__MulWadFailed()`.
                revert(0x00, 0x4)
            }
            z := div(mul(x, y), WAD)
        }
    }
## <a id='L-02'></a>L-02. Incorrect Error Selector used in revert            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L40

https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L53

## Summary
The comments showcase that the author of the library intended the `MathMasters__MulWadFailed()` custom error to be thrown on lines 40 and 53. 

    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, or(div(not(0), y), x))) {
                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
                revert(0x1c, 0x04)
            }
            if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
            z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
        }
    }

However, the value that is loaded into memory is '0xbac65e5b' and that is the incorrect error selector for `MathMasters__MulWadFailed()`. Instead, the error selector that should be used is 

    bytes32 hash_of_error = keccak256("MathMasters__MulWadFailed()");
    bytes4 error_selector = bytes4(hash_of_error);


## Vulnerability Details
The wrong error selector is loaded into memory for the revert. Also note that the free memory pointer is over written as well. Instead, use the scratch space slot. Also note that the place in memory that the revert returns is not the start of a specific slot and is not where '0xbac65e5b'  is stored. 

    function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
        // @solidity memory-safe-assembly
        bytes32 hash_of_error = keccak256("MathMasters__MulWadFailed()");
        bytes4 error_selector = bytes4(hash_of_error);

        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, div(not(0), y))) {
                mstore(0x00, error_selector) // `MathMasters__MulWadFailed()`.
                revert(0x00, 0x4)
            }
            z := div(mul(x, y), WAD)
        }
    }

## Impact
This will not cause any changes in logic, but could be an inconvenience to users. 

## Tools Used
Foundry

## Proof Of Concept

    function  testFailMulWadRevert() public {
        vm.expectRevert(MathMasters.MathMasters__MulWadFailed.selector);
        MathMasters.mulWad(UINT256_MAX, UINT256_MAX);

    }

Note that this test is expected to fail because of the testFail naming convention. 

## Recommendations
As noted, load the correct error selector into the scratch space memory slot. 


