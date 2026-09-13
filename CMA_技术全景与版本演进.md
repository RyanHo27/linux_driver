# Linux CMA 技术全景与版本演进

## 1. 结论先行

CMA（Contiguous Memory Allocator，连续内存分配器）解决的是：系统运行一段时间以后，仍需为不支持 scatter-gather、没有合适 IOMMU 映射能力，或明确要求物理连续内存的设备/子系统，分配较大的连续物理页区间。

它不是一个把整块 RAM 永久闲置给设备的传统 carveout。CMA 在启动早期用 memblock 圈定一段物理地址范围，初始化时把其中的 pageblock 标成 `MIGRATE_CMA` 并交给 buddy allocator。普通的“可迁移”页可以暂时借用这段内存；当 CMA 客户申请连续页时，内核隔离目标范围，回收干净页并把可迁移页迁走，再从 buddy 中摘取连续 PFN。

这个设计的核心交换是：

| 得到的能力 | 付出的代价 |
|---|---|
| 比永久 carveout 更高的空闲内存利用率 | 申请可能触发同步回收和页迁移，延迟不可硬实时保证 |
| 运行期可获得大块物理连续内存 | 即使总空闲内存足够，也可能因不可迁移/长期 pin 页而失败 |
| 能作为默认、设备专用或 NUMA 本地池 | 启动时就要决定容量、地址范围和对齐；常规 CMA 不能运行期扩容 |
| 与通用 DMA API 集成 | CMA 本身不提供缓存一致性、DMA 地址转换或安全隔离 |

对驱动开发最重要的原则是：优先调用通用 DMA API，例如 `dma_alloc_coherent()`、`dma_alloc_attrs()` 或适合流式 DMA 的映射 API，不要因为设备树里配置了 CMA 就直接依赖 `cma_alloc()`。DMA 层会综合设备的 DMA mask、体系结构一致性规则、IOMMU 和设备专用 CMA 区选择正确后端。只有内核子系统级代码确实需要直接管理 `struct page` 连续区间，并能够承担版本兼容责任时，才考虑 CMA 核心 API。

本文本地源码基线是 Linux 6.0.1。该目录没有 `.git` 历史，因此 6.0.1 的机制来自本地代码走读；历史和 6.0.1 之后的变化依据上游提交与官方文档。按照 2026-09-13 的 kernel.org 状态，最新稳定版为 7.2.5、主线为 7.3-rc2。[1]

## 2. CMA 必须掌握的概念

### 2.1 “连续”到底指什么

CMA 返回的是连续的物理页帧号（PFN），也就是 CPU 物理地址范围连续。以下概念不能混为一谈：

- 虚拟地址连续：`vmalloc()` 能做到，但底层物理页通常不连续。
- 物理地址连续：CMA 的核心保证。
- DMA 地址连续：可能由物理连续内存直接得到，也可能由 IOMMU 把离散物理页映射成一个连续 IOVA。
- cache coherent：描述 CPU 与设备观察内存更新的规则，不由 CMA 自动保证。
- 安全隔离：CMA 不是 IOMMU/MPU，不能限制设备只能访问某一片 RAM。

如果硬件支持 scatter-gather，或 IOMMU 能可靠地构造连续 IOVA，通常无需为了 DMA 地址连续而强求物理连续。物理连续仍可能因硬件环形表、固件接口、显示/视频块、巨页或特定共享内存协议而必要。

### 2.2 CMA 与 pageblock、迁移类型

Linux buddy allocator 按阶管理空闲页，并按 pageblock 维护迁移类型。6.0.1 中 `MIGRATE_CMA` 的语义是：

- 只有可迁移分配能够使用 CMA pageblock；
- buddy 不应隐式把 CMA pageblock 改成普通迁移类型；
- CMA 区初始化时以 pageblock 为单位建立该属性；
- 对页分配器而言，`MIGRATE_CMA` 也属于 movable 类别，但有额外的回退和水位控制规则。

源码入口：`include/linux/mmzone.h:42-80`、`mm/page_alloc.c:2305-2323`。6.0.1 中 CMA 区基址和大小必须按 `pageblock_nr_pages` 对齐，见 `include/linux/cma.h:19-25`。

pageblock 通常远大于单页，因此地址和容量设计不能只按 4 KiB 对齐。精确值取决于体系结构、页大小、`pageblock_order`、HugeTLB/THP 配置等，应在目标内核上核实，而不是硬编码“必然为 2 MiB”之类的假设。

### 2.3 “借给系统”不等于“任何分配都能使用”

CMA 空闲页主要只向带有 movable 语义的普通分配开放，例如部分匿名页和页缓存。不可迁移的内核对象、很多 slab、页表和长期固定页不能依赖 CMA 空间满足水位。因此可能出现：机器看起来还有大量 `CmaFree`，但非 movable 分配仍进入内存压力甚至 OOM。

反方向也一样：`CmaFree` 低不等于 CMA 一定无法申请。它只统计当前在 buddy 中真正空闲的 CMA 页；被页缓存等 movable 页面借用的页，在申请时可能成功迁走。能否得到某个尺寸的连续区间，还取决于迁移目标内存、水位、pin 状态、对齐和候选区间。

### 2.4 CMA 不是硬保证

“CMA 总容量大于请求”不是成功充分条件。成功至少受以下因素约束：

- 位图中存在满足请求大小和对齐的候选区间；
- 候选区间内的在用页能够回收或迁移；
- 区间落在一个 zone 内；
- 系统有足够迁移目标页，且水位允许；
- 请求上下文允许睡眠；
- 物理地址满足设备 DMA mask 和平台限制；
- CMA 区在启动时成功激活。

因此 CMA 适合“大块、数量有限、允许阻塞”的连续内存，不适合在 IRQ/atomic 路径临时申请，也不适合作为毫秒级最坏时延必须可证明的硬实时分配器。实时系统通常要在初始化阶段预分配，再在自己的固定池中切分复用。

## 3. Linux 6.0.1 的实现全链路

### 3.1 启动参数与默认容量选择

6.0.1 的 `kernel/dma/contiguous.c` 支持：

```text
cma=<size>
cma=<size>@<base>
cma=<size>@<start>-<end>
cma=0
cma_pernuma=<size>
```

`cma=` 是 early parameter。只给 size 时让 memblock 选址；`size@base` 表示固定基址；`size@start-end` 表示在指定物理范围内选址。`cma=0` 禁用全局默认 CMA。6.0.1 文档见 `Documentation/admin-guide/kernel-parameters.txt:659-669`，解析代码见 `kernel/dma/contiguous.c:72-109`。

如果没有命令行覆盖，默认大小由以下 Kconfig 组合决定：

