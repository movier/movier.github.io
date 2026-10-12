---
author: 小明
pubDatetime: 2026-10-12 20:10:00
title: 在 Windows 11 上安装 PaddleOCR-VL
featured: true
draft: true
tags:
  - AI
  - OCR
  - Ollama
description: 详细记录在 Windows 11 系统下，从零搭建 Python 环境，并成功安装 PaddleOCR-VL 的全流程与避坑指南。
---

OCR（Optical Character Recognition，光学字符识别）是指利用电子设备将图片、扫描件中的手写或印刷文字转换为机器可读文本的技术，说简单点就是识别图片中的文字。

进入 AI 时代后，OCR 成为了大模型感知物理世界数据的“先锋之眼”。尤其在**企业知识库（RAG，检索增强生成）**建设中，大量核心业务资产是以扫描版 PDF、图表研报、合同发票等非结构化形式存在的。传统文本提取方式面对复杂版面、多列排版和嵌套表格往往束手无策，导致喂给大模型的知识碎片化、逻辑混乱。而结合了视觉语言（Vision-Language, VL）能力的现代 OCR（如 PaddleOCR-VL），能够精准解析文档布局、还原表格结构，是保障企业私有知识库质量的关键前置环节。

不过，搭建环境的过程却是一波三折。由于官方推荐使用 Docker 安装（这样可以避免环境问题），再加上自己电脑上安装了 WSL 和 Docker，我想这种安装方式应该没问题，结果运行起来不是报错就是卡住，反复折腾无果，最终放弃了。过了两个月，我又想折腾一下（高质量 OCR 的需求太普遍了），这次在网上搜了很多信息，打算换一种方式安装，回归传统思路——直接在 Windows 本地使用最近极速火爆的 Python 包管理工具 **UV** 来隔离运行环境，按部就班配置后，最终顺利跑通！

本文就来详细记录这次实战过程，带你在 Windows 11 上利用 UV 高效安装 PaddleOCR-VL。


## Table of contents

## 1. 准备工作

在开始之前，介绍一下我的系统环境：

1. **操作系统**：Windows 11
2. **GPU**：Nvidia RTX 3060 Ti
3. **CUDA 版本**：13.1

## 2. 环境管理：安装 UV 并创建隔离环境

为了避免污染全局 Python 环境，这次我们放弃体积臃肿的 Conda，转而采用当下基于 Rust 构建、号称“极速”的现代包管理工具 **UV**。

### 2.1 安装 UV

在 Windows 11 下，无需手动下载安装包，直接打开 PowerShell 执行官方安装脚本即可：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

安装完成后，关闭当前终端并重新打开，输入 `uv --version`，如果能输出版本号说明安装成功。

### 2.2 创建并激活 Python 3.12 虚拟环境

UV 支持无需提前下载 Python 安装包，直接在创建环境时自动拉取并配置指定的 Python 版本，打开终端，进入你的项目目录，然后运行：

```powershell
# 1. 创建基于 Python 3.12 的虚拟环境
uv venv --python 3.12

# 2. 激活虚拟环境，终端最前方出现目录名表示激活成功
.\.venv\Scripts\activate

# 退出虚拟环境
deactivate
```

