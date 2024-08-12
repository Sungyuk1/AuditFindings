# First Flight #16: Mafia Takedown - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Any "gangmember" can force out any other "gangmember" from having the "gangmember" role in the kernel. ](#M-01)
- ## Low Risk Findings
    - ### [L-01. Deployer script does not grant "godFather" the "gangmember" role in the kernel. ](#L-01)
    - ### [L-02. "godFather" cannot retrieve admin privileges in kernel. ](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #16

### Dates: May 23rd, 2024 - May 30th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-05-mafia-take-down)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 2



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Any "gangmember" can force out any other "gangmember" from having the "gangmember" role in the kernel.             

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/src/policies/Laundrette.sol#L84

## Summary
The policy contract has an external function called quitTheGang() which allows an address with the "gangmember" role in the kernel to revoke the role. 

    function quitTheGang(address account) external onlyRole("gangmember") {
        kernel.revokeRole(Role.wrap("gangmember"), account);
    }

However, when calling this function, any address can be provided for the account parameter. This means any address with the "gangmember" role can force out other addresses with the "gangmember" role. The biggest problem of this is if the "godFather" address is forced out of the "gangmember" role. In this case, the "godFather" will no longer be able to call the takeGuns() function in the policy "Laundrette".
 
## Vulnerability Details
Here is a quick proof of concept : 

    function test_ForceQuit() public {
        //Note : assumes that deployer correctly gives godFather the gangmember role
        vm.prank(kernel.admin());
        kernel.grantRole(Role.wrap("gangmember"), godFather);


        //1) Have godFather add two new addresses to the gang.
        address Bob = makeAddr("Hi I'm Bob");
        address Alice = makeAddr("Hi I'm Alice");

        vm.prank(godFather);
        laundrette.addToTheGang(Bob);
        vm.prank(godFather);
        laundrette.addToTheGang(Alice);

        assertEq(kernel.hasRole(Bob, Role.wrap("gangmember")), true);
        assertEq(kernel.hasRole(Alice, Role.wrap("gangmember")), true);

        //2) Bob can force Alice to quit the gang
        vm.prank(Bob);
        laundrette.quitTheGang(Alice);

        assertEq(kernel.hasRole(Bob, Role.wrap("gangmember")), true);
        assertNotEq(kernel.hasRole(Alice, Role.wrap("gangmember")), true);

        //3) Bob can also force the godFather to quit the gang. 
        vm.prank(Bob);
        laundrette.quitTheGang(godFather);

        assertEq(kernel.hasRole(Bob, Role.wrap("gangmember")), true);
        assertNotEq(kernel.hasRole(Alice, Role.wrap("gangmember")), true);

        //Now godFather can no longer call key functions. 
        vm.expectRevert();
        vm.prank(godFather);
        laundrette.addToTheGang(Alice);

        vm.expectRevert(abi.encodeWithSelector(Policy_OnlyRole.selector, Role.wrap("gangmember")));
        vm.prank(godFather);
        laundrette.takeGuns(godFather, 1);
    }

## Impact
Any address with the role "gangmember" in the kernel can force out any other address with the role "gangmember". This especially causes problems when "godFather" is removed as a "gangmember".

## Tools Used
Foundry

## Recommendations
Make the function quitTheGang() in the policy Laundrette only effect the caller

    function quitTheGang() external onlyRole("gangmember") {
        kernel.revokeRole(Role.wrap("gangmember"), msg.sender);
    }

# Low Risk Findings

## <a id='L-01'></a>L-01. Deployer script does not grant "godFather" the "gangmember" role in the kernel.             

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/script/Deployer.s.sol#L25

## Summary
The deploy() function in Deployer.s.sol is used to deploy the protocol. However, in the process of deploying the protocol, deployer forgets to grant "godFather" the role of "gangmember". In doing so, the "godFather" will not be able to run the addToTheGang() function or the takeGuns() function in the policy contract "Laundrette". According to the ReadMe, the "godFather" should be the Owner, and have all the rights.

