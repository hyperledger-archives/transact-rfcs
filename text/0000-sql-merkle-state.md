- Feature Name: sql-merkle-state
- Start Date: 2021-06-14
- RFC PR:
- Transact Issue:

# Summary
[summary]: #summary

This RFC presents a method for improving performance of `MerkleState` when
backed by a relational database through the use of recursive queries and
indexes implemented with slowly changing dimensions.

# Motivation
[motivation]: #motivation

The current implementation of relational-database-backed state storage uses the
Database abstraction. This abstraction expects every underlying store to act as
a basic key-value store.  In the case of a RDBMS, this is not an efficient use
of their capabilities.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The current Database abstraction acts as a simple key-value store, where both
the keys and the values are arbitrary bytes.  This does not allow for various
optimizations that can be made via more complicated SQL queries. This RFC
proposes several tables which will accurately reflect the data and allow for
those optimizations, where possible.

One such operation is made by representing the merkle radix tree nodes with an
array type for the children.  This allows for the use of a recursive query to
find the complete node path of a given address/data pair.

The second optimization is through the use of a custom index table, where the
leaves the tree are referenceable by state root hash via a slowly-changing
dimensions style table.  Each state root is assigned an order, from parent to
child.  Each leaf then has a snapshot that is valid for a given state root hash.

Insertion, while still requiring the same number of records as the key-value
abstraction, can be done with bulk executions, thereby limiting the round trips
to and from the database.

## Trait Extraction

In order to provide interchangeable use of either the existing Database
abstraction-backed `MerkleState` implementation or the new SQL-backed
`MerkleState`, a new trait will need to be added. This trait will provide ways
to iterate over the leaves of the tree. The existing State traits already cover
the remaining functionality.

This new trait would be used by such API's as Sawtooth's list state client
operation or the Scabbard Splinter service's state REST API endpoint.

## Code Organization

As this RFC will result in multiple merkle implementations, the existing
implementation should be pushed down into a sub-module, and a new submodule
should contain the SQL-database-backed model

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following tables will be used for storage:

```sql
CREATE TABLE IF NOT EXISTS merkle_radix_leaf (
    id INTEGER PRIMARY KEY,
    address STRING NOT NULL,
    data BLOB
)

-- The representation of the tree relationships between hashes and address parts
CREATE TABLE IF NOT EXISTS merkle_radix_tree_node
    hash STRING PRIMARY KEY,
    leaf_id INTEGER,
    children string[] not null, -- and size 256, if that is specifiable.
    FOREIGN KEY(leaf_id) REFERENCES leaf(id),
)

-- An ordering table for state root hashes
CREATE TABLE IF NOT EXISTS merkle_radix_state_root (
    id INTEGER PRIMARY KEY,
    state_root STRING NOT NULL,
    parent_state_root STRING NOT NULL
    FOREIGN KEY(state_root) REFERENCES merkle_radix_tree_node (hash)
)

-- An "index" for the relationship between a state root hash and
-- the leaf nodes under that root.  This uses type 2 slowly changing dimensions
-- to manage which values of a leaf are associated with a state root.
CREATE TABLE IF NOT EXISTS merkle_radix_state_root_leaf_index (
    id INTEGER PRIMARY KEY,
    leaf_id INTEGER NOT NULL,
    from_state_root_id INTEGER NOT NULL,
    to_state_root_id INTEGER,
    FOREIGN KEY(from_state_root_id) REFERENCES merkle_radix_state_root(id),
    FOREIGN KEY(leaf_id) REFERENCES merkle_radix_leaf (id)
)
```

The `merkle_radix_leaf` table contains the data at the complete address.

The `merkle_radix_tree_node` table provides the parent-child relationships, based on
the hashes.  As an address in the merkle radix tree is made up of 70 hex
characters and each node has a branch factor of 256, the max depth of this tree
is 35. The portion of the address for a given depth is the index into the
children array.  This is equivalent behaviour to the current key-value
implementation.

Note that in Sqlite, the children array will require the json extension.

