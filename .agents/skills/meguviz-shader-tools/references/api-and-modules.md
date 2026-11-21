# MeguvizShaderTools API 与模块速查

本文档汇总 `MeguvizShaderTools`（`0.1.0` / `--version-json` 输出 `0.1.0-dev`）的宿主机依赖、CMake 封装宏、CLI 全子命令、退出码以及 `native-vulkan` 与 `ncnn-compat` 两条编译规范的完整规则。

## 1. 宿主机依赖与 CMake 接口

### 1.1 锁定上游依赖（`deps.lock.json`）

通过 `python tools/bootstrap.py --root ../dependencies` 拉取并锁定以下源码（CMake 配置阶段校验精确 commit 与干净工作区）：

| 组件 | 变量键名 | 锁定 Commit |
| --- | --- | --- |
| `glslang` | `glslang` | `780b6a209132dc74e99458f62ebe5fa7e57c41cf` |
| `SPIRV-Tools` | `spirv_tools` | `b707790a898e44038547df54580022fc1cf89c3d` |
| `SPIRV-Headers` | `spirv_headers` | `29981f65241605e08b0ede4cfeb999fe3b723c6a` |
| `SPIRV-Reflect` | `spirv_reflect` | `563dc9a16d0e86145e12a66bb61f0695ca4a14b9` |
| `ncnn` | `ncnn` | `7e36d25cb153092ba87a60530455c7376fedf452` |

### 1.2 CMake 预设与导出接口

- **预设列表（`CMakePresets.json`）**：`host-windows-release`、`host-windows-debug`、`host-linux-release`、`host-linux-debug`（均使用 `Ninja` 生成器）。
- **导入目标**：`find_package(MeguvizShaderTools CONFIG REQUIRED)` 后提供 `MeguvizShaderTools::mgst`（`IMPORTED` 可执行文件）。
- **封装函数**：`mgst_compile_shaderpack(REQUEST <req> OUTPUT <prefix> OUT_PACK <out_var> [DEPENDS <files...>])`
  - 自动读取 `REQUEST` 内容、`DEPENDS` 文件哈希与 `mgst` 二进制哈希，计算生成目录 `${OUTPUT}.generations/${generation}`。
  - 自动生成 `index.json`，添加 `mgst compile ... --reuse`（附带 `DEPFILE source.d`）与 `mgst pack ... --reuse` 自定义构建规则。
  - 将最终打包目录 `${OUTPUT}.generations/${generation}/pack` 写入 `<out_var>` 返回给调用作用域。

## 2. `mgst` 全子命令速查

所有子命令支持全局参数：`--report FILE`、`--diagnostics-json FILE`、`--jobs 1..64`、`--offline`。默认向标准输出打印结构化 JSON 诊断报告（匹配 `schemas/diagnostic.schema.json`）。

| 子命令 | 必需 / 可选参数 | 功能说明 |
| --- | --- | --- |
| `--help` / `help` | 无 | 打印命令行用法摘要 |
| `--version-json` | 无 | 输出版本、`cli_protocol (1)`、`toolchain_lock_hash` 及锁定依赖详情 |
| `doctor` | `--lock FILE [--offline]` | 校验外部 `deps.lock.json` 与内置锁定指纹或完整工具链指纹是否匹配 |
| `hash` | `--file FILE` | 使用内置 FIPS 180-4 SHA-256 计算文件摘要 |
| `compile` | `--request FILE --out-dir DIR [--reuse]` | 解析并校验请求，编译 GLSL、做 SPIRV-Tools 校验与 SPIRV-Reflect 反射，输出 `shader.spv`、`reflection.json`、`artifact.json`、`receipt.json`、`source.d` 等 |
| `render-ncnn` | `--request FILE --out-dir DIR` | 仅针对请求渲染预处理源码（`rendered.comp` 与 `render.json`），不生成 SPIR-V |
| `validate` | `--spirv FILE [--target-env ENV] [--profile FILE]` | 用 `SPIRV-Tools` 校验 `.spv` 合法性；传入含 `profile_id` 的 JSON 时可同步执行 `ncnn-compat` 布局检查 |
| `reflect` | `--spirv FILE --out FILE [--target-env ENV]` | 校验并导出规范化反射 JSON（`entry_point`、`target_env`、`required_caps`、`layout_abi`、`layout_hash`） |
| `lint-abi` | `--spirv FILE --profile FILE --expect FILE [--target-env ENV]` | 检查 `.spv` 是否符合指定 profile，并严格比对实际 `layout_abi` 与 `--expect` JSON |
| `diff-abi` | `--before FILE --after FILE` | 比较两个反射 JSON 或 `layout_abi` 文件是否完全一致并返回 `layout_hash` |
| `pack` | `--index FILE --artifacts DIR --out-dir DIR [--reuse]` | 聚合多个已编译产物生成 Shaderpack（含 `blobs/*.spv`、`reflection/*.json` 与带 `manifest_hash` 的 `manifest.json`） |
| `verify` | `--pack DIR` | 端到端校验 `manifest.json` 哈希、SPIR-V 文件大小与哈希、反射字节哈希与内容一致性及 profile 布局约束 |
| `inspect` | `--pack DIR [--format json\|text]` | 执行与 `verify` 相同的完整性校验并按 JSON 或简洁文本列出包内产物 |
| `reproduce` | `--request FILE --baseline DIR --work-dir DIR` | 在新目录重编译并比对 `artifact.json` 与 `shader.spv` 是否逐字节可重现 |
| `coverage` | `--catalog FILE --requests FILE --pack DIR [--strict]` | 核对观测请求轨迹（支持 JSON 或 JSONL）与声明变体目录在 pack 中的覆盖度；缺项退出码为 `10` |

