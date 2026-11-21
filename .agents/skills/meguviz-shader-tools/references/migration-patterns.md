# MeguvizShaderTools 迁移对照与排障

本文档提供从手动 `glslangValidator` / 运行期编译迁移到 `mgst` 离线管线的对照模板、规范化 JSON 编写规则、多算子打包方案与高频排障指引。

## 1. 典型迁移对照

### 范式 A：从手动 `glslangValidator` 脚本迁入 `mgst_compile_shaderpack`

**迁移前（零散调用系统工具、缺少锁定指纹与 ABI 校验）**：
```cmake
find_program(GLSLANG_VALIDATOR glslangValidator REQUIRED)
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/kernel.spv
  COMMAND ${GLSLANG_VALIDATOR} -V ${CMAKE_CURRENT_SOURCE_DIR}/kernel.comp -o ${CMAKE_CURRENT_BINARY_DIR}/kernel.spv
)
```

**迁移后（版本锁定的确定性打包，见 `examples/external_consumer/CMakeLists.txt`）**：
```cmake
find_package(MeguvizShaderTools CONFIG REQUIRED)
get_target_property(mgst_path MeguvizShaderTools::mgst IMPORTED_LOCATION_RELEASE)
if(NOT mgst_path)
  get_target_property(mgst_path MeguvizShaderTools::mgst IMPORTED_LOCATION_DEBUG)
endif()
execute_process(COMMAND "${mgst_path}" --version-json OUTPUT_VARIABLE version_json COMMAND_ERROR_IS_FATAL ANY)
string(JSON lock_hash GET "${version_json}" result toolchain_lock_hash)

file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/inputs")
configure_file(kernel.comp "${CMAKE_CURRENT_BINARY_DIR}/inputs/kernel.comp" COPYONLY)
file(SHA256 "${CMAKE_CURRENT_SOURCE_DIR}/kernel.comp" source_hash)
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/inputs/request.json" "{\"schema_version\":1,\"logical_shader_id\":\"my_app/kernel\",\"owner\":{\"repository\":\"my_app\",\"revision\":\"v1\"},\"source\":{\"path\":\"kernel.comp\",\"sha256\":\"${source_hash}\"},\"includes\":[],\"profile_id\":\"native-vulkan\",\"profile_version\":\"1\",\"entry_point\":\"main\",\"target_env\":\"vulkan1.0\",\"effective_options\":{},\"numeric_profile\":\"reference-fp32\",\"toolchain_lock_hash\":\"${lock_hash}\",\"required_flags\":[],\"optimization_passes\":[],\"resource_recipe\":\"glslang-default-pinned\"}\n")

mgst_compile_shaderpack(
  REQUEST "${CMAKE_CURRENT_BINARY_DIR}/inputs/request.json"
  OUTPUT  "${CMAKE_CURRENT_BINARY_DIR}/kernel-pack"
  OUT_PACK kernel_pack
  DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/kernel.comp"
)
add_custom_target(shaderpack ALL DEPENDS "${kernel_pack}/manifest.json")
```

### 范式 B：为 `ncnn-compat` 准备规范请求模板

`ncnn-compat` 必须显式提供 `ncnn_revision`、`target_facts` 以及完整 15 项 `effective_options`（可从 `MeguvizCompute` 的 `meguviz::ncnn_shader_options_json` 与 `meguviz::ncnn_shader_target_facts` 生成）。注意 `target_env` 须与特性一致：
- `target_facts.maintenance4_enabled == true` → 必须设为 `"vulkan1.3"`
- 否则若 `effective_options.use_subgroup_ops` 或 `use_cooperative_matrix` 为 `true` → 必须设为 `"vulkan1.1"`
- 其余情况 → 必须设为 `"vulkan1.0"`