- `CONFIG_CMA_SIZE_MBYTES`
- `CONFIG_CMA_SIZE_PERCENTAGE`
- `CONFIG_CMA_SIZE_SEL_MBYTES`
- `CONFIG_CMA_SIZE_SEL_PERCENTAGE`
- `CONFIG_CMA_SIZE_SEL_MIN`
- `CONFIG_CMA_SIZE_SEL_MAX`

实践中的优先关系是：命令行 `cma=` 高于设备树的默认 CMA；设备树 `linux,cma-default` 高于构建时默认值；设备专用的非默认 CMA 节点不等同于全局默认池。6.0.1 在检测到命令行覆盖时会跳过设备树的 default CMA 节点，见 `kernel/dma/contiguous.c:400-431`。

### 3.2 早期预留

体系结构启动代码在 memblock 仍可用时调用 `dma_contiguous_reserve()`。它选择容量、基址和上限，再调用：

```text
dma_contiguous_reserve_area()
  -> cma_declare_contiguous()
    -> cma_declare_contiguous_nid()
      -> memblock_reserve()，或 memblock_alloc_range_nid()
      -> cma_init_reserved_mem()
```

关键检查包括：

- 大小非零；
- alignment 为 2 的幂，并至少是 CMA 最小对齐；
- fixed 区域不能因对齐被悄悄改变；
- 不能跨 lowmem/highmem 边界；
- 不能超过 limit 和物理内存末尾；
- `order_per_bit` 与页数对齐；
- CMA area 数量不能超过 `MAX_CMA_AREAS`。

非 fixed 选址在 64 位大内存机器上优先尝试从 4 GiB 以上、靠近 node 起点的位置做 bottom-up 分配。这既避免干扰 DMA/DMA32 受限 zone，又让后续 compaction 更倾向于把页迁出 CMA，而不是迁入它。

### 3.3 激活：从 reserved 变成可借用

`cma_init_reserved_areas()` 是 `core_initcall`。每个 CMA area 会：

1. 为 CMA 自有位图分配内存；
2. 验证整片 CMA 区属于同一个 zone；
3. 逐个 pageblock 调用 `init_cma_reserved_pageblock()`；
4. 清除 `PageReserved`、建立正确 refcount；
5. 把 pageblock 标成 `MIGRATE_CMA` 并释放给 buddy；
6. 更新 managed page 和 CMA 页统计。

位图只记录“已被 CMA 客户占用”的粒度块，不记录 buddy 暂时放在 CMA 区中的 movable 页面。后者由普通页管理结构维护，等真正 CMA 申请时再迁走。

如果位图分配失败、PFN 无效或区域跨 zone，area 激活失败。6.0.1 默认会把这些页暴露给 buddy，避免整片内存泄漏；某些必须保留该物理区的用户可通过 `cma_reserve_pages_on_error()` 改变处理。

### 3.4 分配：位图保留 + alloc_contig_range

6.0.1 的 `cma_alloc()` 在 `mm/cma.c:424-523` 中完成：

```text
校验 area/count
  -> 根据 align 和 order_per_bit 计算 mask、offset、bitmap_count
  -> 在位图中寻找连续零区
  -> 先置位，防止另一个 CMA 请求竞争同一候选区
  -> alloc_contig_range(pfn, pfn + count, MIGRATE_CMA, GFP_KERNEL...)
       -> 把相关 pageblock 临时设为 MIGRATE_ISOLATE
       -> drain PCP
       -> 隔离可迁移页
       -> 回收干净页、迁移其余 movable 页
       -> 验证目标页全部隔离
       -> 从 buddy 摘取目标 PFN 区间
       -> 恢复外围 pageblock 的迁移类型
  -> 成功：返回首个 struct page
  -> -EBUSY：清位图并尝试下一个候选区
  -> 其他错误：结束并计数
```

`alloc_contig_range()` 的实现位于 `mm/page_alloc.c:9202-9344`。它明确要求目标范围属于单个 zone，采用同步迁移 `MIGRATE_SYNC`，所以耗时与要搬走的页面数量、页面类型、系统压力直接相关。

6.0.1 使用全局 `cma_mutex` 串行化 `alloc_contig_range()` 阶段；不同 CMA area 的并发大分配也会互相等待。这是后续版本改成 per-CMA 锁的直接动机。

### 3.5 释放

`cma_release()` 校验起始页是否属于 area，调用 `free_contig_range()` 把页归还 buddy，随后清除 CMA 位图并发出 trace event。调用者必须使用原 area、原起始页和正确页数；错误 count 可能越界或破坏分配记账，不能把 CMA 返回区间当成可随意拆分释放的一般堆。

### 3.6 DMA 层的池选择

6.0.1 `dma_alloc_contiguous()` 的选择顺序是：

1. `dev->cma_area`，即设备专用 CMA；
2. 对单页请求直接返回 NULL，让上层走普通页分配，避免浪费和碎片化 CMA；
3. 若启用 per-NUMA CMA，尝试设备所在 node 的 CMA；
4. 全局默认 CMA；
5. 返回 NULL，由更上层 DMA 实现决定是否走 buddy、IOMMU、swiotlb 等其它路径。

它首先检查 `gfpflags_allow_blocking()`。这就是为什么 CMA 不能在不允许睡眠的上下文使用。入口见 `kernel/dma/contiguous.c:282-370`。

### 3.7 设备树

全局默认 CMA 示例：

```dts
/ {
    reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;

        linux_cma: linux,cma {
            compatible = "shared-dma-pool";
            reusable;
            size = <0x0 0x10000000>;       /* 256 MiB */
            alignment = <0x0 0x00200000>; /* 示例；须按目标内核核实 */
            alloc-ranges = <0x0 0x80000000 0x0 0x40000000>;
            linux,cma-default;
        };
    };
};
```

设备专用池示例：

```dts
/ {
    reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;

        video_cma: video-pool@90000000 {
            compatible = "shared-dma-pool";
            reusable;
            reg = <0x0 0x90000000 0x0 0x08000000>; /* 128 MiB */
        };
    };

    video-codec@... {
        memory-region = <&video_cma>;
    };
};
```

约束：

- CMA `shared-dma-pool` 必须是 `reusable`；
- 不能同时设置 `no-map`；
- `reg` 用于固定区域；`size` 配合可选的 `alignment`、`alloc-ranges` 用于动态选址；
- `linux,cma-default` 只应有一个有效默认池；
- 设备通过 `memory-region` 绑定专用区域，reserved-memory 框架最终设置 `dev->cma_area`；
- `linux,dma-default` 表示 consistent DMA allocator 的默认池，与 `linux,cma-default` 不是同一概念。

绑定依据见 `Documentation/devicetree/bindings/reserved-memory/reserved-memory.yaml`、`shared-dma-pool.yaml`，6.0.1 初始化逻辑见 `drivers/of/of_reserved_mem.c:90-145` 和 `kernel/dma/contiguous.c:372-441`。[2]

