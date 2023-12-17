# EnderTreasury - Code Functionality Report

This report serves the purpose of describing the code used in `EnderTreasury.sol` contract in common terms while also providing techniacal information.

<br>

# Contract Flow

This is how the functionality of this contract starts off with:

  <br>
  
### Deposits & Withdrawals

#### `Deposit`

- The function `EnderBond::deposit`() is called in `EnderBond` contract, which transfers the deposited funds to `EnderTreasury` and calls `_deposit`
- `EnderBond::_deposit`() triggers `depositTreasury()` in treasury, passing in the deposit details as `param` and `amountRequired` is calculated using `EnderBond::getLoopCount`(). If this `amountRequired` is greater than 0 then we withdraw that amount from strategy using `withdrawFromStrategy`, else we update `epochDeposit` and `fundsInfo` mapping, which tracks amount of tokens that are deposited per token address.

#### `Withdrawal`

- The `withdraw()` function is called inside of `EnderBond::_withdraw()`.
- Its checked of amount to be withdrawn is greater than what treasury balance has then we withdraw from strategies, else
- `epochWithdrawl` is increased by the token amount and the `fundsInfo` of that token is subtracted by that amount.
- `_transferFunds()` is a the function actually responsible for sending the asset to the user.

  <br>

### Strategies

- **Desposits** in strategies are then made with `depositInStrategy()` . Function lets the treasury sends the funds to Instadapp lite. The deposit amount is added in that strategy specific total depoit. In case of v1 `instaDappDepositValuations` will keep track of it.

> `NOTE`: Function `depositInStrategy` is public without any msg.sender = owner check.

- `totalAssetStakedInStrategy` mapping tracks all the staked tokens in all strategies, its key is `_asset` address.

- **Withdrawals** in strategies are handled in a similar fashion, `withdrawFromStrategy()` is used to withdraw from specified strategy. The withdrawal amount `_withdrawAmt` will be subtracted from `totalAssetStakedInStrategy` with that `_asset` as key.

  <br>

### Returns

- Total returns are calculated in `calculateTotalReturn()`. We calculate `totalReturn` by calling is calculated by getting stEth balance of `EnderStaking` contract and we add `epochWithdrawal` which is all the withdrawals added together, also add `instaDappDepositValuations` which is total amount of deposit in our strategy, also add `stReturn` and subracts both `epochDeposit` and `balanceLastEpoch`.

- Meanwhile to calculate the `depositReturns` which is all the rewards collected on bonded assets. First we calculate `totalReturn` using the previously mentioned function. The math for this return is like - `totalReturn` is multiplied by ( `fundsInfo` of that token \* 100000 ) / total balance of that asset in our `EnderTreasury` contreact + `instaDappDepositValuations` - `totalReturn` and all of the above calculation is divided by `100000`.

<details>

```javascript
depositReturn =
  (totalReturn *
    ((fundsInfo[_stEthAddress] * 100000) /
      (IERC20(_stEthAddress).balanceOf(address(this)) +
        instaDappDepositValuations -
        totalReturn))) /
  100000
```

</details>

<br>

### Rebase Rewards

The rebasing rewards are calculated inside `stakeRebasingReward()` method. This is called by our `EnderStaking` contract during `EnderStaking::epochStakingReward()` and returns `rebaseReward`. It takes `_tokenAddress` as input.

Values required to calculate `rebaseReward` are calculated through:

- `bondReturn` - the number of END tokens minted is called on `EnderBond::endMint()`.
- `depositReturn` - by running `calculateDepositReturn` using `_tokenAddress`.
- `balanceLastEpoch` - by checking current token balance of `_tokenAddress` in our `EnderTreasury` contract.
- `ethPrice` and `priceEnd` are provided by the oracle.
- update `depositReturn` and `bondReturn` by multiplying the `ethPrice` and `priceEnd` and divide by (10 ** `ethDecimal`) and (10 ** `endDecimal`) respectively.

<details>

