# miniKV Core Hashing Strategy

This document summarizes the planned hashing strategies for the core miniKV engine.

Scope:

- Core database logic only
- No TCP
- No client/server
- No protocol
- No persistence yet

miniKV treats the hash-table index as two possible approaches:

```text
Core miniKV Engine
├── Dynamic approach
│   └── Universal hashing + chaining + resizing
│
└── Static approach
    └── Perfect hashing
```

---

## 1. Dynamic Approach

Use this when the key set $S$ is **not known in advance**.

This is the normal Redis-like miniKV mode.

### Idea

The database changes over time:

```text
SET a 1
SET b 2
DEL a
SET c 3
```

So the active key set is dynamic:

$$
S_0, S_1, S_2, \ldots
$$

We use:

```text
Universal hashing + chaining + resizing
```

---

### General Algorithm

At any moment:

$$
N = |S|
$$

$$
m = \text{number of buckets}
$$

$$
\alpha = \frac{N}{m}
$$

The table stores a hash function:

$$
h \in \mathcal{H}_m
$$

where:

$$
h : U \to \{0, \ldots, m - 1\}
$$

The family depends on the table size:

$$
\mathcal{H}_m
$$

When $m$ changes, miniKV chooses a new hash function from the new matching family.

---

### `SET(k, v)`

```text
i = h(k)

search bucket[i]

if k exists:
    replace value

else:
    insert new entry into bucket[i]
    N++

if alpha > threshold:
    resize m
    choose new h from H_m
    rehash all entries
```

---

### `GET(k)`

```text
i = h(k)
search bucket[i]
return value if found
```

---

### `DEL(k)`

```text
i = h(k)
search bucket[i]

if found:
    remove entry
    N--
```

---

## Implementation Suggestion

Use:

```text
bucket array
    |
    v
linked chains of entries
```

Each entry stores, conceptually:

```text
key bytes
value bytes
next pointer
```

The store should own the key/value memory.

Hashing should be binary-safe:

```text
hash(key bytes, key length)
```

When resizing:

```text
m changes
|
v
choose new h from H_m
|
v
rehash all keys
```

Reason:

$$
h : U \to \{0, \ldots, m - 1\}
$$

If $m$ changes, the image of the hash function changes too.

---

## Runtime

For chaining with universal hashing:

$$
E[\text{operation time}] = O(1 + \alpha)
$$

where:

$$
\alpha = \frac{N}{m}
$$

So the correct base result is:

```text
Expected runtime = O(1 + alpha)
```

It becomes expected $O(1)$ only if miniKV keeps:

$$
\alpha = O(1)
$$

by resizing.

| Operation | Expected runtime |
|---|---:|
| `GET` | $O(1 + \alpha)$ |
| `SET` without resize | $O(1 + \alpha)$ |
| `DEL` | $O(1 + \alpha)$ |
| `SIZE` | $O(1)$ |

If $\alpha$ is bounded by a constant:

$$
O(1 + \alpha) = O(1)
$$

---

## Space

Space is:

$$
\Theta(m + N + \text{total key bytes} + \text{total value bytes})
$$

If miniKV keeps:

$$
m = \Theta(N)
$$

then structural overhead is:

$$
\Theta(N)
$$

---

## Worst Case

Worst case is still:

$$
O(N)
$$

Example:

```text
all keys land in one bucket
```

Then one chain contains almost all keys.

Universal hashing gives expected guarantees, not absolute worst-case guarantees.

---

## How miniKV Handles the Worst Case

Current plan:

```text
if alpha too high:
    resize

if one chain is too long while alpha is fine:
    choose new universal hash function
    rebuild table
```

Meaning:

```text
Too many keys globally -> resize m
Bad local distribution -> choose new h
```

---

## Future Improvements

Later we can handle bad chains by:

- converting long chains into balanced trees,
- using open addressing as an alternative backend,
- using stronger randomized/seeded hashing,
- adding collision statistics,
- adding automatic rebuild policies.

---

# 2. Static Approach

Use this when the key set $S$ is **known in advance** and does not change.

This is not the default Redis-like mode. It is a second option.

---

## Idea

If we know all keys before building the table, we can create a collision-free structure.

This is called:

```text
perfect hashing
```

Good examples:

```text
reserved keywords
fixed command names
frozen database snapshot
```

---

## General Algorithm

Given fixed set:

$$
S = \{k_1, \ldots, k_N\}
$$

Build a two-level table.

---

### Level 1

Choose primary hash function:

$$
h
$$

Place keys into primary buckets:

```text
bucket i contains all keys where h(k) = i
```

---

### Level 2

For each bucket $i$, suppose it contains:

$$
n_i
$$

keys.

Create a secondary table of size roughly:

$$
m_i = n_i^2
$$

Choose a secondary hash function:

$$
h_i
$$

until no collisions happen inside that secondary table.

Lookup:

```text
primary index = h(k)
secondary index = h_i(k)
check exact key
```

---

## Implementation Suggestion

Have two possible index strategies:

```text
DynamicHashIndex
PerfectHashIndex
```

Dynamic index supports mutation.

Perfect index is for frozen/static data.

The static version should mainly support:

```text
GET
EXISTS
SIZE
```

It should not naturally support:

```text
SET new key
DEL key
```

unless the whole perfect hash index is rebuilt.

---

## Runtime

For a successfully built perfect hash table:

| Operation | Runtime |
|---|---:|
| `GET` | worst-case $O(1)$ |
| `EXISTS` | worst-case $O(1)$ |
| `SIZE` | $O(1)$ |
| `SET new key` | not supported / rebuild |
| `DEL` | not supported / rebuild |

The benefit is stronger lookup performance:

```text
Dynamic hashing: expected O(1 + alpha)
Perfect hashing: worst-case O(1) lookup
```

---

## Space

With the standard two-level scheme, expected total space is:

$$
O(N)
$$

But an individual secondary table may use:

$$
n_i^2
$$

space for bucket $i$.

The theory works because, with a good primary universal hash function, the expected sum of secondary table sizes stays linear.

---

## Worst Case

The construction may choose bad hash functions.

Possible problems:

```text
too many collisions
secondary table not collision-free
space too large
```

---

## How miniKV Handles It

For static mode:

```text
try random hash functions
measure collisions / space
retry until acceptable
```

Since $S$ is known and fixed, retrying is possible.

---

## Future Improvements

Later we can add:

- frozen snapshots using perfect hashing,
- read-only optimized tables,
- minimal perfect hashing,
- compressed static indexes,
- hybrid dynamic + frozen architecture.

---

# Final Design Conclusion

miniKV core should support two conceptual paths.

## Main Path

```text
Dynamic key set
-> universal hashing
-> separate chaining
-> resizing by load factor
-> expected O(1 + alpha)
-> expected O(1) only when alpha is bounded
```

## Second Path

```text
Static known key set
-> perfect hashing
-> no collisions after construction
-> worst-case O(1) lookup
-> mutation requires rebuild
```

The main Redis-like miniKV engine uses the **dynamic approach**.

The **static approach** is an advanced optional index for frozen data.
