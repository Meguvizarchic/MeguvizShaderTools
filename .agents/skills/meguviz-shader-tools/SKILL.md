---
name: meguviz-shader-tools
description: 在 CMake、C/C++ 或 Vulkan 工程中构建并接入 MeguvizShaderTools（0.1.0，命令行工具 mgst），或将运行期 glslang 编译、旧版着色器构建脚本迁移到离线编译、SPIR-V 校验反射、ncnn ABI 检查与 Shaderpack 打包流程。当需要生成 compile-request、获取 toolchain_lock_hash、调用 mgst_compile_shaderpack 或排查 ncnn-compat 布局约束与严格 JSON 报错时触发；运行期加载 Shaderpack 与调度计算转交 MeguvizCompute。
---

# MeguvizShaderTools 接入与迁移

`MeguvizShaderTools`（包版本 `0.1.0`，`mgst --version-json` 输出 `0.1.0-dev`）是宿主端离线 Vulkan 着色器编译、SPIR-V 校验反射与 Shaderpack 打包工具，不依赖目标 GPU 设备即可完成确定性构建与静态检查。

## 第 1 阶段：盘点工程现状

改造着色器构建流程前，先读取工程配置并输出简短评估：

1. **检查现有编译路径**：梳理当前 `.comp` 着色器源码、`#include` 依赖树、构建脚本（`CMakeLists.txt` / Python / Gradle）以及是否依赖系统 `glslangValidator` 或运行期编译。
2. **核对宿主机工具链基线**：
   - 本工具**仅支持宿主机原生构建**（`CMakeLists.txt` 显式拒绝交叉编译与 `ANDROID` 目标）。
   - 需要 CMake `3.24+`、C++17、Ninja、Python 3 与 Git；Windows 使用 MSVC x64 工具链与静态 CRT，Linux 使用 `host-linux-release` 预设。
   - 若尚未安装 `mgst`，须先按 `deps.lock.json` 获取锁定依赖并完成编译安装。
3. **确定目标 Profile 与边界**：区分着色器是供通用 Vulkan 管线使用（`native-vulkan`）还是供 `ncnn` Vulkan 算子使用（`ncnn-compat`），确认是否已有 `target_facts` 与 `effective_options`。
4. **明确增量迁移方案**：先挑选 1 个独立算子着色器接入 `mgst` 离线打包验证，保留既有构建逻辑作为对照，再逐步替换全量着色器。

## 第 2 阶段：选择模块与接入依赖

先构建并安装宿主机 `mgst` SDK，再按接入方式二选一：

```sh
python tools/bootstrap.py --root ../dependencies
cmake --preset host-windows-release   # Linux 请改用 host-linux-release
cmake --build --preset host-windows-release --target mgst
cmake --install build/host-windows-release --prefix ./sdk
```

| 接入方式 | 入口 / Target | 适用场景 | 边界与前提 |
| --- | --- | --- | --- |
| CMake 构建期自动编译打包 | `find_package(MeguvizShaderTools CONFIG REQUIRED)` + `mgst_compile_shaderpack(...)` | C/C++ 工程在构建期自动产出并复用 Shaderpack | 通过 `CMAKE_PREFIX_PATH` 引入宿主机安装前缀；交叉编译工程需指向宿主机 `mgst` 包 |
| 独立命令行流水线 | `<prefix>/bin/mgst` CLI | CI 离线生成 pack、独立检查或校验存量 `.spv` | 须先执行 `mgst --version-json` 提取当前工具链的 `toolchain_lock_hash` |

**Profile 选型**：
- `native-vulkan`（`profile_version: "1"`）：面向通用 Vulkan 管线，允许自定义 descriptor set/binding 与特化常量布局。
- `ncnn-compat`（`profile_version: "20261020"`）：面向 `ncnn` Vulkan 算子，须提供 `ncnn_revision` 与 `target_facts`，且强制约束 `set == 0`、连续 `binding`、命名为 `parameter` 的 push constant 块及保留调试名称。

**兄弟库边界**：
- **运行时加载与执行**：`MeguvizShaderTools` 只负责离线生成并校验 pack；在应用内打开并读取 `MgcShaderpack` 请转交 `MeguvizCompute`（`<meguviz/shaderpack.h>` 与 `ncnn_shaderpack_pipeline`）。
- **算法归属**：上层模型与业务着色器源码仍归各自拥有者（如 `MeguvizCompute`、`MeguvizRIFE`）维护，运行期编译器回退策略仍需在引擎侧保留。

## 第 3 阶段：执行最小迁移与生命周期对齐

### 方式 A：CMake 自动编译与打包（参考 `examples/external_consumer/CMakeLists.txt`）

