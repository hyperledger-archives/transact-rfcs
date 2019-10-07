- Feature Name: Multi-Database Smart Contracts
- Start Date: 2019-10-07
- RFC PR: (leave this empty)
- Transact Issue: (leave this empty)

# Summary
[summary]: #summary

Allow smart contracts to read state entries from multiple databases.

# Motivation
[motivation]: #motivation

Some use cases require reading data across blockchain networks (Sawtooth,
Etherium, etc.); one example would be a smart contract running on a small,
restricted-access network that requires access to data on a larger, public
network. It may also be desirable to read from relational databases, or to read
block information from the Sawtooth block store without replicating the
blockstore to state itself. By allowing smart contracts to read from multiple
databases, we can support all of these use cases.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Various transact components will need to be updated to support knowledge of
multiple databases and associated state IDs.

For each ContextManager, there will be one writable database called the
“primary” database; there can additionally be any number of read-only,
“secondary” databases. Each of the secondary databases will be identified with a
unique name that is represented as a string. The ContextManager’s get methods
will be updated to take a new storage address type that specifies the database
to retrieve values from, as well as the keys of the values within those
databases. ContextManager’s set and delete methods will only modify entries in
the primary database.

When Schedulers are created, a state ID for the primary database and state IDs
for all secondary databases known to the ContextManager must be specified; since
the Scheduler does not have knowledge of the kinds of transactions it contains,
it must assume that the transactions may read from all secondary databases.

Like Schedulers, Contexts will be created with state IDs for all databases known
to the ContextManager (primary database and all secondary databases). Context’s
get methods will be updated to take the same storage address type as the
ContextManager’s get methods, which will support specifying which database to
retrieve values from. Again, Context’s set and delete methods will only modify
the primary database. Additionally, Contexts will store the storage address and
corresponding values of all entries that are retrieved from state by the
corresponding transaction.

The TransactionContext trait and its implementing structs will be updated to
take the storage address type as well; its set and delete methods will only
interact with the primary database.

TransactionReceipts will contain a new field that stores a list of all
database/key/value sets that are retrieved from state by the corresponding
transaction; this will allow for easier replay-ability of the transactions.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## ContextManager/ContextLifecycle Changes

Instead of having a single database, the ContextManager will have multiple
databases. Only one of the databases (the “primary” database) will be writable;
the rest (“secondary” databases, which are optional) will be read-only and
named. The ContextManager’s `database` field will be replaced with the following
fields:

```rust
primary_database: Box<
    dyn Read<StateId = String, Key = String, Value = Vec<u8>>,
>,
secondary_database: HashMap<
  String,
  Box<dyn Read<StateId = String, Key = String, Value = Vec<u8>>>,
>,
```

The key of the HashMap of secondary databases will be the names of the
databases.

A new Rust enum will be introduced to represent addresses across databases:

```rust
enum StorageAddress {
    Primary(String),
    Secondary(String, String),
}
```

The ContextManager’s get method will take a list of `StorageAddress`es for
entries to get. If any Secondary address is specified with a database name that
the ContextManager does not have, it will return a new `ContextManagerError`
type:

```rust
MissingDatabaseError(String)
```

In this error, the string will be the name of the database that could not be
found.

The ContextManager will record the values retrieved by calls to its get method
when the values are retrieved from one of the databases; these will be recorded
by adding them to the Context that the method was called with. Values will not
be recorded if they were modified or retrieved by a previous context; this
prevents frequently retrieved values from being unnecessarily recorded.

The signatures of the ContextManager’s `set_state` and `delete_state` methods
will be unchanged; the methods will modify the primary database only.

The ContextLifecycle will define a new method
`get_secondary_database_names(self) -> Vec<&str>;` the context manager will
implement this method to return a list of the names of its secondary databases.
This is required to ensure that Schedulers are created with `state_id`s for all
of the known databases.

## Scheduler changes

Schedulers will be created with a `state_id` for the primary database and
`state_id`s for all secondary databases. When the Scheduler is created, it will
get the list of secondary databases from the ContextLifecycle and enforce that
`state_id`s are specified for all known secondary databases; if not, an error
will be returned. To accomplish this, the new method of Schedulers will be
updated to:

```rust
pub fn new(
    context_lifecycle: Box<ContextLifecycle>,
    primary_state_id: String,
    secondary_state_ids: HashMap<String, String>,
) -> Result<SerialScheduler, SchedulerError>;
```

The new `SchedulerError` type will be defined as follows:

```rust
pub enum SchedulerError {
    MissingStateIds(Vec<String>)
}
```

In this enum, the contained vector will be a list of the names of the secondary
databases whose state IDs are missing.

Rather than having a single `state_id`, Schedulers will have a primary database
`state_id` and `state_id`s for all secondary databases, mapped by the name of
the associated databases:

```rust
primary_state_id: String,
secondary_state_ids: HashMap<String, String>,
```

