# Benchmark Results

| Keys | Load | Insert | Delete |
| :--- | :--- | :--- | :--- |
| 1K | 2ms | 48ms | 121ms |
| 10K | 76ms | 518ms | 1658ms |
| 100K | 692ms | 6527ms | 20172ms |

## Analysis & Scaling

* **Load:** Scales linearly in O(N) time.
* **Insert and Delete:** Scale at an O(N log N) time.
* **Run Time Hierarchy:** Load < Insert < Delete.
