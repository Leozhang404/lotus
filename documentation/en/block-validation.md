# Incoming block validations

This document reviews the code flow that takes place inside the full node after receiving a new block from the GossipSub `/fil/blocks` topic and traces all of its protocol-related validation logic. We do not include validation logic *inside* the VM, the analysis stops at `(*VM).Invoke()`. The `V:` tag explicitly signals validations throughout the text.

## `modules.HandleIncomingBlocks()`

We subscribe to the `/fil/blocks` PubSub topic to receive external blocks from peers in the network and register a block validator that operates at the PubSub (`libp2p` stack) level, validating each PubSub message containing a Filecoin block header.

`V:` PubSub message is a valid CBOR `BlockMsg`.

`V:` Total messages in block are under `BlockMessageLimit`.

`V:` Aggregate message CIDs, encapsulated in the `MsgMeta` structure, serialize to the `Messages` CID in the block header (`ValidateMsgMeta()`).

`V:` Miner `Address` in block header is present and corresponds to a public-key address in the current chain state.

`V:` Block signature (`BlockSig`) is present and belongs to the public-key address retrieved for the miner (`CheckBlockSignature()`).

## `sub.HandleIncomingBlocks()`

Assemble a `FullBlock` from the received block header retrieving its Filecoin messages.

`V:` Block messages CIDs can be retrieved from the network and decode into valid CBOR `Message`/`SignedMessage`.

## `(*Syncer).InformNewHead()`

Assemble a `FullTipSet` populated with the single block received earlier.

`V:` `ValidateMsgMeta()` (already done in the topic validator).

`V:` Block's `ParentWeight` is greater than the one from the (first block of the) heaviest tipset.

## `(*Syncer).Sync()`

`(*Syncer).collectHeaders()`: we retrieve all tipsets from the received block down to our chain. Validation now is expanded to *every* block inside these tipsets.

`FixMe:` Add DRAND check: `// ensure consistency of beacon entires`.

`V:` Tipset `Parents` CIDs match the fetched parent tipset through block sync. (This check is not enforced correctly at the moment, see [issue](https://github.com/filecoin-project/lotus/issues/1918).)

`FixMe:` Consider mentioning `syncFork()` if relevant.

## `(*Syncer).ValidateBlock()`

This function contains most of the validation logic grouped in separate closures that run asynchronously, this list does not reflect validation order then.

`V:` Block `Timestamp`:
  * Is not bigger than current time plus `AllowableClockDrift`.
  * Is not smaller than previous block's `Timestamp` plus `BlockDelay` (including null blocks).

### Messages

We check all the messages contained in one block at a time (`(*Syncer).checkBlockMessages()`).

`V:` The block's `BLSAggregate` matches the aggregate of BLS messages digests and public keys (extracted from the messages `From`).

`V:` Each `secp256k1` message `Signature` is signed with the public key extracted from the message `From`.

`V:` Aggregate message CIDs, encapsulated in the `MsgMeta` structure, serialize to the `Messages` CID in block header (similar to `ValidateMsgMeta()` call).

`V:` For each message, in `ValidForBlockInclusion()`:
* Message fields `Version`, `To`, `From`, `Value`, `GasPrice`, and `GasLimit` are correctly defined.
* Message `GasLimit` is under the message minimum gas cost (derived from chain height and message length).

`V:` Actor associated with message `From` exists and is an account actor, its `Nonce` matches the message `Nonce`.

### Miner

`V:` Miner address is registered in the `Claims` HAMT of the Power actor.

### Compute parent tipset state

`V:` Block's `ParentStateRoot` CID matches the state CID computed from the parent tipset.

`V:` Block's `ParentMessageReceipts` CID matches receipts CID computed from the parent tipset.

### Winner

`FixMe:` Describe `winnerCheck` validations. Winning ticket, DRAND, randomness, VRF.

### Block signature

`V:` `CheckBlockSignature()` (same signature validation as the one applied to the incoming block).

### Beacon values check

`FixMe:` Describe `beaconValuesCheck`.

### `tktsCheck()`

`FixMe:` Change section title from cryptic function name to a more descriptive text.

`FixMe:` Describe `tktsCheck()`. Draw randomness, VRF.

### Winning PoSt proof

`FixMe:` Describe `VerifyWinningPoStProof`.

## `(*StateManager).TipSetState()`

Called throughout the validation process for the parent of each tipset being validated. The checks here then do not apply to the received new head itself that started the validation process.

`FixMe:` Is a tipset fully validated once we start applying block messages from a new epoch then?

### `(*StateManager).computeTipSetState()`

`FixMe:` Consider mentioning `handleStateForks()` if relevant.

`V:` Every block in the tipset should belong to different a miner.

`FixMe:` Is there a way that a `BlockHeader.Miner` address we check could refer to the same miner but with different address protocol type? In general the answer is no because other checks would fail if this is not an ID address, but that should be made explicit.

### `(*StateManager).ApplyBlocks()`

We create a new VM with the tipset's `ParentStateRoot` (this is then the parent state of the parent of the tipset currently being validated) on which to apply all messages from all blocks in the tipset. For each message independently we apply the validations listed next.

`FixMe:` Check `ApplyImplicitMessage` calls and related logic if they have any relevant validations (it would seem they don't).

### `(*VM).ApplyMessage()`

`V:` Basic gas and value checks in `checkMessage()`:
* Message `GasLimit` is bigger than zero.
* Message `GasPrice` and `Value` are set.

`V:` Message storage gas cost is under the message's `GasLimit`.

`FixMe:` Skipping the checks: `// this should never happen, but is currently still exercised by some tests`, confirm if this is correct.

`V:` Message's `Nonce` matches nonce in actor retrieved from message's `From`.

`V:` Message's maximum gas cost (derived from its `GasLimit`, `GasPrice`, and `Value`) is under the balance of the actor retrieved from message's `From`.

### `(*VM).send()`

`V:` Message's transfer `Value` is under the balance in actor retrieved from message's `From`.