Schedulers will take an additional argument on creation to store arbitrary data
associated with the schedule: `data: Vec<u8>`. This argument will be stored by
the Scheduler. This data field can be used, for instance, to store some proof of
agreement on the input states.

## Context Changes

Like the schedulers that create them, Contexts will have a primary `state_id`
and multiple secondary `state_id`s, mapped by the name of the associated
database. `ContextLifecycle.create_context` will become:

```rust
fn create_context(
    &mut self,
    dependent_contexts: &[ContextId],
    primary_state_id: &str,
    secondary_state_ids: HashMap<String, String>,
) -> ContextId;
```

and the `Context.state_id` field will be replaced with:

```rust
primary_state_id: String,
secondary_state_ids: HashMap<String, String>,
```

Context’s `get_state` method will take a list of `StorageAddress`es of entries
to get.

Contexts will keep a record of all (`StorageAddress`, value) tuples that were
requested and retrieved from state by the associated transaction. They will
store these values in a new field:

```rust
retrieved_state_entries: Vec<(StorageAddress, Vec<u8>)>,
```

To add values to this field, Contexts will provide a new method that will be
used by the ContextManager:

```rust
fn add_retrieved_state_entry(&mut self, address: StorageAddress, value: &[u8]);
```

## TransactionContext Changes

Like Contexts, TransactionContext’s get methods (`get_state_entry` and
`get_state_entries`) will be modified to take `StorageAddress`es. Their
signatures will become:

```rust
fn get_state_entry(
    &self,
    address: StorageAddress
) -> Result<Option<Vec<u8>>, ContextError>;
fn get_state_entries(
    &self,
    addresses: &[StorageAddress]
) -> Result<Vec<(StorageAddress, Vec<u8>)>, ContextError>;
```

## TransactionReceipt Changes

Like Contexts, TransactionReceipts will store all (database, key, value) tuples
that were requested and retrieved from state by the associated transaction. They
will be represented by a new protobuf message:

```protobuf
message StateEntry {
    string database = 1;
    string key = 2;
    bytes value = 3;
}
```

These state entries will be stored in a new field on the TransactionReceipt:
`repeated StateEntry retrieved_state_entries`. When the TransactionReceipt is
created, any state entries that were retrieved from the primary database will
have a value of “primary” for the database field.

# Drawbacks
[drawbacks]: #drawbacks

- Only one database is writable per ContextManager; if more than one database
  must be writable, multiple ContextManagers must be used, and this solution
  does not provide any assistance with that pattern.

# Rationale and alternatives
[alternatives]: #alternatives

Providing a single writable database rather than multiple writable databases
greatly reduces the complexity of the problem, since it doesn’t require a
mechanism for atomically managing changes to multiple databases. While this is
limiting, it is significantly simpler and we do not currently have strong use
cases that require writing to multiple databases simultaneously.

The `StorageAddress` struct is used to provide a convenient and easy to enforce
way of referring to the various databases. One alternative would be to provide
different methods for getting from the primary database or a secondary database
(i.e. separate `get_from_primary(key)` and `get_from_secondary(database, key)`
methods). However, we deemed a unified addressing scheme easier to use, since it
can be passed throughout the system and only the ContextManager has to have
knowledge of whether the addresses are primary or secondary.

Storing all values that were retrieved from state in Contexts and the resulting
TransactionReceipts facilitates replaying transactions at a later time. To
replay old transactions at a later time without these retrieved values, the
system running the transactions would need access to all databases used at the
time the transaction was originally executed; in some scenarios, these databases
may no longer exist. To remove the requirement that all databases that were ever
used must be available, the retrieved values are simply stored in the
transaction receipt, which means that the receipt itself is all that’s needed to
rerun the transaction.

# Prior art
[prior-art]: #prior-art

None

# Unresolved questions
[unresolved]: #unresolved-questions

- Where will the definition of the StorageAddress live? It is used in both an
  API (TransactionContext) and internally.
- When the ContextManager’s `get` method encounters a database that doesn’t
  exist, should it fail fast and just return the error, or would it be desirable
  to get all addresses that do exist and return a `Vec<Result>`? Is there a use
  case where a TP may try to retrieve a value from some database that's
  optional?
- It was discussed to add support for saving some proof of agreement on the
  input `state_id`s for a Scheduler and, for the purpose of replay-ability,
  saving the input `state_id`s themselves somehow. This could be accomplished
  with a new SchedulerReceipt-type object that would have the proof (arbitrary
    bytes), the `state_id`s, and the list of batch IDs. This could be retrieved
    (only after the schedule is finalized) using a new `get_scheduler_receipt()`
    method on the schedule. However, this could also be accomplished by the
    application that uses transact (it just saves the proof and the `state_id`s
    along with the scheduler and handles the storage structure/mechanisms
    itself). Is it desirable to have this functionality built into transact and
    have opinions about how it should look?
- How do transactions know which databases they have access to and which is the
  primary database? Where is this established and how is it enforced?
