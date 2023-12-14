# EnderBond - Code Functionality Report

This report serves the purpose of describing the code used in `EnderBond.sol` contract in common terms while also providing some techniacal information.

EnderBond contract has several dependencies

<!-- # Table of contents -->

<!-- <summary>Introduction, Resources, and Prerequisites</summary>
<ol>
<li><a href="#link-to-video-coming-soon">Link to video: *Coming soon...*</a></li>
<li><a href="#resources-for-this-course">Resources For This Course</a></li>
<li><a href="#prerequisites">Prerequisites</a></li>
<li><a href="#outcome">Outcome</a></li>
<li><a href="#functions">Functions</a>
  <ul>
    <li><a href="#initialize-function">initialize</a></li>
    <li><a href="#deposit-function">deposit</a></li>
  </ul>
</li>
</ol>

<summary><a href="#curriculum">Curriculum</a></summary> -->

## State Variables

##### Mappings

-   `bondableTokens` (address => bool) - keeps track of bondable tokens
-   `bonds` (uint256 => Bond) - basically gives a token ID to individual bond.
<details>

```javascript
struct Bond {
        bool withdrawn; // The withdrawn status of the bond
        uint256 principal; // The principal amount of the bond
        // uint256 endAmt; // The END token amount of deposit
        uint256 startTime; // Timestamp of the bond
        uint256 maturity; // The maturity date of the bond
        address token; // The token used for the bond
        uint256 bondFee; // bond fee self-set
        uint256 depositPrincipal;
        uint256 rewardPrincipal;
        uint256 refractionSIndex;
        uint256 stakingSendIndex;
        uint256 YieldIndex;
    }
```

</details>

## Methods

### initialize()

```javascript
 function initialize(address endToken\_, address \_lido, address \_oracle) public initializer
```

Function `initialize` is the first function to be ran after contract is defloyed. Has the `initializer` check on it that ensures that it can only be ran once.

<details>
Takes 3 parameters -

-   endToken\_ - Sets address of END token
-   \_lido - Takes lido address
-   \_oracle - Takes oracle address

Updates following once ran.

1. `__Ownable_init();` - Ownable contract initializer
2. `__EIP712_init("EnderBond", "1");` - Initializes
3. `rateOfChange = 100` - NEED EXPLANING
4. `lido = _lido;` - set lido address from parameters
5. `minDepositAmount = 1000000000000000;` - 0.001 eth
6. `enderOracle = IEnderOracle(_oracle);` - set oracle address from parameters
7. `bondYieldBaseRate = 100;` set bond yield base rate to 100
8. `SECONDS_IN_DAY = 600;` - 600 is 10 mins (1 per sec)
9. `interval = 10 * 60;` - 10 mins
10. `lastTimeStamp = block.timestamp;` - current timestamp
11. `lastDay = block.timestamp / SECONDS_IN_DAY;`
12. `setBondFeeEnabled(true);` - comments say this function is not used

<details>

```javascript
  function initialize(address endToken_, address _lido, address _oracle) public initializer {
        __Ownable_init();
        __EIP712_init("EnderBond", "1");
        rateOfChange = 100;
        lido = _lido;
        setAddress(endToken_, 2);
        // todo set the value according to doc
        minDepositAmount = 1000000000000000;
        txFees = 200;
        enderOracle = IEnderOracle(_oracle);
        bondYieldBaseRate = 100;
        SECONDS_IN_DAY = 600; // note for testing purpose we have set it to 10 mint
        interval = 10 * 60; // note for testing purpose we have set it to 10 mint
        lastTimeStamp = block.timestamp;
        lastDay = block.timestamp / SECONDS_IN_DAY;

        //this function is not used
        setBondFeeEnabled(true);
    }

```

</details>
</details>

### setInterval()

```javascript
function setInterval(uint256 \_interval) public onlyOwner
```

Takes \_interval as parameter and updates `interval` (uint) global >variable and emits the log.

### setBool()

```javascript
function `setBool`(bool _bool) public onlyOwner
```

Sets state of `isSet` global variable.
Is used in `claimStakingReward`, `claimRefractionRewards`, `calculateBondRewardAmount` as if (isSet) condition.

### setAddress()

```javascript
function setAddress(address _addr, uint256 _type) public onlyOwner
```

Takes `_addr` and `_type` as parameter and updates address of the token the type presents. Types are shown in full function below.

<details>

