- Feature Name: state traits v2
- Start Date:
- RFC PR:
- Transact Issue:

# Summary
[summary]: #summary

This RFC defines new versions of the state traits which are more flexible and
with more consistent associated types.

# Motivation
[motivation]: #motivation

The existing state traits are too restrictive for implementers, as they require
the structs to be `Sync` and `Send`. They also combine several seemingly related
operations that should be separated. Finally, there is a disconnect between
`StateChange` and the other `Key`/`Value` associated types.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Current Traits

The existing transact state traits provide the majority of the functionality
needed to operate on a state system. It does have several drawbacks for
implementers:

The traits have strict required traits. Namely, each state trait must be `Sync`
and `Send`.  This prevents state traits from being implemented over
non-thread-safe items, such as mutable references or, in the case of SQL,
individual database connections.  For the latter, this cannot be mitigated with
an `Arc`-`Mutex` combo, as the diesel connections (currently used in transact)
are not Send.

The trait `transact::state::Read` includes an additional requirement with its
`clone_box` function.  This is used in conjunction with an implementation of
`Clone` over `Box<dyn Read>`.  It is a common pattern to solve implementing
`Clone` on boxed traits, though is fairly inflexible on a what traits the boxed
item can have.

Each trait is allowed to define their own state checkpoint IDs and key/value
types, but they cannot change the types in the `StateChange` struct.  This
struct defines keys and values to be `String` and `Vec<u8>`, respectively. It
also limits the type of operations that may be allowed, namely "set" and
"delete".

The `transact::state::Write` trait combines two operations into a single trait:
commit and computing the next state ID.  While commit makes sense (it is
expected to actually write to state), computing the next state ID doesn't
necessarily require the caller to have write access.  It is essentially a
preview operation.

Likewise, iterating over any sort of ordered state is left to implementers.  For
example, listing leaves of merkle-radix state is accomplished via the trait
`transact::state::merkle::MerkleRadixLeafReader`.  Any other state
implementations require their own version of an iterative reader.

## State Traits V2

The new traits introduced will have new names, so as to provide opt-in
behaviour, as well as to maintain backwards-compatibility with existing
implementations.

Firstly the new traits deal with the trait bounds issue by removing the `Sync`
and `Send` requirements for all traits.  It also removes the use of any form of
`clone_box` from all traits.  Any requirement for `Sync`, `Send` or `Clone`
should be made via generic bounds.  For example:

```rust
#[derive(Clone)]
struct MyThreadSafeStateWrapper<R>
where R: transact::state::Reader + Clone + Sync + Send
{
    reader: R,
    ...
}
```

All the functions in the traits will also return the same `StateError`, which
composes the standard errors found in the `transact::error` module.

### Core Traits

These traits solve the type consistency problem in several ways. In order to
enforce that the operational traits operate on the same types, they all require
the implementation of a `State` trait.  This trait includes the type definitions
for the state checkpoint ID (`StateId`), key (`Key`), and value (`Value`).  It
does not specify any specific operations on its own.

As mentioned above the `Write` trait's combined functionality has been divided
into two traits: `Committer` and `DryRunCommitter`.  The former provides the
`commit` capability and the latter provides the `dry_run_commit` functionality
(replacing `compute_state_id` from `Write`).   The definition of a state change
(`StateChange`) is also an associated type on both traits, allowing that value
to be defined by the implementor.