## 4. 配置项和 API 速查

### 4.1 6.0.1 关键 Kconfig

| 选项 | 作用 | 注意 |
|---|---|---|
| `CONFIG_CMA` | CMA 核心 | 依赖 MMU，选择 migration 和 memory isolation |
| `CONFIG_DMA_CMA` | DMA mapping 与 CMA 集成 | 驱动通常通过 DMA API 间接使用 |
| `CONFIG_CMA_AREAS` | 可建立的 area 数量上限 | 6.0.1 的 `MAX_CMA_AREAS` 还额外加 1；NUMA 默认值更大 |
| `CONFIG_CMA_ALIGNMENT` | DMA 连续缓冲区最大对齐 order | 是 PAGE_SIZE order，不是字节；过大造成浪费 |
| `CONFIG_DMA_PERNUMA_CMA` | 每个 NUMA node 建相同大小池 | 6.0.1 用 `cma_pernuma=` 配置 |
| `CONFIG_CMA_DEBUGFS` | 调试信息和人工 alloc/free | 只建议调试环境开放 |
| `CONFIG_CMA_SYSFS` | 每 area 成功/失败页数统计 | 适合长期健康监控 |
| `CONFIG_CMA_DEBUG` | 打开大量 `pr_debug` | 6.0.1 存在，后续主线已删除 |

定义见 `mm/Kconfig:822-867` 与 `kernel/dma/Kconfig:118-196`。

### 4.2 应用/驱动应该选哪种 API

| 需求 | 首选 | CMA 的角色 |
|---|---|---|
| 大块 coherent DMA 缓冲区 | `dma_alloc_coherent()` | 平台可能在后端使用 CMA |
| 要求 CPU 物理连续的 DMA 分配 | `dma_alloc_attrs(..., DMA_ATTR_FORCE_CONTIGUOUS)`，先确认平台语义 | 可能强制 CMA/连续页路径 |
| streaming DMA，设备支持 SG | `dma_map_sg()` / `dma_map_single()` | 通常不需要 CMA |
| 多个小型 coherent 描述符 | `dma_pool_create/alloc/free()` | 底层大页来源可能间接用 DMA allocator；不要用 CMA 自己切小块，除非有充分理由 |
| 普通内核小对象 | `kmalloc`/slab | 不应直接用 CMA |
| 大块仅虚拟连续内存 | `vmalloc()` | 不要求物理连续 |
| 固定物理地址、绝不让 OS 借用 | reserved-memory `no-map`、平台 carveout、`gen_pool` 等 | 这不是普通 CMA |
| 用户态共享图形/多媒体缓冲 | dma-buf heap / 子系统 allocator | 可能以 CMA heap 为后端 |
| gigantic HugeTLB 页运行期供应 | `hugetlb_cma=` | 专用 CMA 用户，不等于 DMA 默认池 |

直接 CMA API 包括 `cma_declare_contiguous[_nid]()`、`cma_init_reserved_mem()`、`cma_alloc()`、`cma_release()`。在 6.0.1 中这些接口主要面向内核内建用户，且签名随历史演进过。树外模块直接调用会增加升级成本；较新的主线才逐步明确并扩大可导出接口范围。

## 5. CMA 的优势、局限和典型适用场景

### 5.1 与 kmalloc、U-Boot 固定预留区的本质区别

先澄清一个最容易产生误解的地方：**CMA 本身也在启动阶段预留一个物理地址范围**。它和 U-Boot/设备树建立的永久预留区，差别不在于“是否预留”，而在于预留后的管理策略：

```text
kmalloc
    不提前圈定专用范围
    -> 运行期从 slab/buddy 直接找当前可用的连续内存

永久 carveout
    启动前圈定范围
    -> Linux 永远不能把它用于普通页
    -> 设备需要时直接使用，不必迁移

CMA
    启动时圈定范围并保护其可回收属性
    -> 初始化后以 MIGRATE_CMA 交回 buddy
    -> movable 页可暂时借用
    -> 设备申请时迁走借用页，形成连续区间
```

所以 CMA 可以理解为“**可回收、可借用的启动预留区**”，位于完全动态的 `kmalloc` 与完全独占的 carveout 之间。

#### 5.1.1 三种方案总表

| 对比维度 | `kmalloc` | U-Boot/DT 永久预留区 | CMA |
|---|---|---|---|
| 物理连续 | 是，但大块依赖当时 buddy 中存在足够高阶连续页 | 是，启动时固定保证 | 是，申请时通过隔离和迁移整理出来 |
| 典型尺寸 | 小对象；内核文档建议通常用于小于一页的对象 | 可非常大 | 页级到很大连续块，偏大块用途 |
| 长期运行后的大块成功率 | 易受外部碎片影响，通常较差 | 不受 Linux 内存碎片影响 | 通常显著优于直接高阶分配，但仍可能被不可迁移/pin 页阻塞 |
| 空闲时可供 Linux 使用 | 本来就是普通 Linux 内存 | 否 | 是，但只适合 movable 分配 |
| 申请时延 | 小分配通常很低；高阶失败/回收路径可能抖动 | 最低且最可预测；若自建池，通常只是位图/链表操作 | 可能同步回收、compaction 和 migration，长尾最大 |
| 成功确定性 | 小块较好，大块无保证 | 如果容量和生命周期规划正确，确定性最高 | 不是硬保证 |
| 动态共享容量 | 通用内存共享，但无法保护大块连续性 | 通常只由指定设备使用 | Linux movable 页与多个 CMA 客户可共享预留容量 |
| 固定物理地址 | 不适合 | 最适合 | 可固定 area 地址，但每次返回的子块位置仍由 allocator 选择 |
| atomic/IRQ 申请 | 合适的 GFP 和尺寸下可以 | 自建无锁/原子池可以 | 不可以依赖；CMA 路径要求可睡眠 |
| DMA 一致性 | 不自动保证，仍必须走 `dma_map_*()` 或正确 DMA API | 不自动保证，映射与 cache 属性需自行设计 | CMA 核心也不保证；经 `dma_alloc_coherent()` 使用时由 DMA 层处理 |
| 内核集成和统计 | 通用 slab/buddy 统计 | 需要平台和驱动自建管理、所有权、映射和统计 | 已集成 DMA API、DT、NUMA、trace、sysfs/debugfs |
| 主要代价 | 大块连续分配脆弱 | 永久损失这部分通用 RAM，峰值按最坏情况静态预留 | 迁移延迟、失败风险、限制非 movable 内存可用空间 |

本地内核文档明确说明 `kmalloc` 单块最大尺寸有限，并建议把它用于小于页大小的对象，见 `Documentation/core-api/memory-allocation.rst:131-148`。本地 CMA 源码则直接把“大块运行期物理连续分配”和“永久预留区空闲时无法被页系统使用”列为 CMA 要解决的两个问题，见 `kernel/dma/contiguous.c:11-35`。

