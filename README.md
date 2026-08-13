# 🏗️ Ling-3.0-tiny Windows Native Builder

本项目通过 GitHub Actions 实现全自动化的 CI/CD 流水线，专为 **NVIDIA RTX 50 系列（Blackwell 架构）** 深度优化。无需本地配置复杂的 CUDA 开发环境，一键即可编译出支持 `bailingmoe3` 架构的 `llama-server`，实现开箱即用。

## 📊 架构总览

mermaid
graph TD
    %% 样式定义
    classDef runner fill:#2c3e50,stroke:#3498db,stroke-width:2px,color:#ecf0f1;
    classDef env fill:#34495e,stroke:#f39c12,stroke-width:2px,color:#ecf0f1;
    classDef process fill:#16a085,stroke:#1abc9c,stroke-width:2px,color:#ffffff;
    classDef deploy fill:#27ae60,stroke:#2ecc71,stroke-width:2px,color:#ffffff;

    %% 核心流水线
    Runner[🖥️ GitHub Actions Runner (windows-2022)]:::runner -->|安装| Env[🧪 CUDA 12.8 + MSVC + cuBLAS]:::env
    Env -->|Checkout| Code[📂 拉取源码 aetherbird/llama.cpp @ bailingmoe3-support]:::process
    Code -->|准备编译环境| BuildEnv[⚙️ 配置 MSVC vcvars64]
    BuildEnv -->|CMake + Ninja| Compile[🔥 编译 -march=sm_120 + Flash Attention]:::process
    Compile -->|收集产物| Artifact[📦 收集可执行文件 (.exe) + 动态链接库 (.dll)]
    Artifact -->|打包| Package[📦 打包为 zp-llama-server-rt50-cuda12.8-vX.X.X.zip]
    Package -->|发布| Release[🚀 发布到 GitHub Releases]:::deploy

    %% 外部依赖与最终用户流向
    Source[(🔀 源码分支: bailingmoe3-support)] -.->|触发更新| Code
    Release -.->|👤 用户下载解压即用| User((💻 用户本地环境))
    User -->|🚀 运行推理| Inference[⚡ Ling-3.0 tiny RTX 50 推理服务]


## 🚀 快速开始

### 1. 编译最新版本
1. 进入仓库的 **Actions** 页面。
2. 选择 **Build Ling-3.0-tiny Server for RTX 50 Series** 工作流。
3. 点击 **Run workflow**，输入版本号（如 `v1.0.0`），点击绿色确认按钮。
4. 等待约 15-20 分钟，编译完成后会自动在 **Releases** 页面生成下载包。

### 2. 本地运行
下载并解压 `zp-llama-server-rt50-cuda12.8-vX.X.X.zip`，将你的 `Ling-3.0-tiny-BF16.gguf` 模型放入同一目录，在文件夹空白处打开终端（PowerShell/CMD），执行：

powershell
# 基础启动命令（本机访问）
.\llama-server.exe -m "Ling-3.0-tiny-BF16.gguf" -ngl 999 --jinja

# 局域网访问（允许手机/其他电脑连接）
.\llama-server.exe -m "Ling-3.0-tiny-BF16.gguf" --host 0.0.0.0 --port 8080 -ngl 999 --jinja


然后在浏览器访问 `http://localhost:8080` 或 `http://你的局域网IP:8080`。

## 📦 产物内容

编译生成的压缩包包含运行所需的全部依赖，无需安装任何额外运行库：

| 文件 | 说明 | 状态 |
| :--- | :--- | :--- |
| `llama-server.exe` | 推理服务器主程序 | ✅ 核心 |
| `ggml*.dll` | GGML 推理引擎核心 | ✅ 核心 |
| `llama*.dll` | Llama 模型支持库 | ✅ 核心 |
| `mtmd.dll` | 多模态支持 | ✅ 默认开启 |
| `cudart64_*.dll` | CUDA 运行库 | ✅ 必需 |
| `cublas64_*.dll` | CUDA BLAS 数学库 | ✅ 必需 |
| `cublasLt64_*.dll` | CUDA BLASLt 加速库 | ✅ 必需 |
| `vcruntime140*.dll` | VC++ 运行库 | ⚠️ 冗余（静态链接已包含） |
| `msvcp140*.dll` | VC++ 标准库 | ⚠️ 冗余 |
| `concrt140.dll` | 并发运行时 | ⚠️ 冗余 |

