
Total supply can become larger than max supply

## Introduction
There is a bug present in the `OUSD` contract. Because of rounding errors the total supply can become larger than the max supply.

The bug is present in the deployed code on the Ethereum mainnet, located here:
- https://etherscan.io/address/0x2A8e1E676Ec238d8A992307B495b45B3fEAa5e86?utm_source=immunefi

To simplify testing, I'm using the contract in isolation. In the POC below I have removed several `onlyVault` modifiers to make this possible.

## Description
Reproduction:
- Mint enough `OUSD` tokens to allow supply changes
- Change the supply to the max supply (or higher)

The bug resides in our favorite function `changeSupply`, also mentioned in my earlier bugreport.
```solidity
    function changeSupply(uint256 _newTotalSupply)
        external
        onlyVault
        nonReentrant
    {
        require(_totalSupply > 0, "Cannot increase 0 supply");

        if (_totalSupply == _newTotalSupply) {
            emit TotalSupplyUpdatedHighres(
                _totalSupply,
                _rebasingCredits,
                _rebasingCreditsPerToken
            );
            return;
        }

        _totalSupply = _newTotalSupply > MAX_SUPPLY
            ? MAX_SUPPLY
            : _newTotalSupply;

        _rebasingCreditsPerToken = _rebasingCredits.divPrecisely(
            _totalSupply.sub(nonRebasingSupply)
        );

        require(_rebasingCreditsPerToken > 0, "Invalid change in supply");

        _totalSupply = _rebasingCredits
            .divPrecisely(_rebasingCreditsPerToken)
            .add(nonRebasingSupply);

        emit TotalSupplyUpdatedHighres(
            _totalSupply,
            _rebasingCredits,
            _rebasingCreditsPerToken
        );
    }
```

The problem here is the following line:
```solidity
        _totalSupply = _rebasingCredits
            .divPrecisely(_rebasingCreditsPerToken)
            .add(nonRebasingSupply);
```

This line calculates the total supply, dividing `_rebasingCredits` by the (very) rounded off value of `_rebasingCreditsPerToken`.

## Recommendation

After a bit of a deep dive my conclusion is that this last `_totalSupply = ...` statement can be completely removed. It recalculates the total supply based on values which are in turn based on the total supply. Actually, it simply reverses this calculation:
```solidity
        _rebasingCreditsPerToken = _rebasingCredits.divPrecisely(
            _totalSupply.sub(nonRebasingSupply)
        );
```

Theoretically, recalculating the total supply like this should be a no-op because it simply reverses the calculation and "recalculates itself". However, because of rounding errors the result is now a (very rough) approximation of itself.

I hope this explanation makes sense. It took me a while to figure this out and I'm struggling to put it into words. Any questions are welcome.

I'm not 100% sure about my conclusions but I've tested it thoroughly and removing this line fixes the bug and passes the testsuite.

## Recommendations
I recommend taking a close look at `changeSupply` in it's entirity and possible doing a complete rewrite/refactor. The current code is hard to read and error prone.

Removing the last `totalSupply = ...` statement would probably be a good start.

## Impact
The impact is low. With `MAX_SUPPLY` being `(2^128) - 1`, there is still a LONG way to go before this becomes a problem.

It is still a bug that could cause unforeseen problems in the future. These future scenarios could be: refactors, protocol upgrades, migrations, forks, etc.

## Proof of Concept
```javascript
const hre = require("hardhat");
const ethers = hre.ethers;

const { deployBase } = require("../utils/deployment");

async function main() {
  [deployer] = await ethers.getSigners();
  [ousd, vault, usdt] = await deployBase(ethers);

  // Mint 1000 OUSD
  mintAmount = ethers.BigNumber.from(10)
    .pow(await ousd.decimals())
    .mul(1000);
  await ousd.mint(deployer.address, mintAmount);

  // Change the supply to MAX_SUPPLY + 1
  supplyAmount = (await ousd.MAX_SUPPLY()).add(1);
  await ousd.changeSupply(supplyAmount);

  // Print supply
  console.log("total supply " + (await ousd.totalSupply()).toString());
  console.log("max supply   " + (await ousd.MAX_SUPPLY()).toString());

  // Output:
  //   total supply 500000000000000000000000000000000000000
  //   max supply   340282366920938463463374607431768211455
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```