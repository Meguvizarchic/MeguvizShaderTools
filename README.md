<p align="center"><img src="docs/brand/MeguvizShaderTools.svg" width="96" alt="MeguvizShaderTools"></p>

# MeguvizShaderTools

用于离线构建 Vulkan 着色器的命令行工具。将 GLSL 编译为 SPIR-V，检查二进制和资源布局，并把着色器打包供应用使用。

- 编译、验证和反射 SPIR-V。
- 检查 ncnn 着色器的绑定、推送常量及特化常量。
- 创建和校验着色器包，供 Vulkan 应用按需加载。

## 快速开始

需要 C++17、CMake 3.24+、Ninja、Python 3 和 Git。先获取源码依赖，再按平台选择 CMake 预设。

```sh
python tools/bootstrap.py --root ../dependencies
cmake --preset host-windows-release
cmake --build --preset host-windows-release --target mgst
cmake --install build/host-windows-release --prefix ./sdk
```

Windows 上使用 MSVC x64 工具链。Linux 可选 `host-linux-release` 预设。将安装目录中的 `bin` 加入命令搜索路径，即可使用 `mgst`。[构建预设](CMakePresets.json) · [源码依赖版本](deps.lock.json)

准备编译请求后，可以编译并查看资源布局：

```sh
mgst compile --request request.json --out-dir shader-output
mgst reflect --spirv shader-output/shader.spv --out reflection.json
```

## 文档

- [编译请求格式](schemas/compile-request.schema.json)
- [CMake 集成示例](examples/external_consumer/)
- [工具调用与着色器包说明](docs/adr/0001-host-contract.md)

[MIT 许可证](LICENSE)
