# Bw-Tree 概念入门（cih-y2k/BwTree-1）

这篇文档面向第一次接触本仓库的读者：先讲清 Bw-Tree 的核心概念，再说明这个 TUM 教学版实现的组成与限制，最后交代为什么它和 `flode/BwTree` 共用一个 fork。

> 本分支对应 [cih-y2k/BwTree-1](https://github.com/cih-y2k/BwTree-1)，它是 [flode/BwTree](https://github.com/flode/BwTree) 的一个 fork，两者的提交 SHA 与文件树完全相同（慕尼黑工业大学数据库教研组学生项目，2014–2015，Apache 2.0）。本分支 `edidada/BwTree-1` 只增加了本文和 `.github/workflows/build-and-run-cih-y2k.yml`，没有改动源码。

## 1. Bw-Tree 是什么

Bw-Tree 出自微软研究院 2013 年 ICDE 论文 *"The Bw-Tree: A B-tree for New Hardware Platforms"*，是一种**无锁（lock-free / latch-free）内存 B+ 树**，面向多核 CPU 与闪存/大内存。

传统 B+ 树改页要加 latch，高并发时根/内部节点成为瓶颈。Bw-Tree 用两个技巧绕开：

- **映射表（Mapping Table）**：节点之间用逻辑页号 PID 互相引用，通过一张可用 CAS 更新的表映射到物理地址；
- **增量链（Delta Chain）**：不就地改写页面，而是往页头追加一条小的"增量记录"，再用一次 CAS 把映射表指针切到新链头。

写操作 = 一次分配 + 一次原子指令，读者只看快照、不需要等锁。

## 2. 结构（Structure）与源码文件

| 文件 | 作用 |
| --- | --- |
| `main.cpp` / `main.hpp` | benchmark 驱动：随机生成键值、多线程 insert/search，打印耗时与统计 |
| `bwtree.hpp` / `bwtree.cpp` | `BwTree::Tree<Key, Data>` 主类、`Settings` 参数、映射表、SMO 逻辑 |
| `nodes.hpp` | 叶子/内部节点与各类 delta 记录 |
| `epoque.hpp` / `epoche.cpp` | Epoch 回收器，基于 TBB 的 `enumerable_thread_specific` 保存每线程待删列表 |
| `CMakeLists.txt` | CMake 构建，链接 `pthread` 与 `tbb` |

### 六个关键词

- **PID / Mapping Table**：间接寻址层，CAS 表项即原子发布新版本。
- **Delta Record / Delta Chain**：追加式修改；`Settings` 里的阈值决定何时整理链。
- **Consolidation**：链太长时把基础页与所有 delta 压成新的紧凑基础页。
- **SMO（Split / Merge）**：改树形状的操作，要同时动子节点与父节点，需拆成多步并让其他线程 help-along。
- **Epoch GC**：被替换的旧页不能立刻释放（可能还有读者），按 epoch 延后回收 —— 本仓库由 `epoque.hpp` 实现。
- **CAS**：`std::atomic::compare_exchange_*`，无锁生效的关键指令。

`main.cpp` 里的 `Settings("8,8,3,3", 8, {8}, 3, {3})` 控制叶子/内部节点的分裂与合并阈值，是观察 delta 链行为的旋钮。

## 3. 优势（Strengths）与代价

| 优势 | 原因 |
| --- | --- |
| 无锁、多核伸缩性好 | 冲突收敛到映射表上的一次 CAS |
| 读不被写阻塞 | 读者遍历版本快照 |
| 对闪存友好 | 只追加小增量，不整页重写 |
| 代码量小、易读 | 主干只有几个文件，适合照着论文读实现 |

代价：增量链变长会拖慢读；SMO 协议复杂；CAS 失败要重试；旧页回收依赖 epoch 机制。

## 4. 本实现的已知限制

README 明确列出：

- 不实现欠满页面的合并（merge）；
- 映射表不能动态增长；
- consolidate / split 不是异步执行。

所以它适合学习主流程，不是生产级索引。

## 5. 编译与运行

```bash
sudo apt-get install -y g++ cmake libtbb-dev
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel 2
./build/BwTree
```

`./build/BwTree` 默认跑一整套性能测量：100 万初始键，之后 100 万 / 1000 万 / 4200 万键 × 1–8 线程 × 多组参数，耗时与内存占用都很大。因此 CI 只做**有界 smoke run**：启动二进制、确认不崩溃、15 秒后由 `timeout` 停止，退出码 0 或 124 都算通过。想改测试内容可以编辑 `main.cpp`。

本分支的 workflow 在 `edidada/**` push 时自动编译并 smoke run，可用 `gh run list` / `gh run view --log` 查看日志。

## 6. 为什么和 flode 共用一个 fork

GitHub 不允许同一账号在同一个 **fork 网络**里创建两个 fork。`cih-y2k/BwTree-1` 是 `flode/BwTree` 的 fork，二者属于同一网络，所以 `gh repo fork cih-y2k/BwTree-1` 返回的是已经存在的 `edidada/BwTree-flode`。

处理方式是在 `edidada/BwTree-flode` 里保留两条分支：

| 分支 | 对应上游 | workflow | 文档 |
| --- | --- | --- | --- |
| `edidada/master` | `flode/BwTree` | `build-and-run-flode.yml` | `bwtree-concepts-flode.md` |
| `edidada/BwTree-1` | `cih-y2k/BwTree-1` | `build-and-run-cih-y2k.yml` | 本文 |

两条分支基于同一个提交（`c68015e`），内容一致，只是各自带一份独立的 CI 与文档。

## 7. 参考

- Levandoski, Lomet, Sengupta. *The Bw-Tree: A B-tree for new hardware platforms.* ICDE 2013.
- 本仓库 `README.md`（Build instructions / Restrictions / Troubleshooting）。
