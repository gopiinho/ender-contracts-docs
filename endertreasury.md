# EnderTreasury - Code Functionality Report

This report serves the purpose of describing the code used in `EnderTreasury.sol` contract in common terms while also providing techniacal information.

<br>

## Key Features

Purpose of `EnderTreasury` is to hold the bond assets.

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
