* Command Family
* Start Date: YYYY-MM-DD
* RFC PR: (leave this empty)
* Transact Issue: (leave this empty)

# Summary
[summary]: #summary

The Command Family is a transaction family that can exercise all of the
operations provided on the TransactionContext in addition to other actions, such
as sleeping or marking the transaction as invalid.  This family will be used for
creating a wide variety of tests, both for correctness and performance of the
Transact library.

# Motivation
[motivation]: #motivation

This family will be used for creating a wide variety of tests, both for
correctness and performance.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Command family is a Transact transaction family that provides the following
operations, based on operations defined in its transaction payloads.  These
operations, or commands, cover the following actions:

* Read state entries
* Write state entries
* Add an event to the receipt
* Add arbitrary receipt data
* "Return" InvalidTransaction
* "Return" InternalError
* Sleep for a specified duration

The above operations provide test coverage of all the possible behaviours of a
transaction handler.

Additionally, a Command Family workload tool will be provided to generate a
stream of command family transactions.  The transactions themselves will be
described by a yaml file declaration.  The tool will accept arguments to adjust
the rates, domain of arguments and random seed of a given workload run.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Transaction Payload

The transaction payload for the command family will be as follows:

```protobuf
// A transaction payload for the Command Family
message CommandPayload {
    // The list of commands to execute
    repeated Command commands = 1;
}

// A key-value pair where the value is a bytes field.
message BytesEntry {
    // The key, for instance a state address or a string value, depending on
    // the target of the entry.
    string key = 1;
    // The byte value to be stored
    bytes value = 2;
}

// The list of state keys to be retrieved via a GET_STATE command.
message GetState {
    // a non-empty list of valid state keys
    repeated string state_keys = 1;
}

// The list of state entries to be set via a SET_STATE command.
message SetState {
    // a non-empty list of BytesEntry instances, where the key field is a
    // valid state key.
    repeated BytesEntry state_writes = 1;
}

// The list of state keys to be deleted via a DELETE_STATE command.
message DeleteState {
    // a non-empty list of valid state keys
    repeated string state_keys = 1;
}

// Describes an event to add to the receipt via an ADD_EVENT command.
message AddEvent {
    // The event type
    string type = 1;
    // The attribute entries
    repeated BytesEntry attributes = 2;
    // Arbitrary opaque event data
    bytes data = 3;
}

// The opaque byte data to add to the receipt via an ADD_RECEIPT_DATA
// command. 
message AddReceiptData {
    // a non-empty byte array
    bytes receipt_data = 1;
}

// The duration and method of sleep to apply during a SLEEP command.
message Sleep {
    enum SleepType {
        // This will wait, releasing cpu utilization to other processes 
        WAIT = 0;
        // This will consume a cpu core for the specified time
        BUSY_WAIT = 1;
    }

    // The duration to sleep, in milliseconds
    u32 duration_millis = 1;
    SleepType sleep_type = 2;
}

// The details to return on an RETURN_INVALID command.
message ReturnInvalid {
    // An optional error message
    string error_message = 1;
}

// The details to return on an RETURN_INTERNAL_ERROR command.
message ReturnInternalError {
    // An optional error message
    string error_message = 1;
}

// A command to be executed
message Command {
    enum CommandType {
        UNSET_COMMAND_TYPE = 0;
        GET_STATE = 1;
        SET_STATE = 2;
        DELETE_STATE = 3;
        ADD_EVENT = 4;
        ADD_RECEIPT_DATA = 5;
        SLEEP = 6;
        RETURN_INVALID = 7;
        RETURN_INTERNAL_ERROR = 8;
    }

    // The type of the command, which indicates which of the following 
    // fields will be set.
    CommandType type = 1;

    GetState get_state = 2;
    SetState set_state = 3;
    DeleteState delete_state = 4;
    AddEvent add_event = 5;
    AddReceiptData add_receipt_data = 6;
    Sleep sleep = 7;
    ReturnInvalid return_invalid = 8;
    ReturnInternalError return_internal_error = 9;
}
```

## Command Execution

A Command Family transaction consists of an ordered list of Commands. The
commands are executed in order and only if the preceding command was successful.
Commands which return an explicit result (invalid or internal error) will
short-circuit the remaining commands in the list.

Each command performs a single operation, from the perspective of
TransactionContext, or from the result returned.  The sleep command is currently
an exception, in that the transaction handler will pause for the specified
duration before proceeding to the next command, if any.