> **踩坑提示：激活脚本报错“无法加载模块/禁止运行脚本”**
>
> 在 Windows 11 的 PowerShell 中执行激活命令时，你可能会遇到如下红字报错：
> `File ...\activate cannot be loaded because running scripts is disabled on this system...`（在此系统上禁止运行脚本）
> 
> **解决方案**：这是 Windows 系统的安全执行策略限制。只需在当前 PowerShell 窗口中运行以下命令，放开当前用户的脚本执行权限即可解决：
> ```powershell
> Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

## 3. 安装 PaddlePaddle 核心库与 PaddleOCR

确保你已经激活了上面创建的虚拟环境，接下来我们将使用 `uv pip` 极速完成依赖的安装。

### 步骤一：安装 PaddlePaddle 核心库

首先需要根据显卡驱动支持的 CUDA 版本选择对应的轮子（Wheel）。可以先在终端输入命令确认支持的最高 CUDA 版本：

```powershell
nvidia-smi
```

根据 [飞桨官方快速安装指南](https://www.paddlepaddle.org.cn/install/quick)，选择安装对应的版本，CUDA 版本是向下兼容的，所以虽然我的 CUDA 版本是 13.1，但是安装 PaddlePaddle 支持的最新 CUDA 版本 12.9 也是没有问题的。使用 `uv pip` 命令执行安装：

```powershell
uv pip install paddlepaddle-gpu==3.3.1 -i https://www.paddlepaddle.org.cn/packages/stable/cu129/
```

> **验证 GPU 是否正确调用：**  
> 安装完成后，执行以下测试命令：
> ```powershell
> python -c "import paddle; paddle.utils.run_check()"
> ```
> 终端正确输出 `PaddlePaddle is installed successfully!` 且检测到了你的显卡型号，就说明安装成功了。

### 步骤二：安装 PaddleOCR 全功能套件

核心库就绪后，直接使用 UV 一键安装带有所有扩展依赖（包含版面分析、视觉多模态支持等）的 PaddleOCR 完整版：

```powershell
uv pip install "paddleocr[all]"
```

至此我们已经安装了 PaddleOCR-VL 运行所需要的所有依赖，甚至把 PaddleOCR 的所有工具包都安装上了，接下去可以尝试运行 PaddleOCR-VL 了。

## 4. 快速测试与验证

安装完成后，我们直接使用 PaddleOCR 提供的命令行工具进行多模态文档解析（`doc_parser`）验证即可。

### 4.1 准备测试文件

在当前项目根目录下放置一份待测试的文档，例如名为 `sample.pdf` 的文档（也可以是图片格式，如 `.png`、`.jpg`）。

### 4.2 运行文档解析命令

在激活了虚拟环境的终端中，执行以下命令开始推理：

```powershell
paddleocr doc_parser -i "./sample.pdf" --save_path ./output
```

### 4.3 首次运行说明

初次执行此命令时，系统会自动从 Paddle 官方仓库下载所需的检测、识别、版面分析以及表格还原相关的预训练模型。
- 这些模型会自动下载并缓存在本地目录：`C:\Users\<你的用户名>\.paddlex\` 下。
- 耐心等待下载完成后，终端便会输出文档每一页的解析进度日志。

推理结束后，在 `./output` 目录下你可以看到解析出来的结构化 Markdown 文本、提取出的独立表格以及标注了版面区域的切片图片。至此，PaddleOCR-VL 验证成功跑通！

## 5. 使用 VLM 推理服务

上述方法只适用于快速测试与验证，为了提升推理性能，并改善生产环境下的资源利用和服务稳定性，还是推荐使用 VLM 推理服务。

在本地运行大模型，最方便、资源占用最轻量的工具莫过于 **Ollama**。借助开源社区将模型量化后的 GGUF 权重，我们可以直接拉取模型：[AuditAid/PaddleOCR-VL-1.6-0.9B](https://ollama.com/AuditAid/PaddleOCR-VL-1.6-0.9B)。

### 步骤 1：下载模型

确保你已经在 Windows 上安装并启动了 [Ollama](https://ollama.com/)。打开终端，拉取多模态视觉模型镜像：

```powershell
ollama pull AuditAid/PaddleOCR-VL-1.6-0.9B
```

下载完成后，Ollama 默认会在本地后台开启提供兼具 OpenAI 兼容 API 格式的端口服务（默认监听 `http://localhost:11434`）。

### 步骤 2：使用 VLM 推理服务

现在，我们就可以使用 VLM 推理服务了，在 PowerShell 中执行如下命令：

```powershell
paddleocr doc_parser --input "./sample.pdf" --vl_rec_backen llama-cpp-server --vl_rec_server_url http://localhost:11434/v1 --vl_rec_api_model_name "AuditAid/PaddleOCR-VL-1.6-0.9B" --save_path ./output
```

通过观察 GPU 资源使用情况，可以发现在使用 VLM 推理服务后资源占用得到了极大的改善。

## 结语

我一直对小模型有着深深地迷恋。在这个动辄几百上千亿参数的大模型狂飙时代，大家似乎习惯了用庞大的算力堆砌性能。但作为一个只有 **0.9B（约 9 亿参数）** 的小模型，PaddleOCR-VL 无疑是垂直场景下“以小博大”的绝佳典范。

它不仅没有牺牲实用性，反而将有限的参数量死死聚焦在“文档理解与图文排版重构”这一垂直领域，在消费级显卡上就能以极低的时延流畅跑完长文档。在搭建企业私有知识库（RAG）的前置工程中，它证明了我们未必要为了一次文档清洗去调用高昂的大模型商业 API，一个极其轻巧但足够专注的本地小模型，就已经能构筑起一道坚实、高质量的数据入口。

技术的魅力往往不在于“越大越好”，而在于“恰到好处”。希望这篇基于 Windows 11 本地纯纯踩坑出来的实战笔记，能帮你省去折腾容器的那些夜晚，真正让这个精致的多模态小模型为你所用！

## 参考

* [本地 OCR 到 VLM：四个 PDF 识别方案实测](https://h89.cn/archives/596.html)
