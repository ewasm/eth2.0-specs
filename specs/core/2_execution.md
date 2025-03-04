# Ethereum 2.0 Phase 2 -- Execution

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- TOC -->

TBA

<!-- /TOC -->

### Introduction

This document describes a proposal for modifications to the beacon chain, and definition of the shard state and shard state transition function, for phase 2 (state and transaction execution) of ethereum 2.0. The general ethos of the proposal is to have a relatively minimal consensus-layer framework, that still provides sufficient capabilities to develop complex frameworks that give us all of the smart contract capabilities that we need on top as a second layer. The first part of the document describes the framework; the second part gives a basic example of implementing in-shard ETH transfers on top of the framework.<br>
The basic idea of the approach is that contracts as a base-layer concept exist only on the beacon chain, and ETH only exists on the beacon chain (ETH can be held by either beacon chain contracts [which we'll call "execution scripts"] or by validator accounts). Each beacon chain contract can maintain a 32-byte state per shard. A transaction on a shard must specify which contract it calls, and the in-shard state transition function executes the specific execution script code using the transaction as data, with the code execution being a pure function `f(prev_state: bytes32, block: bytes) -> (bytes32, List[DepositData])`. It turns out that this provides sufficient functionality to allow an execution environment that supports smart contracts in shards, cross shard communication and all of the other features that we expect to be built using a beacon chain contract.

### Out of scope for now

* EIP 1559 on eth2
* Shard rewards and penalties more generally
* Implementation of standardized SSZ Merkle proofs
* Generalization to blocks having multiple parts that use different execution environments

### Standardized SSZ Merkle proofs

* `DepositReceiptProof`: proves that a given `DepositData` was included as part of the receipts generated by the execution of a `ShardBlock` and included in a crosslink that has been included in the beacon chain
* `WithdrawalReceiptProof`: proves that a given `WithdrawalReceipt` was included as part of a `BeaconState` which either is in the current `latest_state_roots` or is an ancestor of a state in the current `latest_state_roots`

### Beacon Chain Changes

On the beacon chain we add three new transaction types:

`NewExecutionScript` (basically, creates a new execution script whose code lives on the beacon chain and which can hold ETH):

```python
class NewExecutionScript(Container):
    sender: uint64
    slot: uint64
    code: bytes
    pubkey: BLSPubkey
    signature: BLSSignature
```

`NewValidator` (adds a new validator using ETH taken from an execution script's balance; the operation is authorized via that execution script issuing a receipt in a shard):

```python
class NewValidator(Container):
    executor: uint64
    receipt: ShardReceipt
    proof: DepositReceiptProof
```

`Withdrawal` (withdrawal of a beacon chain validator, transferring its ETH to an execution script and issuing a receipt):

```python
class Withdrawal(Container):
    validator_index: uint64
    target: uint64
    data: bytes
    pubkey: BLSPubkey
    signature: BLSSignature
```

We add to the `BeaconState` two new data structures:

`ExecutionScript`:

```python
class ExecutionScript(Container):
    code: bytes
    balance: Gwei
```

`WithdrawalReceipt`:

```python
class WithdrawalReceipt(Container):
    receipt_index: uint64
    withdrawal: Withdrawal
    amount: Gwei
```

The beacon state stores a list of each object; the latter is emptied at the beginning of every slot. The beacon state also stores a counter `next_withdrawal_receipt_index`. We also add to `DepositData` a field `min_timestamp`; we add to the `process_deposit` function a requirement that the beacon chain's observed timestamp must be at least that value and less than that value plus `MIN_VALIDATOR_PERSISTENCE_TIME` (set to 1 year). This plus the pubkey uniqueness requirement are used for replay protection.

The new transaction types are processed as follows:

```python
def process_new_execution_script(state: BeaconState,
                                 new_execution_script: NewExecutionScript) -> None:
    # Verify there is sufficient balance
    fee = NEW_CODE_FEE + NEW_CODE_BYTE_FEE * len(code)
    assert state.balances[new_execution_script.sender] >= fee
    # A new-execution-script is valid in only one slot
    assert state.slot == new_execution_script.slot
    # Sender must be not yet eligible for activation or withdrawn
    sender_acct = state.validator_registry[new_execution_script.sender]
    assert (
        sender_acct.activation_eligibility_epoch == FAR_FUTURE_EPOCH or
        get_current_epoch(state) >= sender_acct.withdrawable_epoch
    )
    # Verify that the pubkey is valid
    assert (
        sender_acct.withdrawal_credentials ==
        BLS_WITHDRAWAL_PREFIX_BYTE + hash(new_execution_script.pubkey)[1:]
    )
    # Verify that the signature is valid
    assert bls_verify(
        pubkey=new_execution_script.pubkey,
        message_hash=signing_root(new_execution_script), 
        signature=new_execution_script.signature,
        domain=get_domain(state, DOMAIN_TRANSFER)
    )
    # Verify that the code is valid WASM and not too long
    assert (
        verify_wasm(new_execution_script.code) and
        len(new_execution_script.code) <= MAX_CODE_LEN
    )
    # Add the new execution script to the beacon state
    decrease_balance(state, new_execution_script.sender, fee)
    state.execution_scripts.append(ExecutionScript(0, new_execution_script.code))
```

```python
def process_new_validator(state: BeaconState, new_validator: NewValidator) -> None:
    # Verify the receipt proof
    assert verify_deposit_receipt_proof(
        state,
        new_validator.receipt,
        new_validator.proof
    )
    # Receipt target 2**256-1 corresponds to new validator
    assert new_validator.receipt.target == 2**256 - 1
    # Interpret receipt data as DepositData object
    deposit_data = deserialize(new_validator.recent.data, DepositData)
    # Check that there's enough ETH in the execution script
    assert new_validator.executor < len(state.execution_scripts)
    new_validator_acct = state.execution_scripts[new_validator.executor]
    assert new_validator_acct.balance >= deposit_data.amount
    # Equivalent to `process_deposit` except it removes the initial code 
    # that verifies the parts of the Deposit outside the DepositData
    assert process_deposit_data(state, deposit_data)
    # Subtract the ETH from the execution script's balance
    new_validator_acct.balance -= deposit_data.amount
```

```python
def process_withdrawal(state: BeaconState, withdrawal: Withdrawal) -> None:
    # Sender must be withdrawable
    withdrawer_acct = state.validator_registry[withdrawal.validator_index]
    withdrawer_balance = state.balances[withdrawal.validator_index]
    assert get_current_epoch(state) >= withdrawer_acct.withdrawable_epoch
    # Verify that the pubkey is valid
    assert (
        withdrawer_acct.withdrawal_credentials ==
        BLS_WITHDRAWAL_PREFIX_BYTE + hash(withdrawal.pubkey)[1:]
    )
    # Verify that the signature is valid
    assert bls_verify(
        pubkey=withdrawal.pubkey,
        message_hash=signing_root(withdrawal),
        signature=withdrawal.signature,
        domain=get_domain(state, DOMAIN_WITHDRAWAL)
    )
    # Add a withdrawal receipt
    state.withdrawal_receipts.append(WithdrawalReceipt(
        receipt_index=state.next_receipt_index,
        withdrawal=withdrawal,
        amount=withdrawer_balance
    ))
    # Transfer funds to the execution script
    assert withdrawal.target < len(state.execution_scripts)
    state.execution_scripts[withdrawal.target].balance += withdrawer_balance
    # Delete the validator
    state.balances[withdrawal.validator_index] = 0
    state.validator_registry[withdrawal.validator_index] = Validator()
```

### Shard processing

The `ShardState` has the following format:

```python
class ShardState(Container):
    # 32 bytes per exec env
    exec_env_states: List[bytes32]
    # Current slot
    slot: uint64
    # Parent block
    parent_block: ShardBlockHeader
    # Some recent historical state roots since the last known crosslink
    latest_state_roots: List[bytes32, LATEST_STATE_ROOTS_LENGTH]
```


The shard state has a simple state transition function:

```python
def process_block(state: ShardState,
                  beacon_state: BeaconState,
                  block: Union[Null, ShardBlock]):
    # Fill in state to accumulator
    pre_state_root = hash_tree_root(state)
    state.latest_state_roots[state.slot % LATEST_HASHES_LENGTH] = pre_state_root
    # Increment state slot
    state.slot += 1
    # Process block
    if block:
        exec_code = beacon_state.execution_scripts[block.env].code
        while block.env >= len(state.exec_env_states):
            state.exec_env_states.append(ZERO_HASH)
        pre_state = state.exec_env_states[block.env]
        post_state, deposits = execute_code(exec_code, [pre_state, block.data])
        state.exec_env_states[block.env] = post_state
    else:
        deposits = []
    assert block.state_root == hash_tree_root(state)
    assert block.deposit_root == hash_tree_root(deposits)
```

We define the function `flatten_block` which creates the `BLOCK_SIZE * 2 ` byte representation of a shard block that goes into a crosslink for custody and data availability and fraud proving.

```python
def flatten_block(block: ShardBlock,
                  pre_state: ShardState,
                  beacon_state: BeaconState) -> bytes:
    # Block header
    section_1 = zpad(serialize(get_block_header(block)), BLOCK_SIZE // 8)
    # State transition proofs
    section_2 = zpad(
        serialize(SSZMerklePartial(process_block, pre_state, beacon_state, block)),
        BLOCK_SIZE * 3 // 8
    )
    # Block data
    section_3 = zpad(serialize(block.data), BLOCK_SIZE // 2)
    return section_1 + section_2 + section_3
```

Note that the above data is sufficient to verify as a fraud proof:

```python
def verify_correctness(data: bytes) -> bool:
    block_header = deserialize(ShardBlockHeader, data[:BLOCK_SIZE // 8])
    _, shard_state_partial, beacon_state_partial = (
        deserialize(
            SSZMerklePartial[ShardBlock, ShardState, BeaconState],
            data[BLOCK_SIZE//8: BLOCK_SIZE//2]
        )
    )
    block_data = deserialize(bytes, data[BLOCK_SIZE//2:])
    block = ShardBlock(block_header, block_data)
    assert process_block(shard_state_partial, beacon_state_partial, block)    
```

### Implementing in-shard ETH transfers

On top of the above base, it's possible to implement an entire fully fledged smart-contract-capable state execution framework through higher layers of software abstraction. To start off, here is how one might set up the simplest possible framework, one that simply allows users to deposit an ETH balance to a shard, move the ETH around, and then later withdraw it. In the beginning, we will make the simplifying assumption that state objects last forever, so pokes cannot happen; later we will relax this assumption.

We first define a few SSZ class:

`EthAccount`:

```python
class EthAccount(Container):
    pubkey: BLSPubkey
    nonce: uint64
    value: uint64
```

`MyWithdrawal`:

```python
class MyWithdrawal(Container):
    receipt: WithdrawalReceipt
    proof: WithdrawalReceiptProof
    state_witness: SSZMerklePartial[BigState]
```

`MyTransfer`:

```python
class MyTransfer(Container):
    sender: bytes32
    nonce: uint64
    target: bytes32
    amount: uint64
    signature: BLSSignature
    state_witness: SSZMerklePartial[BigState]
```

`MyDeposit`:

```python
class MyDeposit(Container):
    address": bytes32
    nonce": uint64
    signature: BLSSignature
    deposit: DepositData
    state_witness: SSZMerklePartial[BigState]
```

We also define `BigState` as `List[EthAccount, 2**256]`.

We can now define our functions (which will all be in the WASM code of the execution environment). First we define the function `processWithdrawal`, which "consumes" a withdrawal receipt and publishes the ETH into an account on the desired shard, which is intended to be called by a transaction in the desired shard with the transaction's `data` containing the encoded function call, and with the `executor` being the ID of this execution script. Here is the code:

```python
def processWithdrawal(state_root: bytes32,
                      tx: MyWithdrawal) -> bytes32:
    receipt, proof, witness = tx.receipt, tx.proof, tx.witness
    # Verify Merkle proof of the withdrawal receipt
    assert verify_withdrawal_receipt_root_proof(
        get_recent_beacon_state_root(proof.root_slot),
        receipt,
        proof
    )
    # Interpret receipt data as an object in our own format
    receipt_data = deserialize(receipt.withdrawal.data, FormattedReceiptData)
    # Check that this function is being executed on the right shard
    assert receipt_data.shard_id == getShard()
    # Check witness validity
    assert state_root == witness.root
    # Save the balance
    address = hash(receipt_data.pubkey)
    new_state = EthAccount(
        pubkey=receipt_data.pubkey,
        nonce=current_state.nonce,
        value=current_state.value + receipt.amount
    )
    witness[address] = new_state
    return witness.root
```

Now `processTransfer` for transferring ETH between accounts, hopefully self-explanatory without comments:

```python
def processTransfer(state_root: bytes32,
                    tx: MyTransfer) -> bytes32:
    sender, nonce, target, amount, signature, witness = (
        tx.sender, tx.nonce, tx.target, tx.amount, tx.signature, tx.witness
    )
    assert witness[sender].nonce == nonce
    assert witness[sender].value >= amount
    assert bls_verify(
        pubkey=witness[sender].pubkey,
        message_hash=hash(nonce, target, amount, getShard()),
        signature=signature
    )
    witness[sender] = EthAccount(
        pubkey=sender_account.pubkey,
        nonce=sender_account.nonce + 1,
        value=sender_account.value - amount
    )
    witness[target] EthAccount(
        pubkey=witness[target].pubkey,
        nonce=witness[target].nonce,
        value=witness[target].value + amount
    ))
    return witness.root
```

Now `sendToValidatorDeposit`, for sending the ETH in an account back into a validator slot:

```python
def processDeposit(state_root: bytes32,
                   tx: MyDeposit) -> bytes32:
    address, nonce, signature, deposit_data, witness = (
        tx.address, tx.nonce, tx.signature, tx.deposit_data, tx.witness
    )
    # Verify that the provided deposit data is valid
    assert verify_deposit_data(deposit_data)
    # Verify balance sufficiency
    assert witness[address].value >= deposit_data.amount
    # Verify the signature
    assert bls_verify(
        message_hash=hash_tree_root({nonce: nonce, deposit_data: deposit_data}),
        pubkey=witness[address].pubkey,
        signature=signature
    )
    # Save the reduced balance
    witness[address] = EthAccount(
        pubkey=witness[address].pubkey,
        nonce=witness[address].nonce + 1,
        value=witness[address].value - deposit_data.amount
    )
    return witness.root
```

Now we put it all together and make the block processing function:

```python
def process_block(state: bytes32,
                  block: ShardBlock) -> Tuple[bytes32, List[DepositData]]:
    txs = deserialize(List[Union[MyWithdrawal, MyTransfer, MyDeposit]], block.data)
    deposits = []
    for tx in txs:
        if isinstance(tx, MyWithdrawal):
            state = processWithdrawal(state, tx)
        elif isinstance(tx, MyTransfer)
            state = processTransfer(state, tx)
        elif isinstance(tx, MyDeposit)
            state, new_deposit = processDeposit(state, tx)
            deposits.append(new_deposit)
    return state, deposits
```

### From ETH transfers to complete state execution

To create a complete framework, we would need to add the following components on top of this (ie. this is all more executor code, no changes required to the above consensus layer):

* Add a `crossShardMessage` function, which creates a receipt specifying a destination shard in addition to target and state, and a function for reviving with these messages
* Add a proper `Transaction` object, with the main components being `revives` (list of accounts to revive + receipts for reviving them), `gasPrice` (transaction fee per gas), `operations` (list of actions the transaction takes), `witness`. Signature verification is no longer BLS-specific; instead, we use the abstracted `assert executeCode(account.witness_verifier, (account.state, tx.witness)) == 1`.
* Add a proper transaction execution state transition function, which processes all of these parts of a transaction
* In addition to sending ETH, add another type of operation, a contract call. A contract call roughly works by starting with `stack = operations[::-1]`, then `while len(stack > 0)`, pop the top operation off the stack and run `executeCode(target.code, (target.state, operation.calldata))`; this would be expected to return `(new_state, continuation)`; the state transition function would set the state to equal the new state, and then add the continuation to the stack if any.
* Add the conditional state logic from https://ethresear.ch/t/fast-cross-shard-transfers-via-optimistic-receipt-roots/5337
* Add a mechanism for expiring accounts and converting them into receipts, and for reviving accounts from those receipts (with a bitfield to prevent double-spending)

### Generic fee payment

To allow validators to be able to collect transaction fees without every client needing to implement every layer-2 scheme, we can create a generic abstraction layer as follows. We create a specialized layer-2 scheme where anyone can publish a message of the form "If you create a block in shard X at slot Y, where the previous state root is Z, then I will give you N gwei", and processing this kind of conditional transaction is the only operation. Then for each user-side layer-2 there can be a separate class of users, which we call **relayers**, that gather transactions and publish packages that bid using this system.

Relayers would be responsible for maintaining the `BigState` (or equivalents in other execution environment implementations), and one would expect that users send "naked" transactions _without_ the witness (the `SSZMerklePartial` of the `BigState`) and it is the relayers that add the witness.

To prevent proposers from simply copying data from relayer proposals without rewarding relayers, we can use JMRS: https://ethresear.ch/t/optimised-proposal-commitment-scheme/1314. Essentially, (i) relayers make proposals, publish only hashes with fees, (ii) block proposer makes a pre-commitment (punishable with slashing if they break it) to only include a proposal which is one of the proposals submitted, (iii) relayers submit bodies.

Note that this market can even be implemented (and be useful) during phase 1, though the code would sit on the PoW chain rather than a layer-2 scheme.
