# Slotted-Page B+ Tree Nodes (BNode)

## Overview

In SparrowOne, BNode implements a Slotted-Page architecture backed directly by a fixed-size byte[] buffer. Instead of allocating managed objects for keys, values, or child pointers, BNode reads and writes primitives directly into the byte buffer via BinaryPrimitives (little-endian). This allows pages to be flushed to and loaded from disk with zero serialization overhead.

## Physical Page Layout

The byte buffer is partitioned into four contiguous regions. Unlike classic slotted pages where the slot directory grows forward from the header and the data heap grows backward from the end of the page, both the Slot Table and the Data Heap grow forward from fixed starting offsets determined by the B+ Tree Order:

```text
+----------------+-----------------------+-----------------+-----------------------+
|     Header     |   Kid Table (Pages)   |   Slot Table    |       Data Heap       |
|   (16 Bytes)   |  ((Order + 1) * 4B)   |  (Order * 4B)   |  (Remaining Bytes)    |
+----------------+-----------------------+-----------------+-----------------------+
0                16                 SlotTableStart   DataAreaStart            PageSize
```

### 1. Header (0 - 15 bytes)
Stores node metadata as four 32-bit little-endian integers:

| Offset | Constant | Size | Description |
| :--- | :--- | :--- | :--- |
| 0 | OFF_LEAF | 4B | 1 for a Leaf node; 0 for an Internal node. |
| 4 | OFF_PAGE_ID | 4B | Unique page identifier on disk. |
| 8 | OFF_NUM_KEYS | 4B | Current number of active keys (N). |
| 12 | OFF_NEXT_LEAF | 4B | Page ID of the right sibling leaf (-1 if none or if internal). |

### 2. Kid Table (KidTableStart = 16)
* **Size:** (Order + 1) * 4 bytes (KID_SIZE = 4).
* **Purpose:** Stores child PageId pointers for internal nodes ($N$ keys require $N + 1$ child branches).
* **Leaf Nodes:** This region is still reserved in leaf buffers so SlotTableStart and DataAreaStart remain identical across node types, keeping CalculateMaxOrder uniform.

### 3. Slot Table (SlotTableStart = 16 + (Order + 1) * 4)
* **Size:** Order * 4 bytes (SLOT_SIZE = 4).
* **Purpose:** Acts as a sorted indirection directory mapping a logical index (0 to NumKeys - 1) to a physical byte offset in the Data Heap.
* **Slot Entry Layout (4 bytes):**
  * **Bytes 0-1 (Int16):** Absolute byte offset of the record in Buffer.
  * **Bytes 2-3 (Int16):** Length of the record in bytes (RECORD_SIZE = 8).

### 4. Data Heap (DataAreaStart = SlotTableStart + Order * 4)
* **Size:** PageSize - DataAreaStart bytes.
* **Purpose:** Stores fixed-size 8-byte key-value payloads (4-byte int Key + 4-byte int Data).
* **Ordering:** Records in the Data Heap are not physically sorted. New records are appended sequentially to the end of the heap while the Slot Table maintains logical sort order.

## Core Operations

### Search & Navigation
* **Leaf Lookup (FindIndex):** Binary searches keys via GetKey(mid) (which resolves Slot[mid] heap offset). Returns the matching index or the bitwise complement (low) of the insertion point.
* **Internal Routing (FindChildBranchIndex):** Binary searches separator keys so that keys < Key[i] route to Kid[i] and keys >= Key[i] route to Kid[i + 1].

### Insertion (InsertAt)
1. Checks logical capacity (NumKeys >= Order) and ensures physical tail space exists via GetNextFreeDataOffset(). If fragmented, triggers Compact() before writing.
2. Shifts existing slot entries from pos onward right by SLOT_SIZE (Buffer.BlockCopy).
3. For internal nodes, shifts child pointers right via ShiftChildrenRight(pos) and writes childPageId into Kid[pos + 1].
4. Appends the 8-byte record to the Data Heap and writes its (offset, length) pair into Slot[pos].
5. Increments NumKeys.

### Deletion & Compaction (RemoveKeyAndChildAt / Compact)
* Removing a key shifts the Slot Table (and Kid Table, if internal) left by one slot, clears the trailing dead slot entry, and resets unused child pointers to -1.
* Compact() defragments the page by extracting active Element pairs in logical order, rewriting them contiguously starting at DataAreaStart, updating every slot offset, and zeroing out stale tail bytes.

### Node Splitting (Split)
* **Leaf Split:** Copies the middle separatorKey (midIndex = NumKeys / 2) up to the parent while retaining it in the right sibling leaf, then updates the NextLeafId singly-linked list pointers.
* **Internal Split:** Promotes separatorKey to the parent and removes it from both children to preserve the N keys, N + 1 children invariant. Right-hand child pointers are bulk-transferred via CopyChildrenTo, and stale pointers on the left node are cleared to -1.

## Critical Invariants & Edge Cases
* **Unused Child Pointers Must Be -1:** Never zero out inactive child slots with 0 or .Clear(), because 0 is a valid PageId. Always use SetChildId(index, -1) or ClearChildPointers(...).
* **Maximum Page Size (32,767 bytes):** Slot offsets are stored as signed Int16 (short). Configuring PageSize > 32,767 will overflow slot offset writes.
* **Order Sizing:** Always compute Order using BNode.CalculateMaxOrder(pageSize) so fixed headers, slot directories, and records are guaranteed not to collide in the buffer.