## 3. 退出码速查

| 退出码 | 语义分类 | 典型触发原因 |
| --- | --- | --- |
| `0` | `PASS` | 命令成功（若 `--reuse` 生效则报告 `reused_existing: true`） |
| `2` | 命令行参数错误 | 未知子命令、未知或重复参数、`--source`/`--profile`/`--entry` 与 `compile` 冲突、`--jobs` 超出 `1..64` |
| `3` | 不受支持的能力或配方 | 不支持的 `profile_id` / `profile_version` / `target_env` / `resource_recipe`、非空 `required_flags` / `optimization_passes`、入口非 `main`、`ncnn` 有效目标环境不匹配 |
| `4` | 请求格式与路径安全错误 | JSON 语法错误、浮点数/不安全整数/重复键/超深嵌套、缺少或多余字段、相对路径违规或符号链接越出请求目录 |
| `5` | GLSL 编译失败 | `#version` 缺失或前导语法错误、`#include` 未在 `includes` 中声明、GLSL 语法错误 |
| `6` | SPIR-V 二进制或校验失败 | 魔数不是 `0x07230203`、长度非 4 的倍数或不足 20 字节、`SPIRV-Tools` 校验不通过 |
| `7` | 反射与 ABI 约束失败 | 多入口或非 Compute 阶段、`ncnn-compat` 布局违规、`expected_abi` 不匹配、缺少 `OpName`/`OpMemberName` |
| `8` | 哈希与版本锁不匹配 | `toolchain_lock_hash`、`ncnn_revision`、源码或 include 哈希不匹配、`manifest_hash` / `reflection_sha256` 篡改、`--reuse` 校验不通过 |
| `9` | 文件 I/O 与不可变目录冲突 | 文件缺失或超过上限（JSON 16 MiB、单资源/SPIR-V 64 MiB）、输出目录非空且已存在、`--report` 写入失败 |
| `10` | 覆盖率检查未通过（`coverage`） | 存在 pack 缺失的 `variant_key`，或启用 `--strict` 时有声明变体/条件未被轨迹覆盖 |
| `11` | 重现差异（`reproduce`） | 重编译的 `artifact.json` 或 `shader.spv` 与基线不一致 |
| `12` | 内部运行期异常 | `glslang::InitializeProcess` 失败或其他未预期异常 |

## 4. `compile-request.schema.json` 核心字段与双 Profile 差异

### 4.1 请求 JSON 公共必需字段

- `schema_version`: 必须为 `1`
- `logical_shader_id`: 非空逻辑标识字符串
- `owner`: JSON 对象（记录仓库与修订信息）
- `source`: `{"path": "<相对路径>", "sha256": "<64位小写hex>"}`
- `includes`: 数组，每项为 `{"name": "<include名>", "path": "<相对路径>", "sha256": "<64位小写hex>"}`（上限 65,536 项，总闭包 <= 64 MiB）
- `profile_id`: `"native-vulkan"` 或 `"ncnn-compat"`
- `profile_version`: `native-vulkan` 为 `"1"`；`ncnn-compat` 为 `"20261020"`
- `entry_point`: 固定 `"main"`
- `target_env`: `"vulkan1.0"` | `"vulkan1.1"` | `"vulkan1.3"`
- `effective_options`: JSON 对象
- `numeric_profile`: 非空字符串（如 `"reference-fp32"`）
- `toolchain_lock_hash`: 来自目标 `mgst --version-json` 的 `result.toolchain_lock_hash`
- `required_flags` / `optimization_passes`: 现阶段均须为空数组 `[]`
- `resource_recipe`: `native-vulkan` 为 `"glslang-default-pinned"`；`ncnn-compat` 为 `"ncnn-20261020"`
- 可选字段：`expected_abi`（编译后自动比对 `layout_abi`）、`target_facts`、`ncnn_revision`

### 4.2 `native-vulkan` 与 `ncnn-compat` 规则对照

| 维度 | `native-vulkan` | `ncnn-compat` |
| --- | --- | --- |
| `profile_version` | `"1"` | `"20261020"` |
| `resource_recipe` | `"glslang-default-pinned"` | `"ncnn-20261020"`（采用锁定 `profiles/ncnn_20261020/resource.hpp`） |
| 额外必需字段 | 无 | `ncnn_revision`（`"7e36d25cb153092ba87a60530455c7376fedf452"`）与 `target_facts` |
| `effective_options` | 任意有效字段对象 | 必须严格包含 `use_bf16_storage` 等 15 个布尔开关，且与 `target_env` 联动校验 |
| 源码渲染预处理 | 原样送入 `glslang` | 在 `#version` 行后注入 `#extension`、`#define NCNN_*`、`#define ncnn_*` 与 `ncnn_glsl_ext.comp`，并内置 `vulkan_activation.comp` |
| Descriptor Set / Binding | 支持多 set、稀疏 binding、数组描述符 | 仅允许 `set == 0`、`binding` 从 `0` 连续递增（最多 16 个）、`count == 1`、`type` ∈ `{1, 3, 7}` |
| Push Constant | 任意标准结构 | 最多 1 个块（`offset == 0`），`OpName` 必须为 `"parameter"`，全部成员须有 `OpMemberName` 且为连续 32 位标量 |
| Specialization Constant | 任意合法 `constant_id` | `id` 必须从 `0` 连续递增且 `< 233`，宽度 `32` 位、大小 `4` 字节 |
| 调试信息保留 | 不剥离调试名 | 不剥离调试名（删除 `OpName`/`OpMemberName` 将直接触发布局校验失败） |