```javascript
  function setAddress(address _addr, uint256 _type) public onlyOwner {
        if (_addr == address(0)) revert ZeroAddress();

        if (_type == 1) endTreasury = IEnderTreasury(_addr);
        else if (_type == 2) endToken = _addr;
        else if (_type == 3) bondNFT = IBondNFT(_addr);
        else if (_type == 4) endSignature = _addr;
        else if (_type == 5) lido = _addr;
        else if (_type == 6) stEth = _addr;
        else if (_type == 7) keeper = _addr;
        else if (_type == 8) endStaking = IEnderStaking(_addr);

        emit AddressSet(_type, _addr);
    }

```

</details>

### setMinDepAmount

```javascript
function setMinDepAmount(uint256 _amt) public onlyOwner
```

Updates the minimum amount amount of any asset on `minDepositAmount`

### setTxFees()

```javascript
function setTxFees(uint256 _txFees) public onlyOwner
```

Updates `txFee` global variable.

### setBondYieldBaseRate()

```javascript
function setBondYieldBaseRate(uint256 _bondYieldBaseRate) public onlyOwner
```

Owner updates the `bondYieldBaseRate` global variable.

### getAddress()

```javascript
function getAddress(uint256 _type) external view returns (address addr)
```

Function returns `address` of the called `_type`

<details>

```javascript
   function getAddress(uint256 _type) external view returns (address addr) {
        if (_type == 1) addr = address(endTreasury);
        else if (_type == 2) addr = endToken;
        else if (_type == 3) addr = address(bondNFT);
        else if (_type == 4) addr = endSignature;
        else if (_type == 5) addr = lido;
    }
```

</details>

### setBondFeeEndbled

```javascript
function `setBondFeeEnabled`(bool _enabled) public onlyOwner
```

Owner swtitches whether bondFee is enabled or not. Updates `bondFeeEnabled`

### setBondableTokens()

```javascript
 function setBondableTokens(address[] calldata tokens, bool enabled) external onlyOwner
```

Owner can use multiple tokens in form of `address[]` calldata tokens and can set whetcher they are bondable or not. Updates the mapping `bondableTokens`

### getInterest()

```javascript
function getInterest(uint256 maturity) public view returns (uint256 rate)
```

Function takes `maturity` as parameter and returns `rate` that is being calculated by

rate = bondYieldBaseRate \* maturityModifier

Maturity represents `days` of the bond. Does not update state of any variable but just returns a `rate` value. The formula used is in function code below.

<details>

```javascript
     function getInterest(uint256 maturity) public view returns (uint256 rate) {
        uint256 maturityModifier;
        //Todo make it dynamic in phase 2
        unchecked {
            if (maturity >= 360) maturityModifier = 150;
            if (maturity >= 320 && maturity < 360) maturityModifier = 140;
            if (maturity >= 280 && maturity < 320) maturityModifier = 130;
            if (maturity >= 260 && maturity < 280) maturityModifier = 125;
            if (maturity >= 220 && maturity < 260) maturityModifier = 120;
            if (maturity >= 180 && maturity < 220) maturityModifier = 115;
            if (maturity >= 150 && maturity < 180) maturityModifier = 110;
            if (maturity >= 120 && maturity < 150) maturityModifier = 105;
            if (maturity >= 90 && maturity < 120) maturityModifier = 100;
            if (maturity >= 60 && maturity < 90) maturityModifier = 90;
            if (maturity >= 30 && maturity < 60) maturityModifier = 85;
            if (maturity >= 15 && maturity < 30) maturityModifier = 80;
            if (maturity >= 7 && maturity < 15) maturityModifier = 70;
            rate = bondYieldBaseRate * maturityModifier;
        }
    }
```

</details>

## deposit()

```javascript
function deposit(uint256 principal, uint256 maturity, address token, uint256 bondFee) private returns (uint256 tokenId)
```

One of the most important functions for the whole protocol. Allows `user` to create `bonds` by depositing assets (stETH, ETH) for certain amount of time. This unit of time is presented as `maturity` which defines the days of bond maturity time. It also gives you option to add a fee on top of it presented as `bondFee`.

Simple techinal breakdown below:

<details>

Takes following parameters:

-   `uint256 principal` - the principal amount of bond the user wants to deposit.
-   `uint256 maturity` - the number of days of the bond the user wants to deposit.
-   ` address token` - the address of the bond the user wants to deposit.
-   `uint256 bondFee` - the amount of bond the user wants to deposit.

and returns `tokenId`

