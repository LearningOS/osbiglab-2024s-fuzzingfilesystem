# 第 12 周

## 本周进展

1. 开发：

   * 之前 extent 只支持单一根节点。实现了 extent 的插入和节点分裂，使 extent 树可以自动扩展。

   * 将原 `BlockDevice` 的 `read_offset` 和 `write_offset` 接口修改为 `read_block` 和 `write_block`。定义 `struct Block` 和 `trait AsBytes` 对数据块的操作进行抽象和封装，便于后续实现块缓存。

2. 重构：

   * 重构 `block_group` 模块，定义了 `struct BlockGroupRef`，并仿照 `InodeRef` 实现了块组的载入、修改、写回操作。

3. 修复：

   * 修复了文件读中偏移位置的 bug。
   * 修复了新建目录项中的逻辑错误。
   * 构造大文件读写测例，修复了 extent 节点分裂的错误。

## 下周计划

1. extent 树的主要操作已基本实现，可在此基础上实现删除文件和删除目录功能。
2. 基于新的 `BlockDevice` 实现块缓存。
3. 对已实现的功能进行测试。