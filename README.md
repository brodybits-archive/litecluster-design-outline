# Litecluster DRAFT design proposal outline

Using SQLite to store application data in a scalable, failure-safe cluster environment

Author: Christopher J. Brody <brodybits@litehelpers.net>

License of this design outline: CC BY-ND 4.0 <http://creativecommons.org/licenses/by-nd/4.0/>

Software license: TBD. Lower-level data storage and distribution layer(s) expected to be distributed under UNLICENSE (a public domain license based on the SQLite licensing statement). Higher-level framework layer(s) expected to be distributed under permissive licensing schemes such as ISC, MIT, and/or Apache 2.0.

## Major requirements

- massively scalable with NO single-node bottlenecks
- data storage safety (ACID)
- system safety: a fried node should not cause any damage (bonus: multiple fried nodes up to a certain proportion should not cause any damage)
- ability to do snapshot read to get all or part of the data accurate at a certain point of time
- failure-safe transactions across multiple records, multiple locations

## Data storage basics

Key-value application data storage (similar to W3 LocalStorage and Redis), enhanced.

For data storage, data is stored with the following fields in a table: RecordKey, FieldKey, FieldClass (classification), FieldValue

Alternative to using FieldClass for field classification: for each storage table, keep a storage info table that keeps field classification and possible description info

Data stored in any field may be high-precision NUMERIC (INTEGER or REAL), TEXT string, binary BLOB, or NULL (as defined by SQLite).

Possible to have multiple key-value data storages by using multiple tables

ACID-compliant transactions possible within the same storage table, among multiple tables within the same (SQLite) database file, and among multiple tables in multiple database files on the same system (node).

Designed to work over SQLite but could be adapted to work over another table-based data storage mechanism.

## Data clustering

Distribution of data based on record key hashing

For the same storage, one (or more) table(s) per node

Data safety: each piece of data is stored in both a primary node and a secondary node. Data is not considered "stored" until there is confirmation that it is saved in both a primary location and a secondary location.

Optional (open): data safety in case multiple nodes are fried (up to a certain proportion)

## Higher-layer SQL library

The basics described should allow a higher-layer SQL library that can interpret a simple SQL subset and emulate table-based data storage using some form of key-value storage. But to be truly SQL-compliant a database system must support arbitrary multi-node transactions as well.

## Snapshot read

Temporary read lock storage, may be across multiple nodes

Each read lock storage record has a GUID, timestamp, state (TBD is explicit state necessary?) and should be considered expired after a certain time

Once a read is finished, the read lock should be marked finished and eventually deleted (TBD maybe just immediately deleted)

For each storage table, need the storage info table with locking info

Possible to either lock the full table or lock a limited number of records for reading

A full table lock would wait for existing record locks to be freed (or expired)

Must specify an order in which records and tables may be locked (ascending key order, for example) to avoid possible deadlock situation

## Multi-node transactions

Temporary transaction (tx) lock storage, may be across multiple nodes, with GUID key, read lock reference, state, and timestamp. A transaction lock record can have states such as ACTIVE (in progress), COMPLETE, ABORTED

A tx lock is always based on a read lock and should be considered ABORTED if it is kept open for too long, the read lock is expired, or the read lock is removed.

For each table keep rollback journal table to store backup of records being changed within ongoing transaction(s)

When starting a multi-node transaction:
- create a read lock
- create a transaction lock record based on the read lock (similar to how SQLite does transaction locking)
- mark the records to be queried and/or changed
- store backup of the records to be changed

Once a transaction is completed:
- set the tx lock state to COMPLETE
- remove the backup records for the transaction from the rollback journal table
- delete the tx lock (TBD alternative: simply delete the tx lock once the transaction is completed)

When reading a record that is marked with a tx lock, first check the state of the tx lock then use the info from the rollback journal table if the tx lock exists and is not COMPLETE.

### Alternative transaction idea

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

## Possible Map-Reduce mechanism

TBD TODO (to combine and present query results in proper format)

## Major TODO(s)

- certain data needed about the distributed database itself (meta data), to be kept in meta data tables
- in case a node is fried, procedure how to introduce a new node and sync it

## Comparison to other data storage libraries and frameworks

TBD TODO

High-integrity, high-reliability libraries/frameworks for comparison: MySQL, Postgres, CockroachDB, MongoDB, MariaDB

Other(s): redis

## FUTURE TBD Other major needs and ideas

- Meteor-like client-server data synchronization
- Same or similar architecture design among client and cluster server layers
- Dynamic updating of client-side applications
- Ability to migrate from another data storage system (free, commercial, or custom-built)
- Ability to migrate to another data storage architecture in the future
- Possibility to access a database storage file from multiple threads and/or multiple processes