#### 5.1.2 CMA 相对 kmalloc 的优势

`kmalloc()` 的优势是简单、低开销，适合控制结构、描述符、小缓冲和一般内核对象。它返回的内存 CPU 虚拟地址连续，底层物理页也连续；但当尺寸跨越多页时，最终要依赖 buddy 的高阶连续块。系统运行越久、物理页越碎片化，几十 MiB 甚至更大的直接连续分配越不现实。`GFP_KERNEL` 可以回收和 compact，却不能保证随时凑出目标高阶块。

CMA 比 `kmalloc` 更适合大块连续内存，原因不是它使用了一个更快的普通堆，而是它从启动时就保护了一批 pageblock：只允许容易迁移的页面进入，避免普通不可迁移内核对象长期污染这片地址空间。申请时 CMA 还会针对具体 PFN 范围执行隔离、回收和迁移，而不是仅在 buddy free list 中等待“碰巧存在”的大块。

因此，两者不是同尺寸下的平级替代：

- 数百字节、几 KiB 的普通内核对象：选择 `kmalloc/kzalloc`；
- 小型 DMA 缓冲：使用 DMA API 或 `dma_pool`，不要直接把 `kmalloc` 地址的物理地址交给设备；
- 几 MiB 到数百 MiB、设备要求物理连续、又需要运行期建立/销毁：CMA 才体现优势；
- 设备支持 SG/IOMMU：通常优先使用离散页和 DMA 映射，可能完全不需要 CMA。

虽然 `kmalloc` 得到的内存可以按 DMA API 规则用于 streaming DMA，但 CPU 地址、CPU 物理地址和 DMA 地址不是同一个概念。驱动仍应调用 `dma_map_single()` 等接口并使用返回的 `dma_addr_t`，不能用 `virt_to_phys()` 代替 DMA 映射。本地 DMA 指南见 `Documentation/core-api/dma-api-howto.rst:105-143`。

#### 5.1.3 CMA 相对 U-Boot 固定预留区的优势

如果 U-Boot 阶段直接划出 512 MiB，并通过 `/memreserve/`、`reserved-memory`、缩减可见 DRAM 等方式让 Linux 永远不管理它，那么无论设备是否工作，这 512 MiB 都不能用于页缓存、应用匿名内存或其它内核需求。如果设备只在少数场景短时间达到峰值，永久 carveout 的平均利用率会很低；多个设备各按最坏峰值独占预留时，浪费会叠加。

CMA 的优势就是减少这部分机会成本：设备空闲时，Linux 可以把 CMA area 用作 page cache 等 movable 内存；多个峰值不同时发生的设备还可以共用全局 CMA。对内存容量紧张、视频/摄像头只在部分时间工作、缓冲区生命周期动态变化的设备，这往往比永久 carveout 更有价值。

CMA 还提供标准化集成：

- DMA 层能根据 `dev->cma_area`、NUMA node 和默认池选择 area；
- 设备树 `memory-region` 可以把设备绑定到专用 CMA；
- `dma_alloc_coherent()` 等上层 API 能继续负责 DMA mask、映射和 cache 一致性；
- 释放后内存重新进入 Linux 管理，而不需要驱动自己维护完整的大块 allocator；
- sysfs、debugfs、vmstat 和 tracepoints 能观测成功率、容量和延迟。

但永久预留区确实有 CMA 无法取代的优点：没有迁移步骤、不会被普通页面暂借，容量规划正确时成功与延迟更可预测；它也更容易满足“固件和设备始终在固定地址访问”“Linux 绝不能建立普通映射”“复位期间设备仍在读写”“安全/可靠性认证需要静态内存地图”等要求。

因此“预留区域按理更好”只在评价指标是**确定性和隔离**时成立；如果评价指标包含**系统可用内存和平均利用率**，永久预留往往更差。两者的交换关系是：

```text
永久 carveout：用固定的内存浪费，换取更强的可预测性
CMA：用申请时迁移成本和非零失败概率，换取更高的内存利用率
```

#### 5.1.4 U-Boot 预留和 CMA 也可以组合

“U-Boot 划区域”不一定与 CMA 对立。常见的正确组合是：U-Boot 根据板卡内存布局修正设备树，向 Linux 传递一个 `reserved-memory` 节点；该节点使用：

```dts
compatible = "shared-dma-pool";
reusable;
linux,cma-default;
```

Linux 随后把 U-Boot 指定的物理区初始化为 CMA。这样 U-Boot/固件负责决定安全的地址窗口，Linux CMA 负责运行期复用和分配。

如果节点设置 `no-map`，或完全不带 `reusable`，它通常是永久 carveout，不是 CMA。`no-map` 与 `reusable` 不能同时使用。如果 U-Boot 只是私下保留内存，却没有通过 DT、EFI/固件 memory map 或其它启动协议告诉 Linux，Linux 仍可能把该区域纳入 buddy 并覆盖它，这是错误实现，不应作为一种可比较的方案。

#### 5.1.5 为什么要用 CMA——以及什么时候不该用

不存在“一定要用 CMA”。只有同时出现下面大部分条件时，CMA 才是很自然的选择：

1. 消费者需要真正的 CPU 物理连续页，而不仅是虚拟连续或连续 IOVA；
2. 单次缓冲区很大，`kmalloc`/普通高阶页在长期运行后成功率不足；
3. 缓冲区在运行期动态申请，无法只在早期一次性固定分配；
4. 设备有明显空闲期，希望把预留容量暂时还给 Linux 使用；
5. 申请路径可以睡眠，也能接受 migration 带来的延迟长尾和偶发失败；
6. 希望继续通过标准 DMA API、设备树和内核统计管理该内存。

以下情况应优先考虑永久预留或启动期预分配，而不是把 CMA 当成强制方案：

- 分配绝不能失败，且最坏时延必须严格可界定；
- 硬件/固件协议要求固定物理地址；
- 设备在 Linux 启动前、崩溃后或复位期间仍访问该内存；
- 内存必须与普通 Linux 映射完全隔离；
- 预留容量很小或几乎一直被设备占用，CMA 的复用收益很低；
- 可以在驱动初始化时一次性 `dma_alloc_coherent()`，随后自己做固定粒度池化，运行期不再向 CMA 申请。

以下情况通常三者都不是最优解：设备本来支持 scatter-gather，或者 IOMMU 可以把离散物理页映射成连续 IOVA。这时直接使用 SG/DMA 映射能避免预留大块物理连续 RAM。

一个实用决策顺序是：

