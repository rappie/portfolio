## Title
`OUSD.burn()` allows for destroying supply while balance remains

## Introduction
There is a bug present in the `OUSD` contract. Because of rounding errors there is a possible situation where `OUSD.burn()` burns from the total supply and **not** from the balance. Luckily this is only possible with 1 unit of tokens at the time.

The bug is present in the deployed code on the Ethereum mainnet, located here:
- https://etherscan.io/address/0x2A8e1E676Ec238d8A992307B495b45B3fEAa5e86?utm_source=immunefi

To simplify testing, I'm using the contract in isolation. In the POC below I have removed several `onlyVault` modifiers to make this possible.


## Description
As seen in earlier reports it's clear by now that `OUSD` has to deal with certain rounding errors. Instead of going over this again, let's dive straight into the `OUSD._burn()` code:
```solidity
function _burn(address _account, uint256 _amount) internal nonReentrant {
  require(_account != address(0), "Burn from the zero address");
  if (_amount == 0) {
      return;
  }

  bool isNonRebasingAccount = _isNonRebasingAccount(_account);
  uint256 creditAmount = _amount.mulTruncate(_creditsPerToken(_account));
  uint256 currentCredits = _creditBalances[_account];

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

  // Remove from the credit tallies and non-rebasing supply
  if (isNonRebasingAccount) {
      nonRebasingSupply = nonRebasingSupply.sub(_amount);
  } else {
      _rebasingCredits = _rebasingCredits.sub(creditAmount);
  }

  _totalSupply = _totalSupply.sub(_amount);

  emit Transfer(_account, address(0), _amount);
}
```

This function basically has 2 tasks:
- 1) Remove amount to burn from user balance
- 2) Remove amount to burn from total supply

For the outside world this amount needs to be the same in both cases.

Let's take a look at the specific code for these cases when dealing with non-rebasing tokens.

Removing amount to burn from user balance:
```solidity
  uint256 creditAmount = _amount.mulTruncate(_creditsPerToken(_account));
  ...
  _creditBalances[_account] = _creditBalances[_account].sub(creditAmount);
```

Removing amount to burn from total supply
```solidity
  nonRebasingSupply = nonRebasingSupply.sub(_amount);
```

As you can see, the amounts here are not the same. This makes sense: because `nonRebasingSupply` is not part of the credit & rebasing mechanism it can use the unmodified `_amount` variable.

However, this means one case is subjected to the rounding problems and one is not.

Now let's look at the situation where you're burning `1` unit of token. In the case of the non rebasing supply it's easy: just subtract `1`. The credit amount is a bit more tricky, because we're dealing with the "credits per token" multiplier. If this multiplier is smaller than `1e18` (meaning `1` in practice) the amount will be a tiny bit smaller than `1`, which gets rounded down to `0`.

The end result here is a situation where the supply becomes smaller while the balance stays the same.

See the PoC below for more info.

## Recommendations
I don't see one clear solution, mostly because the rounding problem is so deeply interwoven within the code. I don't think this can be fixed as a whole without a full rewrite, if at all.

I have thought about `OUSD.burn()` a lot though, and I've come up with a couple of suggestions. None of which are tested.
- 1) You could add some upfront calculation to `_burn` . Here you can calculate both amounts, compare them and take actions accordingly
- 2) You could add a minimum amount to `Vault.redeem()`. This will solve the entire rounding problem for redeeming/burning. I have seen this in other protocols, for example [here](https://github.com/buttonwood-protocol/tranche/blob/main/contracts/BondController.sol#L30)
- 3) Last minute idea: perhaps simply rounding up instead of down is enough?

## Impact
Hard to say, as usual :)

The impact seems to be higher compared to my other reports, because there is a real attack vector here. However, I still don't see any real world scenario where such an attack will be executed as you need to call `Vault.redeem()` 1e18 times (or 1e27?) to destroy just one dollar.

This is why I have decided to assign Medium severity.

Besides the obvious attack vector, there are also other side effects. For example: as the `nonRebasingSupply` can become smaller than the sum of all non-rebasing balances, `OUSD.rebaseOptIn` can fail because of this line:
```solidity
nonRebasingSupply = nonRebasingSupply.sub(balanceOf(msg.sender));
```

See also:
https://github.com/OriginProtocol/origin-dollar/blob/master/contracts/contracts/token/OUSD.sol#L504

I have have also sent out some questions to my friends (without mentioning any specifics, ofcourse) asking if they see any reason for more concern. If the impact turns out to be higher, I trust the issue and reward will be retroactively changed accordingly.

## Proof of Concept

```solidity
const hre = require("hardhat");
const ethers = hre.ethers;

const { deployBase } = require("../utils/deployment");

async function main() {
  [deployer] = await ethers.getSigners();
  [ousd, vault, usdt] = await deployBase(ethers);

  // Mint some OUSD
  mintAmount = 1000000000000;
  await ousd.mint(deployer.address, mintAmount);

  // Change supply by arbitrary amount to cause rounding errors
  supplyAmount = 1014774251433;
  await ousd.changeSupply(supplyAmount);

  // Opt out
  await ousd.rebaseOptOut();

  // Burn X times 1 unit of tokens. This destroys X supply.
  burnAmount = 20;
  for (let i = 0; i < burnAmount; i++) {
    await ousd.burn(deployer.address, 1);
  }

  // Print balance vs supply
  totalSupply = await ousd.totalSupply();
  balance = await ousd.balanceOf(deployer.address);
  console.log("total supply " + totalSupply.toString());
  console.log("balance      " + balance.toString());

  // Example script output:
  //   total supply 1014774251413
  //   balance      1014774251433
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```


