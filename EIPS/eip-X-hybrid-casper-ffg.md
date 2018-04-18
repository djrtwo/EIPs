---
eip: X
title: Hybrid Casper FFG
status: Draft
type: Standards Track
category: Core
author: Danny Ryan <danny@ethereum.org>, Chih-Cheng Liang <cc@ethereum.org>
created: 2018-04-18
---

## Simple Summary

Specification of the first step to transition Ethereum main net from Proof of Work (PoW) to Proof of Stake (PoS). The resulting consensus model is a PoW/PoS hybrid.

## Abstract

This EIP specifies a hybrid PoW/PoS consensus model for Ethereum main net. Existing PoW mechanics are used for new block creation, and a novel PoS mechanism called Casper the Friendly Finality Gadget (FFG) is layered on top using a smart contract.

Through the use of Ether deposits, slashing conditions, and a modified fork choice, FFG allows the underlying PoW blockchain to be finalized.  As network security is greatly shifted from PoW to PoS, PoW block rewards are reduced.

This EIP does not contain safety and liveness proofs or validator implementation details, but these can be found in the [Casper FFG paper](https://arxiv.org/abs/1710.09437) and [Validator Implementation Guide](https://github.com/ethereum/casper/blob/master/VALIDATOR_GUIDE.md) respectively.

## Glossary

* **epoch**: The span of blocks between checkpoints.
* **finality**: The point at which a block has been decided upon by a client to _never_ revert. Proof of Work does not have the concept of finality, only of further deep block confirmations.
* **checkpoint**: The start block of an epoch. Rather than dealing with every block, Casper FFG only considers checkpoints for finalization.
* **dynasty**: The number of finalized checkpoints in the chain from root to the parent of a block. The dynasty is used to define when a validator starts and ends validating.
* **slash**: The burning of a validator's deposit. Slashing occurs when a validator signs two conflicting messages that violate a slashing condition. For an in-depth discussion of slashing conditions, see the [Casper FFG Paper](https://arxiv.org/abs/1710.09437).

## Motivation

Transitioning the Ethereum network from PoW to PoS has been on the roadmap and in the [Yellow Paper](https://github.com/ethereum/yellowpaper) since the launch of the protocol. Although effective in coming to a decentralized consensus, PoW consumes an incredible amount of energy, has no economic finality, and has no effective strategy in resisting cartels. Excessive energy consumption, issues with equal access to mining hardware, mining pool centralization, and an emerging market of ASICs each provide a distinct motivation to make the transition as soon as possible.

Until recently, the proper way to make this transition was still an open area of research. In October of 2017 [Casper the Friendly Finality Gadget](https://arxiv.org/abs/1710.09437) was published, solving open questions of economic finality through validator deposits and crypto-economic incentives. For a detailed discussion and proofs of "accountable safety" and "plausible liveness", see the [Casper FFG](https://arxiv.org/abs/1710.09437) paper.

The Casper FFG contract can be layered on top of any block proposal mechanism, providing finality to the underlying chain. This EIP proposes layering FFG on top of the existing PoW block proposal mechanism as a conservative step-wise approach in the transition to full PoS. The new FFG staking mechanism requires minimal changes to the protocol, allowing the Ethereum network to fully test and vet Casper FFG on top of PoW before moving to a validator based block proposal mechanism.

## Parameters

* `HYBRID_CASPER_FORK_BLKNUM`: TBD
* `CASPER_ADDR`: TBD
* `CASPER_CODE`: see below
* `CASPER_BALANCE`: 1e24 wei (1,000,000 ETH)
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
* `BASE_INTEREST_FACTOR`: 6.933e-3
* `BASE_PENALTY_FACTOR`: 2.052e-7
* `MIN_DEPOSIT_SIZE`: 1e21 wei (1,000 ETH)


## Specification

#### Deploying Casper Contract

If `block.number == HYBRID_CASPER_FORK_BLKNUM`, then when processing the block, before processing any transactions:

* set the code of `SIGHASHER_ADDR` to `SIGHASHER_CODE`
* set the code of `PURITY_CHECKER_ADDR` to `PURITY_CHECKER_CODE`
* set the code of `CASPER_ADDR` to `CASPER_CODE`
* set balance of `CASPER_ADDR` to `CASPER_BALANCE`

#### Initialize Epochs

If `block.number >= HYBRID_CASPER_FORK_BLKNUM` and `block.number % EPOCH_LENGTH == 0`, execute a `CALL` with the following parameters before executing any normal block transactions:

* `SENDER`: NULL_SENDER
* `GAS`: 3141592
* `TO`: CASPER_ADDR
* `VALUE`: 0
* `NONCE`: 0
* `GASPRICE`: 0
* `DATA`: `<encoded call {method: 'initialize_epoch', args: [floor(block.number / EPOCH_LENGTH)]}`>`

This `CALL` utilizes no gas and does not increment the nonce of `NULL_SENDER`

#### Casper Votes

If `block.number >= HYBRID_CASPER_FORK_BLKNUM`, then:

* A valid `vote` transaction to `CASPER_ADDR` must satisfy each of the following:
  * Must have the following signature `(CHAIN_ID, 0, 0)` (ie. `r = s = 0, v = CHAIN_ID`)
  * Must have `value == nonce == gasprice == 0`
* When producing and validating a block, when handling `vote` transactions to `CASPER_ADDR`:
  * Only include "valid" `vote` transactions as defined above
  * Place all `vote` transactions at the end of the block
* When applying `vote` transactions to `CASPER_ADDR` to vm state:
  * Set sender to `NULL_SENDER`
  * Must not count gas of `vote` toward the block gas limit
  * Must not increment the nonce of `NULL_SENDER`
* All unsuccessful `vote` transactions to `CASPER_ADDR` are invalid and must not be included in the block

#### Fork Choice

If `block.number >= HYBRID_CASPER_FORK_BLKNUM`, the fork choice rule is the following:
1. Start with client's last finalized checkpoint (details on how a client decides on finalized checkpoints below)
2. From that finalized checkpoint, select the client's highest justified checkpoint (details on how a client assesses justified checkpoints below)
3. Starting from that justified checkpoint, choose the block with the highest PoW score as the new head

_Note_: If the client has no finalized checkpoints, start PoW from the client's highest justified checkpoint. If the client has no justified or finalized checkpoints, simply choose the block with the highest PoW score as in normal PoW.

A client considers a checkpoint finalized if each of the following is true:
* During an epoch, the previous epoch's checkpoint is finalized within the casper contract
* The current dynasty deposits _during the epoch in question*_ were greater than `NON_REVERT_MIN_DEPOSIT`
* The previous dynasty deposits _during the epoch in question*_ were greater than `NON_REVERT_MIN_DEPOSIT`
```python
casper.last_finalized_epoch() == casper.current_epoch() - 1
casper_state_during_epoch_in_question.total_curdyn_deposits_scaled() > NON_REVERT_MIN_DEPOSIT
casper_state_during_epoch_in_question.total_prevdyn_deposits_scaled() > NON_REVERT_MIN_DEPOSIT
```

A client considers a checkpoint justified if each of the following is true:
* During an epoch, the current epoch's checkpoint is justified within the casper contract
* The current dynasty deposits _during the epoch in question*_ were greater than `NON_REVERT_MIN_DEPOSIT`
* The previous dynasty deposits _during the epoch in question*_ were greater than `NON_REVERT_MIN_DEPOSIT`
```python
casper.last_justified_epoch() == casper.current_epoch()
casper_state_during_epoch_in_question.total_curdyn_deposits_scaled() > NON_REVERT_MIN_DEPOSIT
casper_state_during_epoch_in_question.total_prevdyn_deposits_scaled() > NON_REVERT_MIN_DEPOSIT
```

\* When assessing the size of total deposits for epoch finalization and justification, use `total_*dyn_deposits_scaled` from the state of the VM during the epoch. For example, if assessing whether epoch 10 should be finalized and epoch 10 occured from block 1000 to 1050, then get the `total_*dyn_deposits_scaled` during that 1000 to 1050 block range rather than during the current epoch.

#### Block Reward

If `block.number >= HYBRID_CASPER_FORK_BLKNUM`, then `block_reward = NEW_BLOCK_REWARD` and utilize the same formulas for uncle and nephew rewards but with the updated `block_reward`.

#### Validators

The mechanics and responsibilities of validators are not specified in this EIP because they rely upon network transactions to the contract at `CASPER_ADDR` rather than on protocol level implementation and changes.
See the [Validator Implementation Guide](https://github.com/ethereum/casper/blob/master/VALIDATOR_GUIDE.md) for validator details.

#### SIGHASHER_CODE

The source code for `SIGHASHER_CODE` is located [here](https://github.com/ethereum/casper/blob/master/casper/validation_codes/verify_hash_ladder_sig.se).
The source is to be migrated to Vyper LLL before it's bytecode is finalized for this EIP.

The EVM init code is:
```
TBD
```

The EVM bytecode that the contract should be set to is:
```
TBD
```

#### PURITY_CHECKER_CODE

The source code for `PURITY_CHECKER_CODE` is located [here](https://github.com/ethereum/research/blob/master/impurity/check_for_impurity.se).
The source is to be migrated to Vyper LLL before it's bytecode is finalized for this EIP.

The EVM init code is:
```
TBD
```

The EVM bytecode that the contract should be set to is:
```
TBD
```

#### CASPER_CODE

The source code for `CASPER_CODE` is located at
[here](https://github.com/ethereum/casper/blob/master/casper/contracts/simple_casper.v.py).
The contract is to be formally verified and further tested before it's bytecode is finalized for this EIP.

The EVM init code with the above specified params is:
```
TBD
```

The EVM bytecode that the contract should be set to is:
```
TBD
```

#### Client Settings

Clients should be implemented with the following configurable settings:

##### NON_REVERT_MIN_DEPOSIT

The minimum size of total deposits that the client must see in the FFG contract for the state of the contract to affect the client's fork choice. A suggested implementation is `--non-revert-min-deposit WEI_VALUE`.

See "Fork Choice" more details.

##### Exclusion
The ability to exclude a specified blockhash and all of it's descendants from a client's fork choice. A suggested implementation is `--exclude BLOCKHASHES`, where `BLOCK_HASHES` is a comma delimited list of blockhashes to exclude.

##### Join Fork
The ability to manually join a fork specified by a blockhash. A suggested implementation is `--join-fork BLOCKHASH` where the client automatically sets the head to the block defined by`BLOCKHASH` and locally finalizes it.

## Rationale

Naive PoS specifications and implementations have existed since early blockchain days, but most are vulnerable to serious attacks and do not hold up under crypto-economic analysis. Casper FFG solves problems such as "Nothing at Stake" and "Long Range Attacks" through requiring validators to post slashable deposits and through defining economic finality.

#### Minimize Consensus Changes

The finality gadget is designed to minimize changes across clients. For this reason, FFG is implemented within the EVM, so that the contract byte code encapsulates most of the complexity of the fork.

Most other decisions were made to minimize changes across clients. For example, it would be possible to allow `CASPER_ADDR` to mint Ether each time it payed rewards (as compared to creating the contract with `CASPER_BALANCE`), but this would be more invasive and error-prone than relying on existing EVM mechanics.

#### Casper Contract Params

`EPOCH_LENGTH` is set to 50 blocks as a balance between time to finality and message overhead.

`WITHDRAWAL_DELAY` is set to 15000 epochs to freeze a validator's funds for approximately 4 months after logout. This allows for at least a 4 month window to identify and slash a validator for attempting to finalize two conflicting checkpoints. This also defines the window of time with which a client must log on to sync the network due to weak subjectivity.

`DYNASTY_LOGOUT_DELAY` is set to 700 dynasties to prevent immediate logout in the event of an attack from being a viable strategy.

`BASE_INTEREST_FACTOR` is set to 6.933e-3 such that if there are ~10M ETH in total deposits, then validators earn approximately 5% per year in ETH rewards under optimal FFG conditions.

`BASE_PENALTY_FACTOR` is set to 2.052e-7 such that if 50% of deposits go offline, then offline validators lose half of their deposits in approximately 3 weeks, at which the online portion of validators becomes a 2/3 majority and can begin finalizing checkpoints again.

`MIN_DEPOSIT_SIZE` is set to 1000 ETH to form a natural upper bound on the total number of validators, bounding the overhead due to `vote` messages. Using formulas found [here](https://medium.com/@VitalikButerin/parametrizing-casper-the-decentralization-finality-time-overhead-tradeoff-3f2011672735) under "From validator count to minimum staking ETH", we estimate that with 1000 ETH minimum deposit at an assumed ~10M in total deposits there will be approximately 1300 validators at any given time. `vote`s are only sent after the first quarter of an epoch so 1300 votes have to fit into 37 blocks or ~35 `vote`s per block.

#### Issuance

A fixed amount of 1M ETH was chosen as `CASPER_BALANCE` to fund the casper contract. This gives the contract enough runway to operate for approximately 2 years (assuming ~10M ETH in validator deposits). Acting similarly to the "difficulty bomb", this "funding crunch" forces the network to hardfork in the relative near future to further fund the contract. This future hardfork is an opportunity to upgrade the contract and transition to full PoS.

The PoW block reward is reduced to 0.6 ETH/block because the security of the chain is greatly shifted from PoW difficulty to PoS finality and because rewards are now issued to both validators and miners.

Below is a table of deposit sizes with associated annual interest rate and approximate time until funding crunch:

| Deposit Size | Annual Interest | Funding Crunch |
| -------- | -------- | -------- |
| 2.5M ETH | 10.12%   | ~4 years   |
| 10M ETH  | 5.00%    | ~2 years   |
| 20M ETH  | 3.52%    | ~1.4 years |
| 40M ETH  | 2.48%    | ~1 year    |

#### Gas Changes

Successful casper `vote` transactions are included at the end of the block so that they can be processed in parallel with normal block transactions and cost 0 gas for validators.

The call to `initialize_epoch` at the beginning of each epoch requires 0 gas so that this protocol state transition does not take any gas allowance away from normal transactions.

#### NULL_SENDER and Account Abstraction

This EIP implements a limited version of account abstraction for validators' `vote` transactions. The general design was borrowed from [EIP 86](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-86.md). Rather than relying upon native transaction signatures, a validator specifies a signature contract when sending their `deposit` to `CASPER_ADDR`. When casting a `vote`, the validator bundles and signs the parameters of their `vote` according to the requirements of their signature contract. The `vote` method of the casper contract checks the signature of the parameters against the validator's signature contract, exiting the transaction as unsuccessful if the signature is not successfully verified. 

This allows validators to customize their own signing scheme for votes. Use cases include:

* quantum-secure signature schemes
* multisig wallets
* threshold schemes

For more details on validator account abstraction, see the [Validator Implementation Guide](https://github.com/ethereum/casper/blob/master/VALIDATOR_GUIDE.md).

#### Client Settings
##### NON_REVERT_MIN_DEPOSIT

`NON_REVERT_MIN_DEPOSIT` is defined and configurable locally by each client. Clients are in charge of deciding upon the minimum deposits (security) at which they will accept the chain as finalized. In the general case, differing values in the choice of this local constant will not create any fork inconsistencies because clients with very strict finalization requirements will revert to follow the longest PoW chain.

Arguments have been made to hardcode a value into clients or the contract, but we cannot reasonably define security required for all clients especially in the context of massive fluctuations in the value of ETH.

It is worth noting that due to `NON_REVERT_MIN_DEPOSIT` being defined locally it becomes very easy to turn off casper verification in the case of a bug by setting the value to 999999999999.

##### Exclusion

This setting is useful in coordinating minority forks in cases of majority collusion.

##### Join Fork

This setting is to be used by new clients that are syncing the network. Due to weak subjectivity, a blockhash must be supplied to successfully sync the network when initially starting a node.

This setting is also useful for coordinating minority forks in cases of majority collusion.

## Backward Compatibility

This EIP is not forward compatible and introduces backward incompatibilities in the state, fork choice rule, block reward, transaction validity, and gas calculations on certain transactions. Therefore, all changes should be included in a scheduled hardfork at `HYBRID_CASPER_FORK_BLKNUM`.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
