# [2026-03-07-001] CachingHostAllocator 编译失败：PyTorch nightly 重构 HostBlockPool API

- **发现日期**：2026-03-07
- **编号**：2026-03-07-001
- **严重级别**：🔴 编译失败（阻断构建）
- **受影响文件**：`torch_npu/csrc/core/npu/CachingHostAllocator.cpp`
- **触发版本**：PyTorch nightly ≥ 2026-03-07（具体构建待确认）
- **对应 patch**：`patches/0001-fix-CachingHostAllocator-HostBlockPool-api-compat.patch`

---

## 问题描述

在 Ascend/pytorch 主干与 PyTorch 最新 nightly 构建的兼容性 CI 中，`make` 阶段因 `CachingHostAllocator.cpp` 编译失败终止，涉及 21 个 C++ 编译错误。

---

## 根本原因分析

PyTorch 上游对 `at::CachingHostAllocatorImpl` 模板进行了重大重构，导致以下 API 断层：

### 1. `process_events()` 虚函数签名变更

| | 旧 API | 新 API |
|---|---|---|
| 签名 | `virtual void process_events()` | `virtual void process_events(BlockPool& pool)` |

`NPUExpandableHostAllocatorImpl::process_events()` 使用 `override`，但父类虚函数已添加 `pool` 参数，导致 override 失效：

```
error: 'void at_npu::native::NPUExpandableHostAllocatorImpl::process_events()'
       marked 'override', but does not override
```

### 2. `HostBlockPool` 成员重命名与移除

| 成员 | 旧版 | 新版 |
|------|------|------|
| 已映射块集合 | `blocks` (`std::set`) | `blocks_` (`flat_hash_set`) |
| 未映射块集合 | `unmapped` (`std::set`) | **已移除** |
| 新增 | — | `free_list_` (按大小分桶)、`ptr_to_block_`、`events_`、`events_mutex_` |

Ascend 代码中的 `at_npu::native::BlockPool` 与父类继承的 `using BlockPool = HostBlockPool<...>` 产生名字冲突，使得 `NPUExpandableHostAllocatorImpl` 中的 `BlockPool blocks_pool` 被解析为父类类型（`at::HostBlockPool`），而后者已不含 `blocks`/`unmapped` 字段：

```
error: 'struct at::HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
error: 'struct at::HostBlockPool<...>' has no member named 'unmapped'
```

### 3. `AllocParams` 构造函数类型不匹配

因 `BlockPool` 类型名冲突，`AllocParams(size, &blocks_pool)` 中 `&blocks_pool` 被推导为 `at::HostBlockPool*`，而 `at_npu::native::AllocParams` 期望的是 `at_npu::native::BlockPool*`：

```
error: no matching function for call to
  'at_npu::native::AllocParams::AllocParams(const long unsigned int&,
   at::CachingHostAllocatorImpl<...>::BlockPool*)'
```

---

## 错误日志摘要

```
CachingHostAllocator.cpp:888: error: 'process_events()' marked 'override', but does not override
CachingHostAllocator.cpp:580: error: no matching function for call to 'AllocParams::AllocParams(..., BlockPool*)'
CachingHostAllocator.cpp:759: error: 'HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
CachingHostAllocator.cpp:760: error: 'HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
CachingHostAllocator.cpp:792: error: 'HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
CachingHostAllocator.cpp:821: error: 'HostBlockPool<...>' has no member named 'unmapped'
... (共 21 个错误)
make[2]: *** [CMakeFiles/torch_npu.dir/build.make:961: ...CachingHostAllocator.cpp.o] Error 1
make: *** [Makefile:136: all] Error 2
```

---

## 修复方案

见 `patches/0001-fix-CachingHostAllocator-HostBlockPool-api-compat.patch`，核心改动：

1. **重命名** `at_npu::native::BlockPool` → `at_npu::native::ExpandableBlockPool`，消除与继承 typedef 的名字冲突
2. **更新** `process_events()` 为 `process_events(BlockPool& pool) override`，引入私有辅助方法 `do_process_npu_events()` 承载 NPU 事件处理逻辑
3. **更新** 所有相关函数参数和成员声明使用新名称

> **注意**：Ascend/pytorch 的 `ExpandableBlockPool` 仍保留 `blocks`（std::set）和 `unmapped`（std::set）字段用于自身的可扩展分段管理逻辑，与 PyTorch 新版 `HostBlockPool` 的 `blocks_`（flat_hash_set）+ `free_list_` 设计并行存在，互不影响。

---

## 上游参考

- PyTorch PR/Commit：待追踪（`at::CachingHostAllocatorImpl` 重构）
- 相关文件：`aten/src/ATen/core/CachingHostAllocator.h`
