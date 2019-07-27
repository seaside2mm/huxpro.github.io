# Hash Tables

DATA STRUCTURES
Internal Meta-data
Core Data Storage
Temporary Data Structures
Table Indexes



DESIGN DECISIONS
Data Organization
→ How we layout data structure in memory/pages and what information to store to support efficient access.
Concurrency
→ How to enable multiple threads to access the data structure at the same time without causing problems.



HASH TABLES
A hash table implements an associative array
abstract data type that maps keys to values.
It uses a hash function to compute an offset into
the array, from which the desired value can be
found.



### STATIC HASH TABLE
Allocate a giant array that has one slot
for every element that you need to
record.
To find an entry, mod the key by the
number of elements to find the offset
in the array



Design Decision #1: Hash Function
→ How to map a large key space into a smaller domain.
→ Trade-off between being fast vs. collision rate.
Design Decision #2: Hashing Scheme
→ How to handle key collisions after hashing.
→ Trade-off between allocating a large hash table vs.
additional instructions to find/insert keys.



TODAY'S AGENDA
Hash Functions

### STATIC HASHING SCHEMES

#### Approach #1: Linear Probe Hashing

Single giant table of slots.
Resolve collisions by linearly searching for the next free slot in the table.
→ To determine whether an element is present, hash to a location in the index and scan for it.
→ Have to store the key in the index to know when to stop scanning.
→ Insertions and deletions are generalizations of lookups.



NON-UNIQUE KEYS

Choice #1: Separate Linked List
→ Store values in separate storage area for
each key.
Choice #2: Redundant Keys
→ Store duplicate keys entries together in
the hash table



OBSERVATION
To reduce the # of wasteful comparisons, it is
important to avoid collisions of hashed keys.
This requires a hash table with ~2x the number of
slots as the number of elements.



#### Approach #2: Robin Hood Hashing

Variant of linear probe hashing that steals slots
from "rich" keys and give them to "poor" keys.
→ Each key tracks the number of positions they are from
where its optimal position in the table.
→ On insert, a key takes the slot of another key if the first
key is farther away from its optimal position than the
second key.

#### Approach #3: Cuckoo Hashing

Use multiple hash tables with different hash
functions.
→ On insert, check every table and pick anyone that has a
free slot.
→ If no table has a free slot, evict the element from one of
them and then re-hash it find a new location.
Look-ups and deletions are always O(1) because
only one location per hash table is checked.



Make sure that we don’t get stuck in an infinite
loop when moving keys.
If we find a cycle, then we can rebuild the entire
hash tables with new hash functions.
→ With two hash functions, we (probably) won’t need to
rebuild the table until it is at about 50% full.
→ With three hash functions, we (probably) won’t need to
rebuild the table until it is at about 90% full.



### Dynamic Hashing Schemes

The previous hash tables require
knowing the number of elements you
want to store ahead of time.
→ Otherwise you have rebuild the entire
table if you need to grow/shrink.
Dynamic hash tables are able to
grow/shrink on demand.
→ Extendible Hashing
→ Linear Hashing



#### CHAINED HASHING
Maintain a linked list of buckets for each slot in
the hash table.
Resolve collisions by placing all elements with the
same hash key into the same bucket.
→ To determine whether an element is present, hash to its
bucket and scan for it.
→ Insertions and deletions are generalizations of lookups.



The hash table can grow infinitely because you just
keep adding new buckets to the linked list.
You only need to take a latch on the bucket to
store a new entry or extend the linked list



#### EXTENDIBLE HASHING
Chained-hashing approach where we split buckets
instead of letting the linked list grow forever.
This requires reshuffling entries on split, but the
change is localized.



#### LINEAR HASHING
Maintain a pointer that tracks the next bucket to
split.
When any bucket overflows, split the bucket at the
pointer location.
Overflow criterion is left up to the
implementation.
→ Space Utilization
→ Average Length of Overflow Chains



Splitting buckets based on the split pointer will
eventually get to all overflowed buckets.
→ When the pointer reaches the last slot, delete the first
hash function and move back to beginning.
The pointer can also move backwards when
buckets are empty

[参考](https://blog.csdn.net/jackydai987/article/details/6673063)

CONCLUSION

Fast data structures that support O(1) look-ups
that are used all throughout the DBMS internals.
→ Trade-off between speed and flexibility.
Hash tables are usually not what you want to use
for a table index…