The change log, used for state pruning Initially described in Sawtooth RFC
[#8](https://github.com/hyperledger/sawtooth-rfcs/pull/8) can be simply
translated into SQL tables from the current `ChangeLogEntry` protobuf record
defined in `libtransact/protos/merkle.proto`.

## Inserting Records

Like the existing merkle tree implementation, inserts are the most complicated
operation. This is made slightly more difficult then the key-value
implementation as the hashes are generated from constructed values - the bytes
are not stored as part of the `merkle_radix_tree_node` records in the database.

We insert the leaf record first, in order to obtain its row id:

```sql
INSERT INTO leaf (address, data)
VALUES ($ADDRESS, $DATA)
```

We select the path from the tree using a recursive query.  In order to do this,
we need to do a transformation of the address into an array of indexes. An
address is a 70 character hex string, representing 35 bytes, and each byte is
the index into a node's children array. We can convert the address into an array
bytes, which we can then use in our query.

```sql
WITH RECURSIVE tree_path AS
(
    -- This is the initial node
    SELECT hash, leaf_id, children, 0 as depth
    FROM merkle_radix_tree_node
    WHERE hash = $STATE_ROOT_HASH

    UNION ALL

    -- Recurse through the tree
    SELECT c.hash, c.leaf_id, c.children, p.depth + 1
    FROM merkle_radix_tree_node c, tree_path p
    WHERE c.hash = p.children[$ADDRESS[p.depth + 1] + 1] -- 1-indexed arrays
)
select * from tree_path
```

In the SQLite case, we again will have to make use of the json1 extension to use
the arrays.

We can modify these values using the same method as the key-value
implementation.

## Update the State Root and Index

The state root table and the index provide an optimization for looking up or
listing values in the tree by state root hash.

First, we need to assign a new ordering ID to the new state root hash.  Using
the previous state root hash, we check to see if another state root has been
inserted at that position in the order. If one exists, we are creating a new
branch and need to remove the previous branch.

We do the following to remove the old branch:

```sql
-- Restore any deletions
UPDATE merkle_radix_state_root_leaf_index
SET to_state_root_id = NULL,
WHERE to_state_root_id >= (
    SELECT id FROM merkle_radix_state_root
    WHERE parent_state_root = $PREVIOUS_STATE_ROOT);

-- Delete any additions
DELETE FROM merkle_radix_state_root_leaf_index
WHERE from_state_root_id >= (
    SELECT id FROM merkle_radix_state_root
    WHERE parent_state_root = $PREVIOUS_STATE_ROOT);

-- Delete the ordering record for the branch state root hash
DELETE FROM merkle_radix_state_root
WHERE parent_state_root = $PREVIOUS_STATE_ROOT;
```

The new root and updated leaf entries can now be safely written to the index.

Add the new state root hash to the ordering table:

```sql
INSERT INTO merkle_radix_state_root
    (state_root, parent_state_root)
VALUES
    ($STATE_ROOT, $PREVIOUS_STATE_ROOT)
```

Using the ordering ID from the above insert, update or insert the values into
index table:

First, mark leaves that have been changed, either inserted, updated or deleted:

```sql
UPDATE merkle_radix_state_root_leaf_index
SET to_state_root_id = $STATE_ROOT_ID
WHERE to_state_root_id = NULL and leaf_id = (
    SELECT id FROM merkle_radix_leaf.id = $ADDRESS)
```

Finally, insert a new record for each new or updated leaf:

```sql
INSERT INTO merkle_radix_state_root_leaf_index
     (leaf_id, from_state_root_id)
VALUES
     ($LEAF_ID, $STATE_ROOT_ID)
```

## Querying via State Root Hash

Querying for a specific address under a given state root hash is now a simple
query, using the `merkle_radix_state_root_leaf_index` table.

```sql
SELECT DISTINCT ON (l.address) l.address, l.data, i.to_state_root
FROM merkle_radix_leaf l,
     merkle_radix_state_root_leaf_index i,
     merkle_radix_state_root s
WHERE s.state_root = $STATE_ROOT_HASH
  AND i.from_state_root <= s.id
  AND (i.to_state_root IS NULL OR i.to_state_root > s.idf)
  AND l.id = i.leaf_id
  AND l.address = $ADDRESS
ORDER BY t.address, i.to_state_root desc NULLS FIRST
```

A similar query can be made to search for subtrees using a similar query to the
above by changing  `leaf.address = $ADDRESS` to `leaf.address LIKE $PREFIX`
where the `PREFIX`, The full tree's leaves under a given state root hash can be
queried by removing the address.

In the case of a branch that is not in the index, the leaves can be queried via
the `merkle_radix_tree_node` table.  Individual leaves can be queried in a
single recursive query.  The listing of leaves will require tree walking, which
will have similar performance to the a key-value store implemented over SQL.

## MerkleRadixLeafReader trait

The `MerkleRadixLeafReader` trait will provide the ability to iterate over the
leaves of the merkle tree.  In order to support the uses of `MerkleRadixTree`'s
existing method, it should be as follows:

```rust
type IterResult<T> = Result<T, InternalError>;

trait MerkleRadixLeafReader {
    /// Returns an iterator over the leaves of a merkle radix tree.
    /// By providing an optional address prefix, the caller can limit the iteration
    /// over the leaves in a specific subtree.
    fn leaves(&self, state_id: &Self::StateId, subtree: Option<&str>)
        ->
        Result<
            Box<dyn Iterator<Item=IterResult<(Self::Key, Self::Value)>>>,
            InternalError,
        >;
}
```

This could be extended to provide paging, though this is left to future RFCs.

## Modules and Structs

The existing merkle radix tree implementation should be moved to the module
`transact::state::merkle::kv`. It will keep its name for backwards-compatibility
purposes.

The SQL-backed implementation will exist in the module
`transact::state::merkle::sql`. The state struct will be

```rust
pub struct SqlMerkleState {
    …
}
```

and include implementations of the same set of traits - `Read`, `Write`,
`Prune`, and `MerkleRadixLeafReader` - as the existing implementation.

# Drawbacks
[drawbacks]: #drawbacks

The main drawback is the addition of the index table, to provide a relationship
between a given state root hash and the leaf data. This must be copied and
updated on each commit.

# Rationale and alternatives
[alternatives]: #alternatives

The current use of the `Database` abstraction is an alternative. Attempts could
be made to optimize treating a SQL storage system as a key-value store, but any
optimizations that could be made across the whole tree are lost, as the
abstraction has now knowledge of the tree structure itself.

# Prior art
[prior-art]: #prior-art

The existing Transact merkle tree source code.

# Unresolved questions
[unresolved]: #unresolved-questions

Performance comparisons will still need to be made between the resulting
`SqlMerkleState` implementation and the existing implementation with all of the
current key-value database implementations..