The remaining core trait is `Reader`, which maintains the existing `get`
function, but loses the `clone_box` function. It also includes the ability to
iterate over values with an optional filter value (`Filter`). This replaces the
need for the separate trait specific to merkle-radix state (Merkle

### Optional Trait

The current `Prune` trait is considered optional, as its replacement, `Pruner`.
Its functionality is equivalent, with its main change being not defining its own
associated types and returning the new `StateError`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section includes the traits in detail.

## Core Traits

`State` is the core definition of a state system's keys, values and state
checkpoint IDs:

```rust
/// Defines the characteristics of a given state system.
///
/// A State system stores keys and values.  The keys and values may be made
/// available via state IDs, which define a checkpoint in the state. Keys that
/// exist under one state ID are not required to be available under another.
/// How this is handled is left up to the implementation.
///
/// For example, a `State` defined over a merkle database would proved the
/// root merkle hash as its state ID.
///
/// Available with the feature `"state-trait"` enabled.
pub trait State {
    /// A reference to a checkpoint in state. It could be a merkle hash for a
    /// merkle database.
    type StateId;
    /// The Key that is being stored in state.
    type Key;
    /// The Value that is being stored in state.
    type Value;
}
```

`Reader` provides the capability to read state values:

```rust
/// Provides a way to retrieve state values from a particular storage system.
///
/// This trait provides similar behaviour to the [`Read`](super::Read) trait,
/// without the explicit requirements about thread safety.
///
/// Available with the feature `"state-trait-reader"` enabled.
pub trait Reader: State {
    /// The filter used for the iterating over state values.
    type Filter: ?Sized;

    /// At a given `StateId`, attempt to retrieve the given slice of keys.
    ///
    /// The results of the get will be returned in a `HashMap`.  Only keys
    /// that were found will be in this map. Keys missing from the map can be
    /// assumed to be missing from the underlying  storage system as well.
    ///
    /// # Errors
    ///
    /// [`StateError`] is returned if any issues occur while trying to fetch
    /// the values.
    fn get(
        &self,
        state_id: &Self::StateId,
        keys: &[Self::Key],
    ) -> Result<HashMap<Self::Key, Self::Value>, StateError>;

    /// Returns an iterator over the values of state.
    ///
    /// By providing an optional filter, the caller can limit the iteration
    /// over a subset of the values.
    ///
    /// The values are returned in their natural order.
    ///
    /// # Errors
    ///
    /// [`StateError`] is returned if any issues occur while trying to query
    /// the values. Additionally, a [`StateError`] may be returned on each
    /// call to `next` on the resulting iteration.
    fn filter_iter(
        &self,
        state_id: &Self::StateId,
        filter: Option<&Self::Filter>,
    ) -> ValueIterResult<ValueIter<(Self::Key, Self::Value)>>;
}
```

`ValueIterResult` and `ValueIter` are the following type definitions:

```rust
pub type ValueIterResult<T> = Result<T, StateError>;
pub type ValueIter<T> = Box<dyn Iterator<Item = ValueIterResult<T>>>;
```

`Committer` provides the capability to commit state changes, producing a new
state checkpoint ID:

```rust
/// Provides a way to commit changes to a particular state storage system.
///
/// A `StateId`, in the context of `Committer`, is used to indicate the
/// starting state on which the changes will be applied.  It can be thought of
/// as the identifier of a checkpoint or snapshot.
///
/// All operations are made using `StateChange` instances. These are an
/// ordered set of changes to be applied onto the given `StateId`.
///
/// Available with the feature `"state-trait-committer"` enabled.
pub trait Committer: State {
    /// Defines the type of change to apply
    type StateChange;

    /// Given a `StateId` and a slice of `StateChange` values, persist the
    /// state changes and return the resulting next `StateId` value.
    ///
    /// This function will persist the state values to the underlying storage
    /// mechanism.
    ///
    /// # Errors
    ///
    /// [`StateError`] is returned if any issues occur while trying to commit
    /// the changes.
    fn commit(
        &self,
        state_id: &Self::StateId,
        state_changes: &[Self::StateChange],
    ) -> Result<Self::StateId, StateError>;
}
```

`DryRunCommitter` provides the capability to preview a new state checkpoint ID
based on set of state changes:

```rust
/// Predicts future state checkpoints.
///
/// Available with the feature `"state-trait-dry-run-committer"` enabled.
pub trait DryRunCommitter: State {
    /// Defines the type of change to use for the prediction.
    type StateChange;

    /// Given a `StateId` and a slice of `StateChange` values, compute the
    /// next `StateId` value.
    ///
    /// This function will compute the value of the next `StateId` without
    /// actually persisting the state changes.
    ///
    /// Returns the next `StateId` value;
    ///
    /// # Errors
    ///
    /// [`StateError`] is returned if any issues occur while trying to
    /// generate this next id.
    fn dry_run_commit(
        &self,
        state_id: &Self::StateId,
        state_changes: &[Self::StateChange],
    ) -> Result<Self::StateId, StateError>;
}
```

## Optional Trait

`Pruner` provides an optional way to prune unnecessary state checkpoints:

```rust
/// Provides a way to remove no-longer needed state data from a particular
/// state storage system.
///
/// Removing `StateIds` and the associated state makes it so the state storage
/// system does not grow unbounded.
///
/// Available with the feature `"state-trait-pruner"` enabled.
pub trait Pruner: State {
    /// Prune keys from state for a given set of state IDs.
    ///
    /// In storage mechanisms that have a concept of `StateId` ordering, this
    /// function should provide the functionality to prune older state values.
    ///
    /// It can be considered a clean-up or space-saving mechanism.
    ///
    /// It returns the keys that have been removed from state, if any.
    ///
    /// # Errors
    ///
    /// [`StateError`] is returned if any issues occur while trying to prune
    /// past results.
    fn prune(&self, state_ids: Vec<Self::StateId>)
        -> Result<Vec<Self::Key>, StateError>;
}
```

## Errors

All of the functions return the error `transact::state::StateError`, which is an
enum that makes use of transact's common errors (found in the `transact::error`
module).

```rust
#[derive(Debug)]
pub enum StateError {
    Internal(transact::error::InternalError),
    InvalidState(transact::error::InvalidStateError),
}
```

These variants cover the current variants in the existing state errors, without
the unnecessary specificity.

# Drawbacks
[drawbacks]: #drawbacks

This RFC introduces a parallel set of traits, which requires parallel usages or
implementations for the same set of functionality While this does introduce a
new set of traits, if accepted, the previous traits may be deprecated and
removed in future releases.

# Rationale and alternatives
[alternatives]: #alternatives

Alternatively, the trait bounds for `Sync` and `Send` could be removed from the
existing traits, though this would break existing implementations, as well as
require a substantial rewrite of the `ContextManager`, which requires both
bounds as well as the ability to clone a `Box<dyn Read>` implementation.

The introduction of new traits allows for a gradual introduction in other parts
of the transact library.

# Prior art
[prior-art]: #prior-art

The existing traits in `transact::state` provide similar or identical function
signatures.

# Unresolved questions
[unresolved]: #unresolved-questions

- Which traits, if any, can be auto-implemented for the existing traits?