```javascript
depositReturn = (ethPrice * depositReturn) / 10 ** ethDecimal
bondReturn = (priceEnd * bondReturn) / 10 ** endDecimal
```

</details>

- `rebaseReward` are updated by ( `depositReturn` + ((`depositReturn` \* `nominalYield`) / 10000) - `bondReturn`).
- `rebaseReward` is again updated, multiplaying its value to 10 \*\* ethDecimal divided by `priceEnd`.
  > rebaseReward = ((rebaseReward \* 10 \*\* ethDecimal) / priceEnd);

After calculating this final value of `rebaseReward` rest of the state variables are updates, like `epochWithdrawl` and `epochDeposit` is set to 0, `endMint` is also set to 0 using `EnderBond::resetEndMint()` nad `instaDappLastValuation` is updated and `instaDappWithdrawlValuations` and `instaDappDepositValuations` is also both set to 0.

The calculation of `rebaseReward` is important because the return value decides the amount of `END` minted during `EnderStaking::epochStakingReward()` for `EnderStaking` contract.

<br>

### Withdrawals

<br>

## Key Features

Purpose of `EnderTreasury` is to hold the bond assets, deposit them to strategies and keeping track of the assets and mint END tokens for the user.

<br>

# Inheritance

Inherits from `Initializable`, `OwnableUpgradeable` and `EnderELStrategy`.

<br>

# State Variables

<details>

```javascript
    mapping(address => bool) public strategies; // Contains list of strategies and if they are active.
    mapping(address => uint256) public fundsInfo;
    mapping(address => uint256) public totalAssetStakedInStrategy;
    mapping(address => uint256) public totalRewardsFromStrategy;

    mapping(address => address) public strategyToReceiptToken;

    address private endToken;
    address private enderBond;
    address private enderDepositor;
    address public enderStaking;
    address public instadapp;
    address public lybraFinance;
    address public eigenLayer;
    address public priorityStrategy;
    // address public stEthELS;
    IEnderOracle private enderOracle;

    uint256 public bondYieldBaseRate;
    uint256 public balanceLastEpoch;
    uint256 public nominalYield;
    uint256 public availableFundsPercentage;
    uint256 public reserveFundsPercentage;
    uint256 public epochDeposit;
    uint256 public epochWithdrawl;
    uint256 public instaDappLastValuation;
    uint256 public instaDappWithdrawlValuations;
    uint256 public instaDappDepositValuations;
```

</details>

<br>

# Modifiers

## validStrategy()

Checks if strategy is set to `true` in `strategies`

```javascript
 modifier validStrategy(address str) {
        if (!strategies[str]) revert NotAllowed();
        _;
    }
```

## onlyBond()

Checks if caller of method is `enderBond` address.

```javascript
modifier onlyBond() {
        if (msg.sender != enderBond) revert NotAllowed();
        _;
    }
```

## onlyStaking()

Checks if caller of method is `enderStaking` address.

```javascript
modifier onlyStaking() {
        if (msg.sender != enderStaking) revert NotAllowed();
        _;
    }
```

<br>

# Methods

## initializeTreasury()

```javascript
function initializeTreasury(
        address _endToken,
        address _enderStaking,
        address _bond,
        address _instadapp,
        address _lybraFinance,
        address _eigenLayer,
        uint256 _availableFundsPercentage,
        uint256 _reserveFundsPercentage,
        address _oracle
    ) external initializer
```

Function `initialize` is the first function to be ran after contract is deployed. Has the `initializer` check on it that ensures that it can only be ran once.

<details>
Takes 9 parameters -

- `_endToken` - Sets address of END token
- `_enderStaking` - Sets address of EnderStaking contract.
- `_bond` - Sets EnderBond contract.
- `_instadapp` - Sets address of InstaDapp strategy.
- `_lybraFinance` - Sets address of Lybra Finance strategy.
- `_eigenLayer` - Sets address of Eigen layer strategy.
- `_availableFundsPercentage` - Sets available fund percentage.
- `_reserveFundsPercentage` - Sets reserve fund percentage.
- `_oracle` - Sets address of Oracle.

