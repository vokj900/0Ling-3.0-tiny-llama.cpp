# llama-server-builder

> 自动从 [aetherbird/llama.cpp](https://github.com/aetherbird/llama.cpp/tree/bailingmoe3-support) 编译 Ling-3.0-tiny 推理服务器，专为 **RTX 50 系列（Blackwell）** 优化。

## 🏗️ 架构

```
┌─────────────────────────────────────────────────────┐
│              GitHub Actions (windows-2022)         │
├─────────────────────────────────────────────────────┤
│  Step 0: 安装 CUDA Toolkit ${{ inputs.cuda_version }}        │
│  Step 1: Checkout aetherbird/llama.cpp @ bailingmoe3-support │
│  Step 2: Build (MSVC + Ninja + sm_120)           │
│  Step 3: Upload Artifacts                         │
│  Step 4: Download → Zip → Release                │
└─────────────────────────────────────────────────────┘
                        ↓
            📦 Release: llama-server-rtx50-cuda12.8-v1.0.0.zip
```

## 🚀 怎么用

1. 去仓库的 **[Actions](../../actions)** 页面
2. 选 **"Build Ling-3.0-tiny Server for RTX 50 Series"**
3. 点 **"Run workflow"**，填写：
   - `tag_name`：版本号，如 `v1.0.0`
   - `cuda_version`：CUDA 版本，默认 `12.8.1`（一般不用改）
4. 等 **~15-20 分钟**
5. 去 **[Releases](../../releases)** 下载 `llama-server-rtx50-cuda12.8-v1.0.0.zip`

## 📦 产物内容

| 文件 | 说明 |
|------|------|
| `llama-server.exe` | 推理服务器主程序 |
| `ggml*.dll` | GGML 推理引擎核心 |
| `llama*.dll` | Llama 模型支持库 |
| `mtmd.dll` | 多模态支持 |
| `cudart64_*.dll` | CUDA 运行库 |
| `cublas64_*.dll` | CUDA BLAS 数学库 |
| `cublasLt64_*.dll` | CUDA BLASLt 加速库 |
| `vcruntime140*.dll` | VC++ 运行库 |
| `msvcp140*.dll` | VC++ 标准库 |
| `concrt140.dll` | 并发运行时 |

## ⚙️ 编译参数

| 参数 | 值 | 说明 |
|------|-----|------|
| `CMAKE_C_COMPILER` | `cl.exe` (MSVC) | 强制使用 Visual Studio 编译器 |
| `CMAKE_CUDA_COMPILER` | `nvcc.exe` | CUDA 编译器 |
| `CMAKE_CUDA_ARCHITECTURES` | `120` | Blackwell 架构代号 |
| `GGML_CUDA` | `ON` | 启用 CUDA 后端 |
| `GGML_CUDA_FA_ALL_QUANTS` | `ON` | Flash Attention 全量化 |
| `LLAMA_BUILD_SERVER` | `ON` | 构建 llama-server |
| `LLAMA_BUILD_TOOLS` | `ON` | 构建配套工具 |
| `CMAKE_C_FLAGS` | `/MT` | 静态链接 VC 运行库 |
| `OPENSSL_USE_STATIC_LIBS` | `ON` | 静态链接 OpenSSL |

## 🔧 自定义

### 换 CUDA 版本
Run workflow 时把 `cuda_version` 改成 `12.6.3` 或 `13.0` 等。

### 换分支/仓库
编辑 `.github/workflows/build.yml` 里的：
```yaml
repository: aetherbird/llama.cpp   # 改成你的 fork
ref: bailingmoe3-support          # 改成你的分支
```

### 加编译参数
在 `CMake 配置` 步骤的 `cmake -S . -B build ...` 后面加 `-D你的参数=值`。

## 💻 用户端使用

下载 zip → 解压到纯英文目录 → 放入 GGUF 模型 → 运行：

```powershell
.\llama-server.exe -m your_model.gguf --jinja --host 0.0.0.0 --port 8080 -ngl 999 -t 16
```

浏览器访问 `http://localhost:8080` 即可对话。

## ❓ FAQ

**Q: Release 里只有源码 zip，没有 exe？**
A: 检查构建日志最后几步是否成功。如果 `Upload build artifacts` 或 `Create GitHub Release` 报错了，看日志排查。

**Q: 编译失败了怎么办？**
A: 把失败步骤的日志发给我，我帮你定位问题。

**Q: 能编 Linux 版吗？**
A: 可以，但需要改 Runner 为 `ubuntu-22.04` 并调整依赖安装方式。需要的话告诉我。

## 📄 License

本仓库仅含构建脚本，license 遵循上游 [aetherbird/llama.cpp](https://github.com/aetherbird/llama.cpp) 的 MIT 协议。
