# 🏗️ 架构总览

本项目通过高度自动化的 CI/CD 流水线（GitHub Actions），将繁琐的本地编译过程封装在云端。用户只需一键下载 Release 中的压缩包，即可获得针对 NVIDIA RTX 50 系列（Blackwell 架构）深度优化的推理引擎。

mermaid
graph TD
    %% 核心流水线
    Runner[🖥️ GitHub Actions Runner (windows-2022)] -->|安装| Env[🧪 CUDA 12.8 + MSVC + cuBLAS]
    Env -->|Checkout| Code[📂 拉取源码 aetherbird/llama.cpp @ bailingmoe3-support]
    Code -->|准备编译环境| BuildEnv[⚙️ 配置 MSVC vcvars64]
    BuildEnv -->|CMake + Ninja| Compile[🔥 编译 -march=sm_120 + Flash Attention]
    Compile -->|收集产物| Artifact[📦 收集可执行文件 (.exe) + 动态链接库 (.dll)]
    Artifact -->|打包| Package[📦 打包为 zp-llama-server-rt50-cuda12.8-vX.X.X.zip]
    Package -->|发布| Release[🚀 发布到 GitHub Releases]

    %% 外部依赖与最终用户流向
    Source[(🔀 源码分支: bailingmoe3-support)] -.->|触发更新| Code
    Release -.->|👤 用户下载解压即用| User((💻 用户本地环境))
    User -->|🚀 运行推理| Inference[⚡ Ling-3.0 tiny RTX 50 推理服务]


# 📦 文件结构说明

解压后目录包含以下核心文件：

| 文件名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `llama-server.exe` | 可执行文件 | ✅ 核心服务端 |
| `llama.dll` | 动态链接库 | ✅ 核心 |
| `mtmd.dll` | 动态链接库 | ✅ 多模态支持 |
| `cudart64_*.dll` | 动态链接库 | ✅ CUDA 运行库 |
| `cublas64_*.dll` | 动态链接库 | ✅ CUDA BLAS |
| `cublasLt64_*.dll` | 动态链接库 | ✅ CUDA BLASLt |
| `vcruntime140*.dll` | 动态链接库 | ⚠️ 冗余（静态链接已包含） |
| `msvcp140*.dll` | 动态链接库 | ⚠️ 冗余 |
| `concrt140.dll` | 动态链接库 | ⚠️ 冗余 |

> **💡 精简提示**：由于编译时使用了 `/MT` 静态链接参数，上述 VC++ 运行库文件（标注为冗余的）实际上已编译进 `exe` 中，可以安全删除以减小体积（约 30MB）。

# ⚙️ 编译参数详解

本工作流针对 RTX 50 系显卡和 Ling-3.0-tiny 模型进行了深度调优，核心参数如下：

| 参数 | 值 | 作用 |
| :--- | :--- | :--- |
| `-DGGML_CUDA=ON` | `ON` | 启用 CUDA 后端，利用 NVIDIA 显卡加速。 |
| `-DCMAKE_CUDA_ARCHITECTURES` | `"120"` | **关键**：指定 Blackwell (RTX 50) 架构算力，确保原生性能。 |
| `-DGGML_CUDA_FA_ALL_QUANTS=ON` | `ON` | 启用 Flash Attention 全量化内核，MoE 模型（如 Ling）性能提升显著。 |
| `-DGGML_NATIVE=OFF` | `OFF` | 关闭 CPU 原生优化，保证二进制文件在所有 x86-64 机器上通用。 |
| `-DGGML_BACKEND_DL=ON` | `ON` | CUDA 后端动态加载，符合上游 Release 规范。 |
| `-DLLAMA_BUILD_SERVER=ON` | `ON` | 仅构建 `llama-server`，不编译多余工具，减小体积。 |
| `-DCMAKE_C_FLAGS` / `-DCMAKE_CXX_FLAGS` | `"/MT"` | 静态链接 VC++ 运行库，实现“解压即用”。

# 🎮 显存适配参考

| 显存容量 | 适用显卡 | 推荐模型精度 |
| :--- | :--- | :--- |
| 8GB | RTX 5050 / 5060 | Q4_K_M / Q5_K_M |
| 16GB | RTX 5060 Ti / 5070 / 5080 | Q8_0 / BF16 |
| 32GB | RTX 5090 | BF16 / UD-Q8_K_XL |

# 🔧 自定义编译

你可以在 `workflow_dispatch` 触发运行时修改以下参数：

*   **`tag_name`**: Release 版本号（如 `v1.0.1`）。
*   **`cuda_version`**: 切换 CUDA 版本（如 `13.0`），用于测试新驱动兼容性。

# ❓ 常见问题 (FAQ)

**Q: 启动时报错 `unknown model architecture: 'bailingmoe3'`？**
A: 这是因为使用了官方主线版本的 `llama.cpp`。Ling-3.0-tiny 使用了蚂蚁百灵的自定义 MoE 架构，必须使用本仓库编译的、基于 `aetherbird/llama.cpp@bailingmoe3-support` 分支的版本才能识别。

**Q: 日志中出现 `CORS is set to allow all origins` 警告？**
A: 这是安全提示。仅在局域网内使用时无需处理；若需暴露到公网，请添加启动参数 `--cors-allow-origin "https://your-domain.com"` 或设置 `--api-key`。

**Q: 为什么编译出来的包里有这么多 `.dll` 文件？**
A: 为了保证最大的兼容性和性能，我们将 CUDA 运行库和 VC++ 运行库一并打包。虽然体积稍大，但用户无需安装任何额外软件，真正做到“开箱即用”。

---

*基于 [aetherbird/llama.cpp](https://github.com/aetherbird/llama.cpp/tree/bailingmoe3-support) 的 `bailingmoe3-support` 分支构建*