```text
硬件能接受 SG 或 IOMMU 连续 IOVA？
  是 -> 优先普通页 + DMA/SG/IOMMU，不用 CMA
  否 -> 必须物理连续
       |
       +-- 必须固定地址、零失败、确定时延？
       |     是 -> 永久 carveout，或启动期预分配后自建池
       |
       +-- 运行期需要大块，且希望空闲容量供 Linux 使用？
             是 -> CMA
             否 -> 小块用 DMA API/kmalloc；固定大块用 carveout
```

对许多产品，最佳方案是混合设计：少量固件控制区使用固定 `no-map` carveout；大体积、可变生命周期的视频帧使用 CMA；大量小 DMA 描述符使用 `dma_pool`；可 SG 的数据缓冲使用普通页和 IOMMU。

### 5.2 优势

- 大块物理连续：突破 buddy 在长期运行后高阶块容易碎片化的限制。
- 可复用：设备不用时，movable 工作负载可借用，不像永久 carveout 那样完全浪费。
- 通用机制：从 DMA 专用实现演进为可被 HugeTLB、KVM、dma-buf heap 等使用的 CMA 核心。
- 多池隔离：全局池、设备专用池、NUMA 池可以降低互相抢占并满足地址限制。
- 可观测：dmesg、meminfo、vmstat、sysfs、debugfs、tracepoints 能从不同层面分析。
- 启动布局可控：命令行和设备树可限制容量、基址、地址窗口和 node。

### 5.3 局限

- 不保证成功：页面“最初可迁移”不代表申请时仍能迁移；长期 GUP pin、驱动映射、异常 refcount、设备页和某些复合页会成为障碍。
- 延迟长尾：同步 compaction、回收和 migration 会造成毫秒乃至更高长尾，取决于机器和负载。
- 容量静态：常规 area 在启动阶段预留，无法像一般堆一样弹性扩容。
- 非 movable 内存压力：CMA 比例过高会挤压内核不可迁移分配可用空间。
- 地址布局约束：必须单 zone、满足 pageblock 对齐；还可能与 DMA32、crashkernel、固件保留区、内存洞和 hotplug block 冲突。
- 并发限制：6.0.1 的全局 CMA mutex 会串行不同 area 的核心分配阶段。
- 内存下线限制：包含启动期 CMA 区的 memory block 通常不能正常 offline；当前官方 hotplug 文档明确把 CMA 与 ZONE_MOVABLE 的约束联系起来。[3]
- 不是缓存一致性层：DMA 非一致平台仍需遵循 DMA API 的同步规则和内存屏障。
- 不是安全边界：没有 IOMMU/MPU 时，设备仍可能 DMA 到池外。

### 5.4 最合适的场景

- 摄像头帧缓冲、编解码器、显示控制器；
- 不支持 SG 的老旧/简单 DMA 引擎；
- 大块连续固件工作区或硬件表；
- 某些 ARM/ARM64 平台 coherent DMA 后端；
- gigantic HugeTLB 页；
- 必须交付物理连续 dma-buf 的系统。

不适合：大量小对象、高频申请释放、atomic/IRQ 分配、可直接用 SG/IOMMU 的普通数据面，以及要求严格最坏时延但不预分配的路径。

## 6. 失败机理与排查方法

### 6.1 失败类型

| 现象 | 常见根因 | 判断重点 | 改进方向 |
|---|---|---|---|
| 启动时 `Failed to reserve` | 区间重叠、尺寸/对齐错误、地址窗口太小、跨 zone/low-high 边界 | dmesg、DT `reg/size/alignment/alloc-ranges`、内存图 | 改地址窗口；按 pageblock 对齐；避开 crashkernel/固件区 |
| area `could not be activated` | 位图分配失败、无效 PFN、跨 zone | dmesg、zone 边界 | 缩小/移动 area；核对 memory map |
| `ret=-EBUSY` | 候选范围存在无法迁移的页；迁移多次后仍失败 | `cma_alloc_busy_retry`、`test_pages_isolated`、page owner | 找 pin/unmovable 来源；更早预分配；拆池或调布局 |
| `ret=-ENOMEM` | CMA 位图无足够连续候选、系统无迁移目标、请求大于池 | area bitmap/maxchunk、meminfo、水位 | 增池、降峰值、减少长期占用；保留普通迁移空间 |
| atomic 路径始终不用 CMA | CMA 会睡眠 | 调用上下文和 GFP | 初始化期预分配，或用预建原子池 |
| 有大量 `CmaFree` 仍失败 | 对齐/地址限制、候选块被 pin、统计不表示最大连续可迁移块 | debugfs `maxchunk`、trace、DMA mask | 看具体 area 和请求，不只看全局总量 |
| CMA 成功但设备访问异常 | DMA mask、cache sync、地址类型混用，不一定是 CMA 本身 | CPU PA、DMA address、IOMMU、coherency | 回到 DMA API；不要把物理地址直接当 DMA 地址 |

### 6.2 6.0.1 的观测接口

1. 启动日志：

```text
dmesg | grep -i -E 'cma|reserved memory'
```

确认实际容量、基址、area 名称和启动失败。不要只看内核配置，因为命令行或 DT 可能覆盖。

2. 总量和当前 buddy 空闲量：

```text
grep -E 'CmaTotal|CmaFree' /proc/meminfo
```

3. 全局累计事件：

```text
grep -E '^cma_alloc_(success|fail)' /proc/vmstat
```

6.0.1 在 `mm/cma.c:513-520` 按调用次数增加 vm event，但 sysfs 是按请求页数累计，二者单位不同，分析时不能直接相除。

4. 每 area 累计页数：

```text
/sys/kernel/mm/cma/<area>/alloc_pages_success
/sys/kernel/mm/cma/<area>/alloc_pages_fail
```

实现见 `mm/cma_sysfs.c`。这两个数字是累计“页数”，不是调用次数；一次 64 MiB 失败会比多次小请求更显眼。

5. debugfs：

```text
/sys/kernel/debug/cma/cma-<name>/base_pfn
/sys/kernel/debug/cma/cma-<name>/count
/sys/kernel/debug/cma/cma-<name>/order_per_bit
/sys/kernel/debug/cma/cma-<name>/used
/sys/kernel/debug/cma/cma-<name>/maxchunk
/sys/kernel/debug/cma/cma-<name>/bitmap
/sys/kernel/debug/cma/cma-<name>/alloc
/sys/kernel/debug/cma/cma-<name>/free
```

这里以本地 6.0.1 的目录命名实现为准，见 `mm/cma_debug.c:163-200`；较新官方文档展示的命名/多 range 布局已经不同。[4] `alloc/free` 能主动改变内存状态，只用于受控压测，不要当只读监控。

6. tracepoints：

```text
cma:cma_alloc_start
cma:cma_alloc_finish
cma:cma_alloc_busy_retry
cma:cma_release
```

