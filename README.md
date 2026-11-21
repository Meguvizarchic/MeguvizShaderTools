<p align="center"><img src="docs/brand/MeguvizShaderTools.svg" width="96" alt="MeguvizShaderTools"></p>

# MeguvizShaderTools

用于宿主机离线构建 Vulkan 着色器的命令行工具（包版本 `0.1.0`，`mgst --version-json` 输出 `0.1.0-dev`）。将 GLSL 编译为 SPIR-V，完成二进制校验与资源反射，并把着色器产物打包供应用按需加载。

- 离线编译、验证和反射 SPIR-V，不依赖宿主机安装 GPU 驱动。
- 检查 `ncnn-compat` 着色器的描述符绑定、推送常量及特化常量布局。
- 生成并校验带完整哈希凭据的 Shaderpack，也可通过 CMake 宏 `mgst_compile_shaderpack` 集成到消费工程构建流。

## 快速开始

需要 C++17、CMake `3.24+`、Ninja、Python 3 和 Git。工具仅支持宿主机原生构建，先获取锁定依赖再按平台选择 CMake 预设：

```sh
python tools/bootstrap.py --root ../dependencies
cmake --preset host-windows-release
cmake --build --preset host-windows-release --target mgst
ctest --preset host-windows-release
cmake --install build/host-windows-release --prefix ./sdk
```

Windows 端使用 MSVC x64 工具链与静态 CRT；Linux 端可选用 `host-linux-release` 预设。将安装目录下的 `bin` 加入命令搜索路径，即可使用 `mgst`。详见 [构建预设](CMakePresets.json) 与 [源码依赖版本](deps.lock.json)。

先从 `mgst --version-json` 获取当前工具链的 `toolchain_lock_hash` 写入编译请求，即可执行编译与反射：

```sh
mgst --version-json
mgst compile --request request.json --out-dir shader-output
mgst reflect --spirv shader-output/shader.spv --out reflection.json
```

## 文档与接入指南

- [AI 接入与迁移指南](.agents/skills/meguviz-shader-tools/SKILL.md)
- [编译请求格式](schemas/compile-request.schema.json)
- [CMake 集成示例](examples/external_consumer/)
- [宿主机工具契约与架构决策](docs/adr/0001-host-contract.md)

[MIT 许可证](LICENSE) · [第三方声明](THIRD_PARTY_NOTICES.md)
