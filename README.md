# 🎉 AutoGLM GGUF 支持

**世界首个 GLM-4V 视觉语言模型的 GGUF 格式实现**

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10+-green.svg)](https://www.python.org/)
[![CUDA](https://img.shields.io/badge/CUDA-12.4+-orange.svg)](https://developer.nvidia.com/cuda-toolkit)

## 🌟 项目亮点

- ✅ **完整的 GLM-4V → GGUF 转换**：视觉编码器 + 语言模型
- ✅ **llama.cpp 多模态集成**：原生支持图像+文本输入
- ✅ **手机自动化控制**：实时截图分析 + ADB 命令执行
- ✅ **16K 上下文窗口**：支持长对话和复杂任务
- ✅ **高性能推理**：GPU 全加速，~37 tokens/sec

## 🚀 快速开始

### 编译 llama.cpp

```bash
# 1. 克隆仓库
git clone https://github.com/Luckybalabala/llama-cpp-AutoGLM.git
cd llama-cpp-AutoGLM

# 2. 编译（Windows + CUDA）
mkdir build
cd build
cmake .. -DGGML_CUDA=ON
cmake --build . --config Release

# 3. 转换 GLM-4V 模型
python convert_hf_to_gguf.py /path/to/AutoGLM-Phone-9B --outtype q4_k_s
python convert_hf_to_gguf.py /path/to/AutoGLM-Phone-9B --mmproj --outtype f16

# 4. 启动服务器
.\build\bin\Release\llama-server.exe \
  --model AutoGLM-Phone-9B-Q4_K_S.gguf \
  --mmproj mmproj-AutoGLM-Phone-9B-f16.gguf \
  --port 8080 --ctx-size 16384 --n-gpu-layers 99
```

## 📊 性能指标

| 指标 | 数值 |
|------|------|
| **模型大小** | 5.35GB (Q4_K_S) |
| **mmproj 大小** | 1.55GB |
| **上下文长度** | 16,384 tokens |
| **GPU 加速** | 41/41 层 |
| **推理速度** | ~37 tokens/sec |
| **平均任务时长** | 3-10秒 |

测试环境：RTX 4060 8GB, 32GB RAM, CUDA 12.4

## 🎯 核心功能

### 1. GLM-4V GGUF 转换支持

本 fork 添加了完整的 GLM-4V/AutoGLM 模型转换支持：

```bash
# 语言模型转换
python convert_hf_to_gguf.py /path/to/AutoGLM-Phone-9B \
    --outtype q4_k_s \
    --outfile AutoGLM-Phone-9B-Q4_K_S.gguf

# 视觉编码器转换（mmproj）
python convert_hf_to_gguf.py /path/to/AutoGLM-Phone-9B \
    --mmproj \
    --outtype f16
```

**支持的量化格式**：Q4_K_S, Q4_K_M, Q5_K_S, Q8_0, F16

### 2. 多模态推理

使用 OpenAI 兼容 API：

```python
import base64
import requests

# 发送图像+文本请求
with open("image.jpg", "rb") as f:
    image_base64 = base64.b64encode(f.read()).decode()

response = requests.post(
    "http://localhost:8080/v1/chat/completions",
    json={
        "model": "AutoGLM-Phone-9B",
        "messages": [{
            "role": "user",
            "content": [
                {"type": "text", "text": "描述这张图片"},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{image_base64}"}}
            ]
        }]
    }
)
```

### 3. 与 AutoGLM 项目集成

配合 [AutoGLM](https://github.com/THUDM/AutoGLM) 项目使用，实现手机自动化控制。

## 🔧 技术架构

### 核心突破

#### 1. 维度转换
```
GLM-4V 输出: [H×W, 6144]
    ↓ 降维
llama.cpp 期望: [H×W, 4096]
```

#### 2. 层映射修复
```python
# GLM-4V merger 结构
{
    "norm": "mlp.0",      # Layer normalization
    "up_proj": "mlp.1",   # Up projection
    "down_proj": "mlp.3"  # Down projection
}
```

#### 3. 可选 CLS Token
```cpp
if (model.class_embedding != nullptr) {
    // 添加 CLS token
    embeddings = ggml_concat(ctx, class_emb, embeddings, 1);
}
```

### 关键代码修改

| 文件 | 修改内容 | 说明 |
|------|---------|------|
| `convert_hf_to_gguf.py` | 新增 `Glm4vVisionModel` 类 | 支持 GLM-4V mmproj 转换 |
| `tools/mtmd/clip.cpp` | 修改 `build_internvl()` | 支持可选 CLS token |
| `gguf-py/gguf/constants.py` | 添加 INTERNVL projector | GLM-4V 使用此 projector 类型 |

## 📚 相关文档

- 📖 [llama.cpp 构建指南](docs/build.md) - 编译说明
- 📝 [多模态支持](tools/mtmd/README.md) - mmproj 使用指南
- � [AutoGLM 项目](https://github.com/THUDM/AutoGLM) - 手机自动化
- � [GLM-4V 模型](https://huggingface.co/THUDM/glm-4v-9b) - Hugging Face

## 🎓 使用示例

### 使用 llama-cli 测试

```bash
# 文本+图像输入
.\build\bin\Release\llama-cli \
  -m AutoGLM-Phone-9B-Q4_K_S.gguf \
  --mmproj mmproj-AutoGLM-Phone-9B-f16.gguf \
  -p "描述这张图片" \
  --image screenshot.jpg
```

### 使用 curl 测试 API

```bash
# 启动服务器
.\build\bin\Release\llama-server.exe \
  -m AutoGLM-Phone-9B-Q4_K_S.gguf \
  --mmproj mmproj-AutoGLM-Phone-9B-f16.gguf \
  --port 8080 -ngl 99 -c 16384

# 发送请求（需要先将图像转为 base64）
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4-vision-preview",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "这是什么？"},
        {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,<BASE64>"}}
      ]
    }]
  }'
```

## 🔍 技术细节

### 数据流

```
用户任务
   ↓
实时截图 (ADB)
   ↓
Base64 编码
   ↓
Vision Encoder (mmproj)
   ↓ [H×W, 6144]
维度降维
   ↓ [H×W, 4096]
Language Model
   ↓
<think>分析</think>
<answer>操作</answer>
   ↓
解析 + 执行 (ADB)
   ↓
结果反馈
```

### 关键技术

**1. 视觉编码**
- EVA-CLIP-6B 作为视觉编码器
- Pixel unshuffle 空间下采样
- 可学习的位置编码

**2. 视觉-语言对齐**
- MLP merger 降维投影
- FFN (SwiGLU) 激活函数
- 残差连接 + Layer Norm

**3. 推理优化**
- CUDA 图优化
- Flash Attention
- KV Cache 管理

## 🛠️ 故障排除

### 常见问题

**Q: llama-server 启动失败**
```bash
# 检查 CUDA
nvidia-smi

# 重新编译
cd llama.cpp
.\rebuild_llama_server.ps1
```

**Q: 上下文溢出**
```bash
# 增加上下文
--ctx-size 16384  # 或更大
```

**Q: ADB 连接失败**
```bash
# 重启 ADB
adb kill-server
adb start-server

# 无线连接
adb connect 192.168.x.x:port
```

更多问题：查看 [Issues](https://github.com/Luckybalabala/llama-cpp-AutoGLM/issues)

## 🤝 贡献

欢迎各种形式的贡献！

- 🐛 报告 Bug - 通过 GitHub Issues
- 💡 提出功能建议 - 通过 GitHub Discussions
- 📝 改进文档 - 提交 Pull Request
- 🔧 提交代码 - Fork 后提交 PR

查看上游 [llama.cpp 贡献指南](CONTRIBUTING.md) 了解代码规范。

### 贡献者

感谢所有为本项目做出贡献的开发者！

## 📄 许可证

本项目采用 [MIT 许可证](LICENSE)。

## 🙏 致谢

- [llama.cpp](https://github.com/ggerganov/llama.cpp) - 优秀的推理框架
- [AutoGLM](https://github.com/THUDM/AutoGLM) - 原始模型
- [GLM-4V](https://github.com/THUDM/GLM-4) - 视觉语言模型

## 📞 联系方式

- **Issues**: [GitHub Issues](https://github.com/Luckybalabala/llama-cpp-AutoGLM/issues)
- **Discussions**: [GitHub Discussions](https://github.com/Luckybalabala/llama-cpp-AutoGLM/discussions)
- **微信群**: [加入讨论](resources/WECHAT.md)

## 🗺️ 实现状态

### ✅ 已完成
- [x] GLM-4V 语言模型 GGUF 转换
- [x] GLM-4V 视觉编码器 mmproj 转换
- [x] InternVL projector 架构适配
- [x] 可选 CLS token 支持
- [x] 16K 上下文窗口支持
- [x] OpenAI 兼容 API

### � 技术实现
- **Projector Type**: INTERNVL
- **Vision Encoder**: EVA-CLIP-6B
- **Text Model**: GLM-4-9B
- **Context Length**: 16,384 tokens
- **Output Dimension**: 4096 (从 6144 降维)

## 📈 项目状态

![GitHub stars](https://img.shields.io/github/stars/Luckybalabala/llama-cpp-AutoGLM?style=social)
![GitHub forks](https://img.shields.io/github/forks/Luckybalabala/llama-cpp-AutoGLM?style=social)
![GitHub issues](https://img.shields.io/github/issues/Luckybalabala/llama-cpp-AutoGLM)

---

**⭐ 如果这个项目对你有帮助，请给我们一个 Star！**

**🎉 这是世界首个 GLM-4V GGUF 实现，让我们一起推动开源多模态 AI 的发展！**
