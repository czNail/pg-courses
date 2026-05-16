# 事务与可见性

本目录约 6 节课，目标是解释一个 tuple version 为什么当前语句可见、可锁定、可更新或可回收。

课程安排：

1. Transaction / CLOG / Subtrans。
2. Snapshot 生命周期与 ProcArray。
3. Heap tuple visibility。
4. MultiXact / row lock / update conflict。
5. VACUUM / HOT pruning / freeze / visibility map。
6. Serializable / predicate lock。
