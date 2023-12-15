# EnderStaking - Code Functionality Report

This report serves the purpose of describing the code used in `EnderStaking.sol` contract in common terms while also providing some techniacal information.

<br>

## Key Features

Let's the `user` stake and withdraw their `END` tokens in exchange for `sEND`. It is also responsible for calculating the `rebasingIndex`.

<br>

# Inheritance

Inherits from `Initializable` and `OwnableUpgradeable`.

<br>

# State Variables

<details>

```javascript
    uint public bondRewardPercentage; // Reward percentage for bonds
    uint public rebasingIndex;     // Rebase Index

    address public endToken; // END token contract address
    address public sEndToken; // sEND token contract address
    address public enderTreasury; // EnderTreasury contract address
    address public enderBond; // EnderBond contract address
    address public keeper;  // Chainlink Keeper contract address
    address public stEth;    // stETH contract address
```

</details>

<br>

# Methods

<br>

## initialize()

```javascript
    function initialize(address _end, address _sEnd) external initializer
```

Function `initialize` is the first function to be ran after contract is deployed. Has the `initializer` check on it that ensures that it can only be ran once.

<details>
Takes 2 parameters -

- `_end` - Sets address of END token
- `_sEnd` - Sets address of sEND token

Updates following once ran.

1. `__Ownable_init();` - Ownable contract initializer
2. `setAddress(_end, 3);` - sets address of END token from `setAddress()`
3. `setAddress(_sEnd, 4);` - sets address of sEND token from `setAddress()
4. `bondRewardPercentage = 10;` - set initial value of `bondRewardPercentage`

 </details>

 <br>

## setAddress()

```javascript
function setAddress(address _addr, uint256 _type) public onlyOwner
```

Takes `_addr` and `_type` as parameter and updates address of the token the type presents. Types are shown in full function below.

<br>

Full code:

<details>

```javascript
    function setAddress(address _addr, uint256 _type) public onlyOwner {
        if (_addr == address(0)) revert ZeroAddress();

        if (_type == 1) enderBond = _addr;
        else if (_type == 2) enderTreasury = _addr;
        else if (_type == 3) endToken = _addr;
        else if (_type == 4) sEndToken = _addr;
        else if (_type == 5) keeper = _addr;
        else if (_type == 6) stEth = _addr;

        emit AddressUpdated(_addr, _type);
    }

```

</details>

<br>

## setBondRewardPercentage()

```javascript
function setBondRewardPercentage(uint256 percent) external onlyOwner
```

Method can only be called by `onlyOwner` and takes `percent` as parameter. Checks if `perfect` is not being passed as 0 and updates `bondRewardPercentage`.

<details>

<br>

Full code:

```javascript
 function setBondRewardPercentage(uint256 percent) external onlyOwner {
        if (percent == 0) revert InvalidAmount();

        bondRewardPercentage = percent;
        emit PercentUpdated(bondRewardPercentage);
    }
```

</details>

<br>

## stake()

```javascript
function stake(uint256 amount) external
```

Method takes `amount` as input and uses that same amount to let user stake 'END' token and instead mints `sEND`.

<details>

First makes sure `amount` is not 0.

Performs a check if `balanceOf` the `EnderStaking` (address(this)) is 0 for `endToken`:

- calls `transferFrom` on `endToken`, sending END tokens from `user`, to `EnderStaking` contract for total `amount` number of tokens.
- calls `epochStakingReward()` using `stETH` as asset.
- `sEND` amount as `sEndAmount` is calculated by calling `calculateSEndTokens(amount)`.
- the returned amount of `sEndAmount` is minted for `user`.

```javascript
  if (ISEndToken(endToken).balanceOf(address(this)) == 0) {
            ISEndToken(endToken).transferFrom(msg.sender, address(this), amount);
            epochStakingReward(stEth);
            uint256 sEndAmount = calculateSEndTokens(amount);
            console.log("Receipt token:- ", sEndAmount);
            ISEndToken(sEndToken).mint(msg.sender, sEndAmount);
        }
```

Else if `balanceOf` the `EnderStaking` (address(this)) is not 0:

- calls `transferFrom` on `endToken`, sending END tokens from `user`, to `EnderStaking` contract for total `amount` number of tokens.
- `sEND` amount as `sEndAmount` is calculated by calling `calculateSEndTokens(amount)`.
- the returned amount of `sEndAmount` is minted for `user`.

```javascript
else {
            ISEndToken(endToken).transferFrom(msg.sender, address(this), amount);
            uint256 sEndAmount = calculateSEndTokens(amount);
            console.log("Receipt token:- ", sEndAmount);
            ISEndToken(sEndToken).mint(msg.sender, sEndAmount);
            epochStakingReward(stEth);
        }
```

Emits `Stake(msg.sender, amount)`

<br>

Full code:

<details>

```javascript
    function stake(uint256 amount) external {
        if (amount == 0) revert InvalidAmount();
        console.log("End token deposit:- ", amount);
        console.log(ISEndToken(endToken).balanceOf(address(this)));
        if (ISEndToken(endToken).balanceOf(address(this)) == 0) {
            ISEndToken(endToken).transferFrom(msg.sender, address(this), amount);
            epochStakingReward(stEth);
            uint256 sEndAmount = calculateSEndTokens(amount);
            console.log("Receipt token:- ", sEndAmount);
            ISEndToken(sEndToken).mint(msg.sender, sEndAmount);
        } else {
            ISEndToken(endToken).transferFrom(msg.sender, address(this), amount);
            uint256 sEndAmount = calculateSEndTokens(amount);
            console.log("Receipt token:- ", sEndAmount);
            ISEndToken(sEndToken).mint(msg.sender, sEndAmount);
            epochStakingReward(stEth);
        }
        emit Stake(msg.sender, amount);
    }