When method is first activated it is made sure the `availableFundsPercentage` is 70 and `_reserveFundsPercentage` is 30. Else reverts with `InvalidRatio()`

Some of the initial values:

```javascript
strategies[instadapp] = true
strategies[lybraFinance] = true
strategies[eigenLayer] = true
setBondYieldBaseRate(300)
nominalYield = 500
```

Full code:

<details>

```javascript
    function initializeTreasury(
        address _endToken,
        address _enderStaking,
        address _bond,
        address _instadapp,
        address _lybraFinance,
        address _eigenLayer,
        uint256 _availableFundsPercentage,
        uint256 _reserveFundsPercentage,
        address _oracle
    ) external initializer {
        if (_availableFundsPercentage != 70 && _reserveFundsPercentage != 30)
            revert InvalidRatio();
        __Ownable_init();
        enderStaking = _enderStaking;
        enderOracle = IEnderOracle(_oracle);
        instadapp = _instadapp;
        lybraFinance = _lybraFinance;
        eigenLayer = _eigenLayer;
        strategies[instadapp] = true;
        strategies[lybraFinance] = true;
        strategies[eigenLayer] = true;
        setAddress(_endToken, 1);
        setAddress(_bond, 2);
        setBondYieldBaseRate(300);
        priorityStrategy = instadapp;
        nominalYield = 500;
    }
```

</details>

 </details>

 <br>

## setAddress()

```javascript
function setAddress(address _addr, uint256 _type) public onlyOwner
```

Takes `_addr` and `_type` as parameter and updates address of the token the type presents. Types are shown in full function below.

<details>

```javascript
  function setAddress(address _addr, uint256 _type) public onlyOwner {
        if (_addr == address(0)) revert ZeroAddress();

        if (_type == 1) endToken = _addr;
        else if (_type == 2) enderBond = _addr;
        else if (_type == 3) enderDepositor = _addr;
        else if (_type == 4) enderOracle = IEnderOracle(_addr);
        else if (_type == 5) strategyToReceiptToken[instadapp] = _addr;
        else if (_type == 6) strategyToReceiptToken[lybraFinance] = _addr;
        else if (_type == 7) strategyToReceiptToken[eigenLayer] = _addr;
    }
```

</details>

 <br>

## setBondYieldBaseRate()

```javascript
function setBondYieldBaseRate(uint256 _newBaseRate) public onlyOwner
```

This method allows the `owner` to update the `bondYieldBaseRate`.

Full code:

<details>

```javascript
 function setBondYieldBaseRate(uint256 _newBaseRate) public onlyOwner {
        if (_newBaseRate == 0) revert InvalidBaseRate();

        bondYieldBaseRate = _newBaseRate;
        emit BondYieldBaseRateUpdated(_newBaseRate);
    }
```

</details>

<br>

## getAddress()

```javascript
function getAddress(uint256 _type) external view returns (address addr)
```

This method takes `_type` as parameter and returns `addr`, which is address of the contract linked to the `_type`.

Full code:

<details>

```javascript
function getAddress(uint256 _type) external view returns (address addr) {
        if (_type == 1) addr = endToken;
        else if (_type == 2) addr = enderBond;
        else if (_type == 3) addr = enderDepositor;
        else if (_type == 4) addr = address(enderOracle);
    }
```

</details>

<br>

## setStrategy()

```javascript
function setStrategy(address[] calldata _strs, bool _flag) external onlyOwner
```

This method is called by `owner`, allows a list of addresses to be passed as `_strs` and `_flag` as a bool input to set them active/inactive.

Full code:

<details>

```javascript
function setStrategy(
        address[] calldata _strs,
        bool _flag
    ) external onlyOwner {
        if (_strs.length == 0) revert InvalidStrategy();
        unchecked {
            for (uint8 i; i < _strs.length; ++i) {
                strategies[_strs[i]] = _flag;
                emit StrategyUpdated(_strs[i], _flag);
            }
        }
    }
```

