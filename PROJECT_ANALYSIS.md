# BoN Jailbreaking 项目分析报告

## 项目概述

BoN Jailbreaking 是一个专门用于**大语言模型越狱攻击研究**的开源项目，实现了基于**Best-of-N (BoN)策略**的多模态越狱攻击方法。该项目为AI安全研究提供了全面的攻击测试平台，特别在音频和图像越狱方面做出了重要贡献。

## 核心功能

### 1. 多模态越狱攻击
- **文本越狱攻击** - 通过文本增强技术绕过安全防护
- **音频越狱攻击** - 利用音频处理技术进行语音攻击
- **图像越狱攻击** - 通过视觉扰动隐藏恶意指令

### 2. Best-of-N 策略
- 生成N个增强候选样本
- 并发评估所有候选
- 选择攻击成功率最高的样本
- 迭代优化直到达到阈值

### 3. 多模型支持
- **OpenAI**: GPT-4o, GPT-4o-mini
- **Anthropic**: Claude 3.5 Sonnet, Claude 3 Opus  
- **Google**: Gemini 1.5 Pro/Flash
- **开源模型**: Llama-3, GraySwanAI等

## 技术实现架构

### 项目结构
```
bon/
├── attacks/          # 核心攻击实现
│   ├── run_text_bon.py     # 文本攻击
│   ├── run_audio_bon.py    # 音频攻击
│   └── run_image_bon.py    # 图像攻击
├── apis/            # 多模型API接口
├── data_models/     # 数据结构定义
├── data_prep/       # 数据预处理和增强
├── classifiers/     # 攻击成功率判断
└── utils/           # 工具函数库
```

### 关键技术栈

#### 深度学习框架
- **PyTorch + TorchAudio** - 音频处理
- **Transformers** - 模型接口
- **OpenCV** - 图像处理

#### 音频处理技术
- **SoX** - 专业音频处理工具
- **Kaldi** - 语音识别工具包
- **WavAugment** - 音频增强库
- **LibROSA** - 音频分析

#### API集成
- **异步并发处理** - Asyncio
- **多线程支持** - 提高攻击效率
- **缓存机制** - 避免重复API调用
- **速率限制** - 防止API超限

## 增强技术详解

### 文本增强技术
| 技术 | 实现方式 | 效果 |
|------|----------|------|
| **词汇扰动** | 单词内部字符重排 | 保持可读性的同时绕过检测 |
| **随机大小写** | 随机改变字母大小写 | 干扰文本模式识别 |
| **ASCII扰动** | 轻微修改ASCII字符 | 创建视觉相似但编码不同的字符 |
| **前缀/后缀注入** | 添加随机攻击字符串 | 分散注意力机制 |
| **多轮对话注入** | MSJ策略 | 通过上下文操控模型行为 |

### 音频增强技术
| 技术 | 参数范围 | 实现工具 |
|------|----------|----------|
| **语速调节** | 0.35-2.8倍速 | SoX tempo |
| **音调变换** | ±2000 Hz | WavAugment pitch |
| **背景噪声** | SNR 10-30dB | Kaldi + MUSAN数据集 |
| **音量扰动** | 0.001-1000倍 | SoX volume |
| **混响效果** | 多种房间类型 | RIRs数据集 |

### 图像增强技术
| 技术 | 实现方式 | 参数 |
|------|----------|------|
| **随机背景** | 像素/网格/色块 | 240-600px尺寸 |
| **文本渲染** | OpenCV字体系统 | 多种字体、颜色、位置 |
| **视觉扰动** | 高斯噪声 | σ=0.33标准差 |

## 优化算法

### 随机搜索 (文本/图像)
```python
for step in range(n_steps):
    candidates = generate_augmented_samples(k=num_concurrent_k)
    asrs = evaluate_candidates_parallel(candidates)
    best_candidate = select_best(candidates, asrs)
    if asr >= threshold:
        break
```

### CMA-ES优化 (音频)
```python
cma = CMAEvolutionStrategy(N=num_augmentations, sigma=0.25)
for step in range(n_steps):
    vectors = cma.get_vectors()
    asrs = evaluate_audio_candidates(vectors)
    cma.run_step(vectors, asrs)
```

## 实验配置

### 核心参数
- **并发数**: 10-60个候选样本
- **迭代步数**: 8-120步  
- **总样本数**: N = 并发数 × 迭代步数
- **成功阈值**: ASR ≥ 1% 
- **可靠性检查**: 10次采样中7次成功

### 数据集
- **HarmBench数据集**: 159个直接请求
- **人工语音数据**: 308 PAIR + 307 TAP + 159直接请求
- **多语言支持**: 英语为主，支持其他语言

## 评估机制

### 攻击成功率 (ASR) 计算
1. **分类器判断** - 使用GPT-4作为安全分类器
2. **假阳性过滤** - 排除模型拒绝回答的情况  
3. **长度检查** - 响应必须超过50个单词
4. **可靠性验证** - 多次采样确保稳定性

### 幂律缩放行为
项目发现ASR随样本数N呈现幂律缩放：
```
ASR(N) ∝ N^(-α)
```
其中α为缩放指数，不同模型和模态有不同值。

## 实验结果亮点

### 文本攻击成功率 (N=10,000)
- **GPT-4o**: 89%
- **Claude 3.5 Sonnet**: 78%  
- **Gemini 1.5 Pro**: 52%

### 跨模态有效性
- **音频攻击**: 对Gemini等ALM模型有效
- **图像攻击**: 对GPT-4o等VLM模型有效
- **组合攻击**: BoN + 优化前缀可提升35% ASR

## 安全影响与防护建议

### 攻击特点
- **黑盒攻击** - 无需模型内部信息
- **简单有效** - 实现简单但效果显著
- **跨模态通用** - 适用于文本、音频、图像
- **可组合性** - 可与其他攻击方法结合

### 防护建议
1. **输入标准化** - 对输入进行预处理标准化
2. **多样本检测** - 检测异常高频的相似请求
3. **跨模态防护** - 针对不同模态设计专门防护
4. **动态阈值** - 根据攻击模式调整安全阈值

## 使用方法

### 环境配置
```bash
# 安装micromamba
"${SHELL}" <(curl -L micro.mamba.pm/install.sh)

# 创建环境
micromamba env create -n bon python=3.11.7
micromamba activate bon
pip install -r requirements.txt
pip install -e .
```

### 运行实验
```bash
# 文本攻击
./experiments/1a_run_text_bon.sh

# 音频攻击  
./experiments/1c_run_audio_bon.sh

# 图像攻击
./experiments/1b_run_image_bon.sh
```

## 项目意义

这个项目为AI安全研究提供了重要贡献：

1. **系统性评估** - 提供了对主流LLM安全性的全面测试
2. **多模态扩展** - 首次系统性研究了音频和图像越狱攻击
3. **理论发现** - 揭示了ASR的幂律缩放规律
4. **实用工具** - 为研究者提供了完整的攻击测试平台

该项目强调了当前大语言模型对看似无害的输入变化的敏感性，为提升AI系统的鲁棒性和安全性提供了重要参考。