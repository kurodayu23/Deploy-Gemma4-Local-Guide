[English](README.md) | [简体中文](README_zh.md) | [日本語](README_ja.md)

---

# 🚀 Deploying Google Gemma 4 Locally: A Practitioner's Complete Guide

> **Author's Note**: This isn't a copy-paste tutorial. I wrote this after spending a full weekend wrestling with VRAM limits, model quantization trade-offs, and Ollama's quirks on Windows. If you're trying to run Gemma 4 on your own hardware without paying a cent to any cloud provider, this guide will save you hours of pain.

---

## Table of Contents

- [Why Gemma 4?](#why-gemma-4)
- [Prerequisites & Hardware Reality Check](#prerequisites--hardware-reality-check)
- [Phase 1: Setting Up Ollama on Windows](#phase-1-setting-up-ollama-on-windows)
- [Phase 2: Pulling and Running Gemma 4](#phase-2-pulling-and-running-gemma-4)
- [Phase 3: Changing the Default Model Storage Path](#phase-3-changing-the-default-model-storage-path)
- [Phase 4: Testing the Model Interactively](#phase-4-testing-the-model-interactively)
- [Phase 5: Exposing a Local REST API](#phase-5-exposing-a-local-rest-api)
- [Phase 6: Integrating with Python (LangChain / Raw HTTP)](#phase-6-integrating-with-python-langchain--raw-http)
- [Phase 7: Performance Tuning & VRAM Management](#phase-7-performance-tuning--vram-management)
- [Troubleshooting (The Stuff Nobody Tells You)](#troubleshooting-the-stuff-nobody-tells-you)
- [What's Next?](#whats-next)

---

## Why Gemma 4?

Let me be blunt: I've tried a lot of local models. Llama 3, Mistral, Phi-3, Qwen2... they all have their sweet spots. But Google's Gemma 4 hit differently for me. Here's why:

- **Insane code comprehension**. I threw a 500-line C++ Qt file at it and it correctly identified a namespace pollution issue that was causing LNK4042 linker errors. Most 7B models would've hallucinated.
- **Multilingual out of the box**. I work across English, Chinese, and Japanese documentation. Gemma 4 handles all three without quality degradation.
- **Truly open weights**. Google released these under a permissive license. You can fine-tune, deploy commercially, do whatever you want.
- **Efficient inference**. The 4B variant runs comfortably on 8GB VRAM. The 12B variant needs about 10-12GB. No A100 required.

---

## Prerequisites & Hardware Reality Check

Before we start, let's be honest about what you need:

| Component | Minimum (4B model) | Recommended (12B model) | My Setup |
|-----------|-------------------|------------------------|----------|
| GPU | GTX 1660 (6GB) | RTX 3060 (12GB) | RTX 3060 12GB |
| RAM | 16GB | 32GB | 32GB DDR4 |
| Disk Space | 5GB free | 15GB free | External D: drive |
| OS | Windows 10/11 | Windows 10/11 | Windows 11 |

**Important**: If you don't have a dedicated NVIDIA GPU, you *can* still run Gemma 4 on CPU-only mode through Ollama. It'll be slow (maybe 2-5 tokens/second for the 4B), but it works. I've done it on a laptop during a train ride. Not ideal, but functional.

### Software Dependencies

You'll need these installed before proceeding:

```powershell
# Check if you have NVIDIA drivers installed
nvidia-smi
```

If this command returns your GPU info, you're good. If it errors out, go install the latest NVIDIA drivers from [nvidia.com/drivers](https://www.nvidia.com/drivers).

---

## Phase 1: Setting Up Ollama on Windows

Ollama is the runtime that makes local model deployment painless. Think of it as "Docker, but for LLMs."

### Option A: Install via winget (Recommended)

```powershell
winget install --id Ollama.Ollama -e --source winget
```

### Option B: Manual Download

Head to [ollama.com/download](https://ollama.com/download) and grab the Windows installer. Run it, click through the wizard, done.

### Verify Installation

```powershell
ollama --version
```

You should see something like:

```
ollama version is 0.6.2
```

> **Gotcha I hit**: After installing via winget, I had to **restart my terminal** for the `ollama` command to be recognized. The installer adds it to PATH but your current session won't pick it up. Don't panic if it says "command not found" — just open a fresh PowerShell window.

---

## Phase 2: Pulling and Running Gemma 4

This is where the magic happens. Ollama manages model downloads, quantization, and serving in one command.

### Choosing Your Variant

```
gemma3:4b    → ~2.5GB download, needs ~4GB VRAM   (Good for quick tasks)
gemma3:12b   → ~7.5GB download, needs ~10GB VRAM  (Sweet spot for coding)
gemma3:27b   → ~16GB download, needs ~20GB VRAM   (Beast mode, needs serious GPU)
```

> **Why am I showing gemma3 tags?** At the time of writing, Ollama's registry uses `gemma3` as the tag family. Gemma 4 models are pulled using the latest tags under this family. Always check `ollama list` after pulling to confirm the actual model version.

### Pull the Model

```powershell
# For the 12B variant (my recommendation if you have 12GB+ VRAM)
ollama pull gemma3:12b

# For the lightweight 4B variant
ollama pull gemma3:4b
```

This will take a while depending on your internet speed. The 12B model is roughly 7.5GB. Go make coffee. ☕

### Verify the Download

```powershell
ollama list
```

Expected output:

```
NAME            ID              SIZE      MODIFIED
gemma3:12b      a]b2c3d4e5f6    7.5 GB    2 minutes ago
```

---

## Phase 3: Changing the Default Model Storage Path

**This is critical if your C: drive is small.** By default, Ollama stores models in `C:\Users\<you>\.ollama\models`. On my machine, my C: drive is a 256GB SSD that's always near full. I need my models on the D: drive.

### Step 1: Create your target directory

```powershell
New-Item -ItemType Directory -Path "D:\LLM_Models" -Force
```

### Step 2: Set the environment variable (System-level, persistent)

```powershell
# Run PowerShell as Administrator
[System.Environment]::SetEnvironmentVariable("OLLAMA_MODELS", "D:\LLM_Models", "Machine")
```

### Step 3: Restart the Ollama service

```powershell
# Stop the running Ollama process
taskkill /f /im ollama.exe

# Restart it (it will pick up the new path)
ollama serve
```

### Step 4: Verify it's using the new path

```powershell
# Pull a small model to test
ollama pull gemma3:4b

# Check if files appeared in D:\LLM_Models
Get-ChildItem "D:\LLM_Models" -Recurse | Select-Object -First 10
```

If you see `.bin` files appearing under `D:\LLM_Models`, congratulations — your precious C: drive is safe.

> **The mistake I made here**: I initially set the variable as a User variable instead of a Machine variable. Ollama's background service runs under a different context, so it ignored my user-level env var and kept downloading to C:. Use `"Machine"` scope. Trust me.

---

## Phase 4: Testing the Model Interactively

Let's actually talk to the model.

```powershell
ollama run gemma3:12b
```

This drops you into an interactive chat. Try these prompts to verify it's working properly:

```
>>> Explain the difference between asyncio.gather and asyncio.wait in Python. Be specific.
```

```
>>> Write a Dockerfile for a multi-stage C++ build that produces a distroless final image.
```

```
>>> 用中文解释什么是 RAG（检索增强生成），以及它如何减少大模型的幻觉问题。
```

```
>>> Pythonの非同期処理において、async/awaitとスレッドプールの使い分けを説明してください。
```

To exit the interactive session, type `/bye`.

---

## Phase 5: Exposing a Local REST API

Ollama automatically runs a local HTTP server on port `11434`. This is incredibly useful for building applications on top of it.

### Health Check

```powershell
curl http://localhost:11434/
```

Response: `Ollama is running`

### Generate a Completion (Non-streaming)

```powershell
curl http://localhost:11434/api/generate -d '{
  "model": "gemma3:12b",
  "prompt": "What is the capital of France?",
  "stream": false
}'
```

### Generate a Chat Completion

```powershell
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3:12b",
  "messages": [
    {"role": "system", "content": "You are an expert DevOps engineer."},
    {"role": "user", "content": "Write a health check endpoint for a FastAPI app."}
  ],
  "stream": false
}'
```

### List Available Models via API

```powershell
curl http://localhost:11434/api/tags
```

---

## Phase 6: Integrating with Python (LangChain / Raw HTTP)

### Approach 1: Raw HTTP (Zero Dependencies)

This is my preferred method for lightweight tools. No pip install needed.

```python
import json
import urllib.request

def ask_gemma(prompt: str, model: str = "gemma3:12b") -> str:
    """Send a prompt to local Gemma via Ollama's REST API."""
    payload = {
        "model": model,
        "prompt": prompt,
        "stream": False,
        "options": {
            "temperature": 0.3,
            "top_p": 0.9
        }
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

# Usage
answer = ask_gemma("Explain LoRA fine-tuning in 3 sentences.")
print(answer)
```

### Approach 2: LangChain Integration

If you're already in the LangChain ecosystem, this is cleaner:

```python
from langchain_community.llms import Ollama

llm = Ollama(model="gemma3:12b")
response = llm.invoke("Write a Jenkins pipeline that runs SonarQube analysis.")
print(response)
```

### Approach 3: LangChain + ChromaDB RAG Pipeline

This is where it gets powerful. Ground Gemma's responses in your own documents:

```python
from langchain_community.llms import Ollama
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate

# 1. Setup
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
vector_db = Chroma(persist_directory="./my_chroma_db", embedding_function=embeddings)
llm = Ollama(model="gemma3:12b")

# 2. Build Chain
prompt = ChatPromptTemplate.from_template("""
Answer the question based only on the provided context:
<context>{context}</context>
Question: {input}
""")
doc_chain = create_stuff_documents_chain(llm, prompt)
retriever = vector_db.as_retriever(search_kwargs={"k": 3})
rag_chain = create_retrieval_chain(retriever, doc_chain)

# 3. Query
result = rag_chain.invoke({"input": "How do I fix LNK4042 in MSVC?"})
print(result["answer"])
```

---

## Phase 7: Performance Tuning & VRAM Management

### Monitoring GPU Usage in Real-Time

Keep this running in a separate terminal while you interact with the model:

```powershell
# Refresh every 2 seconds
nvidia-smi -l 2
```

Or for a cleaner view:

```powershell
nvidia-smi --query-gpu=name,memory.used,memory.total,utilization.gpu --format=csv -l 2
```

### Tweaking Context Window Size

By default, Ollama uses a 2048-token context window. For coding tasks, I bump this up:

```powershell
# Create a custom Modelfile
@"
FROM gemma3:12b
PARAMETER num_ctx 8192
PARAMETER temperature 0.2
"@ | Out-File -FilePath Modelfile -Encoding utf8

ollama create gemma-code -f Modelfile
ollama run gemma-code
```

> **Warning**: Larger context windows eat more VRAM. If you're on 8GB, stick with 4096 max. I learned this the hard way when my GPU ran out of memory mid-generation and the entire Ollama process crashed silently.

### Offloading Layers to CPU

If you're running a model that's *slightly* too big for your GPU:

```powershell
# In your Modelfile, add:
PARAMETER num_gpu 20
# This tells Ollama to put only 20 layers on GPU, rest on CPU
# Adjust this number based on your VRAM headroom
```

---

## Troubleshooting (The Stuff Nobody Tells You)

### "Error: model requires more system memory than is available"

**Cause**: Your system RAM (not VRAM) is insufficient for the model's CPU-side buffers.

**Fix**: Close Chrome. Seriously. Chrome eats 4-8GB of RAM on most people's machines. Or:

```powershell
# Check what's eating your RAM
Get-Process | Sort-Object -Property WS -Descending | Select-Object -First 10 Name, @{Name="Memory(MB)";Expression={[math]::Round($_.WS/1MB, 2)}}
```

### "CUDA out of memory"

**Cause**: VRAM exhausted. Usually happens with large context windows or the 27B model.

**Fix**: 
1. Reduce `num_ctx` to 2048
2. Use the 4B variant instead
3. Try `num_gpu 15` to offload some layers to CPU
4. Kill any other GPU-using processes (gaming, video encoding, etc.)

### Model generates garbage or stops mid-sentence

**Cause**: Usually a corrupted download.

**Fix**:
```powershell
ollama rm gemma3:12b
ollama pull gemma3:12b
```

### Ollama won't start / port 11434 is already in use

```powershell
# Find and kill whatever's on that port
netstat -ano | findstr 11434
taskkill /f /pid <PID_FROM_ABOVE>

# Restart
ollama serve
```

### Windows Defender is slowing down model loading

This genuinely happened to me. Windows Defender was scanning the 7.5GB model file every time Ollama loaded it.

**Fix**: Add an exclusion:
```powershell
# Run as Administrator
Add-MpExclusion -Path "D:\LLM_Models"
```

---

## What's Next?

Once you have Gemma 4 running locally, the possibilities are ridiculous:

1. **Build a local RAG system** over your private codebase (see my [RAG-LangChain-Tuner](https://github.com/kurodayu23/RAG-LangChain-Tuner) repo)
2. **Integrate with your IDE** — tools like Cursor, Continue.dev, and Claude Code can all point to `localhost:11434`
3. **Fine-tune with LoRA** for your specific domain (medical, legal, your company's internal docs)
4. **Chain it with other tools** via my [VibeOps-Agent](https://github.com/kurodayu23/VibeOps-Agent) for automated code annotation

The whole point is: **you own the model, you own the data, you own the inference**. No API keys, no rate limits, no monthly bills, no data leaving your machine.

That's the future of Vibe Coding. And it starts with `ollama pull`.

---

*Built with real deployment experience. No ChatGPT was asked "write me a tutorial." Every section of this guide comes from actual errors I hit and solutions I found.*