</details>

<br>

## setPriorityStrategy()

```javascript
function setPriorityStrategy(address _priorityStrategy) public onlyOwner
```

This method is called by `owner`, and allows them to set one stratefy as priority strategy, it is defined as `priorityStrategy` as address.

Full code:

<details>

```javascript
function setPriorityStrategy(address _priorityStrategy) public onlyOwner {
        priorityStrategy = _priorityStrategy;
        emit PriorityStrategyUpdated(_priorityStrategy);
    }
```

</details>

<br>

## setNominalYield()

```javascript
function setNominalYield(uint256 _nominalYield) public onlyOwner
```

This method is called by `owner`, and allows them to set one stratefy as priority strategy, it is defined as `priorityStrategy` as address.

Full code:

<details>

```javascript
function setPriorityStrategy(address _priorityStrategy) public onlyOwner {
        priorityStrategy = _priorityStrategy;
        emit PriorityStrategyUpdated(_priorityStrategy);
    }
```

</details>

<br>

## depositTreasury()

```javascript
function depositTreasury(EndRequest memory param, uint256 amountRequired) external onlyBond
```

This method is called by `enderBond` contract during `EnderBond::_deposit`. `EndRequest memory param` gets passed in call here, which is basically a data structure of following:

<details>

```javascript
    // IEnderBase.sol code
    struct EndRequest {
        address account;
        address stakingToken; // non-zero address: stETH, zero address: ETH
        uint256 tokenAmt;
    }
```

It checks if passed `amountRequired` is 0, then it runs `withdrawFromStrategy()`, using `param.stakingToken`, `priorityStrategy`, `amountRequired`.

Else adds `param.tokenAmt` to `epochDeposit`, and updates that specific tokens amount in `fundsInfo` as fundsInfo[`param.stakingToken`] += `param.tokenAmt`;

Full code:

<details>

```javascript
function depositTreasury(
        EndRequest memory param,
        uint256 amountRequired
    ) external onlyBond {
        unchecked {
            if (amountRequired > 0) {
                withdrawFromStrategy(
                    param.stakingToken,
                    priorityStrategy,
                    amountRequired
                );
            }
            epochDeposit += param.tokenAmt;
            fundsInfo[param.stakingToken] += param.tokenAmt;
        }
    }
```

</details>
</details>

<br>

## \_transferFunds()

```javascript
function _transferFunds(address _account, address _token, uint256 _amount) private
```

This method is to transfer funds to an account. Takes `_account`, `_token` and `_amount` and is only callable from inside `EnderTreasury` contract.

Makes sure if `_token` is `address(0)` then calls .call of `_amount` of value on `_account`. Else calls `.transfer` on `_token` to `_account` of `_amount` of tokens.

Full code:

<details>

```javascript
function _transferFunds(
        address _account,
        address _token,
        uint256 _amount
    ) private {
        if (_token == address(0)) {
            (bool success, ) = payable(_account).call{value: _amount}("");
            if (!success) revert TransferFailed();
        } else IERC20(_token).transfer(_account, _amount);
    }

```

</details>

<br>

## stakeRebasingReward()

```javascript
function stakeRebasingReward(address _tokenAddress) public onlyStaking returns (uint256 rebaseReward)
```

This method is only callable by `EnderStaking` contract. Takes `_tokenAddress` as input and returns a uint256 `rebaseReward`.

<details>

Starts by calling `EnderBond::endMint` to get `bondReturn` and `depositReturn` is calculated by calling `calculateDepositReturn` with that `tokenAddress`. Balance of that token in `EnderTreasury` is fetched as `balanceLastEpoch`.

```javascript
uint256 bondReturn = IEnderBond(enderBond).endMint();
uint256 depositReturn = calculateDepositReturn(_tokenAddress);
balanceLastEpoch = IERC20(_tokenAddress).balanceOf(address(this));
```

Its checked if `depositReturn` is 0

