# SparrowOne

SparrowOne is a high-performance, disk-resident B+ Tree storage engine written in C# (.NET 8.0). Designed around a slotted-page physical layout and top-down proactive balancing, it provides backtrack-free insertions and deletions, rapid bulk-loading, and a comprehensive suite of structural integrity diagnostics.

---

## Key Features

* **Slotted-Page Node Architecture (BNode)**: Utilizes fixed-size binary pages featuring a 16-byte header, a child pointer table (for internal nodes), a slot directory, and a contiguous data heap with automatic page-level defragmentation (Compact).
* **Single-Pass Proactive Balancing (BTree)**: Eliminates parent-pointer overhead and backtracking by preemptively splitting full nodes during insertion descent and proactively borrowing or merging under-full nodes during deletion descent.
* **High-Speed Bulk Loading (TreeBuilder)**: Constructs balanced B+ Trees directly from pre-sorted datasets using geometric range partitioning, configurable leaf fill factors, and write-coalescing memory buffers.
* **Sector Reclamation & Compaction**: Tracks decommissioned pages via an in-memory FreeList serialized to disk on disposal, paired with an atomic file-swap Compact() routine to eliminate external fragmentation.
* **Horizontal Leaf Linkage**: Leaf nodes maintain NextLeafId pointers, enabling fast, index-free sequential scans and range queries (EnumerateRange, GetKeyRange, GetElementRange).
* **Built-In Audit & Diagnostics**: Includes a full health auditing suite (PerformFullAudit, ValidateIntegrity) capable of detecting cycles, underflows, leaked pages (Zombies), and dangling pointers (Ghosts).

---

## Physical Storage Layout

### File Structure
1. **File Header (Page 0, 4096 bytes)**: Stores the magic number (0x42542145), branching order, RootId, PageSize, total allocated NodeCount, and FreeList metadata.
2. **Node Pages (Page 1..N)**: Fixed-width binary blocks (default 4096 bytes, minimum 256 bytes) mapped directly via O(1) file offsets:
   Offset = (PageSize x NodeId) + 4096
3. **FreeList Tail**: On Dispose(), recycled page IDs are appended to the end of the file and truncated back out when the database is reopened.

### Slotted-Page Node Layout (`BNode`)
```text
+----------------+------------------------+-----------------+--------------------+
| Header (16B)   | Kid Table (Internal)   | Slot Directory  | Data Heap          |
| Leaf, Id,      | (Order + 1) * 4B       | Order * 4B      | [Key (4B) +        |
| Keys, NextLeaf |                        | [Offset, Len]   |  Data (4B)] ...    |
+----------------+------------------------+-----------------+--------------------+
```

---

## Usage Examples

### Basic CRUD & Range Queries
```csharp
using SparrowOne;

// Open or create a B+ Tree file with 4KB pages.
var tree = new BTree("database.db", pageSize: 4096);

// Insert key-value pairs.
tree.Insert(10, 100);
tree.Insert(20, 200);
tree.AddOrUpdate(15, 150);

// Point lookup.
if (tree.TrySearch(15, out Element result))
{
    Console.WriteLine($"Found Key {result.Key} -> Data {result.Data}");
}

// Fast horizontal leaf-chain range scan.
foreach (Element item in tree.EnumerateRange(10, 20))
{
    Console.WriteLine(item);
}

// Single-pass deletion.
tree.Delete(10, 100);
tree.Commit();
```

### Bulk Loading from Sorted Data

TreeBuilder constructs the tree bottom-up with a target leaf fill factor to leave room for future inserts without immediate splits:

```csharp
var sortedData = new List<Element>
{
    new Element(1, 10),
    new Element(2, 20),
    new Element(3, 30)
};

var builder = new TreeBuilder(pageSize: 4096, leafFill: 0.8);
builder.CreateFromSorted("bulk.db", sortedData);
```

### Auditing & Compaction
```csharp
var tree = new BTree("database.db");

// Run structural validation (throws on invariant violations).
tree.ValidateIntegrity();

// Print a detailed physical and logical health report.
tree.PrintAuditReport();

// Defragment the file and reclaim freed pages.
tree.Compact();
```
---

## License

This project is licensed under the MIT License.