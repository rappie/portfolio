
`Vault.redeem()` fails with only non-rebasing credits in the protocol

## Introduction
There is a bug present in the `Vault` contract. The `redeem` function reverts when there are only non-rebasing credits present in the `OUSD` contract.

The bug is present in the deployed code on the Ethereum mainnet, located here:
- https://etherscan.io/address/0xe75d77b1865ae93c7eaa3040b038d7aa7bc02f70?utm_source=immunefi
- https://etherscan.io/address/0x2A8e1E676Ec238d8A992307B495b45B3fEAa5e86?utm_source=immunefi

For demonstrational purposes I'm using the code in the Origin Dollar Github repository.
https://github.com/OriginProtocol/origin-dollar/tree/f6069b5535597587f5bb57cf5c66db07388ed4a0

The bug is present in both codebases.
	
## Bug Description & Recommendations
The bug gets triggered by calling the `Vault.redeem()` function.
```javascript
  await vault.redeem(redeemAmount, 0);
  ```

There are two conditions necessary to trigger the bug:
- No non-rebasing credits in `OUSD`
- Redeem more than the rebase threshold

See the Proof of Concept below for more information on how to reproduce the error.

After the redeem logic in `Vault.redeem` is complete, a rebase happens.
```solidity
        if (_amount > rebaseThreshold && !rebasePaused) {
            _rebase();
        }
```

After the rebasing logic is complete, the `_rebase` function calls `OUSD.changeSupply`.
```
        if (vaultValue > ousdSupply) {
            oUSD.changeSupply(vaultValue);
        }
```

This is where the actual bug resides.
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

As you can see, there is a check that requires `rebasingCreditsPerToken` to always be greater than zero.
```solidity
        require(_rebasingCreditsPerToken > 0, "Invalid change in supply");
```

If you assume there are always some rebasing credits available, this makes sense. A rebase from something to zero should never happen. However, in our case there are only non-rebasing credits, so having this check at all does not make sense.

A simple bugfix would be to handle both cases seperately using an `if` statement:
```solidity
        if (_rebasingCredits > 0) {
            _rebasingCreditsPerToken = _rebasingCredits.divPrecisely(
                _totalSupply.sub(nonRebasingSupply)
            );

            require(_rebasingCreditsPerToken > 0, "Invalid change in supply");

            _totalSupply = _rebasingCredits
                .divPrecisely(_rebasingCreditsPerToken)
                .add(nonRebasingSupply);
        } else {
            _totalSupply = nonRebasingSupply;
        }
```

This works perfectly, and passes all tests in the testsuite.

I would recommend looking a bit deeper at the code though, as this is not a very elegant solution. For example: there are three different `_totalSupply = ...` statements here. Your devs will most likely do a much better job of refactoring this than me so I'll leave it at this :)

## Impact
The impact is low. Origin Dollar is a mature protocol and it is highly unlikely that there will ever be a situation where there are only non-rebasing credits left in the protocol.

It is still a bug that could cause unforeseen problems in the future. These future scenarios could be refactors, protocol upgrades, migrations, forks, etc.

## Proof of Concept
```javascript
const hre = require("hardhat");
const ethers = hre.ethers;

const { deployBase } = require("../utils/deployment");

async function main() {
  // Deploy everything
  [ousd, vault, usdt] = await deployBase(ethers);

  // First opt out, so we're adding to our non-rebasing credit
  await ousd.rebaseOptOut();

  // The amount we mint doesnt matter, as long as we have enough to redeem.
  mintAmount = ethers.BigNumber.from(10_000);
  mintAmount = mintAmount.mul(
    ethers.BigNumber.from(10).pow(await usdt.decimals())
  );

  // Mint OUSD
  await usdt.approve(vault.address, mintAmount);
  await vault.mint(usdt.address, mintAmount, 0);

  // Redeem less than we have but more than the rebase threshold
  redeemAmount = ethers.BigNumber.from(1_000);
  redeemAmount = redeemAmount.mul(
    ethers.BigNumber.from(10).pow(await ousd.decimals())
  );

  // Add 1 to force some rounding errors.
  redeemAmount = redeemAmount.add(1);

  // Perform redeem. This reverts
  await vault.redeem(redeemAmount, 0);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

Custom deployment script:
```javascript
async function deployBase(ethers) {
  [deployer, attacker0, attacker1, victim] = await ethers.getSigners();

  const OUSD = await ethers.getContractFactory("OUSD");
  const ousd = await OUSD.deploy();
  await ousd.deployed();

  const Vault = await ethers.getContractFactory("Vault");
  const vault = await Vault.deploy();
  await vault.deployed();

  const MockUSDT = await ethers.getContractFactory("MockUSDT");
  const usdt = await MockUSDT.deploy();
  await usdt.deployed();

  const MockChainlinkOracleFeed = await ethers.getContractFactory(
    "MockChainlinkOracleFeed"
  );
  const usdtFeed = await MockChainlinkOracleFeed.deploy(
    ethers.utils.parseUnits("1", 8).toString(),
    18
  );
  await usdtFeed.deployed();

  const OracleRouterDev = await ethers.getContractFactory("OracleRouterDev");
  const oracle = await OracleRouterDev.deploy();
  await oracle.deployed();

  await oracle.setFeed(usdt.address, usdtFeed.address);

  await vault.initialize(oracle.address, ousd.address);
  await ousd.initialize("Origin Dollar USD", "OUSD", vault.address);

  await vault.supportAsset(usdt.address);
  await vault.unpauseCapital();

  await usdt.connect(deployer).mint(100_000_000_000_000);
  await usdt.connect(attacker0).mint(100_000_000_000_000);
  await usdt.connect(attacker1).mint(100_000_000_000_000);
  await usdt.connect(victim).mint(100_000_000_000_000);

  return [ousd, vault, usdt];
}

module.exports = {
  deployBase,
};
```