## Vulnerability Details
Deployer.s.sol forgets to grant the "gangmember" role to "godFather".

    function test_godFatherGangMember() public {
        //1) Check that godFather does not have the gangmember role after deployer deploys the protocol
        console.log("Does godFather have the gangmember role? : ", kernel.hasRole(godFather, Role.wrap("gangmember")));
        assertEq(false, kernel.hasRole(godFather, Role.wrap("gangmember")));


        //2) Check that godFather cannot call core protocol functions such as laundrette.addToTheGang()
        address Alice = makeAddr("Alice");

        vm.expectRevert();
        vm.prank(godFather);
        laundrette.addToTheGang(Alice);
    }

## Impact
The "godFather" cannot call certain core protocol functions in the policy that he is supposed to be able to call. Key examples here would be the addToTheGang() function or the takeGuns() function in the policy. 

## Tools Used
Foundry

## Recommendations
In the Deployer script's deploy() function, just grant the role of "gangmember" to the "godFather" before changing the admin of the kernel to the policy. 

    kernel.grantRole(Role.wrap("gangmember"), godFather);

## <a id='L-02'></a>L-02. "godFather" cannot retrieve admin privileges in kernel.             

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/src/policies/Laundrette.sol#L92

## Summary
The ReadMe states that the policy contract "Laundrette.sol" is "the admin of Kernel.sol to grant and revoke roles. A function permit the godfather to retrieve the admin role when needed." The function provided in the policy for the godfather to retrieve the admin role is retrieveAdmin() : 

    function retrieveAdmin() external {
        kernel.executeAction(Actions.ChangeAdmin, kernel.executor());
    }

This function when called attempts to change the admin of the kernel to kernel.executor(),  which is set to the godFather's address at deployment. However, this function will never be able to set the admin of the kernel to kernel.executor(). This is because kernel.executeAction() is protected by the onlyExecutor modifier : 

    // Role reserved for governor or any executing address
    modifier onlyExecutor() {
        if (msg.sender != executor) revert Kernel_OnlyExecutor(msg.sender);
        _;
    }

This modifier checks that the address that is calling this function is equal to the executor. However, since laundrette.retrieveAdmin() function calls the kernel.executeAction() function, the msg.sender that is checked in the onlyExecutor() contract will always be the address of the policy laundrette. Therefore, the function  laundrette.retrieveAdmin() will never work, even when called by the "godFather".

## Vulnerability Details
Here is a quick proof of concept : 

    error Kernel_OnlyExecutor(address caller_);

    function test_retrieveAdmin() public {
        console.log("executor : ", kernel.executor());
        console.log("admin : ", kernel.admin());
        console.log("godFather : ", godFather);
        console.log(" ");

        //1) Try to retrieve the admin through the policy 
        vm.expectRevert(abi.encodeWithSelector(Kernel_OnlyExecutor.selector, address(laundrette)));
        vm.prank(godFather);
        laundrette.retrieveAdmin();
        assertNotEq(godFather, kernel.admin());

        console.log("executor : ", kernel.executor());
        console.log("admin : ", kernel.admin());
        console.log("godFather : ", godFather);
        console.log(" ");

        //2) Try directly calling the kernel to set the admin
        vm.prank(godFather);
        kernel.executeAction(Actions.ChangeAdmin, godFather);
        assertEq(godFather, kernel.admin());

        console.log("executor : ", kernel.executor());
        console.log("admin : ", kernel.admin());
        console.log("godFather : ", godFather);
        console.log(" ");
    }

## Impact
The laundrette.retrieveAdmin() does not work as intended. It will fail at changing the admin in the kernel unless the policy becomes the executor.  However, the executor can directly call the kernel if it wishes to become the admin as well. 

## Tools Used
Foundry

## Recommendations
Just remove the laundrette.retrieveAdmin() function. It does not work as intended. If the godFather would like to retrieve the Admin, it can just call the kernel directly. 