If the following conditions are true, the transaction will be rejected (revert)

-   `principal` is lower than `minDepositAmount`
-   `maturity` is lower than 7 or is greater than 365
-   `token` is not equal to `address(0)` and the token is not bondable inside `bondableTokens` mapping.
-   `bondFee` is smaller than or equal to 0 or greater than 100.

This checks are performed first and after ensuring these the following function is ran.

```
IEndToken(endToken).distributeRefractionFees();
```

This runs the function `distributeRefractionFees` in endToken address.

<details>

```javascript
function distributeRefractionFees() external onlyRole(ENDERBOND_ROLE) {
        // if (lastEpoch + 1 days > block.timestamp) revert InvalidEarlyEpoch();
        uint256 feesToTransfer = refractionFeeTotal;
        if (feesToTransfer != 0) {
            refractionFeeTotal = 0;
            lastEpoch = block.timestamp;
            if (feesToTransfer != 0) {
                _approve(address(this), enderBond, feesToTransfer);
                IEnderBond(enderBond).epochRewardShareIndex(feesToTransfer);
                // _transfer(address(this), enderBond, feesToTransfer);
                emit RefractionFeesDistributed(enderBond, feesToTransfer);
            }
        }
    }
```

</details>

Then proceeds the actual token transfer, it is checked if the token is native or an erc20.

-   If the `token` == `address(0)` then what follows is first it makes sure if `msg.value` which is amount of native tokens (eth) is equal to the `principal` amount. Then after confirming a transfer call is made to `lido` address

```javascript
(bool suc, ) = payable(lido).call{value: msg.value}(abi.encodeWithSignature("submit()"));
```

This submit `principal` amount into Lido contract using their `submit()` method.

```javascript
function submit(address _referral) payable returns (uint256)
```

Which presumebly mints and sends stETH to `EnderBond` address from Lido address.

The submit call is made to Lido contract which takes \_referral as a parameter. Success of this .call to Lido address is awaited before proceeding. Then .transfer function is called to `stEth` address

```javascript
IERC20(stEth).transfer(address(endTreasury), IERC20(stEth).balanceOf(address(this)));
```

which transfers all the stETH balance of `EnderBond` address to `endTreasury` address and ends the flow of native token to the treasury.

-   Else other condition is if `token` is an ERC20 the `token` of `principal` amount is sent directly to `endTreasury` using

```javascript
IERC20(token).transferFrom(msg.sender, address(endTreasury), principal);
```

Thus ends the flow of deposits itself has ended yet the return of the `tokenId` from the function itself is defined by

```javascript
tokenId = _deposit(principal, maturity, token, bondFee);
```

Call to `_deposit` is made with `principal`, `maturity`, `token`, and `bondFee` as arguments which were all provided by our `depost` function itself.
Lastly

```javascript
epochBondYieldShareIndex();
// IEnderStaking(endStaking).epochStakingReward(stEth);
emit Deposit(msg.sender, tokenId, principal, maturity, token);

```

`epochBondYieldShareIndex` function is called `Deposit` event is called and the method comes to an end.

Full code:

<details>

```javascript
    function deposit(
        uint256 principal,
        uint256 maturity,
        uint256 bondFee,
        address token
    ) external payable nonReentrant returns (uint256 tokenId) {
        console.log("Deposited Amount:- ", principal);
        console.log("Maturity:- ", maturity);
        console.log("Bond Fees:- ", bondFee);
        if (principal < minDepositAmount) revert InvalidAmount();
        if (maturity < 7 || maturity > 365) revert InvalidMaturity();
        if (token != address(0) && !bondableTokens[token]) revert NotBondableToken();
        if (bondFee <= 0 || bondFee > 100) revert InvalidBondFee();
        IEndToken(endToken).distributeRefractionFees();

        // token transfer
        if (token == address(0)) {
            if (msg.value != principal) revert InvalidAmount();
            (bool suc, ) = payable(lido).call{value: msg.value}(abi.encodeWithSignature("submit()"));
            require(suc, "lido eth deposit failed");
            IERC20(stEth).transfer(address(endTreasury), IERC20(stEth).balanceOf(address(this)));
        } else {
            // send directly to the ender treasury
            IERC20(token).transferFrom(msg.sender, address(endTreasury), principal);
        }
        tokenId = _deposit(principal, maturity, token, bondFee);
        epochBondYieldShareIndex();
        // IEnderStaking(endStaking).epochStakingReward(stEth);
        emit Deposit(msg.sender, tokenId, principal, maturity, token);
    }

```