```javascript
if (depositReturn == 0) {
            epochWithdrawl = 0;
            epochDeposit = 0;
            rebaseReward = 0;
            address receiptToken = strategyToReceiptToken[instadapp];
            instaDappLastValuation = IInstadappLite(instadapp)
                .viewStinstaTokens(
                    IERC20(receiptToken).balanceOf(address(this))
                );
            instaDappWithdrawlValuations = 0;
            instaDappDepositValuations = 0;
        }
```

Else , we get `ethPrice` and `endPrice` from `enderOracle`.

`depositReturn` is updated with (`ethPrice` \* `depositReturn`) / (10 \*\* `ethDecimal`)

while `bondReturn` is updated with (`priceEnd` \* `bondReturn`) / (10 \*\* `endDecimal`)

Rebase reward `rebaseReward` is calculated as below:

```javascript
rebaseReward =
  depositReturn + (depositReturn * nominalYield) / 10000 - bondReturn

rebaseReward = (rebaseReward * 10 ** ethDecimal) / priceEnd
```

States are updated and `EnderBond::resetEndMint` is called.

Full code:

<details>

```javascript
function stakeRebasingReward(
        address _tokenAddress
    ) public onlyStaking returns (uint256 rebaseReward) {
        uint256 bondReturn = IEnderBond(enderBond).endMint();
        uint256 depositReturn = calculateDepositReturn(_tokenAddress);
        balanceLastEpoch = IERC20(_tokenAddress).balanceOf(address(this));
        if (depositReturn == 0) {
            epochWithdrawl = 0;
            epochDeposit = 0;
            rebaseReward = 0;
            address receiptToken = strategyToReceiptToken[instadapp];
            instaDappLastValuation = IInstadappLite(instadapp)
                .viewStinstaTokens(
                    IERC20(receiptToken).balanceOf(address(this))
                );
            instaDappWithdrawlValuations = 0;
            instaDappDepositValuations = 0;
        } else {
            //we get the eth price in 8 decimal and  depositReturn= 18 decimal  bondReturn = 18decimal
            (uint256 ethPrice, uint256 ethDecimal) = enderOracle.getPrice(
                address(0)
            );
            (uint256 priceEnd, uint256 endDecimal) = enderOracle.getPrice(
                address(endToken)
            );
            depositReturn = (ethPrice * depositReturn) / (10 ** ethDecimal);
            bondReturn = (priceEnd * bondReturn) / (10 ** endDecimal);

            rebaseReward = (
                (depositReturn +
                    ((depositReturn * nominalYield) / 10000) -
                    bondReturn)
            );

            rebaseReward = ((rebaseReward * 10 ** ethDecimal) / priceEnd);

            epochWithdrawl = 0;
            epochDeposit = 0;
            IEnderBond(enderBond).resetEndMint();
            address receiptToken = strategyToReceiptToken[instadapp];
            instaDappLastValuation = IInstadappLite(instadapp)
                .viewStinstaTokens(
                    IERC20(receiptToken).balanceOf(address(this))
                );
            instaDappWithdrawlValuations = 0;
            instaDappDepositValuations = 0;
        }
    }
```

</details>
</details>

<br>

## depositInStrategy()

```javascript
function depositInStrategy(address _asset, address _strategy, uint256 _depositAmt) public validStrategy(strategy)
```

This method deposits the available funds to strategies. Takes `_asset` address, `_strategy` address and `_depositAmt` of tokens. Has a `validStrategy(strategy)`, so it checks if a strategy is allowed.

<details>

Checks if `_depositAmount` us not 0, the `_asset` and `_strategy` is not address(0).

`_asset` is deposited to the strategy address.

```javascript
if (_strategy == instadapp) {
  IERC20(_asset).approve(_strategy, _depositAmt)
  IInstadappLite(instadapp).deposit(_depositAmt) // note for testing we changed the function sig.
  instaDappDepositValuations += _depositAmt
} else if (_strategy == lybraFinance) {
  IERC20(_asset).approve(lybraFinance, _depositAmt)
  ILybraFinance(lybraFinance).depositAssetToMint(_depositAmt, 0)
} else if (_strategy == eigenLayer) {
  //Todo will add the instance while going on mainnet.
}
```

