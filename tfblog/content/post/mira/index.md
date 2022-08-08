+++
author = "Piyush"
title = "Fixed sized blockchain"
date = "2022-08-01"
description = "Fixed sized blockchain and recursive zk-SNARKS"
tags = [
    "zk-snark"
]
categories = [
    "technical"
]
series = ["ZK-SNARKS"]
aliases = ["zksnarks"]
image = "bg.jpg"
+++

# Fixed sized blockchain
### - Piyush
Recently I came across “Mira Protocol” and found it technically interesting. It promises “Fixed sized blockchain”. How it defines a blockchain is as follows :
    - An unambiguous usable representation (ie. not merely hashes) of the parts of the state a typical user would care about namely the current balance of their account.
    - The data that a node needs in order to verify that this state is real in a trustless manner.
    - The ability to broadcast transactions on the network to make a transfer.
    From : https://minaprotocol.com/blog/22kb-sized-blockchain-a-technical-reference
Based on above definition of blockchain, Mina claims to have a “fixed sized blockchain” that will never exceed 22KB in size.
The main magic is because of two pieces ( in order of magic potion strength :) )
    `Recursive zk-SNARKs +  Proof-of-Stake`
In this article we will try to udnerstand “Recursive zk-Snarks” and how it is helping Mina.

# A quick primer for zk-snark 
### “Zero-Knowledge Succinct Non-Interactive Argument of Knowledge”.

##### zk = Zero Knowledge
There is a paper by Alibaba explaining Zero knowledge proofs https://pages.cs.wisc.edu/~mkowalcz/628.pdf , but basically a prover has secret X and goal is to proove it to verifier that it knows the secret without relieving the secret
##### s = Succinct
The size of proof is very small and shuold get verified in milliseconds.
##### n = Non-Interactive
Zero knowledge proofs are generally interactive, a verifier must ask “questions”. In zk-Snarks , they must be "non-interactive" i.e. there should be minimal interaction between the prover and the verifier.
##### ark = Argument of knowledge
It means that a bad prover has low chances of successfully cheating the system without actually having the knowledge. It should be "computationally sound".

So in short for a zero knowledge protocol to be SNARK , it should be quickly verifiable / non-interactive with low chances of cheating.

The workflow can be described in following 4 steps : 

- The computational problem to be proved is transformed into arithmetic circuits.
- The arithmetic circuits is converted to rank-1 constraint system
- R1CS ( rank-1 constraint system ) is converted to QAP ( Not solvable in polynomial time, can be verified in polynomial time )
- The implementation of zkSNARK algorithm based on QAP

We will quickly try to cover the steps using nodejs, for the sake of article let us assume we need to build a system that proves to witness that we’re able to factor an integer "x". We will be using two nodejs libraries `circom` and `snarkjs`.

Let us first create our circuit file
> vim SecretMultiplier.circom

```js
   template SecretMultiplier() {
       signal private input a;
       signal private input b;
       signal output c;
       x <== a*b;
   }
   component main = SecretMultiplier();
```

which can be compiled using `circom` 

> circom SecretMultiplier.circom --r1cs --wasm --sym

This will provide us with the R1CS  ( rank-1 constraint system ) output for the circuit. We can now use this file to "setup" Snark.

> snarkjs setup -r SecretMultiplier.r1cs

SnarkJS will generate "proving and verification" keys ( proving_key.json & verification_key.json ).

Now imagine that we want to prove that we are able to factor 33. We need to prove that we know two numbers a and b that multiply to give 33.

Since the only two (non-trivial) numbers that multiply to give 33 are 3 and 11, let’s create a file named input.json, with the following content:

> {"a": 3, "b": 11}

Now, run the following command to calculate the witness ( `witness.json` ) :

> snarkjs calculatewitness --wasm SecretMultiplier.wasm --input input.json

Now that we’ve generated the witness, we’re ready to create the proof.

To create the proof, run:

> snarkjs proof --witness witness.json --provingkey proving_key.json

This will generate proof.json ( which contains the actual proof ) & public.json ( which contains the values of the public inputs and outputs – example 33 ).

Finally to verify a proof we can run 

> snarkjs verify --verificationkey verification_key.json --proof proof.json --public public.json

Hence we can see it is possible to proove, to a verifier, that proover knows the secret without relieving the secret.

# What is a recursive ZK-SNARK?

A recursive SNARK system generates proofs for different transactions and aggregates them into one single proof, which means one SNARK can verify other SNARKs. 

*Microsoft also released an implementation in Rust for Recursive zk-Snarks https://github.com/microsoft/Nova . 

{{< figure src="./recursive.png" title="Recursive" >}}

Mira uses these recursive snarks to keep the blockchain`s size fixed. This means that the blockchain is represented with "verifiable" cryptographic proof (zk-SNARK). This proof is much smaller than most other blockchains and represents the state of the whole chain, rather than the latest block. It enables any node / user to verify any transaction without depending on any third party or running full node. It is possible to verify any transaction using a simple mobile app with a fixed data of under 22KB.
