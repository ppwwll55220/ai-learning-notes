# 学习日志：AI API 调用本质 & GPU 基准测试

> 日期：2026-08-15  
> 课程：AI Engineering from Scratch — 阶段 0 / 第 1 课  
> 环境：WSL2 Ubuntu 24.04 + Python 3.12 + PyTorch 2.14.0 (cu130)

---

## 🎯 今日核心收获

**1. AI API 调用的本质**  
你的本地程序通过 HTTP 协议，向云端 AI 服务器发送一个结构化的请求（包含问题、模型选择、参数），服务器在云端用 GPU 计算后，把结果封装成 JSON 返回给你。整个过程可以拆解为 5 个固定步骤：

1. **鉴权**：带上 API 密钥（从 `.env` 读取，绝不写进代码）
2. **端点**：指定 URL（如 `https://api.deepseek.com/chat/completions`）
3. **请求体**：包含 `model`、`messages`（你的问题）、`max_tokens`
4. **发送请求**：使用 HTTP POST 方法
5. **解析响应**：从 `choices[0].message.content` 中提取答案

**2. GPU 加速对比**  
通过 5000×5000 矩阵乘法测试，验证了 GPU 相比 CPU 的加速效果。  
预热后加速比可达 **13 倍以上**，8GB 显存约可容纳 **40 亿参数** 的 fp16 模型。

---

## 💻 代码实战：DeepSeek API 调用

### 前置准备
- 在项目根目录创建 `.env` 文件，写入：
  ```env
  DEEPSEEK_API_KEY=sk-你的真实密钥
确保 .env 被加入 .gitignore

激活虚拟环境：source ~/.venv/bin/activate

方式一：Python SDK（推荐日常使用）
python
# call_deepseek_sdk.py
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com"
)

MODEL = os.environ.get("LLM_MODEL", "deepseek-chat")

response = client.chat.completions.create(
    model=MODEL,
    max_tokens=256,
    messages=[{"role": "user", "content": "用一句话解释什么是神经网络"}]
)

print(response.choices[0].message.content)
输出结果：

text
神经网络是一种通过模拟人脑神经元连接方式、由多层“神经元”自动从数据中学习特征和规律的计算模型。
方式二：原始 HTTP（理解底层）
python
# call_deepseek_http.py
import os
import urllib.request
import json
from dotenv import load_dotenv

load_dotenv()

url = "https://api.deepseek.com/chat/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {os.getenv('DEEPSEEK_API_KEY')}"
}
body = json.dumps({
    "model": "deepseek-chat",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "用一句话解释什么是神经网络"}]
}).encode()

req = urllib.request.Request(url, data=body, headers=headers, method="POST")
with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())
    print(result["choices"][0]["message"]["content"])
输出结果：与 SDK 版本完全一致。

📊 性能测试：CPU vs GPU 矩阵乘法
测试代码（带预热）
python
# benchmark.py
import torch
import time

size = 5000

a_cpu = torch.randn(size, size)
b_cpu = torch.randn(size, size)

start = time.time()
c_cpu = a_cpu @ b_cpu
cpu_time = time.time() - start
print(f"CPU: {cpu_time:.3f}s")

if torch.cuda.is_available():
    a_gpu = a_cpu.to("cuda")
    b_gpu = b_cpu.to("cuda")

    # 预热（消除首次编译开销）
    c_gpu = a_gpu @ b_gpu
    torch.cuda.synchronize()

    # 正式计时
    torch.cuda.synchronize()
    start = time.time()
    c_gpu = a_gpu @ b_gpu
    torch.cuda.synchronize()
    gpu_time = time.time() - start
    print(f"GPU (预热后): {gpu_time:.3f}s")
    print(f"真实加速比: {cpu_time / gpu_time:.0f}x")
运行命令
bash
python benchmark.py
测试结果
设备	耗时	加速比
CPU	0.407s	—
GPU（预热后）	0.032s	13x
显存信息
bash
nvidia-smi
项目	值
GPU 型号	NVIDIA GeForce RTX 5060 Laptop GPU
显存总量	8151 MiB（约 8 GB）
驱动版本	591.91
CUDA 版本	13.1
PyTorch CUDA 版本	13.0
估算：8GB 显存在 fp16 精度下可容纳约 40 亿参数 的模型。

🐛 遇到的坑与解决方案
坑 1：复制代码时混入中文全角括号
报错信息：SyntaxError: invalid character '）' (U+FF09)

原因：复制网页中的代码时，括号 ( 被自动替换为中文全角括号 （

解决：所有代码标点（括号、引号、逗号）必须使用英文半角，切换至英文输入法重新输入。

坑 2：虚拟环境路径混淆
问题：执行 source .venv/bin/activate 提示文件不存在

原因：虚拟环境实际位于 ~/.venv 而非项目目录内

解决：使用 source ~/.venv/bin/activate 激活

坑 3：PyTorch 不兼容 RTX 5060（sm_120）
问题：安装 cu124 版本后，import torch 出现警告，提示不兼容

解决：升级到 cu130 Nightly 版本（pip install --upgrade --pre torch torchvision --index-url https://download.pytorch.org/whl/nightly/cu130），警告消失，GPU 正常识别。

📌 下一步计划
□ 学习 Jupyter Notebook 交互式编程
□ 尝试修改提示词（例如：“用 50 字解释什么是反向传播”）
□ 构建多轮对话（记忆上下文）
□ 将笔记本迁移到 JupyterLab 中运行
📂 相关文件位置
文件	路径
课程仓库	~/ai-engineering-from-scratch
虚拟环境	~/.venv
API 调用脚本	call_deepseek_sdk.py、call_deepseek_http.py
GPU 测试脚本	benchmark.py
笔记仓库	~/ai-learning-notes
