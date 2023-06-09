# Common data structures specification

## Overview

This specification describes several common data structures used across various Aptos Payment Network (LPN) specifications.

## Terminology

* **epoch** is an u64 that identifies the number of reconfiguration operations that have occurred in the past
* **reconfiguration** is a consensus mechanism for supporting changes to agreement on ordering and execution that can be initiated via special reconfiguration transactions.

## Data structures

Similar to all other LPN specifications, we use Rust to describe all data structures below such that serialization is implemented by serializing each field in sequential order according to the BCS specification.

### AccountAddress

This represents a 128-bit Aptos account address.

```rust
struct AccountAddress([u8; 16]);
```

### BlockInfo

A `BlockInfo` contains all the information necessary for tracking a block without having access to its internals (i.e. transactions).

```rust
struct BlockInfo {
    epoch: u64,
    round: u64,
    id: HashValue,
    executed_state_id: HashValue,
    version: u64,
    timestamp_usecs: u64,
    next_validator_set: Option<ValidatorSet>,
}
```

Fields:

* `epoch` is the epoch that this block belongs to
* `round` is the consensus round number of this block. In a chain of certified blocks, round numbers are strictly increasing
* `id` is the unique identifier of the block as defined by the consensus specification (TODO: define how id is generated in the consensus specification)
* `executed_state_id` is the accumulator root hash (TODO: define accumulator root hash implementation) after execution of transaction in this block
* `version` is count of recorded transactions after executing this block
* `timestamp_usecs` is the [Unix time](https://en.wikipedia.org/wiki/Unix_time) in the unit of microseconds at which all transactions are executed at in this block
* `next_validator_set` is an optional field. If it is set, it contains the validators for the start of the next epoch and will also trigger reconfiguration as

### Ed25519Signature

An ed25519 signature as specified by the [ed25519-dalek](https://github.com/dalek-cryptography/ed25519-dalek). TODO: Define the specification this Rust crate implements - (i.e. <https://tools.ietf.org/html/rfc8032>).

```rust
struct Ed25519Signature {
    R: CompressedEdwardsY,
    s: Scalar,
}
```

Fields

* `R` is an `EdwardsPoint`, formed by using an hash function with 512-bits output to produce the digest of:

  * the nonce half of the `ExpandedSecretKey`, and
  * the message to be signed.

  This digest is then interpreted as a `Scalar` and reduced into an element in ℤ/lℤ. The scalar is then multiplied by the distinguished basepoint to produce `R`, and `EdwardsPoint`.

* `s` is `s` is a `Scalar`, formed by using an hash function with 512-bits output to produce the digest of:

  * the `r` portion of this `Signature`,
  * the `PublicKey` which should be used to verify this `Signature`, and
  * the message to be signed.

  This digest is then interpreted as a `Scalar` and reduced into an element in ℤ/lℤ.

### HashValue

This represents a 256-bit output value of a hashing function. The caller should indicate the hashing method employed and upon which data structures.

```rust
struct HashValue {
    hash: [u8; 32],
}
```

### LedgerInfo

This structure contains a hash of the unverified ledger state after a number of transactions (i.e. version) have been executed in `commit_info`. After `LedgerInfo` has been signed by >2f validators, a valid `LedgerInfoWithSignatures` can be generated.

```rust
struct LedgerInfo {
    commit_info: BlockInfo,
    consensus_data_hash: HashValue,
}
```

Fields:

* `commit_info` contains the highest block information known by this structure that will be committed if >2f validators sign this structure and form a `LedgerinfoWithSignatures`.
* `consensus_data_hash` is a value that is generated by the consensus specification that honest validators will agree on

### LedgerInfoWithSignatures

`LedgerInfoWithSignatures` contains verifiable commitment proof of a ledger history under a consensus specification. It is structured as an enum to support future upgrades that support old and new versions of ledger history as well as different consensus specifications or future signature schemes. `V0` is the currently supported enum value.

```rust
enum LedgerInfoWithSignatures {
    V0(LedgerInfoWithV0),
}

struct LedgerInfoWithV0 {
    ledger_info: LedgerInfo,
    signatures: BTreeMap<AccountAddress, Ed25519Signature>,
}
```

Fields:

* `ledger_info` contains the ledger state at a given transaction version
* `signatures` is a sorted map by account addresses that map to ed25519 signatures over the BCS hash value of `ledger_info`. The signatures must have >2f voting power from validator accounts in order to be considered committed by the consensus specification.

### ContractEvent

TODO: add descriptions.

```rust
enum ContractEvent {
    V0(ContractEventV0),
}

struct ContractEventV0 {
    key: EventKey,
    sequence_number: u64,
    type_tag: TypeTag,
    event_data: Vec<u8>,
}
```