</details>

This is how intended flow of the function is according to the code.

-   Accept the basic bond details
-   Check for requirements
-   Check if `token` is whether ETH or ERC20
-   If ETH (address`(0)`)

    -   then ETH is sent to Lido
    -   inturn which mints stETH to `EnderBond` contract
    -   and transfer that stETH to `endTreasury` from `EnderBond` contract.

-   Else

    -   transfer that `token` ERC20 from `user` to `endTreasury`.

-   And finally define tokenId by calling \_deposit function.
-   epochBondYieldShareIndex() is ran and we log the `Deposit` details.

</details>

## \_deposit()

```javascript
function _deposit(uint256 principal, uint256 maturity, address token, uint256 bondFee) private returns (uint256 tokenId)
```

This method is the second component to the bond deposit. It is ran directly from `deposit()` after the token deposit is done. This function returns a (uint256 `tokenId`) and also is responsible for minting `BondNFT` to the `user`.

<details>

It starts off by making an external call to `endTreasury` on `depositTreasury`.

<br>

> EnderTreasury.sol - depositTreasury() :

<details>

```javascript
   function depositTreasury(EndRequest memory param, uint256 amountRequired) external onlyBond {
        unchecked {
            if (amountRequired > 0) {
                withdrawFromStrategy(param.stakingToken, priorityStrategy, amountRequired);
            }
            epochDeposit += param.tokenAmt;
            fundsInfo[param.stakingToken] += param.tokenAmt;
        }
    }
```

The `EndRequest memory param` gets passed in call here, which is basically a data structure of following:

```javascript
    // IEnderBase.sol code
    struct EndRequest {
        address account;
        address stakingToken; // non-zero address: stETH, zero address: ETH
        uint256 tokenAmt;
    }
```

</details>

the `depositTreasury` call is made with following parameters: `msg.sender` (EnderBond) for account, `token` as adress of the token, `principal` as as the total amount of tokens sent. These are all packed into `param` and assed on while the `amountRequired` is calculated by `getLoopCount()`.

```javascript
(EndRequest memory param, uint256 amountRequired)
```

After calling treasury external call we update the `principal` based on the fee set by the `user`.

```javascript
principal = (principal * (100 - bondFee)) / 100;
```

Next step is actually minting the BondNFT. This is done by the following code. `msg.sender` in this case will be the `user` who executed this tx all the way from `deposit`.

```javascript
tokenId = bondNFT.mint(msg.sender);
```

`.mint` will return a `uint256` which we will directly assign to `tokenId`. Which will basically be assigned as main `key` to `bonds` mapping which is again, built using assigning a `uint256` to `Bond` object.

Later some mappings and global storages are updated. `availableFundsAtMaturity` (mapping) is calculated and so is `rewardPrinciple` using `calculateRefractionData` which returns us the value for it.

```javascript
  availableFundsAtMaturity[(block.timestamp + ((maturity - 4) * SECONDS_IN_DAY)) / SECONDS_IN_DAY] += principal;
        (, uint256 rewardPrinciple) = calculateRefractionData(principal, maturity, tokenId);

        rewardSharePerUserIndex[tokenId] = rewardShareIndex;
        rewardSharePerUserIndexSend[tokenId] = rewardShareIndexSend;
        userBondYieldShareIndex[tokenId] = bondYieldShareIndex;
```

With this `rewardSharePerUserIndex`, `rewardSharePerUserIndexSend`, `userBondYieldShareIndex` mappings all assigned key of `tokenId`, are updated with current values of `rewardShareIndex`, `rewardShareIndexSend`, `bondYieldShareIndex` respectively, which are the state varibles.

Then `depositPrincipal` is calculated with the following, which makes use of `getInterest` method

```javascript
uint256 depositPrincipal = (getInterest(maturity) * ((100 + (bondFee))) * rewardPrinciple) / (365 * 100); // note we have to change 1e8 to 1e18
        depositPrincipalAtMaturity[
            (block.timestamp + ((maturity) * SECONDS_IN_DAY)) / SECONDS_IN_DAY
        ] += depositPrincipal;
```

and the following global storages are updated.

```javascript
totalDeposit += principal;
totalRewardPrincipal += depositPrincipal;
userBondPrincipalAmount[tokenId] = depositPrincipal;
totalBondPrincipalAmount += depositPrincipal;
```

Whose pepose is self explanatory by their name.