还应联合 `compaction`、`migration`、`mm_page_alloc`、`test_pages_isolated`、调度延迟和 direct reclaim 事件分析。定义见 `include/trace/events/cma.h`。

### 6.3 建议的定位顺序

1. 确认失败的是哪个 area，而不是只看全局 `CmaTotal`。
2. 记录请求字节数、页数、对齐 order、调用上下文、设备 DMA mask。
3. 对照 dmesg 确认池实际落点和大小。
4. 读取 per-area 成败页数、debugfs `used/maxchunk/bitmap`。
5. 开启 CMA 与 page-isolation trace，区分“没有候选区”和“候选区迁移失败”。
6. 用 page owner、page flags、必要时动态调试定位阻塞 PFN 的所有者。
7. 检查是否存在长期 `pin_user_pages()`、V4L2/DRM buffer 长时间占用、驱动泄漏或错误的 free count。
8. 在冷启动、稳定负载、极限内存压力三种阶段复现，比较延迟分布，不只统计平均值。

## 7. 容量和布局设计

### 7.1 容量公式

建议从并发峰值而非单次最大请求估算：

```text
CMA需求 ≈ Σ(各业务峰值并发缓冲区 × 单缓冲实际页对齐尺寸)
        + 管线重叠/双缓冲/三缓冲
        + 生命周期交叠
        + 对齐与位图粒度损耗
        + 迁移与碎片安全余量
```

不要把“理论图像数据大小”当实际申请大小。应包含 stride、plane 对齐、metadata、压缩头、guard、固件和并行通道。用真实 trace 的高分位峰值反推容量更可靠。

### 7.2 全局池还是专用池

全局池适合请求峰值错开的设备共享容量，内存利用率高；缺点是一个驱动能把池占满，故障域大。专用池能保证地址窗口和容量隔离，便于按设备监控；缺点是不同设备间不能自然借用已经由 CMA 客户占用的容量，预留总量更大。

如果多个设备 DMA mask 不同，应优先分开布局。把全局池全部放到 4 GiB 以上，会让 32 位寻址设备无法使用；把巨大的池压在 DMA32，又会挤压有限低地址内存。NUMA 系统还要考虑设备亲和 node，否则远端 coherent buffer 会增加延迟。

### 7.3 order_per_bit

一个 CMA 位图 bit 可以代表 `2^order_per_bit` 页。较大的 `order_per_bit` 减小位图、适合只做大粒度分配，但请求会向上取整并增加内部碎片，部分释放也更受限。普通 DMA CMA 常用 0；除非有明确的大页/平台需要，不宜随意调大。

### 7.4 生产建议

- 关键 DMA 缓冲区在设备 probe/stream start 等可控阶段预分配，避免在帧关键路径申请。
- 设置正确的 coherent/streaming DMA mask，并检查返回 DMA 地址，不自行 `virt_to_phys()` 代替 DMA API。
- 给 CMA 分配建立超时观测和降级路径，例如降低分辨率/并发数，而不是无界重试。
- 持续监控每 area 的失败页数增量和分配延迟分位数。
- 在接近业务上限时做内存压力与长期 pin 联合测试。
- 升级内核后重新验证，因为 compaction、migration、folio、pageblock 和 CMA 锁均持续演进。

## 8. 版本发展史

“每一个 Linux 小版本”并不都修改 CMA。下面按首次包含重要 CMA 能力或语义变化的发布代际整理；未列出的版本主要是零散修复、机械重构或没有 CMA 架构变化。版本边界以主线合入周期归类，精确回移情况仍应以具体发行版内核的 commit 为准。

