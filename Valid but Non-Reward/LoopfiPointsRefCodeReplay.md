 # Lines of code

https://urldefense.com/v3/__https://github.com/code-423n4/2024-05-loop/blob/40167e469edde09969643b6808c57e25d1b9c203/src/PrelaunchPoints.sol*L172__;Iw!!HXCxUKc!y9fFMrMVVF0VULXmRw17gorAVJFTy-Jp3xXS02sVBrTbUBB1JVWowkxS8-C71OwvkDxL5K42fqLkp6sPGUPRjR__e6o$


# Vulnerability details

## Summary
The referral code system is vulnerable to replay attacks.

## Vulnerability Details
The contest description states the following : "When staking, users can use a referral code encoded as bytes32 that will give the referral extra points." This is translated into code in the internal function _processLock(address _token, uint256 _amount, address _receiver, bytes32 _referral) which is called by functions which deposit/lock collateral from users.

However, the function _processLock() emits the Lock event :

event Locked(address indexed user, uint256 amount, address token, bytes32 indexed referral);

We see that the bytes32 for the referral parameter is indexed. This means that referral codes will be easily indexable offchain as the referral code will be stored as one of the topics in emitted Log structs. Note that variables of type bytes are hashed when indexed in events, but since bytes32 is a fixed-sized type, and since it already fits within the 32-byte limit of a log topic, it is stored directly without hashing.  


A replay attack is the following : "A replay attack is a network attack when an attacker intercepts a network communication between two parties to delay, redirect, or repeat it. Then, the cybercriminal pretends to be one of the legitimate parties and retransmits the traffic to replicate or manipulate the original action."

Essentially, the fact that the referral code is emitted as an event allows anyone to observe and use a provided referral code even if they were not the intended user. 

https://urldefense.com/v3/__https://github.com/code-423n4/2024-05-loop/blob/40167e469edde09969643b6808c57e25d1b9c203/src/PrelaunchPoints.sol*L172__;Iw!!HXCxUKc!y9fFMrMVVF0VULXmRw17gorAVJFTy-Jp3xXS02sVBrTbUBB1JVWowkxS8-C71OwvkDxL5K42fqLkp6sPGUPRjR__e6o$

## Impact

Not too much information is provided about the referral code system, considering most of it is implanted off-chain. Therefore, the design and structure of the referral code system is unknown to us auditors. However, here are some common designs where this could cause harm.

1) Referral Codes provide boasted yield : If the users of the referral code are given a boasted yield, similar to the provider of the referral code, then the user will be able to earn higher yield than they deserve by observing the blockchain and depositing with a replay of a successful referral code. It is common practice to provide users of a referral code with some sort of benefit, since otherwise users are unlikely to deal with the trouble of inputing the code. This will have the effect of the protocol having to spend more of its capital to pay out a higher yield to certain users when it could have avoided spending that capital.

2) Referral Codes are limited : Similar to how Fortnite launched its mobile app, in some referral code systems users are limited in the amount of referral codes they may provide to users. If this is the case, then this replay attack will provide significant grievances to users.

## Proof of Concept
Here is a quick proof of concept using foundry for a user replaying a referral code

function testReferralReplay(uint256 refCodeSeed) public {
    uint256 lockAmount = 20;
    vm.deal(address(this), lockAmount);
   
    //1) Generate a random referral code
    bytes32 randomReferral = bytes32(uint256(refCodeSeed));

    //2) Call lockEth and record the emitted event
    vm.recordLogs();
    prelaunchPoints.lockETH{value: lockAmount}(randomReferral);

    //3) Get the emitted Locked event. Assert that the referral code found in the
    //   event matches the referral code used.
    Vm.Log[] memory entries = vm.getRecordedLogs();
    assertEq32(randomReferral, entries[0].topics[2]);

    //4) Now have a random user call lockEth() using the referral code used by the first user
    address someRandomUser = vm.addr(1);
    vm.deal(someRandomUser, lockAmount);
    vm.recordLogs();
    vm.prank(someRandomUser);
    prelaunchPoints.lockETH{value: lockAmount}(entries[0].topics[2]);

    //5) Assert that the referral codes used by both users are the same.
    //   Also assert that the addresses used for the two events are different
    Vm.Log[] memory entries2 = vm.getRecordedLogs();
    assertEq32(randomReferral, entries2[0].topics[2]);
    assertNotEq32(entries[0].topics[1], entries2[0].topics[1]);
}

## Tools Used
Foundry

## Recommended Mitigation Steps
Considering a significant amount of the referral code system is already built off-chain, it might be worthwhile to just move the entire system off-chain. The vast majority of users will choose to interface with a smart contract using a provided frontend rather than directly through the contract. The frontend can ask for the referral code, and just store a mapping of (address -> referral code) on the backend. This will allow us to remove the indexed referral code from the Locked event which will also reduce gas. Since the address of the user is indexed in the Locked event, we can easily index the blockchain for Locked events with a certain address and use our backend mapping to associate addresses with the referral codes they used.



## Assessed type

Other