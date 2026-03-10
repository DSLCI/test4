# [2026-03-07-002] NPUSHMEMSymmetricMemory 编译失败：GroupInfo 删除 rank_to_global_rank 且 SymmetricMemory 新增纯虚函数

- **发现日期**：2026-03-07
- **编号**：2026-03-07-002
- **严重级别**：🔴 编译失败（阻断构建）
- **受影响文件**：
  - `torch_npu/csrc/distributed/symm_mem/NPUSHMEMSymmetricMemory.cpp`
  - `torch_npu/csrc/distributed/symm_mem/NPUSHMEMSymmetricMemory.hpp`
- **触发版本**：PyTorch nightly 2026-03-07
- **对应 patch**：`patches/0002-fix-NPUSHMEMSymmetricMemory-GroupInfo-api-compat.patch`
- **前置 patch**：`patches/0001-fix-CachingHostAllocator-HostBlockPool-api-compat.patch`（已应用，本次构建推进到 70% 后暴露此问题）

---

## 问题描述

在应用 0001 patch 修复 `CachingHostAllocator` 问题后，构建推进到 70% 时在分布式对称内存模块出现新的编译失败，涉及 8 个错误，集中在 `NPUSHMEMSymmetricMemory.cpp`。

---

## 根本原因分析

### 1. `c10d::symmetric_memory::GroupInfo` 删除了 `rank_to_global_rank` 字段

PyTorch 上游精简了 `GroupInfo` 结构体，新版只保留三个字段：

```cpp
// PyTorch nightly 新版 GroupInfo
struct GroupInfo {
  int rank;
  int world_size;
  c10::intrusive_ptr<c10d::Store> store;
};
```

旧版含有 `std::vector<int> rank_to_global_rank`，Ascend/pytorch 将其用作跨构造调用的进程映射缓存，移除后产生 6 个错误：

```
NPUSHMEMSymmetricMemory.cpp:49: error: 'struct c10d::symmetric_memory::GroupInfo' has no member named 'rank_to_global_rank'
NPUSHMEMSymmetricMemory.cpp:50: error: 'struct c10d::symmetric_memory::GroupInfo' has no member named 'rank_to_global_rank'
... (共 6 处)
```

### 2. `SymmetricMemory` 基类新增 `get_offset()` 纯虚函数

PyTorch 上游在 `SymmetricMemory` 基类中新增了纯虚方法：

```cpp
virtual size_t get_offset() = 0;
```

`NPUSHMEMSymmetricMemory` 未实现该方法，导致类成为抽象类，无法实例化：

```
error: invalid new-expression of abstract class type 'c10d::symmetric_memory::NPUSHMEMSymmetricMemory'
```

---

## 修复方案

见 `patches/0002-fix-NPUSHMEMSymmetricMemory-GroupInfo-api-compat.patch`，核心改动：

1. **`rank_to_global_rank` 缓存迁移**：在 `.cpp` 文件模块级别引入
   `static std::unordered_map<std::string, std::vector<int>> rank_to_global_rank_cache`（加 mutex 保护），
   替代原本存储在 `GroupInfo` 中的缓存逻辑，语义完全等价。

2. **实现 `get_offset()`**：在 `.hpp` 中为 `NPUSHMEMSymmetricMemory` 添加
   `size_t get_offset() override { return 0; }`。
   NPU 对称内存不使用子缓冲区偏移量，返回 0 是正确语义。
