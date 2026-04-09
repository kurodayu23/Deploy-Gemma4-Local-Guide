[English](README.md) | [简体中文](README_zh.md) | [日本語](README_ja.md)

---

# 🚀 本地部署 Google Gemma 4：一份来自实战的完整指南

> **写在前面**: 这不是一份复制粘贴的教程。这是我花了整整一个周末，在 VRAM 限制、模型量化取舍、以及 Ollama 在 Windows 上的各种坑里反复挣扎之后写下的。如果你想在自己的硬件上零成本运行 Gemma 4，这份指南能帮你省下好几个小时的折腾时间。

---

## 目录

- [为什么选 Gemma 4？](#为什么选-gemma-4)
- [前置条件和硬件实力检查](#前置条件和硬件实力检查)
- [第一阶段：在 Windows 上装好 Ollama](#第一阶段在-windows-上装好-ollama)
- [第二阶段：拉取并运行 Gemma 4](#第二阶段拉取并运行-gemma-4)
- [第三阶段：把模型存储路径迁移到 D 盘](#第三阶段把模型存储路径迁移到-d-盘)
- [第四阶段：交互式测试模型](#第四阶段交互式测试模型)
- [第五阶段：暴露本地 REST API](#第五阶段暴露本地-rest-api)
- [第六阶段：与 Python 集成（LangChain / 原生 HTTP）](#第六阶段与-python-集成langchain--原生-http)
- [第七阶段：性能调优与显存管理](#第七阶段性能调优与显存管理)
- [疑难杂症排查（那些没人告诉你的坑）](#疑难杂症排查那些没人告诉你的坑)
- [接下来做什么？](#接下来做什么)

---

## 为什么选 Gemma 4？

说实话，本地模型我试了一大堆：Llama 3、Mistral、Phi-3、Qwen2……每个都有自己的甜区。但 Google 的 Gemma 4 确实让我眼前一亮，原因如下：

- **代码理解能力极其恐怖**。我把一个 500 行的 C++ Qt 文件扔给它，它居然准确找出了一个导致 LNK4042 链接错误的命名空间污染问题。大部分 7B 模型只会在那里胡说八道。
- **天生多语言**。我日常要处理中英日三种文档，Gemma 4 在三种语言之间切换毫无质量损失。
- **真正的开源权重**。Google 以宽松协议发布，你可以微调、商用、随便搞。
- **推理效率高**。4B 版本在 8GB 显存上跑得很舒服，12B 版本需要大约 10-12GB，完全不需要 A100。

---

## 前置条件和硬件实力检查

在开始之前，我们先来点实话：

| 硬件 | 最低配置（4B 模型） | 推荐配置（12B 模型） | 我的配置 |
|------|-------------------|---------------------|---------|
| GPU | GTX 1660 (6GB) | RTX 3060 (12GB) | RTX 3060 12GB |
| 内存 | 16GB | 32GB | 32GB DDR4 |
| 磁盘空间 | 5GB 空闲 | 15GB 空闲 | 外置 D 盘 |
| 系统 | Windows 10/11 | Windows 10/11 | Windows 11 |

**重要提示**：如果你没有独立 NVIDIA 显卡，通过 Ollama 你*依然可以*用纯 CPU 模式运行 Gemma 4。会很慢（4B 大概每秒 2-5 个 token），但它能跑。我在火车上用笔记本跑过，体验不算理想，但能用。

### 必备软件

在继续之前，先确认这些：

```powershell
# 检查是否安装了 NVIDIA 驱动
nvidia-smi
```

如果这个命令返回了你的 GPU 信息，说明没问题。如果报错，去 [nvidia.com/drivers](https://www.nvidia.com/drivers) 安装最新驱动。

---

## 第一阶段：在 Windows 上装好 Ollama

Ollama 是让本地模型部署变得无痛的运行时。你可以把它理解成"LLM 界的 Docker"。

### 方式 A：通过 winget 安装（推荐）

```powershell
winget install --id Ollama.Ollama -e --source winget
```

### 方式 B：手动下载

访问 [ollama.com/download](https://ollama.com/download)，下载 Windows 安装器，一路点下一步就行。

### 验证安装

```powershell
ollama --version
```

你应该看到类似这样的输出：

```
ollama version is 0.6.2
```

> **我踩过的坑**：通过 winget 安装后，我必须**重新打开终端** `ollama` 命令才能被识别。安装器把它加入了 PATH，但当前会话不会自动更新。如果显示"无法识别的命令"，别慌——开个新的 PowerShell 窗口就好。

---

## 第二阶段：拉取并运行 Gemma 4

这一步才是真正的高潮。Ollama 把模型下载、量化和服务全部整合在一条命令里。

### 选择你的版本

```
gemma3:4b    → 约 2.5GB 下载量, 需要 ~4GB 显存   (适合快速任务)
gemma3:12b   → 约 7.5GB 下载量, 需要 ~10GB 显存  (写代码的甜区)
gemma3:27b   → 约 16GB 下载量, 需要 ~20GB 显存   (野兽模式，需要牛逼的显卡)
```

### 拉取模型

```powershell
# 12B 版本（如果你有 12GB+ 显存，我强烈推荐）
ollama pull gemma3:12b

# 轻量 4B 版本
ollama pull gemma3:4b
```

这需要一点时间，具体取决于你的网速。12B 模型大约 7.5GB。去泡杯咖啡吧。☕

### 验证下载

```powershell
ollama list
```

预期输出：

```
NAME            ID              SIZE      MODIFIED
gemma3:12b      a]b2c3d4e5f6    7.5 GB    2 minutes ago
```

---

## 第三阶段：把模型存储路径迁移到 D 盘

**如果你的 C 盘空间紧张，这一步至关重要。** Ollama 默认把模型存放在 `C:\Users\<用户名>\.ollama\models`。我的 C 盘是个 256GB 的 SSD，常年接近爆满。我需要把模型塞到 D 盘。

### 步骤 1：创建目标目录

```powershell
New-Item -ItemType Directory -Path "D:\LLM_Models" -Force
```

### 步骤 2：设置环境变量（系统级，永久生效）

```powershell
# 以管理员身份运行 PowerShell
[System.Environment]::SetEnvironmentVariable("OLLAMA_MODELS", "D:\LLM_Models", "Machine")
```

### 步骤 3：重启 Ollama 服务

```powershell
# 强制关闭正在运行的 Ollama 进程
taskkill /f /im ollama.exe

# 重新启动（会自动读取新路径）
ollama serve
```

### 步骤 4：验证是否生效

```powershell
# 拉取一个小模型测试
ollama pull gemma3:4b

# 检查 D:\LLM_Models 下是否出现了文件
Get-ChildItem "D:\LLM_Models" -Recurse | Select-Object -First 10
```

如果你在 `D:\LLM_Models` 下看到了 `.bin` 文件，恭喜——你珍贵的 C 盘保住了。

> **我在这里犯的错**：我最初把环境变量设成了用户级（User）而不是机器级（Machine）。Ollama 的后台服务运行在不同的上下文中，所以它完全无视了我的用户级环境变量，继续往 C 盘疯狂下载。请用 `"Machine"` 作用域。相信我。

---

## 第四阶段：交互式测试模型

让我们真正跟模型对话试试：

```powershell
ollama run gemma3:12b
```

这会进入一个交互式聊天界面。试试这些提示词来验证模型是否正常工作：

```
>>> 解释 Python 中 asyncio.gather 和 asyncio.wait 的区别，要具体。
```

```
>>> 写一个多阶段构建的 Dockerfile，用于编译 C++ 项目，最终镜像用 distroless。
```

```
>>> 什么是 RAG（检索增强生成），它如何减少大模型的幻觉？
```

要退出交互会话，输入 `/bye`。

---

## 第五阶段：暴露本地 REST API

Ollama 会自动在端口 `11434` 上运行一个本地 HTTP 服务器。这对于在它之上构建应用非常有用。

### 健康检查

```powershell
curl http://localhost:11434/
```

返回：`Ollama is running`

### 生成补全（非流式）

```powershell
curl http://localhost:11434/api/generate -d '{
  "model": "gemma3:12b",
  "prompt": "法国的首都是哪里？",
  "stream": false
}'
```

### 列出可用模型

```powershell
curl http://localhost:11434/api/tags
```

---

## 第六阶段：与 Python 集成（LangChain / 原生 HTTP）

### 方法 1：原生 HTTP（零依赖）

这是我在轻量工具中的首选方法。不需要 pip install 任何东西。

```python
import json
import urllib.request

def ask_gemma(prompt: str, model: str = "gemma3:12b") -> str:
    """通过 Ollama 的 REST API 向本地 Gemma 发送提示词"""
    payload = {
        "model": model,
        "prompt": prompt,
        "stream": False,
        "options": {"temperature": 0.3, "top_p": 0.9}
    }
    
    data = json.dumps(payload).encode("utf-8")
    req = urllib.request.Request(
        "http://localhost:11434/api/generate",
        data=data,
        headers={"Content-Type": "application/json"}
    )
    
    with urllib.request.urlopen(req) as resp:
        result = json.loads(resp.read().decode("utf-8"))
        return result["response"]

# 使用示例
answer = ask_gemma("用三句话解释 LoRA 微调的原理。")
print(answer)
```

### 方法 2：LangChain 集成

如果你已经在 LangChain 生态里了，这样更简洁：

```python
from langchain_community.llms import Ollama

llm = Ollama(model="gemma3:12b")
response = llm.invoke("写一个集成了 SonarQube 质量检查的 Jenkins 流水线。")
print(response)
```

---

## 第七阶段：性能调优与显存管理

### 实时监控 GPU 使用率

在另一个终端保持运行：

```powershell
nvidia-smi --query-gpu=name,memory.used,memory.total,utilization.gpu --format=csv -l 2
```

### 调整上下文窗口大小

默认 2048 token 的上下文窗口。对于编码任务，我会调大：

```powershell
@"
FROM gemma3:12b
PARAMETER num_ctx 8192
PARAMETER temperature 0.2
"@ | Out-File -FilePath Modelfile -Encoding utf8

ollama create gemma-code -f Modelfile
ollama run gemma-code
```

> **警告**：更大的上下文窗口会吃更多显存。如果你是 8GB 显存，最多 4096。我曾把上下文调到 16384，结果 GPU 显存在生成到一半时直接爆了，整个 Ollama 进程静默崩溃，什么错误信息都没有。

---

## 第八阶段：将 Gemma 4 接入龙虾引擎（AI 驱动的数据采集）

这里才是真正有意思的部分。如果你看过我的 [Project-Lobster](https://github.com/kurodayu23/Project-Lobster) 仓库，你就知道它是一个高并发的异步数据采集引擎。但单独的 Lobster 只是个暴力拉取器——它能又快又稳地抓到原始数据。然而，原始数据在你理解它之前就只是噪音。

核心理念：**龙虾负责大规模采集，Gemma 4 负责本地思考。** 全程零数据外泄。

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                     你的本地机器                          │
│                                                          │
│  ┌──────────────┐    原始 JSON   ┌───────────────────┐  │
│  │   龙虾引擎    │ ────────────► │  Gemma 4 (Ollama) │  │
│  │  (aiohttp)    │               │  localhost:11434   │  │
│  │  50 个 worker │ ◄──────────── │  结构化输出        │  │
│  └──────────────┘    分析洞察    └───────────────────┘  │
│         │                                │               │
│         ▼                                ▼               │
│  ┌──────────────┐               ┌───────────────────┐   │
│  │  原始数据     │               │  分析结果          │   │
│  │  (JSON 转储)  │               │  (Markdown/CSV)    │  │
│  └──────────────┘               └───────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 步骤 1：为龙虾扩展 AI 分析层

在原始引擎旁边创建一个新文件 `lobster_with_gemma.py`：

```python
import asyncio
import aiohttp
import json
import urllib.request
import logging
from typing import List, Dict

logging.basicConfig(level=logging.INFO, format="[Lobster+Gemma] %(asctime)s - %(message)s")

class GemmaAnalyzer:
    """封装本地 Gemma 4（通过 Ollama），用于采集后的智能分析。"""
    
    def __init__(self, model: str = "gemma3:12b"):
        self.model = model
        self.endpoint = "http://localhost:11434/api/generate"
    
    def analyze(self, raw_data: dict) -> str:
        payload = {
            "model": self.model,
            "prompt": f"""你是一个数据分析师。分析以下原始 API 响应，
提取：1) 提到的关键实体，2) 情感倾向（正面/负面/中性），
3) 一句话总结。

原始数据：
{json.dumps(raw_data, indent=2)[:2000]}

以合法 JSON 格式返回，键名为：entities, sentiment, summary。""",
            "stream": False,
            "options": {"temperature": 0.1}
        }
        
        data = json.dumps(payload).encode("utf-8")
        req = urllib.request.Request(self.endpoint, data=data, 
                                     headers={"Content-Type": "application/json"})
        try:
            with urllib.request.urlopen(req, timeout=30) as resp:
                result = json.loads(resp.read().decode("utf-8"))
                return result.get("response", "")
        except Exception as e:
            logging.error(f"Gemma 分析失败: {e}")
            return '{"error": "analysis_failed"}'

class SmartLobsterEngine:
    """龙虾 v2：现在有脑子了。先大规模采集，再逐条通过本地 Gemma 4 分析。"""
    
    def __init__(self, concurrency: int = 10):
        self.semaphore = asyncio.Semaphore(concurrency)
        self.analyzer = GemmaAnalyzer()
    
    async def fetch_single(self, session, target_id: int) -> Dict:
        url = f"https://jsonplaceholder.typicode.com/posts/{target_id}"
        async with self.semaphore:
            try:
                async with session.get(url, timeout=5) as resp:
                    resp.raise_for_status()
                    data = await resp.json()
                    return {"id": target_id, "status": "harvested", "payload": data}
            except Exception as e:
                return {"id": target_id, "status": "failed", "error": str(e)}
    
    async def harvest_batch(self, count: int) -> List[Dict]:
        async with aiohttp.ClientSession() as session:
            tasks = [self.fetch_single(session, i) for i in range(1, count + 1)]
            return await asyncio.gather(*tasks)
    
    def enrich_with_gemma(self, harvested: List[Dict]) -> List[Dict]:
        """第二阶段：顺序 AI 分析（瓶颈在 GPU，不在网络）"""
        enriched = []
        successful = [r for r in harvested if r["status"] == "harvested"]
        for i, result in enumerate(successful):
            logging.info(f"正在分析 [{i+1}/{len(successful)}]...")
            result["gemma_analysis"] = self.analyzer.analyze(result["payload"])
            enriched.append(result)
        return enriched

if __name__ == "__main__":
    engine = SmartLobsterEngine(concurrency=20)
    raw_results = asyncio.run(engine.harvest_batch(10))
    smart_results = engine.enrich_with_gemma(raw_results)
    
    with open("lobster_gemma_output.json", "w", encoding="utf-8") as f:
        json.dump(smart_results, f, indent=2, ensure_ascii=False)
    logging.info(f"流水线完成。{len(smart_results)} 条记录已保存。")
```

### 步骤 2：运行集成流水线

先确保 Ollama 在跑着 Gemma 4：

```powershell
# 终端 1：确保 Ollama 运行中
ollama serve

# 终端 2：运行聪明的龙虾
python lobster_with_gemma.py
```

### 这里到底发生了什么

1. **龙虾的异步蜂群** 通过 `aiohttp` + `asyncio.Semaphore` 并发发射 10-50 个 HTTP 请求。50 个目标大概只需 1-2 秒。
2. **每个采集到的数据包** 随后被喂给你本地的 Gemma 4 实例进行结构化提取（实体、情感、摘要）。
3. 结果保存为一个富化后的 JSON 文件。全程零数据外泄。

### 为什么这对你的作品集很重要

大多数人要么只会写爬虫，要么只会玩大模型。几乎没有人展示它们**在一个真实管线中的协同工作**。这种集成证明你理解：
- 网络密集型任务需要异步 I/O
- GPU 密集型推理有同步约束
- 如何让架构中的每个组件各司其职

> **性能小贴士**：瓶颈在这里发生了翻转。采集阶段是网络瓶颈（异步并发大显身手）；Gemma 分析阶段是 GPU 瓶颈（在单 GPU 上并行化 LLM 调用只会 OOM）。所以第二阶段是故意做成顺序的。**真正的工程素养，是知道什么时候不该并行化。**

---

## 疑难杂症排查（那些没人告诉你的坑）

### "Error: model requires more system memory than is available"

**原因**：系统内存（不是显存）不够用了。

**解法**：关掉 Chrome。说真的，Chrome 在大多数人的电脑上吃 4-8GB 内存。或者：

```powershell
Get-Process | Sort-Object -Property WS -Descending | Select-Object -First 10 Name, @{Name="Memory(MB)";Expression={[math]::Round($_.WS/1MB, 2)}}
```

### "CUDA out of memory"

**解法**：
1. 把 `num_ctx` 降到 2048
2. 换用 4B 版本
3. 用 `num_gpu 15` 把部分层卸载到 CPU
4. 杀掉其他占用 GPU 的进程

### Windows Defender 拖慢了模型加载速度

这个我是真遇到了。Windows Defender 每次 Ollama 加载模型时都会扫描那个 7.5GB 的文件。

**解法**：添加排除项：

```powershell
# 以管理员身份运行
Add-MpExclusion -Path "D:\LLM_Models"
```

---

## 接下来做什么？

一旦你在本地跑起了 Gemma 4，可玩的东西就太多了：

1. **在你的私有代码库上构建本地 RAG 系统**（参见我的 [RAG-LangChain-Tuner](https://github.com/kurodayu23/RAG-LangChain-Tuner) 仓库）
2. **与你的 IDE 集成** — Cursor、Continue.dev、Claude Code 都可以指向 `localhost:11434`
3. **用 LoRA 微调**，针对你的特定领域（医疗、法律、公司内部文档）
4. **与其他工具串联**（参见我的 [VibeOps-Agent](https://github.com/kurodayu23/VibeOps-Agent)）

核心就一句话：**模型是你的、数据是你的、推理也是你的**。没有 API Key，没有速率限制，没有月账单，没有数据外泄。

这就是 Vibe Coding 的未来。而它始于 `ollama pull`。

---

*基于真实部署经验编写。这篇指南的每一个章节，都来自我实际踩过的坑和找到的解法。*
