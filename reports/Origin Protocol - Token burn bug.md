
## Bug description
There is a bug present in the `OUSD` contract. Calling the `burn` function on an address with zero balance causes the `totalSupply` to go down. This happens because of an incorrect/incomplete check for rounding errors.

The end result is a situation where the sum of all balances exceeds the total supply. This is a dangerous situation which can lead to many unforeseen consequences.

The same `burn` function also contains another (smaller) bug, causing an revert because of arithmetic overflow.

I will explain both bugs in more detail below.

The bug is present in the deployed code on the Ethereum mainnet:
https://etherscan.io/address/0x33db8d52d65f75e4cdda1b02463760c9561a2aa1#code

For demonstrational purposes I will be using the code from the `origin-dollar` repository on Github:
https://github.com/OriginProtocol/origin-dollar/tree/f94d5567c57e8c541e7d1e80d1362e19fc453ac7

The bug is present in both codebases.

## Impact
Under certain circumstances the `totalSupply` can become arbitrary. This is a dangerous situation which can lead to many unforeseen consequences.

## Recommendation

### Main bug - incorrect `totalSupply`

The bug resides in the following code block:
```solidity
// Remove the credits, burning rounding errors
if (
    currentCredits == creditAmount || currentCredits - 1 == creditAmount
) {
    // Handle dust from rounding
    _creditBalances[_account] = 0;
} else if (currentCredits > creditAmount) {
    _creditBalances[_account] = _creditBalances[_account].sub(
        creditAmount
    );
} else {
    revert("Remove exceeds balance");
}
```

The main reason for the bug is this line:
```solidity
currentCredits == creditAmount || currentCredits - 1 == creditAmount
```
https://github.com/OriginProtocol/origin-dollar/blob/f94d5567c57e8c541e7d1e80d1362e19fc453ac7/contracts/contracts/token/OUSD.sol#L411

This line is the check of the first `if` statement, and is meant for situations where small rounding errors will get burned. However, it does not account for the sitation where `currentCredits` is `0`. In this case the code below will set the balance to zero (which it already is) **and** lowers the total supply (which shouldn't happen).

A simple extra `if` statement fixes the issue:
```solidity
if (creditAmount == 0) {
    revert("Burn from zero balance");
}
```

Thanks to this `if` nothing will happen when you try to burn tokens from an address with balance `0`, as it should be.

I'm sure there are more elegant and gas-efficient solutions. This code fixes the bug and passes the testsuite.


### Side-bug - revert from arithmetic overflow
This bug has no security related impact. It's still a bug though, so why not fix it while we're at it :)

It resides in the same line of code:
```solidity
currentCredits == creditAmount || currentCredits - 1 == creditAmount
```

This time it is the right part of the or (`||`). When the balanze is zero, `currentCredits` will be 0 and `currentCredits - 1` will underflow. This causes a revert in Solidity 0.8. We can prevent this by calculating the difference upfront using an "iif".

Proposed solution, containing both bugfixes:
```solidity
uint256 creditDifference = currentCredits > creditAmount
    ? currentCredits - creditAmount
    : creditAmount - currentCredits;

if (creditAmount == 0) {
    revert("Burn from zero balance");
}

if (creditDifference <= 1) {
    // Handle dust from rounding
    _creditBalances[_account] = 0;
} else if (currentCredits > creditAmount) {
    _creditBalances[_account] = _creditBalances[_account].sub(
        creditAmount
    );
} else {
    revert("Remove exceeds balance");
}
```

## Proof of Concept
```javascript
const hre = require("hardhat");
const ethers = hre.ethers;

async function main() {
  // Get signer
  [deployer] = await ethers.getSigners();

  // Deploy OUSD token
  const OUSD = await ethers.getContractFactory("OUSD");
  const token = await OUSD.deploy();
  await token.deployed();

  // Initialize token
  await token.initialize("Name", "Symbol", deployer.address);

  // Initial mint
  await token.mint(deployer.address, ethers.BigNumber.from(10).pow(14));

  // Scale the supply so rounding errors start occuring
  await token.changeSupply(ethers.BigNumber.from(10).pow(17));

  // Burn from a random address with zero tokens
  const RANDOM_ADDRESS = "0x00000000000000000000000000000000deadbeef";
  await token.burn(RANDOM_ADDRESS, 42);

  // Print token amounts. Deployer balance is now higher than total supply.
  const balance = await token.balanceOf(deployer.address);
  const totalSupply = await token.totalSupply();
  console.log("Total balance: ", balance.toString());
  console.log("Total supply: ", totalSupply.toString());

  // Script output:
  //
  // $ yarn hardhat run scripts/poc-changesupply-burn.js
  // Total balance:  100000000000000000
  // Total supply:  99999999999999958
  //
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });

```