`totalAssetStakedInStrategy` of that asset is updated with `_depositAmt` and emits `StrategyDeposit`(`_asset`, `_strategy`, `_depositAmt`).

Full code:

<details>

```javascript
    function depositInStrategy(
        address _asset,
        address _strategy,
        uint256 _depositAmt
    ) public validStrategy(strategy) {
        // stEthBalBeforeStDep = IERC20(_asset).balanceOf(address(this));
        if (_depositAmt == 0) revert ZeroAmount();
        if (_asset == address(0) || _strategy == address(0))
            revert ZeroAddress();
        if (_strategy == instadapp) {
            IERC20(_asset).approve(_strategy, _depositAmt);
            IInstadappLite(instadapp).deposit(_depositAmt); // note for testing we changed the function sig.
            instaDappDepositValuations += _depositAmt;
        } else if (_strategy == lybraFinance) {
            IERC20(_asset).approve(lybraFinance, _depositAmt);
            ILybraFinance(lybraFinance).depositAssetToMint(_depositAmt, 0);
        } else if (_strategy == eigenLayer) {
            //Todo will add the instance while going on mainnet.
        }
        totalAssetStakedInStrategy[_asset] += _depositAmt;
        emit StrategyDeposit(_asset, _strategy, _depositAmt);
    }
```

</details>
</details>

<br>

## withdrawFromStrategy()

```javascript
function withdrawFromStrategy(address _asset, address _strategy, uint256 _withdrawAmt) public validStrategy(_strategy) returns (uint256 _returnAmount)
```

This method withdraw the available funds from strategies. Takes `_asset` address, `_strategy` address and `_depositAmt` of tokens. Has a `validStrategy(strategy)`, so it checks if a strategy is allowed.

<details>

Checks if `_depositAmount` us not 0, the `_asset` and `_strategy` is not address(0).

`receiptToken` is set from `strategyToReceiptToken`[`_strategy`];

Withdraw request is made to the `_strategy` to `receiptToken` address.

```javascript
if (_strategy == instadapp) {
  //Todo set the asset as recipt tokens and need to check the assets ratio while depolying on mainnet
  _withdrawAmt = IInstadappLite(instadapp).viewStinstaTokensValue(_withdrawAmt)
  IERC20(receiptToken).approve(instadapp, _withdrawAmt)
  _returnAmount = IInstadappLite(instadapp).withdrawStinstaTokens(_withdrawAmt)
  instaDappWithdrawlValuations += _returnAmount
} else if (_strategy == lybraFinance) {
  IERC20(receiptToken).approve(lybraFinance, _withdrawAmt)
  _returnAmount = ILybraFinance(lybraFinance).withdraw(
    address(this),
    _withdrawAmt
  )
}
```

Updates the balances and emits log.

```javascript
totalAssetStakedInStrategy[_asset] -= _withdrawAmt;
        if (_returnAmount > 0) {
            totalRewardsFromStrategy[_asset] += _returnAmount;
        }
        emit StrategyWithdraw(_asset, _strategy, _returnAmount);
```

Full code:

<details>

```javascript
function withdrawFromStrategy(
        address _asset,
        address _strategy,
        uint256 _withdrawAmt
    ) public validStrategy(_strategy) returns (uint256 _returnAmount) {
        if (_withdrawAmt == 0) revert ZeroAmount();
        if (_asset == address(0) || _strategy == address(0))
            revert ZeroAddress();
        address receiptToken = strategyToReceiptToken[_strategy];
        if (_strategy == instadapp) {
            //Todo set the asset as recipt tokens and need to check the assets ratio while depolying on mainnet
            _withdrawAmt = IInstadappLite(instadapp).viewStinstaTokensValue(
                _withdrawAmt
            );
            IERC20(receiptToken).approve(instadapp, _withdrawAmt);
            _returnAmount = IInstadappLite(instadapp).withdrawStinstaTokens(
                _withdrawAmt
            );
            instaDappWithdrawlValuations += _returnAmount;
        } else if (_strategy == lybraFinance) {
            IERC20(receiptToken).approve(lybraFinance, _withdrawAmt);
            _returnAmount = ILybraFinance(lybraFinance).withdraw(
                address(this),
                _withdrawAmt
            );
        }
        totalAssetStakedInStrategy[_asset] -= _withdrawAmt;
        if (_returnAmount > 0) {
            totalRewardsFromStrategy[_asset] += _returnAmount;
        }
        emit StrategyWithdraw(_asset, _strategy, _returnAmount);
    }
```

