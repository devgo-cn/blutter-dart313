# blutter-dart313

[blutter](https://github.com/worawit/blutter) 的 **Dart 3.13.x / Flutter AOT** 适配补丁。

上游 blutter 停在 Dart 3.11 左右，遇到新引擎（本例：**Dart 3.13.2, arm64, android, product, compressed-pointers**）
会在编译期报约 20 个 API 不兼容错误。本仓库记录了全部修复，并以补丁形式提供。

## 目标样本

```
libapp.so  (Flutter AOT, arm64-v8a)
Dart version: 3.13.2
Snapshot hash: 0451907c2eaa8467e848c0067bfe8ed4
flags: product no-asan no-msan no-tsan no-shared_data no-code_comments
       dwarf_stack_traces arm64 android compressed-pointers
```

## 关键差异

### 1. Snapshot 符号改名（`extract_dart_info.py`, `ElfHelper.cpp`）

新引擎把 VM/isolate 两套快照合并成一对：

| 旧 | 新 |
| --- | --- |
| `_kDartVmSnapshotData` + `_kDartIsolateSnapshotData` | `_kDartSnapshotData` |
| `_kDartVmSnapshotInstructions` + `_kDartIsolateSnapshotInstructions` | `_kDartSnapshotText` |

`extract_dart_info.py` 改为按候选名列表查找并带兜底扫描；`ElfHelper.cpp` 用
`#if defined(kVmSnapshotDataAsmSymbol) && !defined(kSnapshotDataAsmSymbol)`
同时兼容两代命名。
（注意：预处理器只能用 `#if defined(X)` 判断宏是否存在，`#ifdef kVmSnapshotDataAsmSymbol` 无效。）

### 2. `Dart_InitializeParams` 去掉 snapshot 字段（`DartLoader.cpp`）

3.13 的 `Dart_InitializeParams` 不再有 `vm_snapshot_data` / `vm_snapshot_instructions`，
合并后的快照改由 `Dart_CreateIsolateGroup()` 传入。旧字段访问包进 legacy 分支。

### 3. `OBJECT_STORE_STUB_CODE_LIST` 被删除（`DartStub.h`, `DartApp.cpp`）

那些 stub 全部搬进了 `VM_STUB_CODE_LIST`。补丁提供空的 `OBJECT_STORE_STUB_CODE_LIST`
占位宏 + `BLUTTER_NO_OBJECT_STORE_STUBS` 标记，并把 `throwStubAddr` 改从
`StubCode::Throw()` 取；`build_*_method_extractor_code` 和 `StubCode::HasBeenInitialized()`
一并移除（新引擎已无此 API）。

### 4. Stub 枚举后缀不一致（`DartStub.h`）

`VM_STUB_CODE_LIST` 经 blutter 的 `DO(name) name ## VMStub` 展开成 `DefaultTypeTestVMStub`，
但 `CodeAnalyzer_arm64.cpp` 里写的是旧拼法 `DefaultTypeTestStub`。
补丁在 `enum Kind` 末尾补了一组 **等值别名**，保持数值同一性：

```cpp
DefaultTypeTestStub = DefaultTypeTestVMStub,
AllocateMintSharedWithFPURegsStub = AllocateMintSharedWithFPURegsVMStub,
LateInitializationErrorSharedWithoutFPURegsStub = LateInitializationErrorSharedWithoutFPURegsVMStub,
WriteBarrierWrappersStub = WriteBarrierWrappersVMStub,
// ... 共 11 个
```

### 5. `Mint::value()` → `Mint::Value()`、`Record::GetRecordType(TypeVisibility)`

3.13 已统一为 `Value()`，且 `GetRecordType` 需要可见性参数。
对应 `UNIFORM_INTEGER_ACCESS` 宏，仓库里默认打开（`CMakeLists.txt`）。

### 6. Closure 布局改为变长尾部数组（`pch.h`）

`AOT_Closure_context_offset` / `AOT_Closure_delayed_type_arguments_offset` 已不存在。
新布局（arm64）：

| 字段 | 偏移 |
| --- | --- |
| `entry_point_` | 0x8 |
| `length_and_flags` | 0x10 |
| `hash` | 0x18 |
| `function` | 0x20 |
| `elements_start`（尾部数据起点） | 0x28 |

尾部各槽含义由 closure flags 决定（`UntaggedClosure::ContextIndex()`）：
`data[0]`=delayed type args，`data[1]`=instantiator type args，`data[2]`=function type args，
之后才是捕获值与 context。两个旧常量都塌缩到 `AOT_Closure_elements_start_offset`，
补丁以 `#define` 别名等价实现。

### 7. 运行期容错（`DartTypes.cpp`, `CodeAnalyzer_arm64.cpp`）

新引擎会抛出上游没建模的类型/池条目，直接 `FATAL` 会让整个 dump 失败。改为降级：

- `FindOrAdd(AbstractTypePtr)` 遇到未知 cid → 打印 cid 并回退到 `dart::Type::DynamicType()`
- `ObjectPool` 的 `kNativeFunction` 条目（纯 C 函数指针，非 Dart 对象）
  → 当作 `VarInteger(NativeInt)` 输出，不再抛异常

## 使用

```bash
git clone https://github.com/worawit/blutter
cd blutter
git apply /path/to/blutter-dart313.patch

# 依赖：cmake / ninja / g++>=13 / libicu-dev / libcapstone-dev
python3 blutter.py <dir-with-libapp.so> <outdir>
```

`python3 blutter.py` 只在二进制不存在时才重新编译；改过源码后需要手动
`ninja -C blutter/build && cmake --install blutter/build`，否则会跑到旧二进制。

### 内存：真正的拦路虎

**必须在足够内存的机器上跑。** blutter 会把**每个函数**的反汇编文本
（`AsmText`，约 92–104 B/条）和 IL **全部常驻内存**，直到 `DumpCode()` 才消费：

```cpp
// CodeAnalyzer.cpp:12 AnalyzeAll() —— 串行循环，无任何流式落盘
dartFn->SetAnalyzedData(std::make_unique<AnalyzedFnData>(app, *dartFn, convertAsm(asm_insns)));
```

几十万个函数 × 每条几十~几百个 `AsmText` = 上千万条对象，仅 asmTexts 就数 GB。
实测某 Flutter AOT 应用（44 MB APK）：

| 阶段 | RSS |
| --- | --- |
| VM 引导 + 类表 dump | ~500 MB |
| 反汇编分析中 | 4.5 GB → 6.5 GB 持续爬升 |
| 触碰 **8 GiB cgroup 上限** | 卡死（`D` 状态，0% CPU，I/O 阻塞） |

在 **8 GiB 容器里这个应用跑不完**，不是补丁的问题，是上游架构的固定内存模型。
建议 **≥16 GB** 内存，或按下面的分片方式跑。

### 分片（可选，缓解内存）

补丁给 `AnalyzeAll()` 加了 `BLUTTER_SHARD="i/N"`：只分析函数列表的第 i/N 片。

```bash
for i in $(seq 0 7); do
  BLUTTER_SHARD="$i/8" ./blutter_dartvm3.13.2_android_arm64 -i libapp.so -o out_$i &
done
wait
```

每个分片内存约 1/N。**注意**：分片输出的是**各自的残缺结果**，
`pp.txt` / `asm/` / Frida / IDA 脚本都只含本片函数，需要自己按函数名合并。

## 验证进展

以大师兄影视 `libapp.so`（Dart 3.13.2）实测：

- [x] Dart VM 成功初始化并读取快照
- [x] 类表 / 库 / 函数 / 类型全部 dump 成功
- [x] 进入 ARM64 反汇编阶段，分析上千个函数（1650 条非致命 analysis warning）
- [ ] 完整跑完并落盘 `pp.txt` / `asm/` —— **受阻于 8 GiB 内存上限**

补丁本身的编译期与 VM 初始化问题已全部解决；剩余阻塞是上游的内存模型，
不属于本补丁范畴。

## 备注

- 上游仓库：<https://github.com/worawit/blutter>
- 补丁基于上游 `4a60ac6`（Merge PR #217）
- 文件 `blutter-dart313.patch` 即完整 diff（9 文件, +157 / -14）