> **💡 精简提示**：由于编译时使用了 `/MT` 静态链接参数，上述 VC++ 运行库文件（标注为冗余的）实际上已编译进 `exe` 中，可以安全删除以减小体积（约 30MB）。

## ⚙️ 编译参数详解

本工作流针对 RTX 50 系显卡和 Ling-3.0-tiny 模型进行了深度调优，核心参数如下：

| 参数 | 值 | 作用 |
| :--- | :--- | :--- |
| `-DGGML_CUDA=ON` | `ON` | 启用 CUDA 后端，利用 NVIDIA 显卡加速。 |
| `-DCMAKE_CUDA_ARCHITECTURES` | `"120"` | **关键**：指定 Blackwell (RTX 50) 架构算力，确保原生性能。 |
| `-DGGML_CUDA_FA_ALL_QUANTS=ON` | `ON` | 启用 Flash Attention 全量化内核，MoE 模型（如 Ling）性能提升显著。 |
| `-DGGML_NATIVE=OFF` | `OFF` | 关闭 CPU 原生优化，保证二进制文件在所有 x86-64 机器上通用。 |
| `-DGGML_BACKEND_DL=ON` | `ON` | CUDA 后端动态加载，符合上游 Release 规范。 |
| `-DLLAMA_BUILD_SERVER=ON` | `ON` | 仅构建 `llama-server`，不编译多余工具，减小体积。 |
| `-DCMAKE_C_FLAGS` / `-DCMAKE_CXX_FLAGS` | `"/MT"` | 静态链接 VC++ 运行库，实现“解压即用”。 |

## 🎮 显存适配参考

| 显存容量 | 适用显卡 | 推荐模型精度 |
| :--- | :--- | :--- |
| 8GB | RTX 5050 / 5060 | Q4_K_M / Q5_K_M |
| 16GB | RTX 5060 Ti / 5070 / 5080 | Q8_0 / BF16 |
| 32GB | RTX 5090 | BF16 / UD-Q8_K_XL |

## 🔧 自定义编译

你可以在 `workflow_dispatch` 触发运行时修改以下参数：

*   **`tag_name`**: Release 版本号（如 `v1.0.1`）。
*   **`cuda_version`**: 切换 CUDA 版本（如 `13.0`），用于测试新驱动兼容性。

## ❓ 常见问题 (FAQ)

**Q: 启动时报错 `unknown model architecture: 'bailingmoe3'`？**
A: 这是因为使用了官方主线版本的 `llama.cpp`。Ling-3.0-tiny 使用了蚂蚁百灵的自定义 MoE 架构，必须使用本仓库编译的、基于 `aetherbird/llama.cpp@bailingmoe3-support` 分支的版本才能识别。

**Q: 日志中出现 `CORS is set to allow all origins` 警告？**
A: 这是安全提示。仅在局域网内使用时无需处理；若需暴露到公网，请添加启动参数 `--cors-allow-origin "https://your-domain.com"` 或设置 `--api-key`。

**Q: 为什么编译出来的包里有这么多 `.dll` 文件？**
A: 为了保证最大的兼容性和性能，我们将 CUDA 运行库和 VC++ 运行库一并打包。虽然体积稍大，但用户无需安装任何额外软件，真正做到“开箱即用”。

---

*基于 [aetherbird/llama.cpp](https://github.com/aetherbird/llama.cpp/tree/bailingmoe3-support) 的 `bailingmoe3-support` 分支构建*