```cmake
find_package(MeguvizShaderTools CONFIG REQUIRED)
get_target_property(mgst_path MeguvizShaderTools::mgst IMPORTED_LOCATION_RELEASE)
if(NOT mgst_path)
  get_target_property(mgst_path MeguvizShaderTools::mgst IMPORTED_LOCATION_DEBUG)
endif()
execute_process(COMMAND "${mgst_path}" --version-json OUTPUT_VARIABLE version_json COMMAND_ERROR_IS_FATAL ANY)
string(JSON lock_hash GET "${version_json}" result toolchain_lock_hash)

file(SHA256 "${CMAKE_CURRENT_SOURCE_DIR}/source.comp" source_hash)
# 按 schemas/compile-request.schema.json 生成严格 JSON（键按字母序排列且禁止浮点数）
# ... 写入 ${CMAKE_CURRENT_BINARY_DIR}/inputs/request.json ...

mgst_compile_shaderpack(
  REQUEST "${CMAKE_CURRENT_BINARY_DIR}/inputs/request.json"
  OUTPUT  "${CMAKE_CURRENT_BINARY_DIR}/identity-pack"
  OUT_PACK identity_pack
  DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/source.comp"
)
add_custom_target(shaderpack ALL DEPENDS "${identity_pack}/manifest.json")
```

### 方式 B：独立命令行编包流水线

```sh
mgst --version-json                                                    # 提取 result.toolchain_lock_hash
mgst compile --request request.json --out-dir build/artifacts/kernel   # 生成 shader.spv 与 reflection.json
mgst pack --index index.json --artifacts build/artifacts --out-dir build/pack
mgst verify --pack build/pack
```

### 核心输入规范与输出约束铁律

1. **严格 JSON 与数值限制**：请求与配置 JSON 禁止浮点数（浮点特化常量须编码为类型位串）、禁止重复键、禁止 `[-9007199254740991, 9007199254740991]` 之外的不安全整数、禁止前导零与非法 UTF-8/孤立代理项。
2. **路径封闭与完整 Include 闭包**：`source.path` 与 `includes[].path` 必须使用相对 `/` 路径（禁止 `..`、`.`、盘符或反斜杠），解析符号链接后仍须位于 `request.json` 所在目录内；着色器用到的所有 `#include` 文件都必须完整列在 `includes` 中并附带准确 SHA-256。
3. **绑定 `toolchain_lock_hash`**：必须填入从目标 `mgst --version-json` 读出的 `result.toolchain_lock_hash`，该哈希同时绑定了 `deps.lock.json` 与宿主机编译器构建指纹，不可手填或复用其他机器的值。
4. **不可变输出与 `--reuse`**：`compile`、`render-ncnn`、`pack` 与 `reproduce` 要求目标目录不存在或为空目录（通过临时目录原子重命名发布）；CMake 的 `mgst_compile_shaderpack` 会按输入与工具哈希隔离生成 `.generations/<hash>/` 并自动传入 `--reuse` 做一致性复验。
5. **保留调试命名（`ncnn-compat` 关键约束）**：`ncnn-compat` 依赖 SPIR-V 中的 `OpName`（如 `"parameter"`）与 `OpMemberName` 反射 push constant，外部优化或剥离调试名会导致 `lint-abi` 与 `verify` 失败（退出码 `7`）。

> **按需深入阅读**：全量 CLI 子命令、退出码、`compile-request` 字段与 `native-vulkan` / `ncnn-compat` 规则差异见 [API 与模块速查](references/api-and-modules.md)；请求构造范式、多变体覆盖检查及高频报错排查见 [迁移对照与排障](references/migration-patterns.md)。

## 第 4 阶段：验证与交付清单

1. **本库宿主自检**：编译 `mgst` 后在仓库根目录运行宿主契约测试与示例消费测试：
   ```sh
   ctest --preset host-windows-release --output-on-failure
   ```
   若已安装 SDK，可进一步配置并构建 `examples/external_consumer`（`cmake --build ... --target shaderpack verify`）验证 CMake 集成链路。
2. **产物校验与覆盖检查**：对生成的 pack 执行完整验证与检查：
   ```sh
   mgst verify --pack <pack-dir>
   mgst inspect --pack <pack-dir> --format text
   mgst reproduce --request <request.json> --baseline <artifact-dir> --work-dir <repro-dir>
   ```
   若声明了变体覆盖清单，补跑 `mgst coverage --catalog <catalog.json> --requests <trace.json> --pack <pack-dir> --strict`。
3. **回退与边界核对**：确认所有失败退出码（如退出码 `4/5/7/8/9`）均不会覆盖既有有效输出目录，且消费方保留了运行期编译器回退路径。
4. **交付摘要**：列出新增或更新的请求模板、CMake 配置、选用的 `profile_id` 与 `toolchain_lock_hash`、实际执行的 `ctest` / `mgst verify` 结果，并注明离线宿主验证不替代真实 GPU 设备执行。