| 版本/时期 | 主要变化 | 相比上一代的优点 | 当时仍有的不足及后续改进 |
|---|---|---|---|
| 合入前，2009–2011 | 围绕嵌入式多媒体大块物理连续内存，经历多轮 patch 讨论；目标是替代利用率差的静态 carveout | 确立“早期保留、运行期可借用、需要时迁移”的模型 | 与 page allocator、compaction、DMA mapping 的接口和可证明行为尚未成熟 |
| Linux 3.5，2012 | CMA 正式合入；同时引入/完善 `MIGRATE_CMA`、page isolation、`alloc_contig_range()`，ARM 与 x86 DMA mapping 接入。[5][6] | 系统运行后仍能请求大块连续物理内存；空闲池可被 movable 页利用 | 当时仍标为 experimental；实现偏 DMA 专用；对齐、锁和失败处理随后频繁修正 |
| 3.6–3.8 | 修复连续区对齐、类型宽度，重构搜索与分配路径 | 避免错误对齐和大物理地址截断，早期实现更可靠 | 单一 DMA 场景耦合仍强；缺少多用途 area 管理 |
| 3.16–3.17，2014 | `cma=` 增加地址放置语法；CMA core 与 DMA API 分离；支持通用自定义 area、alignment、任意 bitmap 粒度。[7] | CMA 从“DMA 私有帮助函数”变为可被 KVM/HugeTLB 等复用的内存核心能力；能建立多个池 | 配置仍偏平台代码，用户难以从 DT 描述；边界和激活错误路径仍在完善 |
| 3.18，2014 | 支持从 reserved-memory 的 `shared-dma-pool` 设备树节点初始化 CMA。[8] | 板级内存拓扑、设备绑定和默认池可由 DT 表达 | DT 对 `reusable/no-map`、zone、对齐和优先级要求严格，错误配置容易只在启动日志暴露 |
| 4.1，2015 | 加入 CMA debugfs、人工 allocation trigger 和 CMA alloc/free trace events。[9] | 首次能从用户空间系统性查看和压测 area | debugfs 不是稳定 ABI；生产监控仍缺少轻量累计指标 |
| 4.11–4.12，2017 | 分配失败打印原因与位图；一度传递 GFP mask；给 area 命名并支持遍历 | 多 area 故障定位改善 | `cma_alloc()` 实际不支持完整 GFP 语义，容易误以为 `__GFP_ZERO` 等有效 |
| 4.19，2018 | 把 `cma_alloc()` 的 `gfp_mask` 改成明确的 `no_warn`，修正 API 误导；DMA mapping 源码迁到 `kernel/dma/`。[10] | API 契约更真实，避免未清零缓冲等错误假设 | 调用者仍重复 count/align/blocking/fallback 逻辑 |
| 5.3，2019 | 增加 `dma_alloc_contiguous()` / `dma_free_contiguous()` 封装；单页请求绕过 CMA。[11] | 统一阻塞检查、对齐、释放和 buddy fallback；降低小请求造成的 CMA 碎片 | NUMA 机器仍主要依赖一个全局池，远端内存访问开销明显 |
| 5.7，2020 | CMA core 增加按 NUMA node 声明 area 的接口 | 为后续本地化 DMA CMA 打下基础，预留可限定到目标 node | DMA 层尚未建立自动的每 node 默认池 |
| 5.10，2020 | DMA 层加入 per-NUMA CMA 和 `cma_pernuma=`。[12] | 设备优先获得本地连续内存；上游测试中 ARM SMMU CMD_SYNC 延迟从约 560 ns 降至 240 ns | 所有 node 只能配置相同大小；area 数量与预留总量增加；失败仍回退全局池 |
| 5.12，2021 | 非固定 CMA area 优先从 node 较低地址、4 GiB 以上 bottom-up 布局。[13] | compaction 更倾向把页迁出 CMA，提升成功率；避开 DMA32 | 小内存或选址失败仍回退旧策略；固定 DT 区不自动获得该布局收益 |
| 5.13，2021 | 加入 `/proc/vmstat` 成败事件、per-area sysfs 成败页数、带 area 名称的 trace 和性能 trace。[14] | 可以做长期健康监控并区分具体池 | 成败累计值不能解释失败 PFN 所有者；仍需 trace/page owner 深挖 |
| 5.18，2022 | 以 `pageblock_order` 统一最小对齐；支持在 area 激活失败时选择不把保留页交给 buddy。[15] | 减少不同体系结构对齐规则分裂；支持 fadump 等必须保留内存的用户 | CMA 失败处理仍需要每个特殊用户明确策略 |
| 6.0.1，2022（本地基线） | 已具备多 area、DT、全局和 per-NUMA DMA CMA、HugeTLB CMA、debugfs/sysfs/vmstat/trace；分配仍使用全局 `cma_mutex` | 功能成熟、可调试，适合大多数嵌入式 DMA 场景 | `cma_pernuma` 只能统一大小；没有多物理 range area、folio API；不同池分配串行 |
| 6.6，2023 | `numa_cma=<node>:<size>,...` 可按指定 node 配置不同容量，per-NUMA CMA 推广到更多体系结构。[16] | 异构 NUMA/设备拓扑可以精细容量规划 | 配置与 fallback 路径更复杂，运维需逐 node 监控 |
| 6.9，2024 | 删除高噪声 `CONFIG_CMA_DEBUG`；sysfs 增加 release 成功页数；修正参数和 trace 边界 | 依靠动态调试/trace 替代编译期刷屏，释放侧记账更完整 | debugfs 仍非稳定 ABI；旧监控脚本需适配 |
| 6.12，2024 | CMA 增加 large-folio 分配/释放接口。[17] | 与 folio 化内存管理和 HugeTLB 连续页处理衔接 | 后续发现 HugeTLB 更适合“冻结 refcount”接口，该 folio 接口又被替换 |
| 6.15，2025 | 可选的单 CMA area 多物理 range（最多 8 个），主要解决超大 hugetlb CMA 遇到物理内存洞；导出 total/free；全局分配锁改为 per-CMA 锁。[18][19] | 大内存机器跨 memory hole 的预留成功率提高；不同 area 真正并发，提交测试从约 7/14/21 s 改善到约 7/8/7 s | 多 range 只对新接口开放，单次返回仍必须落在某个连续 range；普通 DMA 用户不能假定 area 自身整体连续 |
| 6.18，2025 | 拒绝返回 PFN 连续但 `struct page` 数组不连续的特殊 SPARSEMEM 区间。[20] | 明确并强化调用者可顺序遍历 page 的安全契约 | 极少数无 SPARSEMEM_VMEMMAP 的配置可用候选区减少 |
| 7.0，2026 | 为 HugeTLB 增加 `cma_alloc_frozen[_compound]()` / `cma_release_frozen()`，直接以冻结 refcount 形式交接连续页，取代短命的 CMA folio API。[21] | 减少不必要的 refcount 建立/拆除，更贴合 HugeTLB 页生命周期 | 属于新 API，树外代码必须按目标内核适配，不能把 6.0.1 用法直接复制过去 |
| 7.1–7.2，2026 | CMA/DMA contiguous 接口进一步模块化并导出；reusable CMA 与 dma-buf heap 注册、默认 heap 保留、DT/cmdline 协调继续整理；补充激活失败泄漏和 NUMA 参数截断等修复 | CMA 更容易作为模块化内核服务和 dma-buf heap 后端使用，边界错误更少 | 行为仍在快速演进；生产升级应验证 heap 暴露、安全策略、命名和监控路径 |
| 7.3-rc2，2026-09 | 当前主线开发状态；CMA 已拥有多 range、per-area 并发、NUMA 精细配置、冻结页接口和更完整可观测性 | 相比 6.0.1，超大内存、并发与 HugeTLB 场景明显增强 | rc 版本不应直接视为稳定生产基线；应以目标 stable/LTS 的实际 backport 为准 |

说明：厂商内核、Android common、发行版内核会回移部分提交，所以“内核版本号”只能说明首次主线代际。判断项目能力的可靠方法是检查对应配置、函数和 commit，而不是只比较 `uname -r`。

## 9. 6.0.1 与当前主线的关键差异

| 维度 | 本地 6.0.1 | 2026 当前主线 |
|---|---|---|
| area 物理布局 | 每 area 一个连续范围 | 可选多 range，主要供 hugetlb CMA 规避大内存洞 |
| 分配串行化 | 全局 `cma_mutex` | per-CMA allocation mutex，不同 area 可并行 |
| NUMA 参数 | `cma_pernuma=`，所有 node 同大小 | 另有 `numa_cma=`，可指定 node 与不同大小，并持续修复截断/默认配置问题 |
| API 表示 | `struct page *`，普通 refcount；有 `cma_pages_valid()` | large-folio 过渡后引入 frozen/compound 接口；部分核心 API 显式导出；移除旧校验帮助函数 |
| debug | `CONFIG_CMA_DEBUG` + debugfs/sysfs/trace | 删除 `CONFIG_CMA_DEBUG`，依赖动态调试、trace 和更丰富统计 |
| debugfs 布局 | 单 bitmap、`cma-<name>` | 多 range 信息和当前官方文档布局 |
| dma-buf heap | 树中已有 CMA heap 驱动，但注册模型较早 | reusable/default CMA 与 heap 注册进一步统一 |

这意味着从 6.0.1 升级时，驱动若只使用标准 DMA API，迁移成本通常较低；若直接访问 `struct cma` 内部字段、假设单一 `base_pfn/bitmap`、解析 debugfs 路径或调用未导出的 CMA core API，升级风险明显更高。

## 10. 对当前项目的建议清单

当前源码树未发现根目录 `.config`，因此还不能判断你的实际板级 CMA 大小、地址和设备绑定。下一步项目审计建议收集：目标 SoC/架构、最终 `.config`、实际 DTB 对应 DTS、bootargs、启动 dmesg、`/proc/meminfo`、`/proc/vmstat`、`/sys/kernel/mm/cma`、debugfs CMA 目录，以及具体驱动的分配调用栈。