Finally `tokenId` that was generated by minting from `bondNft` contract to the user, is pushed into `bonds` mapping, which again holds info of all bonds directly connected to their tokenId.

```javascript
bonds[tokenId] = Bond(
    false,
    principal,
    block.timestamp,
    maturity,
    token,
    bondFee,
    depositPrincipal,
    rewardPrinciple,
    0, // refractionSIndex
    0, // stakingSendIndex
    0, // YieldIndex
);
```

Full code:

<details>

```javascript
function _deposit(
        uint256 principal,
        uint256 maturity,
        address token,
        uint256 bondFee
    ) private returns (uint256 tokenId) {
        endTreasury.depositTreasury(IEnderBase.EndRequest(msg.sender, token, principal), getLoopCount());
        principal = (principal * (100 - bondFee)) / 100;
        // uint256 timeNow = block.timestamp / SECONDS_IN_DAY;
        // dayToBondYieldShareUpdation[timeNow].push(block.timestamp + (maturity * SECONDS_IN_DAY));

        // mint bond nft
        tokenId = bondNFT.mint(msg.sender);
        availableFundsAtMaturity[(block.timestamp + ((maturity - 4) * SECONDS_IN_DAY)) / SECONDS_IN_DAY] += principal;

        (, uint256 rewardPrinciple) = calculateRefractionData(principal, maturity, tokenId);

        rewardSharePerUserIndex[tokenId] = rewardShareIndex;
        rewardSharePerUserIndexSend[tokenId] = rewardShareIndexSend;
        userBondYieldShareIndex[tokenId] = bondYieldShareIndex;

        uint256 depositPrincipal = (getInterest(maturity) * ((100 + (bondFee))) * rewardPrinciple) / (365 * 100); // note we have to change 1e8 to 1e18
        depositPrincipalAtMaturity[
            (block.timestamp + ((maturity) * SECONDS_IN_DAY)) / SECONDS_IN_DAY
        ] += depositPrincipal;

        totalDeposit += principal;
        totalRewardPrincipal += depositPrincipal;
        userBondPrincipalAmount[tokenId] = depositPrincipal;
        totalBondPrincipalAmount += depositPrincipal;

        // save bond info
        bonds[tokenId] = Bond(
            false,
            principal,
            block.timestamp,
            maturity,
            token,
            bondFee,
            depositPrincipal,
            rewardPrinciple,
            0,
            0,
            0
        );
    }

```

</details>

<br>
After pushing this info the function comes to an end.

This is how intended flow of the function `_deposit` is according to the code.

-   Gets triggered from inside `deposit()` and gets passed the bond details.
-   Bond info is sent into `endTreasury` using `depositTreasury`.
-   `BondNFT` is minted to `user`.
-   Updates `availableFundsAtMaturity` mapping and gets `rewardPrinciple` from `calculateRefractionData()`
-   `rewardSharePerUserIndex` , `rewardSharePerUserIndexSend` and `userBondYieldShareIndex` mappings are updated using `tokenId` as key.
-   Calculates `rewardPrinciple`.
-   Calculate `depositPrincipalAtMaturity` mapping.
-   Update the following `totalDeposit`, `totalRewardPrincipal`, `userBondPrincipalAmount`, `totalBondPrincipalAmount`.
-   Push bond details and functions ends.

</details>

## withdraw()

```javascript
function withdraw(uint256 tokenId) external nonReentrant
```

Takes `tokenId` as parameter, it doesn't hold logic for actual bond withdrawls but instead uses `_withdraw()` method to do it. Logs `Withdrawal` even after \_withdraw is finished.

## \_withdraw()

```javascript
function _withdraw(uint256 _tokenId) private
```

This method is responsible for actually processing bond withdrawals and is executed inside function `withdraw()`

<details>

Info of the bond is used from `storage` using `tokenId` key and is presented as `bond` variable.

Following checks are made -

-   if bond is not already withdrawed.
-   if caller of the tx is indeed `ownerOf` that `_tokenId` from `bondNFT` contract.
-   if bond has matured yet.

After these checks, `distributeRefractionFees()` will be called to `EndToken`.

```javascript
 // update current bond
        bond.withdrawn = true;
        endTreasury.withdraw(IEnderBase.EndRequest(msg.sender, bond.token, bond.principal), getLoopCount());
        uint256 reward = calculateBondRewardAmount(_tokenId, bond.YieldIndex);
        dayBondYieldShareIndex[bonds[_tokenId].maturity] = userBondYieldShareIndex[_tokenId];
```

