# RAG Full Pipeline 教程

这份教程配合 [RAG_Tutorial_Full_Pipeline.ipynb](RAG_Tutorial_Full_Pipeline.ipynb)，从 PDF 文档开始，逐步搭建一个可查询、可检查来源的 RAG（Retrieval-Augmented Generation，检索增强生成）系统。

## 学习目标

- 理解文档加载、文本切分、Embedding、向量索引、检索和生成各阶段的职责。
- 使用 LlamaIndex 组织 RAG 流程，以 Chroma 持久化向量数据。
- 通过阿里云 Model Studio 的 OpenAI 兼容 API 调用 Qwen 对话模型和文本向量模型。
- 检查检索到的文本与来源，并用参考答案评估生成结果。

## 工作流程

```text
PDF → Documents → Chunks → Embeddings → Chroma
                                      ↓
问题 → 检索 → 上下文 → Prompt → Qwen → 答案与来源
```

RAG 的质量取决于整条流程：模型只根据检索到的上下文回答；如果检索内容不相关或缺失，生成模型通常无法给出可靠答案。

## 环境准备

需要 Python 3.9 或更高版本、Jupyter，以及一个可调用 Qwen 对话和 Embedding 模型的 Model Studio API Key。Notebook 会检查 Python 版本和所需模块。

在 PowerShell 中进入本教程目录，并创建虚拟环境：

```powershell
cd "02_RAG tutorial"
py -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

安装后，在 VS Code 中打开 Notebook，并选择 `.venv` 对应的 Jupyter Kernel。若已使用配置好的 Python 环境，可跳过虚拟环境创建，但要确保 Notebook 使用的 Kernel 已安装依赖。

> `requirements.txt` 是本教程目录的共享依赖清单，包含了其他 RAG 示例可能用到的包；本 Notebook 只使用其中一部分。

## API 配置

在 `02_RAG tutorial` 目录下创建 `.env` 文件：

```dotenv
DASHSCOPE_API_KEY=your_api_key
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL=qwen3.7-flash
EMBEDDING_MODEL=qwen3.7-text-embedding
```

Notebook 会从环境变量或 `.env` 读取配置；未设置模型名时使用上面的默认模型名，未设置 Base URL 时使用示例中的默认端点。请确认 API Key 所属地域与 Base URL 匹配。不要把真实 API Key 写入 Notebook，也不要将 `.env` 提交到版本库。

开始处理文档前，Notebook 会分别请求一次对话模型和 Embedding 模型，以验证 API Key、端点和模型名。API 请求可能产生费用；文档片段和问题会发送到配置的模型服务，请勿使用未经授权或不适合外发的资料。

## 准备 PDF

将要建立知识库的 PDF 放在：

```text
02_RAG tutorial/
└── data/
    ├── raw/        # 输入 PDF
    ├── processed/  # 预留目录；当前 Notebook 不读取
    ├── eval/       # 预留目录；当前 Notebook 不读取
    └── vector_db/  # Chroma 持久化数据
```

把 PDF 放入 `data/raw/`。目录为空时 Notebook 会停止并提示缺少 PDF。路径相对于 Notebook 的当前工作目录解析，因此建议从本教程目录打开并运行 Notebook。

## 运行教程

在 VS Code 或 Jupyter 中打开 `RAG_Tutorial_Full_Pipeline.ipynb`，按顺序运行单元格。主要阶段如下：

1. **环境与 API 检查**：确认 Python、依赖、模型配置和 API 连通性。
2. **加载与检查文档**：递归读取 `data/raw/` 中的 PDF，查看解析出的文本、文件名、页码和空文本统计。一个 `Document` 通常对应 PDF 的一页或解析单元，并非最终的检索块。
3. **切分文本**：使用 `SentenceSplitter` 生成节点，默认 `chunk_size=512`、`chunk_overlap=100`。重叠可保留块边界附近的上下文。
4. **向量化与建索引**：通过配置的 Qwen Embedding 模型生成向量，并将节点写入 Chroma 的 `rag_course_demo` collection；数据库位于 `data/vector_db/`。
5. **检查检索**：默认 `top_k=4`，显示命中片段、相似度分数、文件和页码，并高亮问题中的词，帮助判断检索是否真正相关。
6. **生成回答**：把检索片段及来源组成上下文，要求模型仅依据上下文作答；若信息不足，应明确说明知识库无法提供足够信息。结果中包含答案和来源元数据。
7. **测试与复核**：运行不同类型的问题，将回答与 Notebook 中的参考答案对照，并计算 Embedding 余弦相似度作为人工复核信号。
8. **便捷接口对照**：最后用 LlamaIndex 的 `index.as_query_engine()` 执行同类查询，对比手动展开各阶段与框架封装后的调用方式。

## 运行结果与调参

重点观察检索结果，而不只是最终答案：

- 返回的片段是否包含回答问题所需的信息？
- 相似度较高的片段是否真的更有用？
- 来源文件和页码是否符合预期？
- 问题超出知识库范围时，模型是否承认信息不足？

可以尝试调整 `CHUNK_SIZE`、`CHUNK_OVERLAP` 和 `TOP_K`，并更换成与你的 PDF 内容相符的测试问题。每次调整后都应检查检索片段，再判断回答变化来自检索还是生成。

语义相似度不是正确性分数：措辞不同但事实正确的答案可能得分较低；语义相似的回答也可能遗漏关键条件或包含错误事实。请逐条对照原文或参考答案进行判断。

## 重要注意事项

- **重建索引会覆盖 collection：** Notebook 在创建索引时会删除并重建 `rag_course_demo`。该 collection 中已有的数据不会保留；如需保留其他实验结果，请先备份或更改 `COLLECTION_NAME`。
- **索引与 PDF 不会自动同步：** 更换或修改 PDF 后，重新运行索引构建阶段以更新向量库。
- **Notebook 不是自动化评测器：** 测试问题和参考答案在 Notebook 中配置；相似度只用于辅助检查，不代表质量保证。
- **预留目录当前未参与流程：** `data/processed/` 和 `data/eval/` 会被创建，但当前 Notebook 不从中读取数据。

## 常见问题

| 现象 | 检查方式 |
|---|---|
| 找不到 PDF | 确认 PDF 位于 `02_RAG tutorial/data/raw/`，并检查 Notebook 的当前工作目录。 |
| 找不到 API Key | 检查 `.env` 是否在教程目录、变量名是否为 `DASHSCOPE_API_KEY`，然后重启 Kernel 并重新运行配置单元。 |
| API 请求失败 | 检查网络、API Key 有效性、地域与 Base URL 是否匹配，以及模型名是否可用。 |
| 缺少 Python 模块 | 确认依赖安装到了 Notebook 当前选用的 Kernel 环境，再运行 `pip install -r requirements.txt`。 |
| 回答不相关或不完整 | 先检查检索片段；再考虑切分参数、`TOP_K`、PDF 文本解析质量和问题表达。 |

## 延伸阅读

- [RAG Test Questions and Answer.txt](RAG%20Test%20Questions%20and%20Answer.txt)：教程测试问题对应的参考答案。
- [LlamaIndex 文档](https://docs.llamaindex.ai/)
- [Chroma 文档](https://docs.trychroma.com/)
- [阿里云 Model Studio 文档](https://help.aliyun.com/zh/model-studio/)