# Litecluster DRAFT design outline

Using SQLite to store application data in a scalable, failure-safe cluster environment

Author: Christopher J. Brody <brodybits@litehelpers.net>

License of this design outline: CC BY-ND 4.0 <http://creativecommons.org/licenses/by-nd/4.0/>

Software license: TBD. Lower-level data storage and distribution layer(s) expected to be distributed under UNLICENSE (a public domain license based on the SQLite licensing statement). Higher-level framework layer(s) expected to be distributed under permissive licensing schemes such as ISC, MIT, and/or Apache 2.0.

## Basics

Key-value application data storage, similar to W3 LocalStorage and Redis.

Clustering: distribution of data based on key hashing

Data safety: each piece of data is stored in both a primary node and a secondary node. Data is not considered "stored" until there is confirmation that it is saved in both a primary location and a secondary location.

Optional (open): data safety in case multiple nodes are fried (up to a certain proportion)

Mandatory value data types: high-precision integer and floating-point numbers, text strings, binary BLOBs

Optional (under consideration): Ability to store multiple key-value pairs for each key. In SQLite this could be achieved by having multiple columns, for example: RecordKey, RecordSubKey, RecordSubValue

SQLite provides an ACID data storage mechanism that would apply to individual key-value pairs.

## Transactions

The basics described should allow a higher-layer SQL library that can interpret a simple SQL subset and emulate table-based data storage using some form of key-value storage. But to be truly SQL-compliant a database system must support arbitrary multi-record transactions as well.

Proposed change transaction idea:

- Use WAL (write-ahead logging) technique: for each change write a new record that marks the key and what the change is. Old change records could be deleted upon confirmation that they are no longer needed. This does NOT require SQLite to be used with WAL logging enabled.
- Special key-value tables to keep track of ongoing changes (may be distributed among multiple nodes to be scalable)
- Before starting a change transaction, create a new transaction record that marks the transaction with an in-progress status
- Change records are added with transaction record references
- When a change transaction is considered complete:
  - Update the transaction record state to indicate that that the change is complete
  - Clear the transaction record reference from the change records
  - Then it is possible to delete the transaction record.
- A transaction may be aborted, in which case the change records should be deleted before the transaction record may be deleted.
- When reading data (by key for exampe), for each active change status reference, check the change status record and ignore the change if it is marked "in-progress"

## Map-Reduce mechanism

TBD TODO

## Dynamic updating of client-side applications

TBD TODO

## Comparison to other data storage libraries and frameworks

TBD TODO

## FUTURE TBD Other major needs and ideas

- Meteor-like client-server data synchronization
- Same or similar architecture design among client and cluster server layers
- Ability to migrate from another data storage system (free, commercial, or custom-built)
- Ability to migrate to another data storage architecture in the future

