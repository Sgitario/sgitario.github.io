---
layout: post
title: Ethereum And Solidity Getting Started
date: 2018-10-22
tags: [ Ethereum, Solidity ]
---

# Introduction

Ethereum networks are a set of nodes or machines mining transactions within [blockchains](https://anders.com/blockchain/). The purpose of each transaction is tied to a smart contract or business definition. Because the machines need power to work, this mining work has a cost or gast limit which is measured in ether (or units of ether like wei). This cost depends on the number of operations our smart contract has. 

# Installation

In order to invest/browse in the Ethereum networks, we can use [Metamask](https://chrome.google.com/webstore/search/metamask). This is a browser extension and is intented to use only by final users (not for programming).

In order to deploy contracts and make actions over them in the Etheresum networks, we use the [web3.js](https://github.com/ethereum/web3.js/). 

In order to design/compile our custom smart contracts, we use [Solidity](https://solidity.readthedocs.io/en/v0.4.25/).

## Metamask

- Install Metamask [in Chrome](https://chrome.google.com/webstore/search/metamask).
- Accept terms and create an account. *Save the secret phrase or mnemonic account in a safe place*. 
- Use the correct network: for production: the "Main Ethereum" network, it costs real money!. For testing, we'll use the "Rinkeby Test Network" network.

The secret phrase is really like a mnemonic account and one mnemonic account can have one or many accounts that we can create later on. Each account can have different money and works in a different purpose.

### How to receive Ether

Go to http://rinkeby-faucet.com and fill the address input (we can find it next to My account in Metamask plugin).

**For testing, sometimes you'll need much more money. If so, go to http://faucet.rinkeby.io**

- What happened in between?

1. Click 'submit' on form
2. Address sent to Backend server
3. Backend server used web3.js library to create a 'transaction' object
4. Backend server sent 'transaction' object to the Rinkeby test network
5. Backend server waited for transaction to be confirmed. The transcation goes to one node in the network, the node groups this transaction with other transactions that are running at the same time into a 'block' and start validating (or mining!)
6. Backend server sent success message back to the browser

- What is a 'transaction'?
 
| nonce: how many times the sender has sent a transaction
| to: address of account this money is going to
| value: amount of ether to send to the target address
| gasPrice: amount of ether the sender is willing to pay per unit gas to get this transaction processed
| startGas/gasLimit: units of gas that this transaction can consume
| v |
| r | -> cryptographic pieces of data that can be used to generate the senders account address. Generated from the sender's private key.
| s |

But then, why we need Ether's for? Why this complexity about transactions? We are missing another concept: smart contracts. 

## Smart Contracts

Smart Contracts is a code that lives in each transaction. Actually, when deploying a new contract, this is done like another regular transaction. 

### Solidity Programming Language

The smart contracts are written in [Solidity Programming Language](https://solidity.readthedocs.io/en/v0.4.25/).

We can test and play with the smart contracts using [this online editor](http://remix.ethereum.org)

Let's write this contract:

```
pragma solidity^0.4.17;

contract Inbox {
    address public owner;
    string public message;

    constructor(string initialMessage) public {
        owner = msg.sender;
        message = initialMessage;
    }
    
    function setMessage(string newMessage) public {
        message = newMessage;
    }
}
```

- Storage vs Memory

The *storage* keyword makes variables point out directly to input variable. It's like using reference addresses in the oriented programming language world.

```
contract Numbers{
    int[] public numbers;

    function Numbers() public {
        numbers.push(1);
        numbers.push(2);

        int[] storage other = numbers; // numbers is the input of other field.
        other[0] = 3;
    }
}
```

Output for numbers and other is the same because they are pointing to the same address location: [3, 4].

On the other hand, the *memory* keyworld will make a copy of the input variable in a different address location:

```
contract Numbers{
    int[] public numbers;

    function Numbers() public {
        numbers.push(1);
        numbers.push(2);

        int[] memory other = numbers; // numbers is the input of other field.
        other[0] = 3;
    }
}
```

Now, the data in other is [3, 2] and in numbers is [1, 2].

The *memory* keyword is equivalent to:

```
contract Numbers{
    int[] public numbers;

    function Numbers() public {
        numbers.push(1);
        numbers.push(2);

        changeArray(numbers);
    }

    function changeArray(int[] other) private {
        other[0] = 3;
    }
}
```

However, we can change this behaviour by using the *storage* keyword as:

```
contract Numbers{
    int[] public numbers;

    function Numbers() public {
        numbers.push(1);
        numbers.push(2);

        changeArray(numbers);
    }

    function changeArray(int[] storage other) private {
        other[0] = 3;
    }
}
```

- Types:
    - string
    - bool
    - int or int256 (positive or negative non decimal numbers). Also, we can specify the range doing int8, int16, int32, ...
    - uint (positive non decimal numbers)
    - fixed (decimal numbers)
    - address (example: 0x01ab1212... for accounts/clients)
    - fixed array (int[3] --> [1, 2, 3])
    - dynamic array (int[] --> [1, 2, 3])
    - mapping ( mapping(string => string) )

This is like a key-value collection:

```
contract MyContract {
    mapping(address => bool) public members;
    
    function addMember() public payable {
        require(msg.value > 0);
        members[msg.sender] = true;
    }

    function isActive() public {
        require(members[msg.sender]);
    }
}
```

    - struct (struct Car { string make; string model; })

```
contract MyContract {
    struct MyType {
        string name;
        bool flag;
    }

    MyType public instance;
    
    function method(string name, bool flag) public payable {
        MyType memory myType = MyType({
            name: name,
            flag: flag
        });

        instance = myType;
    }
}
```

- Message Global Context

All the messages contain the account of the sender and the transaction itself. We can access to the sender account from the smart contract by doing:

    - msg.data
    - msg.gas: amount of gas the current function invocation has available
    - msg.sender: sender account address
    - msg.value: amount of ether (in wei) that was sent along with the function invocation

The msg instance is directly available in the smart contract constructor with doing nothing.

- Payable Function Type

Mark a public method as payable and use the require function inside as:

```
pragma solidity^0.4.17;

contract Inbox {
    ...

    function doSomethingThatRequiresMoney() public payable {
        require(msg.value > .01 ether, "Must have at least 0.01 ether");
        // ..
    }
}
```

- How to send money to an address

The current amount of money the contract has is in "this.balance":

```
contract Lottery {
    address public manager;

    // ...
    
    function sendMoneyToManager() public {
        manager.transfer(this.balance);
    }
}
```

- Modifiers

The modifiers are used to reduce the duplicity of the code we use to write our smart contracts:

```
function fnc1() public mymodifier {
    // ...
}

function fnc2() public mymodifier {
    // ... 
}

modified mymodifier() {
    // do the common bits and then:
    _;
}
```

- Other functions

    - now: current time
    - block.difficulty: a level to resolve the current block
    - random function:

```
function random() private view returns (uint) {
    return uint(keccak256(block.difficulty, now, an_array));
}
```

### What about the cost of a transaction?

If we rewrite our smart contract to include a new function like:

```
pragma solidity^0.4.17;

contract Inbox {
    ...
    
    function doMath(int a, int b) {
        a + b // ADD: Costs 3 gas
        a - b // SUB: Costs 3 gas
        a * b // MULTIPLY: Costs 5 gas
        a == b // EQ: Costs 3 gas
    }
}
```

We can use [this spreadsheet](https://docs.google.com/spreadsheets/d/1n6mRqkBz3iWcOlRem_mO09GtSKEKrAsfO7Frgx18pNU/edit#gid=0) to calculate the cost of gas of our contract depending on the operations/store data. If we say in a transaction we are up to spend 10 gas unit, the above transaction could not complete. 

### How Do We Deploy Smart Contracts? 

Earlier we said we'll work using a test network Rinkeby, but how we deploy our smart contract into it? Let's introduce Truffle because Remix only runs in the browser.

- Compiler: we need to setup our JS compiler to Solidity compiler
- Testing: we need to use mocha test runner and Ganache/TestRPC (local test network). Also web3 to have access to our contract in the local test network.
- Deployment: we need a deploy script. We'll use Infura as provider.

Project Structure:
    inbox ->
        contracts ->
            inbox.sol
        test ->
            inbox.test.js
        package.json
        compile.js
        deploy.js

Steps:
1. Installation

```bash
npm init
npm install --save solc fs-extra
```

2. Write compile.js:

```js
const path = require('path'); // for cross platform 
const fs = require('fs-extra');
const solc = require('solc');

const buildPath = path.resolve(__dirname, 'build');

// Remove build
fs.removeSync(buildPath);

// Create build folder if it does not exist
fs.ensureDirSync(buildPath);

const inboxPath = path.resolve(__dirname, 'contracts', 'inbox.sol');
let source = fs.readFileSync(inboxPath, 'utf8');
let output = solc.compile(source, 1).contracts;

// Copy contracts inside the .sol file into the build folder
for (let contract in output) {
    fs.outputJsonSync(
        path.resolve(buildPath, contract.replace(':', '') + '.json'), 
        output[contract]
    );
}
```

3. Run compile

```bash
node compile.js
```

4. Prepare for testing:

```bash
npm install --save mocha ganache-cli web3@1.0.0-beta.26
```

5. Write test/inbox.test.js:

```js
const assert = require('assert');
const ganache = require('ganache-cli');
const Web3 = require('web3');

const provider = ganache.provider();
const web3 = new Web3(provider);

const Inbox = require('../build/Inbox.json');

let accounts;
let inbox;

beforeEach(async () => {
    // Get a list of all accounts
    accounts = await web3.eth.getAccounts();

    // Use one of those accounts to deploy the contract
    inbox = await new web3.eth.Contract(JSON.parse(Inbox.interface))
        .deploy({ data: Inbox.bytecode, arguments: [ 'Hello world!'] })
        .send({ from: accounts[0], gas: '1000000' });

    inbox.setProvider(provider)
});

describe('Inbox', () => {
    it ('deploys a contract', () => {
        // To operate with the contract, we need the address:
        assert.ok(inbox.options.address);
    });
    it('has a default message', async () => {
        const message = await inbox.methods.message().call();
        assert.equal(message, 'Hello world!');
    });
    it('can change the message', async () => {
        await inbox.methods.setMessage('This is a new message').send({ from: accounts[0], gas: '1000000' });
        const message = await inbox.methods.message().call();
        assert.equal(message, 'This is a new message'); 
    });
});
```

Change the package json file to use mocka as test runner:

```json
"scripts": {
    "test": "mocha"
  },
```

Run the test by doing: npm run test

6. Preparing For Deployment

We want to avoid somebody modify the contract code. The best solution is to create another contract as a factory:

```
pragma solidity^0.4.17;

contract InboxFactory {
    address[] public deployedInbox;

    function createInbox(string initialMessage) public {
        address newInbox = new Inbox(initialMessage); // send msg.sender if our contract wans the original sender
        deployedInbox.push(newInbox);
    }

    function getDeployedInbox() public view returns (address[]) {
        return deployedInbox;
    }
}

contract Inbox {
    // ...
}
```

7. Write deployment script:

We'll use Infura and our account that we created in Metamask. Infura will find the right node in the target ethereum network and will ease the smart contract transaction.

- Signup Infura: https://infura.io/
- Create a new project (aka "Rinkeby API") and make sure we select Rinkeby Network
- Install truffle module (this is the provider)

```bash
npm install --save truffle-hdwallet-provider
```

- Write the deploy.js file:

```js
const HDWalletProvider = require('truffle-hdwallet-provider');
const Web3 = require('web3');
const Inbox = require('build/Inbox.json');

const provider = new HDWalletProvider(
    'inhale require dry moment bubble deposit seed embark flock opinion fragile just', // mnemonic account
    'https://rinkeby.infura.io/v3/58f26bf5af6a4a97b6d5daa81b044266'
);

const web3 = new Web3(provider);

const deploy = async () => {
    const accounts = await web3.eth.getAccounts();

    console.log('Attempting to deploy from account', accounts[0]); // we'll use the first account we have in the mnemonic account.

    const result = await new web3.eth.Contract(JSON.parse(Inbox.interface))
        .deploy({ data: '0x' + Inbox.bytecode, arguments: [ 'Hello world!'] })
        .send({ gas: '1000000', from: accounts[0] });

    console.log('Contract interface: ' + interface);
    console.log('Contract deployed to', result.options.address); // we need the address to work with this contract!
};

deploy();
```

Output:

```
Attempting to deploy from account 0xA94044008e2C5E01A473571FBd95A357f47557bC
Contract interface: [{"constant":true,"inputs":[],"name":"manager","outputs":[{"name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[],"name":"pickWinner","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"getPlayers","outputs":[{"name":"","type":"address[]"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[],"name":"enter","outputs":[],"payable":true,"stateMutability":"payable","type":"function"},{"constant":true,"inputs":[{"name":"","type":"uint256"}],"name":"players","outputs":[{"name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"inputs":[],"payable":false,"stateMutability":"nonpayable","type":"constructor"}]
Contract deployed to 0xF9b1a29BCc53517247cc35790b130f90C44431cF
```

**Copy the contract address "0xdAf3aA08075E5eed4b323B87EA976B7DA27f0beC". And also the interface. We'll be using it later in this tutorial!**

- Check your contract in: https://rinkeby.etherscan.io and use the contract address to look for. For the main network, use https://etherscan.io.
- Also, we can use remix.ethereum.org to test our deployed contract. All we need is again the address.

# Frontend Development

We'll use React for the frontend which is very easy well integrated with web3 and metamask. 

- Install react in our project and create the react project

```
npm install -g create-react-app
create-react-app inbox-react
cd inbox-react
npm start
```

We'll see our UI in localhost:3000 directly in our browser.

- Install dependencies to connect with Ethereum

```
npm install --save web3@1.0.0-beta.35
```

- Configure web3

Web3 works as a provider of Ethereum network (Rinkeby, Main or any other). 

**We'll use assume the user has Metamask installed!!!!**. So, note that Metamask will inject the web3 libraries in our page and there is nothing we can do in order to stop this. Therefore, we need to be sure that we'll use the web3 version library we installed just above and then we'll use the provider linked to the web3 Metamask version.

Let's create a new library in src/web3.js:

```js
import Web3 from 'web3';

const web3 = new Web3(window.web3.currentProvider); // use provider from Metamask extension;

export default web3;
```

And import this library in src/App.js:

```js
import React, { Component } from 'react';
import logo from './logo.svg';
import './App.css';
import web3 from './web3';

class App extends Component {
  render() {
    console.log(web3.version);
    web3.eth.getAccounts().then(console.log)

    // ...
  }
}

export default App;
```

- Use the inbox contract

We'll use the same contract that we used before (run: node deploy.js and see the interface and the address again). 

Create a new library src/inbox.js
```js
import web3 from './web3';

// contract address
const address = '0xF9b1a29BCc53517247cc35790b130f90C44431cF';
// contract interface: abi
const abi = [{"constant":true,"inputs":[],"name":"manager","outputs":[{"name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[],"name":"pickWinner","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"getPlayers","outputs":[{"name":"","type":"address[]"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[],"name":"enter","outputs":[],"payable":true,"stateMutability":"payable","type":"function"},{"constant":true,"inputs":[{"name":"","type":"uint256"}],"name":"players","outputs":[{"name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"inputs":[],"payable":false,"stateMutability":"nonpayable","type":"constructor"}];

export default new web3.eth.Contract(abi, address);
```

And import the library into the src/App.js:

```js
import React, { Component } from 'react';
import logo from './logo.svg';
import './App.css';
import web3 from './web3';
import inbox from './inbox';

class App extends Component {
  state = {
    owner: '',
    balance: ''
  };

  async componentDidMount() {
    const owner = await inbox.methods.owner().call();
    const balance = await web3.eth.getBalance(inbox.options.address); // get the money inside of the contract
    this.setState( { owner, balance });
  }

  render() {
    
    console.log(web3.version);
    // See accounts in Metamask
    web3.eth.getAccounts().then(console.log);

    return (
      <div>
        <h2>Inbox Contract</h2>
        <p>This contract is owner by {this.state.owner}.
        The balance in the contract is {web3.utils.fromWei(this.state.balance, 'ether')} ether!</p>
      </div>
    );
  }
}

export default App;
```

- Let's update the contract from UI:

```js
// ..

class App extends Component {
  state = {
    owner: '',
    balance: '',
    message: ''
  };

  // ..

  onSubmit = async (event) => {
    event.preventDefault(); // Stop the event in the html tree.

    const accounts = await web3.eth.getAccounts();
    await inbox.methods.setMessage(this.state.message).send({
      from: accounts[0],
      gas: '1000000' 
    });
  };

  render() {
    
    // ..

    return (
      <div>
        <h2>Inbox Contract</h2>
        <p>
            This contract is owner by {this.state.owner}.
            The balance in the contract is {web3.utils.fromWei(this.state.balance, 'ether')} ether!
        </p>

        <h2>Message: {this.state.message}</h2>

        <hr />

        <form onSubmit={this.onSubmit}>
          <h4>Want to change the message?</h4>
          <div>
            <label>Type the new message</label>
            <input value={this.state.message}
              onChange={event => this.setState({ message: event.target.value })}>
            </input>
          </div>

          <button>Enter</button>
        </form>
      </div>
    );
  }
}
```

In your contract has a payable function, you need to pass the value field in the send method by doing: 

```js
onSubmit = async (event) => {
    // ..
    await inbox.methods.setMessage(this.state.message).send({
      from: accounts[0],
      value: web3.utils.toWei(this.state.value, 'ether')
    });
  };
```

# Source Code

See [my Github repository](https://github.com/Sgitario/blockchain-full-guide/lottery-react) for a full example.

