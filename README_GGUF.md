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

### 5 分钟上手

```bash
# 1. 克隆仓库
git clone https://github.com/Luckybalabala/llama-cpp-AutoGLM.git
cd llama-cpp-AutoGLM

# 2. 安装依赖
pip install -r requirements.txt

# 3. 启动服务器（新终端）
cd llama-cpp-bin
.\llama-server.exe --model ../models/converted/AutoGLM-Phone-9B-Q4_K_S.gguf \
  --mmproj ../models/converted/AutoGLM-Phone-9B-mmproj.gguf \
  --port 8080 --ctx-size 16384 --n-gpu-layers 99

# 4. 执行任务
python main.py --base-url http://localhost:8080/v1 "打开微信"
```

详细教程：[快速开始指南](QUICK_START_GGUF.md)

## 📊 性能指标

| 指标 | 数值 |
|------|------|
| **模型大小** | 5.35GB (Q4_K_S) |
| **mmproj 大小** | 1.55GB |
| **上下文长度** | 16,384 tokens |
| **GPU 加速** | 41/41 层 |
| **推理速度** | ~37 tokens/sec |
| **平均任务时长** | 3-10秒 |

测试环境：RTX 4090, 32GB RAM, CUDA 12.4

## 🎯 核心功能

### 1. GGUF 模型转换

```bash
# 语言模型转换
python llama.cpp/convert_hf_to_gguf.py models/AutoGLM-Phone-9B \
    --outtype q4_k_s \
    --outfile models/converted/AutoGLM-Phone-9B-Q4_K_S.gguf

# 视觉编码器转换
.\generate_mmproj_simple.ps1
```

支持的量化格式：Q4_K_S, Q4_K_M, Q5_K_S, Q8_0, F16

### 2. 多模态推理

```python
from phone_agent.model import ModelClient

client = ModelClient("http://localhost:8080/v1")
response = client.request(
    text="这是什么？",
    image_base64=screenshot_b64
)
```

### 3. 手机自动化

```bash
# 打开应用
python main.py --base-url http://localhost:8080/v1 "打开微信"

# 复杂任务
python main.py --base-url http://localhost:8080/v1 \
  "在微信搜索autoGLM群并发送消息：测试成功"
```

支持操作：Launch, Tap, Type, Swipe, Back, Home

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

### 关键文件修改

| 文件 | 修改内容 | 状态 |
|------|---------|------|
| `llama.cpp/tools/mtmd/clip.cpp` | InternVL 架构适配 | ✅ |
| `llama.cpp/convert_hf_to_gguf.py` | GLM-4V 转换支持 | ✅ |
| `phone_agent/model/client.py` | API 兼容性改进 | ✅ |

## 📚 文档

- 📖 [快速开始](QUICK_START_GGUF.md) - 5分钟上手指南
- 📝 [完整技术文档](GLM4V_GGUF_COMPLETE.md) - 深入技术细节
- 🔄 [更新日志](CHANGELOG_GGUF.md) - 版本变更记录
- 🛠️ [mmproj 生成指南](MMPROJ_GENERATION_GUIDE.md) - 转换教程
- 🤝 [贡献指南](CONTRIBUTING_GGUF.md) - 参与开发

## 🎓 使用示例

### 基础用法

```python
# 示例 1: 打开应用
python main.py --base-url http://localhost:8080/v1 "打开微信"

# 示例 2: 导航操作
python main.py --base-url http://localhost:8080/v1 "返回主屏幕"

# 示例 3: 搜索和发送
python main.py --base-url http://localhost:8080/v1 \
  "在微信搜索autoGLM并发送：你好"
```

### 高级用法

```python
from phone_agent import PhoneAgent
from phone_agent.model import ModelConfig

# 自定义配置
config = ModelConfig(
    base_url="http://localhost:8080/v1",
    model_name="AutoGLM-Phone-9B",
    temperature=0.0,
    max_tokens=500
)

agent = PhoneAgent(config)
result = agent.execute_task("你的任务")
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

更多问题：[快速开始指南 - 常见问题](QUICK_START_GGUF.md#常见问题)

## 🤝 贡献

欢迎各种形式的贡献！

- 🐛 报告 Bug
- 💡 提出功能建议
- 📝 改进文档
- 🔧 提交代码

查看 [贡献指南](CONTRIBUTING_GGUF.md) 了解详情。

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

## 🗺️ 路线图

### 已完成 ✅
- [x] GLM-4V GGUF 转换
- [x] llama.cpp 集成
- [x] 手机自动化基础功能
- [x] 16K 上下文支持

### 进行中 🚧
- [ ] UI 导航优化
- [ ] 上下文自动管理
- [ ] 更多应用支持

### 计划中 📋
- [ ] Web UI 改进
- [ ] 批量任务处理
- [ ] 性能监控
- [ ] 更多语言支持

## 📈 项目状态

![GitHub stars](https://img.shields.io/github/stars/Luckybalabala/llama-cpp-AutoGLM?style=social)
![GitHub forks](https://img.shields.io/github/forks/Luckybalabala/llama-cpp-AutoGLM?style=social)
![GitHub issues](https://img.shields.io/github/issues/Luckybalabala/llama-cpp-AutoGLM)

---

**⭐ 如果这个项目对你有帮助，请给我们一个 Star！**

**🎉 这是世界首个 GLM-4V GGUF 实现，让我们一起推动开源多模态 AI 的发展！**
