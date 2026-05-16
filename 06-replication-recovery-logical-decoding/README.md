# 复制、恢复与逻辑解码

本目录约 6 节课，目标是理解 WAL 如何支撑 standby、recovery、logical decoding 和 subscription apply。

课程安排：

1. WAL Sender / WAL Receiver。
2. Replication Slot 与 WAL/xmin 保留边界。
3. Recovery / REDO / Hot Standby conflict。
4. Reorder Buffer。
5. Logical Decoding。
6. Logical apply worker / subscription。
