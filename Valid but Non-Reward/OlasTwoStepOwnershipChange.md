 # Consider implementing two-step procedure for updating protocol owners

A simple copy-paste error or a typo may end up bricking the protocol, by either setting the owner to a malicious address, setting the owner to a contract without the ability to perform key ownership functions, or setting the owner to an address with no known private keys. Implementing a two-step procedure for updating protocol ownership roles helps protect the protocol from human errors. Consider implementing a two-step procedure for updating ownership where the recipient is set as pending, and must accept the role of ownership by making an affirmative call.

https://github.com/code-423n4/2024-05-olas/blob/3ce502ec8b475885b90668e617f3983cea3ae29f/tokenomics/contracts/Tokenomics.sol#L546


```
/// @dev Changes the owner address.
/// @param newOwner Address of a new owner.
function changeOwner(address newOwner) external {
    // Check for the contract ownership
    if (msg.sender != owner) {
        revert OwnerOnly(msg.sender, owner);
    }

    // Check for the zero address
    if (newOwner == address(0)) {
        revert ZeroAddress();
    }

    owner = newOwner;
    emit OwnerUpdated(newOwner);
}
```