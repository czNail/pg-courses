# 存储与持久化

本目录约 7 节课，目标是理解 block、buffer、WAL 和文件系统之间的正确性边界。

课程安排：

1. Buffer Manager：pin、dirty、I/O in progress。
2. WAL / XLog：WAL-before-data 与 flush 边界。
3. Storage Manager / fd / md：fork、segment、fd cache。
4. Checkpointer / Background Writer / WAL Writer。
5. HeapAM DML 与 HOT update。
6. B-tree search / insert / split。
7. Index AM API 与 index vacuum。