</details>
</details>

<br>

## withdraw()

```javascript
function withdraw(EndRequest memory param, uint256 amountRequired) external onlyBond
```

This method is called by `enderBond`, Uses `EndRequest` data structure mentioned earlier and `amountRequired` as both inputs.

Checks if `amountRequired` greater than `balanceOf` that `stakingToken` in our treasury contract. Then `withdrawFromStrategy` is called to writhdraw funds from the strategy itself.

```javascript
if (amountRequired > IERC20(param.stakingToken).balanceOf(address(this))) {
  withdrawFromStrategy(param.stakingToken, priorityStrategy, amountRequired)
}
```

`epochWithdrawal` is updated by adding `param.tokenAmt` and `fundsInfo` of that `param.stakingToken` is reduced by `param.tokenAmt`.

Treasury then calls `_transferFunds` to send the transaction that sends the assets back to the user, and emits `TreasuryWithdraw`.

```javascript
epochWithdrawl += param.tokenAmt;
fundsInfo[param.stakingToken] -= param.tokenAmt;

// bond token transfer
 _transferFunds(param.account, param.stakingToken, param.tokenAmt);
emit TreasuryWithdraw(param.stakingToken, param.tokenAmt);
```

Full code:

<details>

```javascript
function withdraw(
        EndRequest memory param,
        uint256 amountRequired
    ) external onlyBond {
        if (
            amountRequired > IERC20(param.stakingToken).balanceOf(address(this))
        ) {
            withdrawFromStrategy(
                param.stakingToken,
                priorityStrategy,
                amountRequired
            );
        }
        epochWithdrawl += param.tokenAmt;
        fundsInfo[param.stakingToken] -= param.tokenAmt;

        // bond token transfer
        _transferFunds(param.account, param.stakingToken, param.tokenAmt);
        emit TreasuryWithdraw(param.stakingToken, param.tokenAmt);
    }
```

</details>

<br>

## collect()

```javascript
function collect(address account, uint256 amount) external onlyBond
```

This method is called by `enderBond` contract and transfers END tokens of `amount` as rewards to `account` user. Emits `Collect` event.

```javascript
function collect(address account, uint256 amount) external onlyBond {
        IERC20(endToken).transfer(account, amount);
        emit Collect(account, amount);
    }
```

 <br>

## mintEndToUser()

```javascript
function mintEndToUser(address _to, uint256 _amount) external onlyBond
```

This method is called by `enderBond` contract and mints END tokens to `_to` address of `_amount`. Emits `MintEndToUser` event.

```javascript
function mintEndToUser(address _to, uint256 _amount) external onlyBond {
        ///just return for temp  should changethe
        IEndToken(endToken).mint(_to, _amount);
        emit MintEndToUser(_to, _amount);
    }
```

 <br>

## calculateTotalReturn()

```javascript
function calculateTotalReturn(address _stEthAddress) internal view returns (uint256 totalReturn)
```

This method calculates total return `totalReturn` of a given asset. Takes `_stEthAddress` as input for asset address.

`Note`: It would be better to update `_stEthAddress`to something like `_tokenAddr` to make it not look like it only accepts stETH as address.

<details>

Starts with creating couple variables as `stReturn` a uint256 and `receiptToken` which is the address. Then we calculate the balance of this `receiptToken` by calling `balanceOf` of token, on our `EnderTreasury`.