```

</details>
</details>

<br>

## withdraw()

```javascript
function withdraw(uint256 amount) external
```

Function lets `user` withdraw their staked `END` tokens and instead burns `sEND`.

Starts off with checking if the `amount` argument in this call is not 0 and also the `balanceOf` the `user` for `sEndToken` is greater than `amount`.

- Calls ` epochStakingReward(stEth)` using sTEth as asset.
- Calculates `reward` by calling `claimRebaseValue(amount)`
- Transfers `END` token to `user` for `reward` amount.
- Burns `sEND` from `user` address for `amount` number of tokens.

<br>

Full code:

<details>

```javascript
    function withdraw(uint256 amount) external {
        if (amount == 0) revert InvalidAmount();
        if (ISEndToken(sEndToken).balanceOf(msg.sender) < amount) revert InvalidAmount();
        // add reward
        epochStakingReward(stEth);
        uint256 reward = claimRebaseValue(amount);
        console.log("Withraw amount of staking contract:- ", reward);
        // transfer token
        ISEndToken(endToken).transfer(msg.sender, reward);
        ISEndToken(sEndToken).burn(msg.sender, amount);
        emit Withdraw(msg.sender, amount);
    }
```

</details>

<br>

## epochStakingReward()

```javascript
function epochStakingReward(address _asset) public
```

Method accepts `_asset` as an address and calculates staking rewards for that epoch.

<details>
Calculates `totalRewards` by calling `stakingRebasingReward(_asset)` on `enderTreasury` contract, and calculates `rw2` by multiplying that `totalRewards` and `bondRewardPercentage` and dividing that by 100.

```javascript
uint256 totalReward = IEnderTreasury(enderTreasury).stakeRebasingReward(_asset);
uint256 rw2 = (totalReward * bondRewardPercentage) / 100;
```

uint256 `sendTokens` the amount of sEND that will be minted to `enderBond` contract. END tokens are of `totalRewards` amount is minted in `EnderStaking` contract.
This amount of sEND `sendTokens` is calculated via calling `EnderStaking::calculateSEndTokens(rw2)` and using `rw2` which we addressed earlier.

```javascript
uint256 sendTokens = calculateSEndTokens(rw2);
        ISEndToken(sEndToken).mint(enderBond, sendTokens);
        ISEndToken(endToken).mint(address(this), totalReward);
```

`EnderBond::epochRewardShareIndexForSend(sendTokens)` is called inside `enderBond` contract and `calculateRebaseIndex()` is called.

Method ends by emitting `EpochStakingReward`(`_asset`, `totalReward`, `rw2`, `sendTokens`)

<br>
Full code:

<details>

```javascript
function epochStakingReward(address _asset) public {
        // if (msg.sender != keeper) revert NotKeeper();
        uint256 totalReward = IEnderTreasury(enderTreasury).stakeRebasingReward(
            _asset
        );
        uint256 rw2 = (totalReward * bondRewardPercentage) / 100;

        uint256 sendTokens = calculateSEndTokens(rw2);
        ISEndToken(sEndToken).mint(enderBond, sendTokens);
        ISEndToken(endToken).mint(address(this), totalReward);
        IEnderBond(enderBond).epochRewardShareIndexForSend(sendTokens);
        calculateRebaseIndex();
        emit EpochStakingReward(_asset, totalReward, rw2, sendTokens);
    }

```

</details>
</details>

<br>

## calculateSEndTokens()

```javascript
function calculateSEndTokens(
        uint256 _endAmount
    ) public view returns (uint256 sEndTokens)
```

This method is responsible for calculating the amount of `sEND` to be minted in both `stake()` and `epochStakingReward()`. It takes uint256 `_endAmount` as parameter and returns `sEndTokens` of the same type.

<details>

It starts by checking if rebasingIndex is equal to 0. In this case `sEndTokens` is equal to `_endAmount`.

Else `sEndTokens` is calculated by (`_endAmount` / `rebasingIndex`)
<br>
Full code:

<details>

```javascript
if (rebasingIndex == 0) {
  console.log("I'm here", _endAmount)
  sEndTokens = _endAmount
  return sEndTokens
} else {
  console.log('else', _endAmount, rebasingIndex)
  sEndTokens = _endAmount / rebasingIndex
  return sEndTokens
}
```

</details>

</details>

<br>

## calculateRebaseIndex()

```javascript
function calculateRebaseIndex() internal
```

This method calculates the rebasing index.

Gets `endBalStaking` which is total END being staked currently, and total supply of of sEND tokens as `sEndTotalSupply`.
Makes a check if either of those values are 0 then sets it as `1`. Else `rebasingIndex` is calculated as (`endBalStaking` \* 1e18) / `sEndTotalSupply`;
<br>
Full code:

<details>

```javascript
function calculateRebaseIndex() internal {
        uint256 endBalStaking = ISEndToken(endToken).balanceOf(address(this));
        uint256 sEndTotalSupply = ISEndToken(sEndToken).totalSupply();
        if (endBalStaking == 0 || sEndTotalSupply == 0) {
            rebasingIndex = 1;
        } else {
            rebasingIndex = (endBalStaking * 10 ** 18) / sEndTotalSupply;
        }
    }
```

</details>

<br>

## claimRebaseValue()

```javascript
function claimRebaseValue(
        uint256 _sendAMount
    ) internal view returns (uint256 reward)
```

This method takes in `_sendAmount` as input and returns calculated `reward`.

This `reward` is calculated by (`_sendAmount` \* `rebaseIndex`).

<br>
Full code:

<details>

```javascript
function claimRebaseValue(
        uint256 _sendAMount
    ) internal view returns (uint256 reward) {
        reward = (_sendAMount * rebasingIndex);
    }
```

</details>
