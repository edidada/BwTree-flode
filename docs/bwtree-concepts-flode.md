# Bw-Tree 概念入门（TUM 教学版 / flode）

这篇文档面向第一次接触本仓库的读者：先讲清 Bw-Tree 的核心概念，再说明这个 2015 年的教学版实现由哪些文件组成、有哪些限制。

> 本仓库 fork 自 [flode/BwTree](https://github.com/flode/BwTree)，是慕尼黑工业大学（TUM）数据库系统教研组的**学生项目**，作者 Florian Scheibner，2014–2015 年，Apache 2.0 许可。当前分支 `edidada/master` 只增加了本文和 `.github/workflows/build-and-run-flode.yml`，没有改动源码。

## 1. Bw-Tree 是什么

Bw-Tree 是微软研究院 2013 年 ICDE 论文 *"The Bw-Tree: A B-tree for New Hardware Platforms"* 提出的**无锁（lock-free / latch-free）内存 B+ 树**，为多核与闪存设计。

传统 B+ 树修改页面时要加 latch，高并发下线程会在根/内部节点排队。Bw-Tree 换成两个思路：

- **映射表（Mapping Table）**：树里存逻辑页号 PID，通过一张可用 CAS 更新的表映射到物理地址；
- **增量链（Delta Chain）**：不就地改页，而是往页头追加一条小记录（"插入了什么/删除了什么"），再用一次 CAS 把映射表指针切到新链头。

写操作 = 一次分配 + 一次 CAS；读者只看快照，不需要加锁。

## 2. 结构（Structure）与本仓库源码

| 文件 | 作用 |
| --- | --- |
| `main.cpp` / `main.hpp` | benchmark 驱动：随机生成键值、多线程执行 insert/search，打印耗时与统计 |
| `bwtree.hpp` / `bwtree.cpp` | `BwTree::Tree<Key, Data>` 主类、`Settings` 参数、映射表、SMO 逻辑 |
| `nodes.hpp` | 节点与 delta 记录类型（叶子/内部节点、各类增量） |
| `epoque.hpp` / `epoche.cpp` | **Epoch 回收器**：基于 TBB 的 `enumerable_thread_specific` 保存每线程的待删列表 |
| `CMakeLists.txt` | CMake 构建，链接 `pthread` 与 `tbb` |

### 关键概念速记

- **PID / Mapping Table**：间接寻址层，CAS 它即原子发布新页面版本。
- **Delta Record / Delta Chain**：追加式修改；`Settings` 里的 split/consolidate 阈值决定何时整理。
- **Consolidation（页合并）**：把基础页和一串 delta 压成新的紧凑基础页。
- **SMO（Split / Merge）**：改树形状的操作，要同时动子节点与父节点，需拆成多步并让其他线程 help-along。
- **Epoch GC**：被换下来的旧页面不能马上释放（可能还有线程在读），按 epoch 延后回收 —— 本仓库用 `epoque.hpp` 加 TBB 的线程本地存储实现。
- **CAS**：`std::atomic::compare_exchange_*`，无锁生效的关键指令。

`Settings("8,8,3,3", 8, {8}, 3, {3})` 这类参数（`main.cpp` 里可见）控制叶子/内部节点的分裂与合并阈值，是研究 delta 链行为的旋钮。

## 3. 优势（Strengths）

| 优势 | 原因 |
| --- | --- |
| 无锁、多核伸缩 | 冲突收敛到映射表上的一次 CAS，没有节点级 latch |
| 读不被写阻塞 | 读者遍历版本快照 |
| 闪存/内存友好 | 只追加小增量，不整页重写 |
| 代码量小、易读 | 教学版，主干只有几个文件，适合照着论文读实现 |

代价：增量链变长会拖慢读；SMO 协议复杂；CAS 失败要重试；旧页回收依赖 epoch 机制。

## 4. 本实现的已知限制

README 明确列出：

- **不实现欠满页面的合并（merge）**；
- **映射表不能动态增长**；
- **consolidate / split 不是异步执行**。

因此它适合用来理解主流程，不适合当生产级索引。

## 5. 编译与运行

```bash
sudo apt-get install -y g++ cmake libtbb-dev
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel 2
./build/BwTree
```

注意 `./build/BwTree` 默认跑的是一整套性能测量：100 万初始键，然后 100 万 / 1000 万 / 4200 万键 × 1–8 线程 × 多组参数，耗时很长、内存占用也大。**CI 因此只做有界的 smoke run**：启动二进制、确认不崩溃、15 秒后由 `timeout` 停止，退出码 0 或 124 都算通过。测试内容本身可以在 `main.cpp` 里改。

本分支的 workflow 在 `edidada/**` push 时自动编译并做 smoke run，可用 `gh run list` / `gh run view --log` 查看日志。

## 6. 关于分支与同源仓库

[cih-y2k/BwTree-1](https://github.com/cih-y2k/BwTree-1) 是 `flode/BwTree` 的一个 fork，两者文件树与提交 SHA 完全相同。GitHub 不允许同一账号在同一 fork 网络内建两个 fork，因此这个 `edidada/BwTree-flode` fork 里为它另开了 `edidada/BwTree-1` 分支，配一份独立的 workflow 与文档。

## 7. 参考

- Levandoski, Lomet, Sengupta. *The Bw-Tree: A B-tree for new hardware platforms.* ICDE 2013.
- 本仓库 `README.md`（Build instructions / Restrictions / Troubleshooting）。
