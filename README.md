# llama-server-builder

自动从 [aetherbird/llama.cpp](https://github.com/aetherbird/llama.cpp/tree/bailingmoe3-support) 编译 Ling-3.0-tiny 推理服务器。

## 怎么用

1. 去 [Actions](../../actions) 页面
2. 选 "Build Ling-3.0-tiny Server for RTX 50 Series"
3. 点 "Run workflow"，输入版本号（如 v1.0.0）
4. 等 ~15-20 分钟，去 [Releases](../../releases) 下载编译好的包

## 编译参数

- CUDA 12.8.1 + sm_120（Blackwell）
- MSVC 静态链接（/MT）
- Ninja 构建系统
- Flash Attention 全量化内核
