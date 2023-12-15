# EnderStaking - Code Functionality Report

This report serves the purpose of describing the code used in `EnderStaking.sol` contract in common terms while also providing some techniacal information.

## Key Features

Let's the `user` stake and withdraw their `END` and `sEND` tokens.

## Inheritance

Inherits from `Initializable` and `OwnableUpgradeable`.

## State Variables

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

## Methods

## initialize()

```javascript
    function initialize(address _end, address _sEnd) external initializer
```

Function `initialize` is the first function to be ran after contract is deployed. Has the `initializer` check on it that ensures that it can only be ran once.

<details>
Takes 2 parameters -

-   `_end` - Sets address of END token
-   `_sEnd` - Sets address of sEND token

Updates following once ran.

1. `__Ownable_init();` - Ownable contract initializer
2. `setAddress(_end, 3);` - sets address of END token from `setAddress()`
3. `setAddress(_sEnd, 4);` - sets address of sEND token from `setAddress()
4. `bondRewardPercentage = 10;` - set initial value of `bondRewardPercentage`

 </details>

## setAddress()

```javascript
function setAddress(address _addr, uint256 _type) public onlyOwner
```

Takes `_addr` and `_type` as parameter and updates address of the token the type presents. Types are shown in full function below.

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

## setBondRewardPercentage()

```javascript
function setBondRewardPercentage(uint256 percent) external onlyOwner
```

Method can only be called by `onlyOwner` and takes `percent` as parameter. Checks if `perfect` is not being passed as 0 and updates `bondRewardPercentage`.

<details>

Full code:

```javascript
 function setBondRewardPercentage(uint256 percent) external onlyOwner {
        if (percent == 0) revert InvalidAmount();

        bondRewardPercentage = percent;
        emit PercentUpdated(bondRewardPercentage);
    }
```

</details>

## stake()

```javascript
function stake(uint256 amount) external
```

Method takes `amount` as input and uses that same amount to let user stake 'END' token and instead mints `sEND`.

<details>

First makes sure `amount` is not 0.

Performs a check if `balanceOf` the `EnderStaking` (address(this)) is 0 for `endToken`:

-   calls `transferFrom` on `endToken`, sending END tokens from `user`, to `EnderStaking` contract for total `amount` number of tokens.
-   calls `epochStakingReward()` using `stETH` as asset.
-   `sEND` amount as `sEndAmount` is calculated by calling `calculateSEndTokens(amount)`.
-   the returned amount of `sEndAmount` is minted for `user`.

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

-   calls `transferFrom` on `endToken`, sending END tokens from `user`, to `EnderStaking` contract for total `amount` number of tokens.
-   `sEND` amount as `sEndAmount` is calculated by calling `calculateSEndTokens(amount)`.
-   the returned amount of `sEndAmount` is minted for `user`.

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

## withdraw()

```javascript
function withdraw(uint256 amount) external
```

Function lets `user` withdraw their staked `END` tokens and instead burns `sEND`.

Starts off with checking if the `amount` argument in this call is not 0 and also the `balanceOf` the `user` for `sEndToken` is greater than `amount`.

-   Calls ` epochStakingReward(stEth)` using sTEth as asset.
-   Calculates `reward` by calling `claimRebaseValue(amount)`
-   Transfers `END` token to `user` for `reward` amount.
-   Burns `sEND` from `user` address for `amount` number of tokens.

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

## epochStakingReward()

```javascript
function epochStakingReward(address _asset) public
```

Method accepts `_asset` as an address and calculates staking rewards for that epoch.

Calculates `totalRewards` by calling `stakingRebasingReward(_asset)` on `enderTreasury` contract, and calculates `rw2` by multiplying that `totalRewards` and `bondRewardPercentage` and dividing that by 100.

```javascript
uint256 totalReward = IEnderTreasury(enderTreasury).stakeRebasingReward(_asset);
uint256 rw2 = (totalReward * bondRewardPercentage) / 100;
```
