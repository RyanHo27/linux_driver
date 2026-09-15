# Linux/glibc `dlopen` 实现源码分析

## 1. 结论

在 GNU/Linux 系统中，`dlopen()` 不是 Linux 系统调用，也不是由 Linux 内核直接实现的“加载动态库”接口。

它由 glibc 和运行时动态链接器 `ld.so` 实现：

- ELF 解析、依赖展开、符号查找、符号版本检查、重定位计算、TLS 管理和构造函数调度，主要由用户态的 `ld.so` 完成。
- 打开文件、读取文件、建立虚拟内存映射、修改页面权限和处理缺页异常，需要通过系统调用或异常进入内核态。

因此，最准确的说法是：

> `dlopen` 的动态链接算法在用户态执行，但整个加载过程会多次进入内核态，请求文件系统和虚拟内存服务。

Linux 中不存在 `dlopen` 系统调用。它是用户态动态链接器主导、内核提供基础机制的一项协作过程。

本文以 glibc 当前源码主线为基础。不同 glibc 版本的内部细节可能变化，但总体流程基本一致。

## 2. 核心源码位置

- [`dlfcn/dlopen.c`](https://github.com/bminor/glibc/blob/master/dlfcn/dlopen.c)：公开 `dlopen` API 的入口
- [`dlfcn/dlerror.c`](https://github.com/bminor/glibc/blob/master/dlfcn/dlerror.c)：`dlerror` 和错误捕获接口
- [`elf/dl-open.c`](https://github.com/bminor/glibc/blob/master/elf/dl-open.c)：`dlopen` 主控制流程
- [`elf/dl-load.c`](https://github.com/bminor/glibc/blob/master/elf/dl-load.c)：查找、打开、校验和解析 ELF
- [`elf/dl-map-segments.h`](https://github.com/bminor/glibc/blob/master/elf/dl-map-segments.h)：映射 `PT_LOAD` 段
- [`elf/dl-deps.c`](https://github.com/bminor/glibc/blob/master/elf/dl-deps.c)：处理 `DT_NEEDED` 依赖
- [`elf/dl-version.c`](https://github.com/bminor/glibc/blob/master/elf/dl-version.c)：符号版本检查
- [`elf/dl-reloc.c`](https://github.com/bminor/glibc/blob/master/elf/dl-reloc.c)：通用重定位流程
- [`elf/dl-lookup.c`](https://github.com/bminor/glibc/blob/master/elf/dl-lookup.c)：动态符号查找
- [`elf/dl-init.c`](https://github.com/bminor/glibc/blob/master/elf/dl-init.c)：执行共享库构造函数
- [`elf/dl-close.c`](https://github.com/bminor/glibc/blob/master/elf/dl-close.c)：卸载和失败回滚
- [`include/link.h`](https://github.com/bminor/glibc/blob/master/include/link.h)：内部 `link_map` 扩展定义
- [`sysdeps/generic/ldsodefs.h`](https://github.com/bminor/glibc/blob/master/sysdeps/generic/ldsodefs.h)：动态链接器内部接口和全局状态
- [`sysdeps/x86_64/dl-machine.h`](https://github.com/bminor/glibc/blob/master/sysdeps/x86_64/dl-machine.h)：x86-64 架构相关重定位

glibc 完整源码镜像：[`bminor/glibc`](https://github.com/bminor/glibc)

glibc 官方项目入口：[The GNU C Library](https://sourceware.org/glibc/)

接口语义参考：[`dlopen(3)` Linux manual page](https://man7.org/linux/man-pages/man3/dlopen.3.html)

## 3. 总体调用链

```text
应用程序调用 dlopen(filename, mode)
        │
        ▼
___dlopen
        │
        ▼
dlopen_implementation
        │
        ├─ 建立 dlerror 所需的错误捕获环境
        ▼
dlopen_doit
        │
        ▼
GLRO(dl_open) → ld.so 中的 _dl_open
        │
        ├─ 校验参数并取得加载锁
        ├─ 确定命名空间和调用者 link_map
        ├─ 检查目标对象是否已经加载
        ▼
dl_open_worker
        │
        ▼
dl_open_worker_begin
        │
        ├─ _dl_map_new_object       查找、打开、验证并映射 ELF
        ├─ _dl_map_object_deps      展开 DT_NEEDED 依赖
        ├─ _dl_check_map_versions   检查符号版本
        ├─ _dl_relocate_object      符号解析与重定位
        ├─ 更新符号作用域和 TLS
        ├─ 通知调试器和审计模块
        └─ _dl_init                 执行构造函数
        │
        ▼
返回 link_map 指针作为不透明 handle
```

## 4. 公开入口 `dlopen`

入口源码位于 `dlfcn/dlopen.c`。

保留核心含义的等价精简代码如下。它用于说明调用关系，不是对 glibc 文件的逐字复制：

```c
void *conceptual_dlopen_entry(const char *file, int mode)
{
    void *caller = get_return_address();

    if (dynamic_linker_hook_exists())
        return hook_dlopen(file, mode, caller);

    struct args args = {
        .file = file,
        .mode = mode,
        .caller = caller,
    };

    if (run_with_dlerror_handler(dlopen_worker, &args))
        return NULL;

    return args.result;
}
```

这一层主要完成：

1. 保存调用 `dlopen()` 的代码地址。
2. 检查 `mode` 是否只包含允许的标志。
3. 建立线程局部的错误捕获环境。
4. 通过动态链接器函数表调用 `_dl_open()`。

保存调用者地址很重要。当文件名没有 `/` 时，动态链接器可能需要使用发起调用的共享对象的 `RPATH/RUNPATH`；`$ORIGIN` 的展开也需要知道调用者属于哪个 ELF 对象。

从 glibc 2.34 开始，`dlopen` 的主符号位于 `libc.so.6` 中。glibc 仍保留旧 `libdl` 符号的兼容版本。

### 4.1 `file == NULL`

当 `file` 为 `NULL` 时，`dlopen` 返回代表主程序/全局符号作用域的句柄，而不是加载一个新文件。

## 5. 进入 `_dl_open`

主函数位于 `elf/dl-open.c`：

```c
void *conceptual_dl_open(file, mode, caller, namespace)
{
    validate_mode(mode);
    lock_recursive(loader_lock);

    caller_map = find_map_containing(caller);
    map = find_already_loaded_object(namespace, file);

    if (map_is_fully_open(map, mode)) {
        map->direct_open_count++;
        unlock(loader_lock);
        return map;
    }

    error = catch_exception(dl_open_worker, &args);

    if (error) {
        rollback_with_dl_close_worker(args.map);
        unlock(loader_lock);
        raise_dlerror(error);
    }

    unlock(loader_lock);
    return args.map;
}
```

### 5.1 校验模式

`mode` 必须至少指定一种绑定方式：

- `RTLD_LAZY`：允许延迟解析可延迟的 PLT 函数引用。
- `RTLD_NOW`：`dlopen` 返回前完成所需符号解析。

其他常见标志：

- `RTLD_LOCAL`：默认行为，新对象通常不成为后续对象的全局符号提供者。
- `RTLD_GLOBAL`：把新对象及适用依赖加入全局符号作用域。
- `RTLD_NOLOAD`：只查询对象是否已经加载，不加载新对象。
- `RTLD_NODELETE`：`dlclose` 后仍保留对象映射和状态。
- `RTLD_DEEPBIND`：优先使用新对象自身作用域中的定义，是 GNU 扩展。

### 5.2 获取加载锁

glibc 使用递归加载锁保护：

- 命名空间中的 `link_map` 链表；
- 全局和局部符号作用域；
- TLS 模块编号及相关表；
- 对象引用和打开计数；
- 调试器可见的加载状态。

采用递归锁，是因为共享库构造函数、IFUNC resolver 或审计接口可能再次触发 `dlopen()`。

### 5.3 确定调用者和命名空间

动态链接器根据传入的调用者地址寻找其所属 `link_map`。该结果影响：

- 使用哪个动态链接命名空间；
- 从哪个对象的 `RPATH/RUNPATH` 开始搜索；
- `$ORIGIN` 相对于哪个对象展开。

### 5.4 已加载对象快速路径

`_dl_lookup_map()` 遍历相应命名空间中的已加载对象，比较请求名称、别名和 `DT_SONAME`。

如果目标已经完整加载，通常只执行：

```c
++map->l_direct_opencount;
return map;
```

不会再次执行 `mmap`、重定位和构造函数。如果本次新增了 `RTLD_GLOBAL` 或 `RTLD_NODELETE`，glibc 仍可能升级对象状态。

## 6. 查找共享库文件

新对象由 `elf/dl-load.c` 中的 `_dl_map_new_object()` 处理。

### 6.1 文件名包含 `/`

例如：

```c
dlopen("./plugin/libdemo.so", RTLD_NOW);
dlopen("/opt/demo/libdemo.so", RTLD_NOW);
```

动态链接器会：

1. 展开 `$ORIGIN`、`$LIB`、`$PLATFORM` 等动态字符串 token。
2. 直接打开展开后的路径。
3. 不执行普通的共享库目录搜索。

### 6.2 文件名不包含 `/`

例如：

```c
dlopen("libdemo.so", RTLD_NOW);
```

主要搜索顺序为：

1. 如果调用者没有 `DT_RUNPATH`，沿加载者链查找旧式 `DT_RPATH`。
2. 在适用情况下检查主程序的 `DT_RPATH`。
3. 检查 `LD_LIBRARY_PATH`。
4. 检查调用者对象的 `DT_RUNPATH`。
5. 查询 `/etc/ld.so.cache`。
6. 搜索默认目录，例如 `/lib64`、`/usr/lib64` 或对应架构目录。

需要注意：

- `DT_RPATH` 可以沿加载者链参与搜索。
- `DT_RUNPATH` 通常只用于该对象的直接依赖。
- 对象存在 `DT_RUNPATH` 时，不再使用该对象自己的 `DT_RPATH`。
- 安全执行模式会忽略或限制某些环境变量。
- `DF_1_NODEFLIB` 可以禁止默认目录。
- `LD_AUDIT` 模块可以观察甚至修改对象搜索名称。

## 7. 打开和验证 ELF

找到候选文件后，`open_verify()` 和 `_dl_map_object_from_fd()` 会验证：

- ELF magic `0x7f E L F`；
- ELF 是 32 位还是 64 位；
- 大端或小端；
- ELF 格式版本；
- 目标体系结构，例如 `EM_X86_64`；
- 程序头是否合法；
- 是否具有动态段；
- ELF 类型是否适合动态加载；
- 是否设置了禁止动态打开的 `DF_1_NOOPEN`；
- 是否错误地把 PIE 可执行文件作为共享库加载；
- 实际文件是否已经通过其他名字加载。

普通共享库一般是 `ET_DYN`。普通可执行文件和不允许被动态打开的 ELF 对象会被拒绝。

这部分会通过 `openat`、`read/pread`、`fstat` 和 `close` 等系统调用进入内核态。文件权限、挂载属性、LSM 和其他安全检查也由内核完成。

## 8. 创建 `link_map`

每个已加载 ELF 对象都对应一个动态链接器内部的 `struct link_map`。概念上包含：

```c
struct conceptual_link_map {
    ElfW(Addr) load_bias;
    char *file_name;
    ElfW(Dyn) *dynamic_section;
    struct conceptual_link_map *next;
    struct conceptual_link_map *previous;

    /* glibc 内部还保存：
       程序头、地址范围、依赖关系、符号作用域、
       TLS 信息、重定位状态、构造函数状态和引用计数等。 */
};
```

glibc 会根据实际文件身份再次去重，防止同一文件通过软链接、相对路径或不同名称重复映射。

`dlopen()` 返回的 `void *handle` 在 glibc 内部实质上对应一个 `link_map` 指针。但它对应用程序是不透明的，应用程序不应解释其内部布局，只应传给 `dlsym()`、`dlclose()` 或 `dlinfo()`。

## 9. 解析 ELF program header

动态链接器读取 program header，重点处理：

- `PT_LOAD`：需要映射到进程地址空间的段；
- `PT_DYNAMIC`：动态链接元数据；
- `PT_PHDR`：程序头自身的位置；
- `PT_TLS`：线程局部存储模板；
- `PT_GNU_RELRO`：重定位后需要变为只读的区域；
- `PT_GNU_STACK`：栈权限要求；
- `PT_GNU_PROPERTY`：CET、BTI 等体系结构属性；
- `PT_NOTE`：ABI、build-id 和 GNU property 等信息。

运行时装载主要依据 program header，而不是 section header。`.text`、`.data`、`.got` 等 section 对链接、反汇编和调试很有帮助，但真正指导运行时映射的是 `PT_LOAD` 等 segment。

## 10. 映射 `PT_LOAD` 段

核心实现位于 `elf/dl-map-segments.h`。

对于 `ET_DYN` 对象，主要过程是：

1. 根据所有 `PT_LOAD` 计算完整映射范围。
2. 第一个映射通常不使用固定地址，由内核选择可用地址，从而支持 ASLR。
3. 计算加载偏移：

   ```text
   l_addr = 实际映射起点 - ELF 第一个 LOAD 段虚拟地址
   ```

4. 按各 `PT_LOAD` 的相对位置映射其他段。
5. 根据 `p_flags` 设置保护权限：

   ```text
   PF_R → PROT_READ
   PF_W → PROT_WRITE
   PF_X → PROT_EXEC
   ```

6. 对文件中没有内容、内存中需要存在的 `.bss` 区域进行清零或匿名映射。
7. 将 segment 之间需要保留的地址洞设为 `PROT_NONE`。
8. 保存 `l_map_start`、`l_map_end`、`l_addr` 和程序头地址。

运行时地址的基本关系是：

```text
ELF 中的链接地址 + l_addr = 进程中的实际地址
```

例如：

```text
符号 ELF 值：  0x2300
本次加载偏移： 0x7f1200000000
运行时地址：   0x7f1200002300
```

映射动作通过 `mmap` 系统调用由内核完成；如何解释 ELF、如何划分映射以及使用什么权限，则由用户态动态链接器决定。

## 11. 按需分页和缺页异常

`mmap()` 通常不会立即把共享库的全部内容读入物理内存。它主要建立虚拟内存区域和文件映射关系：

```text
进程虚拟地址范围 → 共享库文件的某个偏移范围
```

第一次访问尚未驻留的代码或数据页时，会发生：

```text
CPU 访问共享库页面
        │
        ▼
产生缺页异常并进入内核
        │
        ▼
内核从页缓存或存储设备取得页面
        │
        ▼
更新页表
        │
        ▼
返回用户态，重新执行发生缺页的指令
```

所以即使 `dlopen()` 已经返回，之后第一次执行共享库中的某段代码时，仍可能因为缺页进入内核。

只读代码页还可以在多个进程之间共享物理页；经过私有写入的页面遵循写时复制语义。

## 12. 解析 `PT_DYNAMIC`

映射完成后，glibc 调整动态段指针并读取 `PT_DYNAMIC` 中的标签，包括：

```text
DT_NEEDED       依赖的共享库名称
DT_STRTAB       动态字符串表
DT_SYMTAB       动态符号表
DT_GNU_HASH     GNU 符号哈希表
DT_HASH         System V 符号哈希表
DT_REL/DT_RELA  普通重定位表
DT_JMPREL       PLT 重定位表
DT_PLTGOT       GOT/PLT 信息
DT_INIT         单个初始化函数
DT_INIT_ARRAY   构造函数数组
DT_FINI_ARRAY   析构函数数组
DT_RPATH        旧式搜索路径
DT_RUNPATH      新式搜索路径
DT_SONAME       共享库逻辑名称
DT_VERSYM       符号版本索引
DT_VERNEED      所需符号版本
DT_VERDEF       本对象提供的符号版本
DT_FLAGS        动态标志
DT_FLAGS_1      NOOPEN、NODEFLIB、NODELETE 等标志
```

这些条目被整理并缓存到 `link_map` 的内部字段中，后面的依赖处理、查符号、重定位和初始化都依赖这些信息。

## 13. 展开 `DT_NEEDED` 依赖

入口是 `elf/dl-deps.c` 中的 `_dl_map_object_deps()`。

它会：

1. 把刚加载的对象加入工作列表。
2. 遍历该对象的 `DT_NEEDED`。
3. 对每个依赖调用 `_dl_map_object()`。
4. 继续展开新依赖自身的 `DT_NEEDED`。
5. 去除重复对象。
6. 建立扁平依赖搜索表 `l_searchlist`。
7. 建立构造和析构顺序表 `l_initfini`。

glibc 源码采用类似广度优先的方式展开依赖。例如：

```text
libplugin.so
 ├─ libA.so
 │   └─ libC.so
 └─ libB.so
     └─ libC.so
```

搜索列表可能是：

```text
plugin → A → B → C
```

`libC.so` 只加入一次。随后依赖图还会被排序，以满足重定位、构造函数和析构函数的顺序要求。

每一个尚未加载的依赖都会重复经历“搜索路径、打开、验证、创建 `link_map`、映射 ELF”的过程。

## 14. 检查符号版本

glibc 调用 `_dl_check_map_versions()` 检查：

- `DT_VERNEED`：引用者要求的版本；
- `DT_VERDEF`：提供者定义的版本；
- `DT_VERSYM`：动态符号对应的版本索引。

例如引用者要求：

```text
memcpy@GLIBC_2.14
```

只找到名字为 `memcpy` 的符号还不够，提供者必须同时具有兼容的 `GLIBC_2.14` 版本定义，否则 `dlopen()` 失败，并可能报告：

```text
version `GLIBC_x.y' not found
```

## 15. 建立符号查找作用域

动态链接器为对象建立 `l_scope` 和 `l_searchlist`。它们决定未定义符号在哪些对象中、以什么顺序查找。

作用域受到下列因素影响：

- 主程序和启动阶段加入的全局对象；
- `LD_PRELOAD`；
- 当前对象及其依赖；
- `RTLD_LOCAL` 和 `RTLD_GLOBAL`；
- `RTLD_DEEPBIND`；
- 动态链接命名空间；
- 符号绑定属性和可见性；
- GNU unique symbol；
- 符号版本。

`RTLD_LOCAL` 并不意味着该库不能引用已有的全局库。它主要意味着该库通常不会自动成为以后加载对象的全局符号提供者。

## 16. 重定位

通用入口是 `elf/dl-reloc.c` 中的 `_dl_relocate_object()`。架构相关逻辑位于对应的 `dl-machine.h`。

动态链接器读取：

- `DT_REL` 或 `DT_RELA`；
- `DT_JMPREL`；
- 动态符号表；
- 动态字符串表；
- 符号版本表。

然后按 CPU 架构处理具体重定位类型。

以 x86-64 为例，常见类型包括：

- `R_X86_64_RELATIVE`
- `R_X86_64_GLOB_DAT`
- `R_X86_64_JUMP_SLOT`
- `R_X86_64_64`
- `R_X86_64_DTPMOD64`
- `R_X86_64_DTPOFF64`
- `R_X86_64_TPOFF64`
- `R_X86_64_IRELATIVE`

### 16.1 相对重定位

`R_X86_64_RELATIVE` 通常不需要按名称查找符号：

```text
目标值 = 当前对象的 l_addr + addend
```

这是共享库和 PIE 中非常常见的一类重定位。

### 16.2 外部符号重定位

假设目标库引用：

```c
extern int global_counter;
```

动态链接器会：

1. 从重定位记录中取得符号索引。
2. 从 `.dynsym` 取得符号记录。
3. 从 `.dynstr` 取得符号名称。
4. 调用 `_dl_lookup_symbol_x()`。
5. 按 `l_scope` 指定的顺序搜索对象。
6. 检查版本、绑定属性和可见性。
7. 计算定义所在对象中的实际地址。
8. 把结果写入 GOT 或目标重定位位置。

### 16.3 用户态与内核态边界

一般重定位计算和写回是在用户态完成的。例如动态链接器在用户态计算一个地址，再执行普通内存写指令写入 GOT。

如果目标页面不可写，例如存在 `DT_TEXTREL`，动态链接器可能先通过 `mprotect()` 请求内核临时修改页面权限，完成重定位后再恢复。

## 17. 符号查找

核心函数 `_dl_lookup_symbol_x()` 位于 `elf/dl-lookup.c`。

查找过程中会使用：

- GNU hash 或 SysV hash；
- GNU hash bloom filter；
- 符号名比较；
- `STB_GLOBAL`、`STB_WEAK`、`STB_GNU_UNIQUE`；
- `STV_DEFAULT`、`STV_HIDDEN`、`STV_PROTECTED`；
- 符号版本；
- 当前符号作用域；
- 重定位类型。

找不到普通强符号时，会产生类似错误：

```text
undefined symbol: symbol_name
```

弱未定义符号通常允许解析为空值。

## 18. `RTLD_NOW` 与 `RTLD_LAZY`

### 18.1 `RTLD_NOW`

`dlopen()` 返回之前完成需要立即处理的符号解析，包括 PLT 函数重定位。

特点：

- 未定义符号等错误较早暴露；
- 第一次函数调用不承担延迟绑定开销；
- `dlopen()` 本身可能更慢。

### 18.2 `RTLD_LAZY`

数据重定位仍然必须完成；主要是可延迟的 PLT 函数调用可以暂不解析。

初始 GOT/PLT 被设置为指向动态链接器 resolver。第一次调用某函数时：

```text
call foo@plt
      │
      ▼
PLT 跳转到 ld.so resolver
      │
      ├─ 定位对应的 PLT 重定位记录
      ├─ 查找符号 foo
      ├─ 执行审计和 IFUNC 等逻辑
      ├─ 把真实函数地址写回 GOT
      ▼
跳转到真正的 foo
```

以后的调用可以通过 GOT 直接到达目标函数。

以下情况可以强制立即绑定：

- 环境变量 `LD_BIND_NOW`；
- ELF 中的 `DT_BIND_NOW`；
- `DF_BIND_NOW`。

## 19. GNU IFUNC

GNU IFUNC 符号保存的是 resolver，而不是最终实现地址。例如 libc 可以根据 CPU 能力选择不同的 `memcpy`：

```text
memcpy IFUNC resolver
 ├─ AVX-512 实现
 ├─ AVX2 实现
 └─ 通用实现
```

遇到 `STT_GNU_IFUNC` 或 `R_X86_64_IRELATIVE` 时，动态链接器调用 resolver，并把 resolver 的返回值作为最终函数地址。

IFUNC resolver 在动态链接器的敏感阶段执行，不适合进行复杂操作。递归调用 `dlopen()` 或触发新的延迟绑定可能造成复杂的锁、初始化和依赖问题。

## 20. TLS 处理

如果新对象包含 `PT_TLS`，glibc 需要：

1. 为模块分配 TLS module ID。
2. 扩展全局 TLS slotinfo。
3. 增加 TLS generation。
4. 更新线程 DTV 所需的元数据。
5. 必要时建立静态 TLS 区域。
6. 保存 TLS 初始化镜像。
7. 处理 TLS 相关重定位。

TLS 符号地址与当前线程有关：

```text
TLS 地址 = 当前线程的 TLS 基址 + 模块及变量偏移
```

所以动态加载带 TLS 的共享库远不只是把文件映射到内存。

## 21. RELRO 和页面权限

完成重定位后，glibc 会处理 `PT_GNU_RELRO`：

1. 重定位期间相关区域保持可写。
2. 重定位完成后调用 `mprotect()`。
3. 内核将这些页面改成只读。

这样可以保护 GOT 和部分动态链接元数据，降低它们被覆盖的风险。

页面权限修改必须进入内核，因为最终页表权限由内核控制。

## 22. 更新作用域和 TLS

在新对象向其他线程可见之前，glibc 先完成可能失败的分配：

- 扩展局部 scope；
- 扩展 TLS slotinfo；
- 为 `RTLD_GLOBAL` 扩展全局 scope；
- 建立地址查找结构。

随后进入提交阶段：

1. 激活 `RTLD_NODELETE`。
2. 原子地更新局部符号作用域。
3. 更新地址查找结构。
4. 提交 TLS 信息。
5. 向调试器报告重定位完成。

这种先分配、后发布的两阶段做法，可避免其他线程看见只加载了一部分的对象。

## 23. 通知调试器和审计模块

动态链接器维护 `_r_debug` 和每个命名空间中的 `link_map` 链表。状态大致经历：

```text
RT_CONSISTENT
      │
      ▼
RT_ADD
      │
      ├─ 加入 link_map
      ├─ 映射依赖并重定位
      ▼
RT_CONSISTENT
```

GDB 会通过动态链接器暴露的调试接口重新读取 `link_map` 链表，从而发现刚刚加载的共享库及其调试符号。

如果配置了 `LD_AUDIT`，动态链接器还会执行：

- 对象搜索回调；
- 对象打开回调；
- activity 状态回调；
- 符号绑定回调；
- PLT enter/exit 回调。

## 24. 执行构造函数

完成映射和重定位后，`dl_open_worker()` 调用 `_dl_init()`。实现位于 `elf/dl-init.c`。

顺序原则是依赖优先：

```text
底层依赖库构造函数
        ↓
上层依赖库构造函数
        ↓
本次直接 dlopen 的对象
```

每个对象内部通常按以下顺序执行：

```text
DT_INIT
   ↓
DT_INIT_ARRAY[0]
   ↓
DT_INIT_ARRAY[1]
   ↓
...
```

对应常见 C 写法：

```c
__attribute__((constructor))
static void library_init(void)
{
    /* 在 dlopen 返回前执行 */
}
```

glibc 使用 `l_init_called` 防止构造函数重复执行，并避免循环依赖造成无限递归。

构造函数执行在用户态。构造函数可以再次调用 `dlopen()`，但这可能引入递归装载、循环依赖、部分初始化对象和复杂的符号可见性问题。

## 25. `RTLD_GLOBAL` 的最终发布

对于：

```c
dlopen("libdemo.so", RTLD_NOW | RTLD_GLOBAL);
```

glibc 会把目标对象以及适用依赖加入命名空间的全局符号搜索列表。其整体顺序大致是：

```text
预留全局 scope 空间
        ↓
完成重定位
        ↓
更新局部 scope 和 TLS
        ↓
执行构造函数
        ↓
正式发布到全局 scope
```

两阶段操作可防止内存分配失败后留下部分更新的全局状态。

## 26. 成功返回

所有阶段成功后，`_dl_open()` 返回对应的 `link_map`：

```c
return args.map;
```

应用层看到的是不透明句柄：

```c
void *handle = dlopen("libdemo.so", RTLD_NOW);
```

之后调用：

```c
void *address = dlsym(handle, "function_name");
```

会从该句柄对应的符号作用域中执行显式符号查找。

## 27. 错误处理和回滚

glibc 使用动态链接器内部的异常捕获机制组织错误：

```text
_dl_signal_error
        ↓
_dl_catch_exception
        ↓
清理并回滚
        ↓
保存线程局部错误信息
        ↓
dlopen 返回 NULL
```

以下情况均可能导致失败：

- 找不到目标库或依赖；
- ELF 格式或体系结构不兼容；
- 符号版本不匹配；
- 找不到强符号；
- `mmap` 或内存分配失败；
- TLS 分配失败；
- 重定位失败；
- 对象设置了 `DF_1_NOOPEN`。

失败时 `_dl_open()` 调用 `_dl_close_worker(map, true)` 撤销本次装载，包括：

- 从命名空间链表移除对象；
- 解除依赖引用；
- 释放符号搜索表；
- 清理 TLS 元数据；
- `munmap` 已映射的段；
- 恢复全局作用域计数；
- 恢复调试器一致状态。

错误信息保存在当前线程的状态中，可通过：

```c
const char *message = dlerror();
```

取得。

## 28. 用户态与内核态的完整边界

### 28.1 用户态完成的工作

| 工作 | 主要执行者 |
|---|---|
| 搜索 `RPATH/RUNPATH/LD_LIBRARY_PATH` | `ld.so` |
| 解析 ELF header 和 program header | `ld.so` |
| 解析 `PT_DYNAMIC` 和 `DT_NEEDED` | `ld.so` |
| 构建依赖图和 `link_map` | `ld.so` |
| 构建符号搜索作用域 | `ld.so` |
| 动态符号查找 | `ld.so` |
| 符号版本匹配 | `ld.so` |
| 计算和写入重定位结果 | `ld.so` |
| GOT/PLT 延迟绑定 | `ld.so` |
| TLS 元数据管理 | glibc/`ld.so` |
| 调用 `DT_INIT/DT_INIT_ARRAY` | `ld.so` 调度、库代码执行 |

### 28.2 需要内核完成的工作

| 工作 | 内核机制 |
|---|---|
| 打开共享库和依赖 | `openat` 等系统调用 |
| 读取 ELF 元数据 | `read/pread` 等系统调用 |
| 查询文件身份和权限 | `fstat` 等系统调用 |
| 建立文件内存映射 | `mmap` 系统调用 |
| 建立匿名 `.bss` 页面 | `mmap` 系统调用 |
| 调整页面 R/W/X 权限 | `mprotect` 系统调用 |
| 删除映射 | `munmap` 系统调用 |
| 将文件页装入物理内存 | 缺页异常处理 |
| 更新页表和执行权限检查 | 内核虚拟内存子系统 |
| 文件权限、挂载和 LSM 检查 | 内核安全及文件系统子系统 |

所以调用 `dlopen()` 并不是“一次陷入内核，由内核完成动态链接”。实际情况是：

```text
用户态 ld.so 运行一段动态链接逻辑
        ↓
需要文件或虚拟内存操作时进入内核
        ↓
系统调用返回用户态
        ↓
ld.so 继续解析、计算和修改链接状态
        ↓
再次根据需要进入内核
```

## 29. 整体等价骨架

以下代码将多个 glibc 文件的职责合并成一个便于阅读的模型，不是 glibc 原始源码的替代品：

```c
void *conceptual_dlopen(const char *name, int mode)
{
    validate_mode(mode);
    lock_loader();

    link_map *caller = find_calling_object();
    link_map *map = find_loaded_object(name);

    if (fully_loaded(map, mode)) {
        map->open_count++;
        unlock_loader();
        return map;
    }

    TRY {
        map = find_open_verify_and_mmap_elf(caller, name);

        build_link_map(map);
        parse_dynamic_section(map);

        load_all_dt_needed_dependencies(map);
        build_symbol_search_scopes(map);
        sort_dependency_and_init_order(map);

        check_symbol_versions(map);

        for_each_object_in_relocation_order(map)
            relocate_object(map, mode);

        reserve_scope_and_tls_storage(map);
        commit_nodelete_scope_and_tls(map);

        notify_debugger_and_audit(map);
        run_dependency_constructors_then_map(map);

        if (mode & RTLD_GLOBAL)
            publish_to_global_scope(map);
    }
    CATCH (error) {
        rollback_all_new_objects(map);
        unlock_loader();
        save_error_for_dlerror(error);
        return NULL;
    }

    unlock_loader();
    return map;
}
```

## 30. 如何观察真实加载过程

### 30.1 使用 `LD_DEBUG`

在 Linux shell 中：

```bash
LD_DEBUG=libs,files,reloc,bindings,scopes,versions ./your_program
```

常用分类：

- `libs`：共享库目录搜索；
- `files`：对象打开和引用计数；
- `reloc`：重定位过程；
- `bindings`：符号最终绑定到哪个对象；
- `scopes`：符号搜索作用域；
- `versions`：符号版本检查；
- `statistics`：重定位数量和耗时。

PowerShell 中为：

```powershell
$env:LD_DEBUG = "libs,files,reloc,bindings,scopes,versions"
./your_program
```

如果程序实际在 WSL 或 Linux 虚拟机中运行，应在对应 Linux 环境中设置变量。

### 30.2 观察系统调用

```bash
strace -f -e trace=openat,read,pread64,newfstatat,mmap,mprotect,munmap,close \
  ./your_program
```

这可以显示动态链接器何时进入内核打开文件、建立映射和修改页面权限。

### 30.3 检查 ELF

```bash
readelf -hW libdemo.so
readelf -lW libdemo.so
readelf -dW libdemo.so
readelf -rW libdemo.so
readelf -sW libdemo.so
readelf --version-info libdemo.so
objdump -T libdemo.so
```

- `readelf -h`：ELF 类型和体系结构；
- `readelf -l`：将被映射的 program header/segment；
- `readelf -d`：`DT_NEEDED`、`RPATH`、`RUNPATH` 等；
- `readelf -r`：重定位记录；
- `readelf -s`：符号表；
- `readelf --version-info`：符号版本；
- `objdump -T`：动态符号及其版本。

### 30.4 查看实际映射

程序运行期间可以检查：

```bash
cat /proc/<pid>/maps
```

常见共享库会出现多条映射，对应只读头部、可执行代码、只读数据和可写数据等不同权限的 `PT_LOAD` 段。

## 31. 最终总结

一次典型的运行时 `dlopen()` 可以概括为：

```text
校验参数
  → 获取动态链接器锁
  → 确定调用者与命名空间
  → 检查对象是否已加载
  → 搜索并打开共享库
  → 验证 ELF
  → 创建 link_map
  → mmap PT_LOAD 段
  → 解析 PT_DYNAMIC
  → 展开全部 DT_NEEDED
  → 检查符号版本
  → 建立符号作用域
  → 执行符号查找和重定位
  → 配置 TLS 和 RELRO
  → 发布对象并通知调试器
  → 按依赖顺序执行构造函数
  → 返回不透明 handle
```

其中，`ld.so` 决定“加载什么、到哪里找、怎样解析、如何链接”；内核负责“文件能否打开、地址怎样映射、页面是否驻留、页面具有哪些访问权限”。