```javascript
uint256 stReturn;
        address receiptToken = strategyToReceiptToken[instadapp];
        uint256 receiptTokenAmount = IInstadappLite(receiptToken).balanceOf(
            address(this)
        );
```

Balance of the asset in question `receiptToken`'s balance of `EnderTreasury` contract is checked to be greater than 0:

- Then `stReturn` is calculated by calling `viewStinstaTokems` on `receiptToken` address + `instaDappWithdrawlValuations` - `instaDappDepositValuations` - `instaDappLastValuation`.

```javascript
if (IInstadappLite(receiptToken).balanceOf(address(this)) > 0) {
  stReturn =
    IInstadappLite(receiptToken).viewStinstaTokens(receiptTokenAmount) +
    instaDappWithdrawlValuations -
    instaDappDepositValuations -
    instaDappLastValuation
}
```

Finally `totalReturn` is calculated by checking balance of passed `_stEthAddress` token address on our `EnderTreasury` contract + `epochWithddrawal` + `instaDappDepositValuations`- `stReturn` - `epochDeposit` - `balanceLastEpoch`.

```javascript
totalReturn =
  IERC20(_stEthAddress).balanceOf(address(this)) +
  epochWithdrawl +
  instaDappDepositValuations +
  stReturn -
  epochDeposit -
  balanceLastEpoch
```

Full code:

<details>

```javascript
function calculateTotalReturn(
        address _stEthAddress
    ) internal view returns (uint256 totalReturn) {
        uint256 stReturn;
        address receiptToken = strategyToReceiptToken[instadapp];
        uint256 receiptTokenAmount = IInstadappLite(receiptToken).balanceOf(
            address(this)
        );
        if (IInstadappLite(receiptToken).balanceOf(address(this)) > 0) {
            stReturn =
                IInstadappLite(receiptToken).viewStinstaTokens(
                    receiptTokenAmount
                ) +
                instaDappWithdrawlValuations -
                instaDappDepositValuations -
                instaDappLastValuation;
        }
        //todo add stlogic
        totalReturn =
            IERC20(_stEthAddress).balanceOf(address(this)) +
            epochWithdrawl +
            instaDappDepositValuations +
            stReturn -
            epochDeposit -
            balanceLastEpoch;
    }
```

</details>
</details>

<br>

## calculateDepositReturn()

```javascript
function calculateDepositReturn(address _stEthAddress) public view returns (uint256 depositReturn)
```

This method calculates the deposit returns `depositReturn` based on total return, takes `_stEthAddress` as address of token.

<details>

Sets a `totalReturn` value which is calculated using `calculateTotalReturn`. If `totalReturn` or `balanceLastEpoch` is 0 then `depositReturn` is also 0.

Else `depositReturn` is calculated using:

(`totalReturn` \* (( total funds of that specific token \* 100000) / `balanceOf` that token inside `EnderTreasury` + `instaDappDepositValuations` - `totalReturn`)) / 100000

```javascript
depositReturn =
  (totalReturn *
    ((fundsInfo[_stEthAddress] * 100000) /
      (IERC20(_stEthAddress).balanceOf(address(this)) +
        instaDappDepositValuations -
        totalReturn))) /
  100000
```

Full code:

<details>

```javascript
function calculateDepositReturn(
        address _stEthAddress
    ) public view returns (uint256 depositReturn) {
        uint256 totalReturn = calculateTotalReturn(_stEthAddress);
        if (totalReturn == 0 || balanceLastEpoch == 0) {
            depositReturn = 0;
        } else {
            //here we have to multiply 100000and dividing so that the balanceLastEpoch < fundsInfo[_stEthAddress].depositFunds
            depositReturn =
                (totalReturn *
                    ((fundsInfo[_stEthAddress] * 100000) /
                        (IERC20(_stEthAddress).balanceOf(address(this)) +
                            instaDappDepositValuations -
                            totalReturn))) /
                100000;
        }
    }
```

</details>
</details>