按风险优先级检查：

1. 驱动是否绕过 DMA API 直接使用 `cma_alloc()`，以及为何必须这样做。
2. 分配是否发生在 atomic/IRQ、持锁或实时关键路径。
3. 是否正确设置 DMA mask/coherent mask，是否误把 CPU 物理地址当 DMA 地址。
4. 设备专用池是否真的通过 `memory-region` 绑定，还是所有设备争用全局池。
5. CMA 区是否位于设备可寻址窗口，是否挤压 DMA32 或跨 zone。
6. 是否存在长期 pin、泄漏、缓冲区生命周期重叠或错误释放页数。
7. 容量是否按并发峰值而非单帧估算。
8. 压力测试是否覆盖长期运行后的碎片状态和低内存状态。
9. 失败是否有可执行降级路径，并记录请求 size/align/area/耗时。
10. 若计划升级到 6.6/6.12/6.18/7.x，是否能利用 `numa_cma`、per-area 并发或新 HugeTLB 接口，且避免依赖 debugfs/内部结构。

## 11. 6.0.1 源码阅读地图

建议按下面顺序阅读：

1. `kernel/dma/contiguous.c:1-36`：设计动机。
2. `mm/Kconfig:822-867`、`kernel/dma/Kconfig:118-196`：能力开关和默认值。
3. `kernel/dma/contiguous.c:72-203`：命令行、per-NUMA 和早期默认池预留。
4. `mm/cma.c:162-378`：area 建立、地址/zone/对齐约束与 memblock 选址。
5. `mm/cma.c:96-153`、`mm/page_alloc.c:2305-2323`：area 激活与交回 buddy。
6. `include/linux/mmzone.h:42-80`：`MIGRATE_CMA` 定义。
7. `mm/cma.c:415-523`：位图选择、重试、统计。
8. `mm/page_alloc.c:9143-9344`：`alloc_contig_range()` 隔离和迁移本体。
9. `kernel/dma/contiguous.c:245-370`：DMA 层选择设备池、NUMA 池和全局池。
10. `kernel/dma/contiguous.c:372-441`、`drivers/of/of_reserved_mem.c:90-145`：设备树 reserved-memory 集成。
11. `mm/cma_debug.c`、`mm/cma_sysfs.c`、`include/trace/events/cma.h`：观测与压测。
12. `drivers/dma-buf/heaps/cma_heap.c`、`mm/hugetlb.c`：两个重要的实际 CMA 用户。

## 12. Sources

1. Linux Kernel Archives, [kernel.org 首页与当前版本状态](https://www.kernel.org/), accessed 2026-09-13.
2. Linux kernel, [Reserved-memory shared DMA pool binding](https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/reserved-memory/shared-dma-pool.yaml).
3. Linux kernel documentation, [Memory Hot(Un)Plug](https://docs.kernel.org/admin-guide/mm/memory-hotplug.html).
4. Linux kernel documentation, [CMA Debugfs Interface](https://docs.kernel.org/admin-guide/mm/cma_debugfs.html).
5. Marek Szyprowski et al., [drivers: add Contiguous Memory Allocator](https://github.com/torvalds/linux/commit/c64be2bb1c6eb43c838b2c6d57b074078be208dd), Linux mainline commit.
6. LWN, [3.5 merge window part 2](https://lwn.net/Articles/498693/), 2012; and [KS2012: ARM: DMA mapping](https://lwn.net/Articles/513939/), 2012.
7. Joonsoo Kim, [DMA, CMA: separate core CMA management codes from DMA APIs](https://github.com/torvalds/linux/commit/3162bbd7e65b9cc57b660796dd3409807bfc9070), Linux mainline commit.
8. Marek Szyprowski, [drivers: dma-contiguous: add initialization from device tree](https://github.com/torvalds/linux/commit/de9e14eebf33a60712a52a0bc6e08c043c0aba53), Linux mainline commit.
9. Sasha Levin, [mm: cma: debugfs interface](https://github.com/torvalds/linux/commit/28b24c1fc8c22cabe5b8a16ffe6a61dfce51a1f2), Linux mainline commit.
10. Marek Szyprowski, [mm/cma: remove unsupported gfp_mask parameter from cma_alloc()](https://github.com/torvalds/linux/commit/6518202970c1052148daaef9a8096711775e43a2), Linux mainline commit.
11. Nicolin Chen, [dma-contiguous: add dma_{alloc,free}_contiguous() helpers](https://github.com/torvalds/linux/commit/b1d2dc009dece4cd7e629419b52266ba51960a6b), Linux mainline commit.
12. Barry Song, [dma-contiguous: provide the ability to reserve per-numa CMA](https://github.com/torvalds/linux/commit/b7176c261cdbced87bed9562577333150ed05b01), Linux mainline commit.
13. Roman Gushchin, [mm: cma: allocate cma areas bottom-up](https://github.com/torvalds/linux/commit/df2ff39e78da74dc23e7187dd58a784d91a876e0), Linux mainline commit.
14. Minchan Kim et al., [mm: cma: support sysfs](https://github.com/torvalds/linux/commit/43ca106fa8ec7d684776fbe561214d3b2b7cb9cb), Linux mainline commit.
15. Hari Bathini, [mm/cma: provide option to opt out from exposing pages on activation failure](https://github.com/torvalds/linux/commit/27d121d0ec6d604d0147c5b579e4181b688a2d64), Linux mainline commit.
16. Yajun Deng, [dma-contiguous: support numa CMA for specified node](https://github.com/torvalds/linux/commit/bf29bfaa54901a4bdee2a18cd10eb951a884a5f9), Linux mainline commit.
17. Yu Zhao, [mm/cma: add cma_{alloc,free}_folio()](https://github.com/torvalds/linux/commit/463586e9ff398f951a9a63d98a3b93f44434d20c), Linux mainline commit.
18. Frank van der Linden, [mm, cma: support multiple contiguous ranges, if requested](https://github.com/torvalds/linux/commit/c009da4258f9885c5a3749fc004870db9c0e7a99), Linux mainline commit.
19. Ge Yang, [mm/cma: using per-CMA locks to improve concurrent allocation performance](https://github.com/torvalds/linux/commit/24ac6fb6e3647fff3646b3ea1811095441380560), Linux mainline commit.
20. David Hildenbrand, [mm/cma: refuse handing out non-contiguous page ranges](https://github.com/torvalds/linux/commit/6972706f95926838f9bd3ec2b2393c034bdb85ba), Linux mainline commit.
21. Kefeng Wang, [mm: cma: add cma_alloc_frozen{_compound}()](https://github.com/torvalds/linux/commit/9bda131c6093e9c4a8739e2eeb65ba4d5fbefc2f), Linux mainline commit.