### Get State

The `GET_STATE` command will use the `state_keys` field of the `GetState`
message to request a read of the addresses specified.  This will make use of the
`TransactionContext`'s `get_state_entries` method. In other words, it will only
make a single call to the context. 

Each entry in the `state_keys` field must be a valid state key (for example, a
valid address in a Sawtooth merkle trie). 

Note, no validation will occur if the requested state value is actually
returned, nor what the entry should or shouldn't contain.

### Set State

The `SET_STATE` command will use the `state_writes` field of the `SetState`
message to request a write of the bytes entries specified.  This will make use
of the `TransactionContext`'s `set_state_entries` method, where the keys and
values are specified by `BytesEntry` messages. In other words, it will only make
a single call to the context.

The key in the `BytesEntry` must be a valid state key (for example, a valid
address in a Sawtooth merkle trie). 

### Delete State

The `DELETE_STATE` command will use the `state_keys` field of the `DeleteState`
message to request a read of the addresses specified.  This will make use of the
`TransactionContext`'s `delete_state_entries` method. In other words, it will
only make a single call to the context. 

Each entry in the `state_keys` field must be a valid state key (for example, a
valid address in a Sawtooth merkle trie). 

### Add Event

The `ADD_EVENT` command will use the event field of the `AddEvent` message to
add an event to the transaction receipt.  This will make use of the
`TransactionContext`'s `add_event` method, using the contents of the `Event`
message contained in the event field to set the matching parameters.

The Event type field must not be empty. The remaining fields, attributes and
data, may be empty.

### Add Receipt Data

The `ADD_RECEIPT_DATA` command will use the `receipt_data` field of the
`AddReceiptData` message to add this data to the transaction receipt.  This will
make use of the `TransactionContext`'s `add_receipt_data` method.

The `receipt_data` field must not be empty.

### Sleep

The `SLEEP` command will use the `duration_millis` field of the `Sleep` message
in order to delay the execution of the transaction.  This field is a millisecond
value.  The `SleepType` indicates whether the transaction handler should
busy-wait (i.e. consume CPU resources) or standard wait.

The provided sleep duration must not be zero.

### Return Invalid Transaction

The `RETURN_INVALID` command will force the return of an `InvalidTransaction`
result from the transaction handler.  It will use the `error_message` field in
the ReturnInvalid message to provide arbitrary feedback. Any remaining commands
will be ignored.

The `error_message` field may be empty.

### Return Internal Error

The `RETURN_INTERNAL_ERROR` command will force the return of an `InternalError`
result from the transaction handler.  It will use the `error_message` field in
the `ReturnInternalError` message to provide arbitrary feedback. Any remaining
commands will be ignored.

The `error_message` field may be empty.

## Transaction Header

The transaction header is a standard header in that it will include the family
name and the family version as normal.

family_name: "command"
family_version: "0.1"

In a Sawtooth context, there is an additional field for namespaces. For the
command family, the namespace should be "any" (i.e. `*`).  The command family
should be allowed to read and write any value from/to state.  

Alternatively, the namespace should be set to the first 6 char of the SHA-512
hash of the family name, "command".  Address values used in `GET`-, `SET`- or
`DELETE_STATE` commands would have to be required to be under that namespace.

## Workload Generator

The workload generator for the command family will generate a stream of batches,
consisting of one or more command transactions.  These transactions generated by
the stream will be described declaratively using a yaml file.

The declarations should describe the commands to be used, with ranges of values
that may be generated.

# Drawbacks
[drawbacks]: #drawbacks

None.

# Rationale and alternatives
[alternatives]: #alternatives

This family is designed to improve the overall testing of the Transact library.
It provides coverage of the operations available to a transaction handler and
therefore improves overall testing of the Transact library.

Alternatively, individual transaction families could be created for specific
test circumstances, using the transaction order as a way to implement operation
order.  This approach has the side-effect of operations potentially occurring in
an unpredictable order, depending on the scheduler implementation.  Likewise, it
would not allow for testing multiple operations before failing the transaction.

# Prior art
[prior-art]: #prior-art

The Command family builds on existing Sawtooth example and test-related
transaction families, such as IntKey and Smallbank.  Both of these exercise
subsets of the operations on the TransactionContext, but not in a broad sense.
Likewise, the workload generation for these families is mainly focused on batch
throughput, and not focused tests on the effects of certain operations.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