Here, in code above `bond` is set as `withdrawn` status, and then `withdraw()` is call on to `endTresury`.END Rewards as `reward` is calculated using `calculateBondRewardAmount` and gets `userBondYieldShareIndex` of `_tokenId`, this statement is set by `dayBondYieldShareIndex[bonds[_tokenId].maturity]`.

Then in `EndTreasury` contract, `mintEndToUser` is called by `EnderBond` and mints END token to `msg.sender` (user in this case) for total `reward` amount of END tokens.

It is checked if `rewardShareIndex` is not equal to `rewardSharePerUserIndex` of `_tokenId`,

-   then `claimRefractionRewards()` is called using \_tokenId and its `bond`.refractionSIndex

Check again if `rewardSharePerUserIndexSend` of `_token` is not equal to `rewardShareIndexSend`.

-   `claimStakingReward` is called using \_tokenId and its `bond`.stakingSendIndex

`userBondPrincipalAmount` on \_tokenId is subtracted from `totalBondPrincipalAmount`.

<br>
Finally these global mappings and state is updated.

<details>

```javascript
userBondPrincipalAmount[_tokenId] == 0; // Bond principal amount of _tokenId
delete userBondYieldShareIndex[_tokenId]; // sets to 0

totalRewardPrincipal -= bond.depositPrincipal; //global variables are subtracted
depositAmountRequired -= bond.depositPrincipal; //global variables are subtracted
totalDeposit -= bond.principal; //global variables are subtracted
amountRequired -= bond.principal; //global variables are subtracted
```

</details>

Full code:

<details>

```javascript
    function _withdraw(uint256 _tokenId) private {
        Bond storage bond = bonds[_tokenId];
        if (bond.withdrawn) revert BondAlreadyWithdrawn();
        if (bondNFT.ownerOf(_tokenId) != msg.sender) revert NotBondUser();
        if (block.timestamp <= bond.startTime + (bond.maturity * SECONDS_IN_DAY)) revert BondNotMatured();
        // require(block.timestamp >= bond.startTime + (bond.maturity * SECONDS_IN_DAY), "Bond is not matured");
        IEndToken(endToken).distributeRefractionFees();
        // update current bond
        bond.withdrawn = true;
        endTreasury.withdraw(IEnderBase.EndRequest(msg.sender, bond.token, bond.principal), getLoopCount());
        uint256 reward = calculateBondRewardAmount(_tokenId, bond.YieldIndex);
        dayBondYieldShareIndex[bonds[_tokenId].maturity] = userBondYieldShareIndex[_tokenId];

        endTreasury.mintEndToUser(msg.sender, reward);
        if (rewardShareIndex != rewardSharePerUserIndex[_tokenId])
            claimRefractionRewards(_tokenId, bond.refractionSIndex);
        if (rewardSharePerUserIndexSend[_tokenId] != rewardShareIndexSend)
            claimStakingReward(_tokenId, bond.stakingSendIndex);
        totalBondPrincipalAmount -= userBondPrincipalAmount[_tokenId];

        userBondPrincipalAmount[_tokenId] == 0;
        delete userBondYieldShareIndex[_tokenId];

        totalRewardPrincipal -= bond.depositPrincipal;
        depositAmountRequired -= bond.depositPrincipal;
        totalDeposit -= bond.principal;
        amountRequired -= bond.principal;
    }
```

</details>
</details>

## getLoopCount()

```javascript
 function getLoopCount() public returns (uint256)
```

This function is ran in both calls made to `endTreasury`, during both `_deposit` and `_withdraw`. It returns a `uint256` called `amountRequired`.

This first calculates `currentDay` by making `block.timestamp` / `SECONDS_IN_DAY` (86400)

Full code:

<details>

```javascript
function getLoopCount() public returns (uint256) {
        // if (msg.sender != address(endTreasury)) revert NotTreasury();
        uint256 currentDay = block.timestamp / SECONDS_IN_DAY;
        if (currentDay == lastDay) return amountRequired;
        for (uint256 i = lastDay + 1; i <= currentDay; i++) {
            if (availableFundsAtMaturity[i] != 0) amountRequired += availableFundsAtMaturity[i];
            if (depositPrincipalAtMaturity[i] != 0) {
                depositAmountRequired += depositPrincipalAtMaturity[i];
            }
        }
        lastDay = currentDay;
        return amountRequired;
    }
```

</details>