```json
{
  "schema_version": 1,
  "logical_shader_id": "my_algo/warp",
  "owner": {"repository": "my_algo", "revision": "v1"},
  "source": {"path": "warp.comp", "sha256": "<64位小写SHA-256>"},
  "includes": [],
  "profile_id": "ncnn-compat",
  "profile_version": "20261020",
  "entry_point": "main",
  "target_env": "vulkan1.0",
  "effective_options": {
    "use_bf16_storage": false, "use_bf16_packed": false,
    "use_fp16_storage": false, "use_fp16_packed": false,
    "use_fp16_uniform": false, "use_fp16_arithmetic": false,
    "use_int8_storage": false, "use_int8_packed": false,
    "use_int8_uniform": false, "use_int8_arithmetic": false,
    "use_int16_storage": false, "use_int16_packed": false,
    "use_shader_local_memory": false,
    "use_subgroup_ops": false, "use_cooperative_matrix": false
  },
  "numeric_profile": "reference-fp32",
  "toolchain_lock_hash": "<mgst --version-json 输出中的 result.toolchain_lock_hash>",
  "required_flags": [],
  "optimization_passes": [],
  "resource_recipe": "ncnn-20261020",
  "ncnn_revision": "7e36d25cb153092ba87a60530455c7376fedf452",
  "target_facts": {
    "provenance": "synthetic",
    "maintenance4_enabled": false,
    "support_fp16_storage": false,
    "shader_int64": false,
    "shader_int16": false,
    "moltenvk": false,
    "device_defines": []
  }
}
```

### 范式 C：多变体打包与覆盖检查

1. 分别调用 `mgst compile --request <req_i.json> --out-dir <artifacts_dir>/<name_i>` 编译每个变体。
2. 编写 `index.json`（同一 pack 内 `profile_id` 与 `profile_version` 须一致）：
   ```json
   {"schema_version": 1, "profile_id": "ncnn-compat", "profile_version": "20261020", "artifacts": ["warp_fp32", "warp_fp16"]}
   ```
3. 执行 `mgst pack --index index.json --artifacts <artifacts_dir> --out-dir <pack_dir>` 与 `mgst verify --pack <pack_dir>`。
4. 用 `mgst coverage --catalog catalog.json --requests trace.jsonl --pack <pack_dir> --strict` 确认预期变体及条件已完整打包。

## 2. 高频报错与排查表

| 报错 `cause_code` / 现象 | 退出码 | 根因与修复办法 |
| --- | --- | --- |
| `json_float_use_typed_bits` | `4` | 请求或配置 JSON 中出现了小数或科学计数法浮点数；改用十六进制位串或字符串形式 |
| `json_unsafe_integer_use_typed_string` / `json_duplicate_key` | `4` | 整数超出 `±9007199254740991` 或对象存在重复键；大整数改用字符串，去重键名 |
| `path_relative_required` / `path_traversal` / `path_escape` | `4` | `source.path` 或 `includes[].path` 含绝对路径、`..`、`.`、`\`、`:` 或符号链接指向请求目录外；统一把源码与头文件复制到请求目录或子目录并使用相对 `/` 路径 |
| `toolchain_lock_mismatch` | `8` | `toolchain_lock_hash` 与当前可执行文件不符；重新执行 `mgst --version-json` 获取新值（修改编译器版本、构建配置或升级 `mgst` 后该值都会变化） |
| `source_hash_mismatch` / `include_hash_mismatch` | `8` | 文件换行符或字节被修改后未更新请求 SHA-256；注意确认 `.comp`/`.glsl` 字节与 `sha256` 完全一致 |
| `version_first_directive` | `5` / `4` | `ncnn-compat` 渲染要求 `#version` 为源码第一条指令（前面仅允许空白与注释） |
| `ncnn_effective_target_mismatch` | `3` | `target_env` 与 `target_facts.maintenance4_enabled` 或 `use_subgroup_ops`/`use_cooperative_matrix` 推导结果不一致；按范式 B 规则对齐 |
| `ncnn_descriptor_set_or_binding` / `ncnn_parameter_names_required` / `ncnn_spec_id_width_reserved` | `7` | 着色器使用了非 `0` set、跳号 binding、`type_name != "parameter"` 的 push constant（或被优化器剥离了 `OpName`/`OpMemberName`）、或 `constant_id >= 233` / 跳号 |
| `immutable_output_exists` | `9` | 目标 `--out-dir` 已存在且非空；改用新目录、先清理目录或配合一致输入加上 `--reuse` |
| `reuse_identity` / `reuse_bytes` | `8` | 传入 `--reuse` 时现有目录内容对应的是另一份请求或已被修改；切换新输出目录重新编译 |

## 3. 安全回退方案

- **构建失败回退**：因为 `compile` 与 `pack` 先写入临时目录并在最终校验通过后才原子发布，任何错误（退出码 `2..12`）都不会破坏已有有效 pack；只需保留上一版已校验的 pack 目录即可安全回滚。
- **设备兼容回退**：离线打包目前主要覆盖主机端编译与静态 ABI 校验；若运行期遇到未涵盖目标设备特性或缺失的变体，消费方（如 `MeguvizCompute` / `MeguvizRIFE`）必须保留既有的运行期编译器回退路径。
