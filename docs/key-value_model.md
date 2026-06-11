# Key-Value Store Model 

## Main idea

# Key–Value Store Model

## Main Idea

miniKV is an in-memory database that stores values under unique keys.

Conceptually, the database is a mapping: [ D : K -> V ]

where:
* (K) is the set of possible keys.
* (V) is the set of possible values.
* A key may be absent, which is why the mapping is partial.
* Each existing key has exactly one current value.

This document describes **what miniKV does**, not how it is implemented.

## Basic Operations

### `SET key value`

Stores a value under a key.

* If the key does not exist, it is inserted.
* If the key already exists, its old value is replaced.

Example:

```text
SET name Ariel
SET name Daniel
```

The current value of `name` is now `Daniel`.

### `GET key`

Returns the value associated with a key.

* If the key exists, return its value.
* If the key does not exist, return `NOT_FOUND`.
* The database state does not change.

### `DEL key`

Removes a key and its value.

* Return `1` if the key existed and was removed.
* Return `0` if the key was already absent.

### `EXISTS key`

Checks whether a key currently exists.

* Return `1` if it exists.
* Return `0` otherwise.
* The database state does not change.

### `SIZE`

Returns the number of keys currently stored.

Replacing the value of an existing key does not increase the size.

### `PING`

Returns `PONG` to verify that the server is available.

`PING` belongs to the server command layer and does not access the key–value database itself.

## Important Behavioral Rules

1. A key has at most one current value.
2. `SET` replaces the previous value of an existing key.
3. `GET`, `EXISTS`, and `SIZE` do not modify the database.
4. `DEL` makes the key absent.
5. An empty value is different from a missing key.
6. `SIZE` counts existing keys, including keys containing empty values.

## Binary-Safe Data

Keys and values should be treated as sequences of bytes with explicit lengths, not only as null-terminated C strings.

Conceptually:

```text
data pointer + data length
```

This allows miniKV to store:

* Text
* Empty values
* Embedded null bytes
* Serialized objects
* Arbitrary binary data

## Architectural Consequence

The key–value engine should be independent of networking and command parsing.

```text
Client
  ↓
Network protocol
  ↓
Command processor
  ↓
Key–value engine
  ↓
Database state
```

The engine should behave correctly whether it is called by:

* A network server
* A local test
* Another C module

## Summary

miniKV represents a collection of unique keys and their current values.

Its initial behavior is defined by:

```text
SET
GET
DEL
EXISTS
SIZE
```

The purpose of this model is to establish a clear contract before choosing the internal data structures and implementation.
