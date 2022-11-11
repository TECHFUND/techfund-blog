+++
author = "Piyush"
title = "Re-Entry issue in Smart Contracts and how to handle it"
date = "2022-11-01"
description = "Handling Re-Entry issue of smart contracts"
tags = [
    "solidity"
]
categories = [
    "technical"
]
series = ["Solidity"]
aliases = ["solidity"]
image = "bg.jpeg"
+++

### Re-Entry issue
A reentrancy issue is a vulnerability in a smart contract that can allow an attacker to repeatedly call the contract and execute arbitrary code. This can be dangerous, as it can allow the attacker to steal funds or manipulate the contract's state in unintended ways. In this blog post, we will discuss what a reentrancy issue is, provide an example of a buggy contract with a reentrancy issue, and discuss how to fix such issues.

## Example of a Buggy Contract

To understand what a reentrancy issue is, let's consider the following example of a buggy contract:

	contract Buggy {
	uint public balance;

	function deposit() public payable {
		balance += msg.value;
	}

	function withdraw() public {
		require(balance >= msg.value);
		msg.sender.transfer(msg.value);
		balance -= msg.value;
	}
	}

This contract has a deposit function that allows users to deposit funds, and a withdraw function that allows users to withdraw funds. However, this contract has a reentrancy issue.

The issue is that the withdraw function calls msg.sender.transfer, which sends the funds to the caller's address. This function is not atomic, which means that it can be interrupted before it

## How to handle 

To fix a reentrancy issue, you need to make sure that the contract's state is updated before calling external functions that could be interrupted. This can be done using a mutex, which is a mechanism that ensures that only one thread of execution can access a shared resource at a time.

To implement a mutex in a smart contract, you can use a boolean variable to track whether the contract is currently executing. Before calling an external function, you can check this variable to make sure that the contract is not already executing. If the contract is already executing, you can prevent the function from being called.

Here is an example of a contract that uses a mutex to fix a reentrancy issue:

	contract Safe {
	uint public balance;
	bool executing;

	function deposit() public payable {
		balance += msg.value;
	}

	function withdraw() public {
		require(balance >= msg.value);
		require(!executing);
		executing = true;
		msg.sender.transfer(msg.value);
		balance -= msg.value;
		executing = false;
	}
	}

In this contract, the withdraw function checks the executing variable before calling msg.sender.transfer. If the contract is already executing, the function will not be called and the withdrawal will not be completed. This prevents the attacker from repeatedly calling the contract and executing arbitrary code.

## Conclusion 

In conclusion, a reentrancy issue is a vulnerability in a smart contract that can allow an attacker to repeatedly call the contract and execute arbitrary code. To fix a reentrancy issue, you need to make sure that the contract's state is updated before calling external functions that could be interrupted. This can be done using a mutex, which is a mechanism that ensures that only one thread of execution can access a shared resource at a time. By using a mutex, you can prevent a reentrancy issue and ensure that your smart contract is secure.