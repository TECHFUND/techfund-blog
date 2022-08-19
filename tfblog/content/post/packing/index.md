+++
author = "Piyush"
title = "Solidity variable packing & saving deployment costs"
date = "2022-08-19"
description = "Saving 1000`s of dollars in Ethereum Smart Contracts by a simple technique of variable packing"
tags = [
    "solidity"
]
categories = [
    "technical"
]
series = ["Solidity"]
aliases = ["solidity"]
image = "bg.jpg"
+++

## Introduction
Solidity variable packing is a technique used in Solidity to pack variables into bytes.
We save state vars on the storage slot by slot, each slot has a space of 32 bytes equal to 256 bits. When we declared a uint8 (unsigned integer of 8bits), and then a uint256 (unsigned integer of 256bits) on the storage since the 8 and 256 uint can not be together on a slot that is 256 bits our contract will take 2 slots and one of them it will be just for an 8 bit. In the end, we will consume 64bits of space.

But the worse part is that: when you have a entire slot and you have just a unit8 due to a lack of proper var packaging the Ethereum virtual machine has to first convert it to uint256 to work it and the conversion cost more Gas!  

## How to optimize variable packing

1. Try to pack variables as closely as possible together. 
2. Try to avoid creating large blocks of unused variables. 
3. Try to use as few variables as possible to achieve the desired effect.  
4. Large blocks of unused variables can lead to memory leaks and can make it difficult to track down errors.

For example let us have a look at these two deployments with just one variable packing difference : 

{{< figure src="./1.png" title="Structure 1" >}}

{{< figure src="./2.png" title="Structure 2" >}}


We will notice for 
`struct Vesting {
    uint8 b;    
    uint256 a;    
    uint256 c;
  }
`
the Gas used was 382563

and for 
`struct Vesting {    
    uint256 a;   
    uint8 b; 
    uint256 c;
}`

the Gas used was 382577

## Tradeoffs
 There are a few trade-offs to optimizing solidity variable packing. One trade-off is that it can make code more difficult to read. Another trade-off is that it can make code more difficult to maintain. 
 
## Conclusion
It is possible to store as many variables into one storage slot, as long as the combined storage requirement is equal to or less than the size of one storage slot, which is 32 bytes. For example, one bool variable takes up one byte. A uint8 is one byte as well, uint16 is two bytes, uint32 four bytes, and so on. The storage requirement of the bytes data type is easy to remember, since for example bytes4 takes exactly four bytes. So theoretically 32 uint8 variables can be stored in the same space as one uint256 can. This only works if the variables are declared one after another in the code, because if one bigger data type has to be stored in between, a new slot in storage is used. 