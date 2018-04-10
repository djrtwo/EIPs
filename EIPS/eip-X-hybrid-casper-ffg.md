---
eip: X
title: Hybrid Casper FFG
status: Draft
type: Standards Track
category: Core
author: Danny Ryan <danny@ethereum.org>, Chih-Cheng Liang <cc@ethereum.org>
created: 2018-04-05
---

## Simple Summary

Specification of the first step to transition Ethereum main net from Proof of Work (PoW) to Proof of Stake (PoS). The resulting consensus model is a PoW/PoS hybrid.

## Abstract

This EIP specifies a hybrid PoW/PoS consensus model for Ethereum main net. Existing PoW mechanics are used for new block creation, and a novel PoS mechanism called Casper the Friendly Finality Gadget (FFG) is layered on top using a smart contract.

Through the use of Ether deposits, slashing conditions, and a modified fork choice, FFG allows the underlying PoW blockchain to be finalized.  As network security is partially shifted from PoW to PoS, PoW block rewards can be reduced. 

This EIP does not contain the safety and liveness proofs or the validator implementation details, but these can be found in the [Casper FFG paper](https://arxiv.org/abs/1710.09437) and [Validator Implementation Guide](https://github.com/ethereum/casper/blob/master/VALIDATOR_GUIDE.md) respectively.

## Glossary

* **epoch**: The span of blocks between checkpoints.
* **finality**: The point at which a block has been decided upon by a client to _never_ revert. Proof of Work does not have the concept of finality, only of further deep block confirmations.
* **checkpoint**: The start block of an epoch. Rather than dealing with every block, Casper FFG only considers checkpoints for finalization.
* **dynasty**: The number of finalized checkpoints in the chain from root to the parent of a block. The dynasty is used to define when a validator starts and ends validating.
* **slash**: The burning of a validator's deposit. Slashing occurs when a validator signs two conflicting messages that violate a slashing condition. For an indepth discussion of slashing conditions, see the [Casper FFG Paper](https://arxiv.org/abs/1710.09437).

## Motivation

Transitioning the Ethereum network from PoW to PoS has been on the roadmap and in the [Yellow Paper](https://github.com/ethereum/yellowpaper) since the launch of the protocol. Although effective in coming to a decentralized consensus, PoW consumes an incredible amount of energy, has no economic finality, and has no effective strategy in resisting cartels. Excessive energy consumption, issues with equal access to mining hardware, mining pool centralization, and an emerging market of ASICs each provide a distinct motivation to make the transition as soon as possible.

Until recently, the proper way to make this transition was still an open area of research. In October of 2017 [Casper the Friendly Finality Gadget](https://arxiv.org/abs/1710.09437) was published, solving open questions of economic finality through validator deposits and crypto-economic incentives. For a detailed discussion and proofs of "accountable safety" and "plausible liveness", see the [Casper FFG](https://arxiv.org/abs/1710.09437) paper.

The Casper FFG contract can be layered on top of any block proposal mechanism, providing finality to the underlying chain. This EIP proposes layering FFG on top of the existing PoW block proposal mechanism as a conservative step-wise approach in the transition to full PoS. The new FFG staking mechanism requires minimal changes to the protocol, allowing the Ethereum network to fully test and vet Casper FFG on top of PoW before moving to a validator based block proposal mechanism.

## Parameters

* `HYBRID_CASPER_FORK_BLKNUM`: TBD
* `CASPER_ADDR`: TBD
* `CASPER_CODE`: see below
* `CASPER_BALANCE`: 5e24 wei (5,000,000 ETH)
* `SIGHASHER_ADDR`: TBD
* `SIGHASHER_CODE`: see below
* `PURITY_CHECKER_ADDR`: TBD
* `PURITY_CHECKER_CODE`: see below
* `NULL_SENDER`: `2**160 - 1`
* `NEW_BLOCK_REWARD`: 6e17 wei (0.6 ETH)
* `NON_REVERT_MIN_DEPOSIT`: amount in wei configurable by client

### Casper Contract Parameters

* `EPOCH_LENGTH`: 50 blocks
* `WITHDRAWAL_DELAY`: 15,000 epochs
* `DYNASTY_LOGOUT_DELAY`: 700 dynasties
* `BASE_INTEREST_FACTOR`: TBD
* `BASE_PENALTY_FACTOR`: TBD
* `MIN_DEPOSIT_SIZE`: 1e21 wei (1,000 ETH)


## Specification

#### Deploying Casper Contract

If `block.number == HYBRID_CASPER_FORK_BLKNUM`, then when processing the block, before processing any transactions:

* set the code of `SIGHASHER_ADDR` to `SIGHASHER_CODE`
* set the code of `PURITY_CHECKER_ADDR` to `PURITY_CHECKER_CODE`
* set the code of `CASPER_ADDR` to `CASPER_CODE`
* set balance of `CASPER_ADDR` to `CASPER_BALANCE` for issuance

#### Initialize Epochs

If `block.number >= HYBRID_CASPER_FORK_BLKNUM` and `block.number % EPOCH_LENGTH == 0`, then execute a call with the following parameters at the start of the block:

* `SENDER`: NULL_SENDER
* `GAS`: 3141592
* `TO`: CASPER_ADDR
* `VALUE`: 0
* `NONCE`: 0
* `GASPRICE`: 0
* `DATA`: <encoded call {method: 'initialize_epoch', args: [floor(block.number / EPOCH_LENGTH)]}`>

This transaction utilizes no gas and does not increment `NULL_SENDER`s nonce

#### Casper Votes

If `block.number >= HYBRID_CASPER_FORK_BLKNUM`, then:

* all `vote` transactions to `CASPER_ADDR`:
  * must have the following signature `(CHAIN_ID, 0, 0)` (ie. `r = s = 0, v = CHAIN_ID`)
  * must have sender as `NULL_SENDER`
  * must have `value = nonce = gasprice = 0`
  * must be included at the end of the block
  * utilize no gas
  * do not increment `NULL_SENDER`s nonce
* all unsuccessful `vote` transactions to `CASPER_ADDR` are considered invalid and are not to be included in the block

#### Fork Choice

If `block.number >= HYBRID_CASPER_FORK_BLKNUM`, the fork choice rule is the following:
1. Start with last finalized checkpoint
2. From that finalized checkpoint, select the casper checkpoint with the highest justified epoch: `casper.last_justified_epoch()`
3. Starting from that justified epoch, choose the block with the highest PoW score as the new head

A client considers a checkpoint finalized if the following hold true:

* During an epoch, the previous epoch is finalized within the casper contract:
```python
casper.last_finalized_epoch() == casper.current_epoch() - 1
```
* The current dynasty deposits _during the proposed finalized epoch_ were greater than `NON_REVERT_MIN_DEPOSIT`:
```python
casper_during_finalized_epoch.total_curdyn_deposits_scaled() > NON_REVERT_MIN_DEPOSIT
```
* The previous dynasty deposits _during the proposed finalized epoch_ were greater than `NON_REVERT_MIN_DEPOSIT`:
```python
casper_during_finalized_epoch.total_prevdyn_deposits_scaled() > NON_REVERT_MIN_DEPOSIT
```

#### Block Reward

If `block.number >= HYBRID_CASPER_FORK_BLKNUM`, then `block_reward = NEW_BLOCK_REWARD` and utilize the same formulas for uncle and nephew rewards but with the updated `block_reward`.

#### Validators

The mechanics and responsibilities of validators are not specified in this EIP because they rely upon network transactions to the contract at `CASPER_ADDR` rather than on protocol level implementation and changes.
See the [Validator Implementation Guide](https://github.com/ethereum/casper/blob/master/VALIDATOR_GUIDE.md) for validator details.

#### SIGHASHER_CODE

The source code for `SIGHASHER_CODE` is located [here](https://github.com/ethereum/casper/blob/master/casper/validation_codes/verify_hash_ladder_sig.se).

The EVM init code is:
```
0x
```

The EVM bytecode that the contract should be set to is:
```
0x
```

#### PURITY_CHECKER_CODE

The source code for `PURITY_CHECKER_CODE` is located [here](https://github.com/ethereum/research/blob/master/impurity/check_for_impurity.se).

The EVM init code is:
```
0x
```

The EVM bytecode that the contract should be set to is:
```
0x
```

#### CASPER_CODE

The source code for `CASPER_CODE` is located at
[here](https://github.com/ethereum/casper/blob/master/casper/contracts/simple_casper.v.py).

The EVM init code with the above specified params is:
```
0x
```

The EVM bytecode that the contract should be set to is:
```
0x
```

## Rationale

Naive PoS specs and implementations have existed since early blockchain days, but most are vulnerable to serious attacks and do not hold up under crypto-economic analysis. Casper FFG solves problems such as "Nothing at Stake" and "Long Range Attacks" through requiring validators to post slashable deposits and through defining economic finality.

#### Minimize Consensus Changes
The finality gadget is designed to minimize changes across clients. For this reason, FFG is implemented within the EVM, so that the contract byte code encapsulates most of the complexity of the fork.

Other were also designed to minimize changes across clients. For example, it would be possible to allow `CASPER_ADDR` to mint Ether each time it payed rewards (as compared to creating the contract with `CASPER_BALANCE`), but this would be more invasive and error-prone than relying on existing EVM mechanics.

#### Economic Constants
*insert: Discuss economic constants*

`WITHDRAWAL_DELAY` is set to 15000 epochs to freeze a validator's funds for approximately 4 months after logout. This allows for at least a 4 month window to identify and slash a validator for attempting to finalize two conflicting checkpoints. This defines the window of time with which a client must log on to sync a network due to weak subjectivity.
```python
delay_in_months = (15000 epochs) * (50 blocks/epoch) * (14 seconds/block) * (1/86400 days/second) * (1/30.4 month/day)
round(delay_in_months, 2) == 4.00
```

`DYNASTY_LOGOUT_DELAY` is set set to 700 dynasties to prevent immediate logout in the event of an attack from being a viable strategy.

#### Issuance
A fixed amount of 5 million ether was chosen as `CASPER_BALANCE` to fund the casper contract. This only gives the contract enough runway to operate for approximately _insert time based on economic constants_, acting similarly to the "difficulty bomb". This "funding crunch" forces the network to hard-fork in the relative near future to further fund the contract. This future hard fork is a good opportunity to upgrade the contract and likely transition to full PoS.

The PoW block reward is further reduced to 0.6 eth/block because security of the chain is greatly shifted from PoW to PoS finality and because rewards are now issued to both stakers and miners.

#### Gas Changes
Successful casper `vote` transactions are included at the end of the block so that they can be processed in parallel with normal block transactions and cost 0 gas for validators.

The call to `initialize_epoch` at the beginning of each epoch requires 0 gas so that this protocol state transition does not take any gas allowance away from normal transactions.

#### NULL_SENDER and Account Abstraction
This EIP implements a limited version of account abstraction for validator's `vote` transactions. The general design was borrowed from [EIP 86](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-86.md).

## Backwards Compatibility
This EIP is not forward compatible and introduces backwards incompatibilities in the state, fork choice rule, block reward, transaction validity, and gas calculations on certain transactions. Therefore, all changes should be included in a scheduled hardfork at a `HYBRID_CASPER_FORK_BLKNUM`.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
