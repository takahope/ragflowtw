# RAGFlow GraphRAG 功能規格文件

**版本**: 1.0
**更新日期**: 2025-01-14
**專案**: RAGFlow - Knowledge Graph Enhanced RAG System

---

## 目錄

- [1. 系統概述](#1-系統概述)
- [2. 系統架構](#2-系統架構)
- [3. 核心模組](#3-核心模組)
- [4. API 規格](#4-api-規格)
- [5. 配置規格](#5-配置規格)
- [6. 數據模型](#6-數據模型)
- [7. 工作流程](#7-工作流程)
- [8. 集成指南](#8-集成指南)
- [9. 性能考慮](#9-性能考慮)
- [10. 限制與最佳實踐](#10-限制與最佳實踐)

---

## 1. 系統概述

### 1.1 簡介

RAGFlow GraphRAG 是一個基於知識圖譜的檢索增強生成 (Retrieval-Augmented Generation) 系統，旨在通過構建和查詢知識圖譜來支持多跳問答和複雜的語義檢索任務。

### 1.2 核心特性

- **自動化知識圖譜構建**：從非結構化文本中自動提取實體和關係
- **雙模式提取**：
  - **General 模式**：高精度提取，適用於複雜場景
  - **Light 模式**：高效率提取，基於 LightRAG 優化
- **實體解析**：自動去重和合併相似實體
- **社區檢測**：使用 Leiden 算法進行圖社區分析
- **社區報告生成**：為每個社區自動生成摘要報告
- **多跳查詢**：支持 N-hop 關係檢索
- **混合排序**：結合 PageRank 和語義相似度排序
- **分佈式鎖**：支持並發圖譜構建

### 1.3 技術基礎

- **參考實現**：
  - [Microsoft GraphRAG](https://github.com/microsoft/graphrag)
  - [LightRAG](https://github.com/HKUDS/LightRAG)
  - [MiniRAG](https://github.com/HKUDS/MiniRAG)
- **圖數據結構**：NetworkX
- **向量存儲**：Elasticsearch / Infinity
- **緩存層**：Redis
- **並發框架**：Trio (async/await)

---

## 2. 系統架構

### 2.1 整體架構

```
┌─────────────────────────────────────────────────────────────┐
│                     RAGFlow Frontend                         │
│              (Vue.js + TypeScript)                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ HTTP/REST API
                        │
┌───────────────────────▼─────────────────────────────────────┐
│                   API Layer (Flask)                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │ Document   │  │ Knowledge  │  │ Chat       │            │
│  │ Service    │  │ Base       │  │ Service    │            │
│  │            │  │ Service    │  │            │            │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘            │
└────────┼───────────────┼───────────────┼────────────────────┘
         │               │               │
         │               │               │
┌────────▼───────────────▼───────────────▼────────────────────┐
│              GraphRAG Core Engine                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Index Module (graphrag/general/index.py)           │   │
│  │  - run_graphrag()                                    │   │
│  │  - generate_subgraph()                               │   │
│  │  - merge_subgraph()                                  │   │
│  │  - resolve_entities()                                │   │
│  │  - extract_community()                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Search Module (graphrag/search.py)                  │   │
│  │  - KGSearch.retrieval()                              │   │
│  │  - query_rewrite()                                   │   │
│  │  - get_relevant_ents_by_keywords()                   │   │
│  │  - get_relevant_relations_by_txt()                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Extractors                                          │   │
│  │  ┌────────────────┐  ┌────────────────┐             │   │
│  │  │ General        │  │ Light          │             │   │
│  │  │ GraphExtractor │  │ GraphExtractor │             │   │
│  │  └────────────────┘  └────────────────┘             │   │
│  │  ┌────────────────┐  ┌────────────────┐             │   │
│  │  │ Entity         │  │ Community      │             │   │
│  │  │ Resolution     │  │ Reports        │             │   │
│  │  └────────────────┘  └────────────────┘             │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       │
┌──────────────────────▼───────────────────────────────────────┐
│                  Storage Layer                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │ Elastic    │  │ NetworkX   │  │ Redis      │            │
│  │ search/    │  │ Graph      │  │ Cache      │            │
│  │ Infinity   │  │ Storage    │  │            │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 模組層級架構

```
graphrag/
├── search.py                    # 查詢接口層
├── utils.py                     # 工具函數層
├── entity_resolution.py         # 實體解析層
├── query_analyze_prompt.py      # Prompt 管理
├── entity_resolution_prompt.py
│
├── general/                     # General 模式實現
│   ├── index.py                # 主流程控制
│   ├── extractor.py            # 基礎提取器
│   ├── graph_extractor.py      # 圖譜提取器
│   ├── graph_prompt.py         # 提取提示詞
│   ├── community_reports_extractor.py
│   ├── community_report_prompt.py
│   ├── leiden.py               # 社區檢測
│   ├── entity_embedding.py     # Node2Vec
│   └── mind_map_extractor.py   # 思維導圖
│
└── light/                       # Light 模式實現
    ├── graph_extractor.py
    └── graph_prompt.py
```

---

## 3. 核心模組

### 3.1 知識圖譜構建模組

#### 3.1.1 主流程控制器 (index.py)

**文件位置**: `graphrag/general/index.py`

**核心函數**:

##### `run_graphrag()`

```python
async def run_graphrag(
    row: dict,                  # 任務數據
    language: str,              # 語言 (如 "English", "Chinese")
    with_resolution: bool,      # 是否啟用實體解析
    with_community: bool,       # 是否生成社區報告
    chat_model,                 # LLM 模型實例
    embedding_model,            # 嵌入模型實例
    callback: Callable,         # 進度回調函數
) -> None
```

**功能**:
- 協調整個 GraphRAG 構建流程
- 管理分佈式鎖以防止並發衝突
- 順序執行子圖生成、合併、實體解析、社區提取

**執行流程**:
1. 從文檔存儲加載文本塊 (chunks)
2. 調用 `generate_subgraph()` 生成子圖
3. 調用 `merge_subgraph()` 合併到全局圖
4. 可選: 調用 `resolve_entities()` 進行實體去重
5. 可選: 調用 `extract_community()` 生成社區報告

##### `generate_subgraph()`

```python
async def generate_subgraph(
    extractor: Extractor,       # LightKGExt 或 GeneralKGExt
    tenant_id: str,
    kb_id: str,
    doc_id: str,
    chunks: list[str],          # 文檔片段列表
    language: str,
    entity_types: list[str],    # 要提取的實體類型
    llm_bdl,                    # LLM bundle
    embed_bdl,                  # Embedding bundle
    callback: Callable,
) -> nx.Graph | None            # 返回 NetworkX 圖對象
```

**功能**:
- 從文本塊中提取實體和關係
- 構建 NetworkX 子圖
- 驗證圖的完整性
- 將子圖存儲到 Elasticsearch/Infinity

**數據流**:
```
chunks (文本) → Extractor → (entities, relations)
                → NetworkX Graph → Elasticsearch
```

##### `merge_subgraph()`

```python
async def merge_subgraph(
    tenant_id: str,
    kb_id: str,
    doc_id: str,
    subgraph: nx.Graph,         # 新生成的子圖
    embedding_model,
    callback: Callable,
) -> nx.Graph                   # 返回合併後的全局圖
```

**功能**:
- 加載現有全局圖
- 合併新子圖到全局圖
- 計算 PageRank 分數
- 更新圖存儲

**合併策略**:
- 節點合併: 合併相同名稱的節點，合併屬性
- 邊合併: 合併相同源-目標的邊，合併描述
- PageRank: 為所有節點重新計算重要性分數

##### `resolve_entities()`

```python
async def resolve_entities(
    graph: nx.Graph,            # 全局圖
    subgraph_nodes: set[str],   # 新增節點集合
    tenant_id: str,
    kb_id: str,
    doc_id: str,
    llm_bdl,
    embed_bdl,
    callback: Callable,
) -> None
```

**功能**:
- 使用 LLM 判斷實體是否重複
- 合併相似實體 (如 "2025" 和 "year of 2025")
- 更新圖存儲和索引

##### `extract_community()`

```python
async def extract_community(
    graph: nx.Graph,            # 全局圖
    tenant_id: str,
    kb_id: str,
    doc_id: str,
    llm_bdl,
    embed_bdl,
    callback: Callable,
) -> tuple[list[dict], list[str]]  # (社區結構, 社區報告)
```

**功能**:
- 使用 Leiden 算法進行社區檢測
- 為每個社區生成摘要報告
- 將社區報告索引到 Elasticsearch

#### 3.1.2 圖譜提取器

##### General 模式 (graph_extractor.py)

**文件位置**: `graphrag/general/graph_extractor.py`

```python
class GraphExtractor(Extractor):
    def __init__(
        self,
        llm_invoker: CompletionLLM,
        language: str | None = "English",
        entity_types: list[str] | None = None,
        join_descriptions: bool = True,
        max_gleanings: int | None = None,  # 默認 2
    ):
        ...

    async def __call__(
        self,
        doc_id: str,
        chunks: list[str],
        callback: Callable | None = None
    ) -> tuple[list[dict], list[dict]]  # (entities, relations)
```

**特點**:
- 使用 Microsoft GraphRAG 的提示詞策略
- 支持多次查詢 (gleanings) 以提取遺漏的實體
- 高精度但 token 消耗較大

**輸出格式**:
- 實體: `{"entity_name": str, "entity_type": str, "description": str}`
- 關係: `{"src_id": str, "tgt_id": str, "description": str, "weight": int}`

##### Light 模式 (light/graph_extractor.py)

**文件位置**: `graphrag/light/graph_extractor.py`

**特點**:
- 基於 LightRAG 優化的提示詞
- 單次查詢，更快更節省 token
- 適用於大規模文檔處理

**提取分隔符**:
```python
DEFAULT_TUPLE_DELIMITER = "<|>"
DEFAULT_RECORD_DELIMITER = "##"
DEFAULT_COMPLETION_DELIMITER = "<|COMPLETE|>"
```

**示例輸出**:
```
("entity"<|>Apple Inc.<|>organization<|>A technology company##
("entity"<|>iPhone<|>product<|>A smartphone product##
("relationship"<|>Apple Inc.<|>iPhone<|>manufactures<|>8)
```

#### 3.1.3 實體解析 (entity_resolution.py)

**文件位置**: `graphrag/entity_resolution.py`

```python
class EntityResolution(Extractor):
    async def __call__(
        self,
        graph: nx.Graph,
        subgraph_nodes: set[str],    # 待解析的節點
        prompt_variables: dict[str, Any] | None = None,
        callback: Callable | None = None
    ) -> EntityResolutionResult
```

**工作原理**:
1. 對每個新節點，找出圖中名稱相似的節點
2. 使用 LLM 判斷是否為同一實體
3. 如果是，合併節點並更新所有相關邊

**示例**:
- "2025" 和 "year of 2025" → 合併為 "2025"
- "Apple" 和 "Apple Inc." → 合併為 "Apple Inc."

#### 3.1.4 社區檢測與報告

##### Leiden 算法 (leiden.py)

**文件位置**: `graphrag/general/leiden.py`

```python
def run(graph: nx.Graph, args: dict[str, Any]) -> dict[int, dict[str, dict]]:
    """
    Args:
        max_cluster_size: 最大社區大小 (默認 12)
        use_lcc: 是否只使用最大連通分量 (默認 True)

    Returns:
        {level: {community_id: {node: level_of_node}}}
    """
```

**算法特點**:
- 分層社區檢測
- 自動控制社區大小
- 支持加權圖

##### 社區報告提取器 (community_reports_extractor.py)

**文件位置**: `graphrag/general/community_reports_extractor.py`

```python
class CommunityReportsExtractor(Extractor):
    async def __call__(
        self,
        graph: nx.Graph,
        callback: Callable | None = None
    ) -> CommunityReportsResult
```

**報告結構**:
```json
{
    "title": "社區標題",
    "summary": "社區摘要 (150字)",
    "rating": 7.5,
    "rating_explanation": "評分說明",
    "findings": [
        {
            "summary": "發現摘要",
            "explanation": "詳細解釋"
        }
    ],
    "entities": ["實體1", "實體2"],
    "weight": 0.85
}
```

### 3.2 知識圖譜查詢模組

#### 3.2.1 KGSearch 類 (search.py)

**文件位置**: `graphrag/search.py`

```python
class KGSearch(Dealer):
    def retrieval(
        self,
        question: str,              # 用戶問題
        tenant_ids: str | list[str], # 租戶 ID
        kb_ids: list[str],          # 知識庫 ID 列表
        emb_mdl,                    # 嵌入模型
        llm,                        # 語言模型
        max_token: int = 8196,      # 最大返回 token 數
        ent_topn: int = 6,          # 返回的實體數量
        rel_topn: int = 6,          # 返回的關係數量
        comm_topn: int = 1,         # 返回的社區報告數量
        ent_sim_threshold: float = 0.3,  # 實體相似度閾值
        rel_sim_threshold: float = 0.3,  # 關係相似度閾值
    ) -> dict
```

**返回格式**:
```python
{
    "chunk_id": str,
    "content_with_weight": str,  # 格式化的實體、關係、社區報告
    "docnm_kwd": "Related content in Knowledge Graph",
    "similarity": 1.0,
    "kb_id": list[str],
    # ... 其他元數據
}
```

#### 3.2.2 查詢重寫 (query_rewrite)

```python
def query_rewrite(
    self,
    llm,
    question: str,
    idxnms: list[str],
    kb_ids: list[str]
) -> tuple[list[str], list[str]]  # (type_keywords, entities_from_query)
```

**功能**:
- 使用 LLM 分析問題
- 提取答案類型關鍵詞 (如 "person", "organization")
- 提取問題中提到的實體

**示例**:
```
Question: "Who is the CEO of Apple?"
Output:
  type_keywords = ["person"]
  entities_from_query = ["Apple", "CEO"]
```

#### 3.2.3 實體檢索方法

##### 基於關鍵詞檢索

```python
def get_relevant_ents_by_keywords(
    self,
    keywords: list[str],        # 關鍵詞列表
    filters: dict,
    idxnms: list[str],
    kb_ids: list[str],
    emb_mdl,
    sim_thr: float = 0.3,       # 相似度閾值
    N: int = 56                 # 檢索數量
) -> dict[str, dict]            # {entity_name: {sim, pagerank, description, n_hop_ents}}
```

**檢索策略**:
- 將關鍵詞連接成查詢字符串
- 使用嵌入模型生成向量
- 在 Elasticsearch 中執行向量相似度搜索
- 過濾低於閾值的結果

##### 基於類型檢索

```python
def get_relevant_ents_by_types(
    self,
    types: list[str],           # 實體類型列表
    filters: dict,
    idxnms: list[str],
    kb_ids: list[str],
    N: int = 56
) -> dict[str, dict]
```

**檢索策略**:
- 精確匹配實體類型
- 按 PageRank 降序排序
- 返回前 N 個結果

#### 3.2.4 關係檢索

```python
def get_relevant_relations_by_txt(
    self,
    txt: str,                   # 查詢文本
    filters: dict,
    idxnms: list[str],
    kb_ids: list[str],
    emb_mdl,
    sim_thr: float = 0.3,
    N: int = 56
) -> dict[tuple[str, str], dict]  # {(from_ent, to_ent): {sim, pagerank, description}}
```

#### 3.2.5 社區報告檢索

```python
def _community_retrival_(
    self,
    entities: list[str],        # 相關實體列表
    condition: dict,
    kb_ids: list[str],
    idxnms: list[str],
    topn: int,
    max_token: int
) -> str                        # 格式化的社區報告文本
```

**檢索策略**:
- 查找包含指定實體的社區
- 按社區權重降序排序
- 返回格式化的報告

### 3.3 工具模組 (utils.py)

**文件位置**: `graphrag/utils.py`

#### 3.3.1 圖操作

##### 圖合併

```python
def graph_merge(
    g1: nx.Graph,               # 目標圖
    g2: nx.Graph,               # 源圖
    change: GraphChange         # 變更跟蹤
) -> nx.Graph                   # 合併後的圖
```

**合併規則**:
- **節點合併**:
  - 如果節點存在，合併 `description` 和 `source_id`
  - 如果節點不存在，添加新節點
- **邊合併**:
  - 如果邊存在，合併 `description`，累加 `weight`
  - 如果邊不存在，添加新邊

##### 圖存取

```python
async def get_graph(
    tenant_id: str,
    kb_id: str,
    exclude_rebuild: list[str] | None = None
) -> nx.Graph | None

async def set_graph(
    tenant_id: str,
    kb_id: str,
    embd_mdl,
    graph: nx.Graph,
    change: GraphChange,
    callback: Callable
) -> None

async def rebuild_graph(
    tenant_id: str,
    kb_id: str,
    exclude_rebuild: list[str] | None = None
) -> nx.Graph
```

**存儲機制**:
- 從 Elasticsearch 檢索所有 `knowledge_graph_kwd="subgraph"` 的塊
- 反序列化 NetworkX 圖
- 合併所有子圖為全局圖

##### 圖驗證

```python
def tidy_graph(graph: nx.Graph, callback: Callable) -> None
```

**驗證項目**:
1. 所有節點必須有 `entity_name`
2. 所有節點必須有 `description`
3. 所有邊的端點必須存在於節點集合中
4. 所有邊必須有 `description`

#### 3.3.2 數據轉換

##### 節點轉 Chunk

```python
async def graph_node_to_chunk(
    kb_id: str,
    embd_mdl,
    ent_name: str,
    meta: dict,                 # 節點屬性
    chunks: list[dict]          # 輸出列表
) -> None
```

**Chunk 結構**:
```python
{
    "id": str,
    "important_kwd": [entity_name],
    "title_tks": list[int],
    "entity_kwd": entity_name,
    "entity_type_kwd": entity_type,
    "knowledge_graph_kwd": "entity",
    "content_with_weight": json.dumps(meta),
    "content_ltks": list[int],      # token 化的描述
    "content_sm_ltks": list[int],   # 細粒度 token
    "source_id": list[str],
    "kb_id": kb_id,
    "q_{dim}_vec": list[float],     # 嵌入向量
    "rank_flt": float,               # PageRank 分數
    "n_hop_with_weight": str,        # JSON: N-hop 鄰居信息
    "available_int": 0
}
```

##### 邊轉 Chunk

```python
async def graph_edge_to_chunk(
    kb_id: str,
    embd_mdl,
    from_ent_name: str,
    to_ent_name: str,
    meta: dict,                 # 邊屬性
    chunks: list[dict]
) -> None
```

**Chunk 結構**:
```python
{
    "id": str,
    "from_entity_kwd": from_entity_name,
    "to_entity_kwd": to_entity_name,
    "knowledge_graph_kwd": "relation",
    "weight_int": int,          # 關係權重
    "content_with_weight": json.dumps(meta),
    "content_ltks": list[int],
    "important_kwd": [keywords],
    "source_id": list[str],
    "kb_id": kb_id,
    "q_{dim}_vec": list[float],
    "available_int": 0
}
```

#### 3.3.3 緩存管理

##### LLM 緩存

```python
def get_llm_cache(
    llmnm: str,                 # 模型名稱
    txt: str,                   # 系統提示
    history: list[dict],        # 對話歷史
    genconf: dict               # 生成配置
) -> str | None

def set_llm_cache(
    llmnm: str,
    txt: str,
    v: str,                     # LLM 響應
    history: list[dict],
    genconf: dict
) -> None
```

**緩存策略**:
- 使用 xxhash64 生成鍵
- Redis 存儲，24小時過期
- 大幅減少重複 LLM 調用

##### 嵌入緩存

```python
def get_embed_cache(llmnm: str, txt: str) -> np.ndarray | None
def set_embed_cache(llmnm: str, txt: str, arr: np.ndarray) -> None
```

---

## 4. API 規格

### 4.1 主要 API 端點

雖然 GraphRAG 主要通過內部模組調用，但以下是集成點：

#### 4.1.1 文檔上傳與解析

**端點**: `POST /api/v1/dataset/{dataset_id}/document`

**GraphRAG 觸發條件**:
```python
if parser_config.get("graphrag", {}).get("use_graphrag"):
    # 觸發 GraphRAG 構建任務
    queue_raptor_o_graphrag_tasks(doc, "graphrag", priority)
```

#### 4.1.2 知識圖譜查詢

**內部調用**:
```python
from graphrag import search as kg_search

kg = kg_search.KGSearch(settings.docStoreConn)
result = kg.retrieval(
    question=question,
    tenant_ids=tenant_id,
    kb_ids=[kb_id],
    emb_mdl=embedding_model,
    llm=chat_model,
    ent_topn=6,
    rel_topn=6,
    comm_topn=1,
)
```

**集成位置**:
- `rag/app/conversation.py` - 在對話檢索中調用
- `api/apps/sdk/retrieval.py` - 在 SDK 檢索中調用

### 4.2 Python SDK 接口

#### 4.2.1 構建知識圖譜

```python
from graphrag.general.index import run_graphrag
from graphrag.light.graph_extractor import GraphExtractor as LightKGExt
from graphrag.general.graph_extractor import GraphExtractor as GeneralKGExt

# 異步調用
await run_graphrag(
    row={
        "tenant_id": "your_tenant_id",
        "kb_id": "your_kb_id",
        "doc_id": "your_doc_id",
        "kb_parser_config": {
            "graphrag": {
                "method": "light",  # 或 "general"
                "entity_types": ["person", "organization", "geo"],
                "resolution": True,
                "community": True
            }
        }
    },
    language="English",
    with_resolution=True,
    with_community=True,
    chat_model=your_chat_model,
    embedding_model=your_embedding_model,
    callback=lambda msg: print(msg)
)
```

#### 4.2.2 查詢知識圖譜

```python
from graphrag.search import KGSearch
from api import settings

kg = KGSearch(settings.docStoreConn)
result = kg.retrieval(
    question="Who is the CEO of Apple?",
    tenant_ids="your_tenant_id",
    kb_ids=["your_kb_id"],
    emb_mdl=embedding_model,
    llm=chat_model,
    max_token=8196,
    ent_topn=6,
    rel_topn=6,
    comm_topn=1,
    ent_sim_threshold=0.3,
    rel_sim_threshold=0.3,
)

# 結果包含格式化的實體、關係和社區報告
print(result["content_with_weight"])
```

---

## 5. 配置規格

### 5.1 GraphRAG 配置模型

**文件位置**: `api/utils/validation_utils.py:355`

```python
class GraphragConfig(BaseModel):
    use_graphrag: bool = Field(default=False)
    entity_types: list[str] = Field(
        default_factory=lambda: [
            "organization",
            "person",
            "geo",
            "event",
            "category"
        ]
    )
    method: GraphragMethodEnum = Field(default=GraphragMethodEnum.light)
    community: bool = Field(default=False)
    resolution: bool = Field(default=False)
```

### 5.2 配置參數詳解

| 參數 | 類型 | 默認值 | 說明 |
|------|------|--------|------|
| `use_graphrag` | bool | `false` | 是否啟用 GraphRAG 功能 |
| `entity_types` | list[str] | `["organization", "person", "geo", "event", "category"]` | 要提取的實體類型列表，可自定義 |
| `method` | enum | `"light"` | 提取方法：`"light"` 或 `"general"` |
| `community` | bool | `false` | 是否生成社區報告 |
| `resolution` | bool | `false` | 是否啟用實體解析（去重） |

### 5.3 方法對比

| 特性 | Light 模式 | General 模式 |
|------|-----------|--------------|
| **精度** | 中等 | 高 |
| **速度** | 快 | 慢 |
| **Token 消耗** | 低 (約 50%) | 高 |
| **適用場景** | 大規模文檔、成本敏感 | 複雜領域、高精度需求 |
| **額外查詢** | 無 | 支持 (max_gleanings=2) |
| **提示詞來源** | LightRAG | Microsoft GraphRAG |

### 5.4 前端配置界面

**文件位置**: `web/src/components/parse-configuration/graph-rag-items.tsx`

**UI 組件**:
1. **Use GraphRAG** - Switch 開關
2. **Entity Types** - Tag 輸入框，支持添加/刪除
3. **Method** - Radio 單選 (Light / General)
4. **Entity Resolution** - Checkbox 複選框
5. **Community Reports** - Checkbox 複選框

---

## 6. 數據模型

### 6.1 圖數據結構

#### 6.1.1 NetworkX 圖模型

**節點屬性**:
```python
{
    "entity_name": str,         # 實體名稱（節點 ID）
    "entity_type": str,         # 實體類型
    "description": str,         # 實體描述
    "source_id": list[str],     # 來源文檔 ID 列表
    "pagerank": float,          # PageRank 分數
    # 可選屬性:
    "entity_id": str,           # 唯一標識符
    "rank": float,              # Node2Vec rank (如果啟用)
}
```

**邊屬性**:
```python
{
    "description": str,         # 關係描述
    "weight": int,              # 關係權重 (1-10)
    "source_id": list[str],     # 來源文檔 ID 列表
    "src_id": str,              # 源實體名稱
    "tgt_id": str,              # 目標實體名稱
}
```

**圖屬性**:
```python
graph.graph = {
    "source_id": list[str],     # 圖的來源文檔列表
}
```

### 6.2 Elasticsearch 索引結構

#### 6.2.1 實體 Chunk

**knowledge_graph_kwd**: `"entity"`

```json
{
    "id": "uuid",
    "important_kwd": ["entity_name"],
    "title_tks": [101, 2033, 102],
    "entity_kwd": "Apple Inc.",
    "entity_type_kwd": "organization",
    "knowledge_graph_kwd": "entity",
    "content_with_weight": "{\"entity_name\": \"Apple Inc.\", \"entity_type\": \"organization\", \"description\": \"...\"}",
    "content_ltks": [101, 1037, 102],
    "content_sm_ltks": [101, 1037, 102],
    "source_id": ["doc_id_1", "doc_id_2"],
    "kb_id": "kb_id",
    "q_768_vec": [0.1, 0.2, ...],
    "rank_flt": 0.0123,
    "n_hop_with_weight": "[{\"path\": [\"A\", \"B\", \"C\"], \"weights\": [0.8, 0.6]}]",
    "available_int": 0
}
```

**關鍵字段說明**:
- `entity_kwd`: 實體名稱，用於精確匹配
- `entity_type_kwd`: 實體類型，用於類型過濾
- `rank_flt`: PageRank 分數，用於排序
- `n_hop_with_weight`: N-hop 鄰居路徑信息
- `q_{dim}_vec`: 實體描述的嵌入向量

#### 6.2.2 關係 Chunk

**knowledge_graph_kwd**: `"relation"`

```json
{
    "id": "uuid",
    "from_entity_kwd": "Apple Inc.",
    "to_entity_kwd": "iPhone",
    "knowledge_graph_kwd": "relation",
    "weight_int": 8,
    "content_with_weight": "{\"description\": \"manufactures\", \"weight\": 8}",
    "content_ltks": [101, 2033, 102],
    "important_kwd": ["manufactures", "produces"],
    "source_id": ["doc_id_1"],
    "kb_id": "kb_id",
    "q_768_vec": [0.1, 0.2, ...],
    "available_int": 0
}
```

**關鍵字段說明**:
- `from_entity_kwd`, `to_entity_kwd`: 關係的源和目標實體
- `weight_int`: 關係強度 (1-10)
- `q_{dim}_vec`: 關係描述的嵌入向量

#### 6.2.3 社區報告 Chunk

**knowledge_graph_kwd**: `"community_report"`

```json
{
    "id": "uuid",
    "docnm_kwd": "Technology Products Community",
    "title_tks": [101, 2033, 102],
    "entities_kwd": ["Apple Inc.", "iPhone", "iPad"],
    "knowledge_graph_kwd": "community_report",
    "weight_flt": 0.85,
    "content_with_weight": "{\"report\": \"...\", \"evidences\": \"...\"}",
    "content_ltks": [101, 1037, 102],
    "content_sm_ltks": [101, 1037, 102],
    "important_kwd": ["Apple Inc.", "iPhone"],
    "source_id": ["doc_id_1", "doc_id_2"],
    "kb_id": "kb_id",
    "available_int": 0
}
```

**關鍵字段說明**:
- `docnm_kwd`: 社區標題
- `entities_kwd`: 社區包含的實體列表
- `weight_flt`: 社區權重
- `content_with_weight`: JSON 字符串，包含 `report` 和 `evidences`

#### 6.2.4 子圖 Chunk

**knowledge_graph_kwd**: `"subgraph"`

```json
{
    "id": "uuid",
    "knowledge_graph_kwd": "subgraph",
    "content_with_weight": "{\"directed\": false, \"nodes\": [...], \"edges\": [...]}",
    "source_id": ["doc_id_1"],
    "kb_id": "kb_id",
    "removed_kwd": "N",
    "available_int": 0
}
```

**關鍵字段說明**:
- `content_with_weight`: NetworkX `node_link_data()` 序列化的圖
- `removed_kwd`: 是否已刪除 ("Y" / "N")

### 6.3 任務數據模型

**文件位置**: `rag/svr/task_executor.py`

```python
{
    "task_type": "graphrag",
    "tenant_id": str,
    "kb_id": str,
    "doc_id": str,
    "kb_parser_config": {
        "graphrag": {
            "use_graphrag": True,
            "method": "light",
            "entity_types": ["person", "organization"],
            "resolution": False,
            "community": False
        }
    }
}
```

---

## 7. 工作流程

### 7.1 知識圖譜構建流程

```
[文檔上傳]
    ↓
[解析配置]
    ↓
[檢查 use_graphrag]
    ↓ (True)
[加入 Redis 任務隊列] (task_type="graphrag")
    ↓
[TaskExecutor 處理]
    ↓
┌─────────────────────────────────────┐
│  run_graphrag()                     │
│  ┌───────────────────────────────┐  │
│  │ 1. 加載文檔 chunks            │  │
│  │ 2. 選擇提取器                 │  │
│  │    - LightKGExt (light 模式)  │  │
│  │    - GeneralKGExt (general)   │  │
│  │ 3. generate_subgraph()        │  │
│  │    ├─ 調用提取器              │  │
│  │    ├─ 構建 NetworkX 圖        │  │
│  │    └─ 存儲子圖到 ES           │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │ 4. 獲取分佈式鎖               │  │
│  │ 5. merge_subgraph()           │  │
│  │    ├─ 加載全局圖              │  │
│  │    ├─ 合併子圖                │  │
│  │    ├─ 計算 PageRank           │  │
│  │    └─ 更新索引                │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │ 6. resolve_entities() (可選)  │  │
│  │    ├─ LLM 判斷實體相似性       │  │
│  │    ├─ 合併相似實體            │  │
│  │    └─ 更新索引                │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │ 7. extract_community() (可選) │  │
│  │    ├─ Leiden 社區檢測         │  │
│  │    ├─ 生成社區報告            │  │
│  │    └─ 存儲報告到 ES           │  │
│  └───────────────────────────────┘  │
│  8. 釋放分佈式鎖                  │  │
└─────────────────────────────────────┘
```

### 7.2 實體提取詳細流程

```
[文檔 chunks]
    ↓
[GraphExtractor]
    ↓
┌─────────────────────────────────────┐
│ 1. 構建提示詞                       │
│    - 系統提示                       │
│    - 實體類型定義                   │
│    - 文本塊                         │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 2. LLM 調用 (首次)                  │
│    輸出格式:                        │
│    ("entity"<|>name<|>type<|>desc## │
│    ("relationship"<|>src<|>tgt<|>.. │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 3. 解析輸出                         │
│    - 按 ## 分割記錄                 │
│    - 按 <|> 分割字段                │
│    - 驗證格式                       │
└─────────────────────────────────────┘
    ↓ (General 模式 + max_gleanings > 0)
┌─────────────────────────────────────┐
│ 4. 額外查詢 (可選)                  │
│    提示: "是否有遺漏的實體?"        │
│    重複 max_gleanings 次            │
└─────────────────────────────────────┘
    ↓
[返回 (entities, relations)]
```

### 7.3 查詢流程

```
[用戶問題]
    ↓
[KGSearch.retrieval()]
    ↓
┌─────────────────────────────────────┐
│ 1. query_rewrite()                  │
│    - 提取實體類型關鍵詞             │
│    - 提取問題中的實體               │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 2. 並行檢索 (Trio 並發)             │
│  ┌─────────────────────────────┐   │
│  │ get_relevant_ents_by_       │   │
│  │ keywords()                  │   │
│  │ - 向量相似度搜索            │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ get_relevant_ents_by_types()│   │
│  │ - 類型過濾 + PageRank 排序  │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ get_relevant_relations_by_  │   │
│  │ txt()                       │   │
│  │ - 向量相似度搜索關係        │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 3. 結果融合與排序                   │
│    - 計算分數: sim × PageRank       │
│    - 類型匹配加權 (×2)              │
│    - N-hop 路徑擴展                 │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 4. 截斷與格式化                     │
│    - 取 top-N 實體和關係            │
│    - 轉換為 DataFrame CSV 格式      │
│    - 檢索社區報告                   │
└─────────────────────────────────────┘
    ↓
[返回格式化結果]
```

### 7.4 實體解析流程

```
[新增節點集合]
    ↓
[EntityResolution]
    ↓
┌─────────────────────────────────────┐
│ For each new_node:                  │
│  ┌─────────────────────────────┐   │
│  │ 1. 尋找候選節點             │   │
│  │    - 名稱編輯距離 < 0.8     │   │
│  │    - 類型相同（如果有）     │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ 2. LLM 判斷                 │   │
│  │    提示: "這兩個實體是同一個│   │
│  │          實體嗎?"           │   │
│  │    輸出: YES/NO             │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ 3. 合併節點 (如果 YES)      │   │
│  │    - 合併描述               │   │
│  │    - 合併 source_id         │   │
│  │    - 重定向邊               │   │
│  │    - 刪除舊節點             │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
    ↓
[返回 (更新的圖, GraphChange)]
```

### 7.5 社區檢測與報告生成流程

```
[全局圖]
    ↓
[Leiden 算法]
    ↓
┌─────────────────────────────────────┐
│ 1. 社區檢測                         │
│    - 分層 Leiden                    │
│    - 控制社區大小 (max=12)          │
└─────────────────────────────────────┘
    ↓
[社區列表]
    ↓
┌─────────────────────────────────────┐
│ For each community:                 │
│  ┌─────────────────────────────┐   │
│  │ 2. 提取社區子圖             │   │
│  │    - 節點: 社區內所有節點   │   │
│  │    - 邊: 社區內所有邊       │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ 3. 構建提示詞               │   │
│  │    - 社區實體列表           │   │
│  │    - 社區關係列表           │   │
│  │    - 報告格式指令           │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ 4. LLM 生成報告             │   │
│  │    輸出 JSON:               │   │
│  │    - title                  │   │
│  │    - summary                │   │
│  │    - rating (0-10)          │   │
│  │    - findings               │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ 5. 解析並存儲               │   │
│  │    - 轉換為 Chunk           │   │
│  │    - 計算社區權重           │   │
│  │    - 索引到 Elasticsearch   │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

---

## 8. 集成指南

### 8.1 前端集成

#### 8.1.1 配置界面集成

**文件位置**: `web/src/components/parse-configuration/`

**關鍵文件**:
- `graph-rag-items.tsx` - UI 組件
- `graph-rag-form-fields.tsx` - 表單字段定義

**集成步驟**:

1. **導入組件**:
```typescript
import { GraphRagItems } from '@/components/parse-configuration/graph-rag-items';
```

2. **添加到解析配置表單**:
```typescript
<Form.Item name={['graphrag', 'use_graphrag']} valuePropName="checked">
  <Switch />
</Form.Item>

<Form.Item name={['graphrag', 'entity_types']}>
  <GraphRagEntityTypesInput />
</Form.Item>

<Form.Item name={['graphrag', 'method']}>
  <Radio.Group>
    <Radio value="light">Light</Radio>
    <Radio value="general">General</Radio>
  </Radio.Group>
</Form.Item>
```

#### 8.1.2 知識圖譜可視化（未實現）

**建議實現**:
- 使用 D3.js 或 ECharts 可視化圖譜
- 顯示實體、關係和社區
- 支持交互式查詢

### 8.2 後端集成

#### 8.2.1 任務隊列集成

**文件位置**: `api/db/services/document_service.py`

```python
def queue_raptor_o_graphrag_tasks(doc, ty, priority):
    """將 GraphRAG 任務加入 Redis 隊列"""
    task = {
        "task_type": "graphrag",
        "tenant_id": doc["tenant_id"],
        "kb_id": doc["kb_id"],
        "doc_id": doc["id"],
        "kb_parser_config": doc["kb_parser_config"],
    }
    redis_client.lpush(f"task_queue:{priority}", json.dumps(task))
```

#### 8.2.2 檢索集成

**文件位置**: `rag/app/conversation.py`

```python
from graphrag import search as kg_search

# 在檢索流程中添加
kg = kg_search.KGSearch(settings.docStoreConn)
kg_result = kg.retrieval(
    question=question,
    tenant_ids=tenant_id,
    kb_ids=[kb_id],
    emb_mdl=embedding_model,
    llm=chat_model,
)

# 將結果添加到上下文
chunks.append(kg_result)
```

### 8.3 數據庫遷移

**無需額外遷移** - GraphRAG 使用現有的 Elasticsearch/Infinity 存儲，通過 `knowledge_graph_kwd` 字段區分數據類型。

### 8.4 環境變量配置

```bash
# .env 文件
MAX_CONCURRENT_CHATS=10        # LLM 並發限制
REDIS_HOST=localhost           # Redis 緩存主機
REDIS_PORT=6379                # Redis 端口
```

---

## 9. 性能考慮

### 9.1 資源消耗

| 操作 | CPU | 內存 | 網絡 | 存儲 |
|------|-----|------|------|------|
| **Light 提取** | 低 | 低 (100MB) | 中 (LLM API) | 中 |
| **General 提取** | 低 | 中 (200MB) | 高 (LLM API) | 中 |
| **實體解析** | 低 | 中 (500MB) | 高 (LLM API) | 低 |
| **社區檢測** | 高 | 高 (1GB+) | 低 | 低 |
| **社區報告** | 低 | 低 | 高 (LLM API) | 低 |
| **查詢** | 低 | 低 | 中 (ES + LLM) | 無 |

### 9.2 Token 消耗估算

**Light 模式**:
- 每個 chunk (512 tokens): 約 1,000 tokens (輸入 + 輸出)
- 100 chunks 文檔: 約 100,000 tokens

**General 模式**:
- 每個 chunk: 約 2,000 tokens (含額外查詢)
- 100 chunks 文檔: 約 200,000 tokens

**實體解析**:
- 每對實體: 約 200 tokens
- 100 個新實體 × 平均 3 個候選: 約 60,000 tokens

**社區報告**:
- 每個社區: 約 3,000 tokens
- 10 個社區: 約 30,000 tokens

### 9.3 時間複雜度

| 操作 | 時間複雜度 | 備註 |
|------|-----------|------|
| **提取** | O(n) | n = chunk 數量 |
| **合併圖** | O(V + E) | V = 節點數, E = 邊數 |
| **PageRank** | O(k(V + E)) | k = 迭代次數 (約 100) |
| **實體解析** | O(m × c) | m = 新節點數, c = 候選數 (平均 3) |
| **Leiden** | O((V + E) log V) | 分層社區檢測 |
| **查詢** | O(log V) | Elasticsearch 向量搜索 |

### 9.4 優化建議

#### 9.4.1 提取優化

1. **使用 Light 模式**：節省 50% token 和時間
2. **批量處理**：一次處理多個 chunk
3. **並行提取**：多文檔並行構建圖譜

#### 9.4.2 查詢優化

1. **調整 topn 參數**：減少返回的實體/關係數量
2. **提高相似度閾值**：過濾低質量結果
3. **緩存查詢結果**：相同問題重用結果

#### 9.4.3 存儲優化

1. **定期清理**：刪除舊的子圖 chunk
2. **壓縮描述**：限制描述長度
3. **索引優化**：合理配置 Elasticsearch 分片

### 9.5 擴展性

#### 9.5.1 橫向擴展

- **任務隊列**: Redis + 多個 TaskExecutor 實例
- **LLM 服務**: 使用 LLM 網關實現負載均衡
- **Elasticsearch**: 分片和副本擴展

#### 9.5.2 圖規模限制

| 節點數 | 邊數 | PageRank 時間 | 內存消耗 |
|--------|------|--------------|----------|
| 1,000 | 5,000 | < 1s | 50MB |
| 10,000 | 50,000 | 2-5s | 200MB |
| 100,000 | 500,000 | 30-60s | 2GB |
| 1,000,000 | 5,000,000 | 10-20min | 20GB |

**建議**: 對於超大規模圖 (> 100K 節點)，考慮:
- 分區圖譜 (按知識庫或領域)
- 增量 PageRank 計算
- 使用圖數據庫 (如 Neo4j)

---

## 10. 限制與最佳實踐

### 10.1 已知限制

1. **單一圖結構**: 每個知識庫一個全局圖，無多版本支持
2. **同步 PageRank**: 每次更新都重新計算所有節點
3. **無事務支持**: 圖更新不是原子操作
4. **LLM 依賴**: 實體解析和社區報告依賴 LLM 質量
5. **內存限制**: 大規模圖需要大量內存
6. **無增量更新**: 文檔刪除需要重建整個圖

### 10.2 最佳實踐

#### 10.2.1 配置選擇

```python
# 小規模文檔 (< 100 文檔)
{
    "use_graphrag": True,
    "method": "general",        # 高精度
    "entity_types": ["person", "organization", "product"],
    "resolution": True,         # 啟用去重
    "community": True           # 生成報告
}

# 大規模文檔 (> 1000 文檔)
{
    "use_graphrag": True,
    "method": "light",          # 高效率
    "entity_types": ["person", "organization"],  # 減少類型
    "resolution": False,        # 禁用去重以節省時間
    "community": False          # 按需生成
}
```

#### 10.2.2 實體類型設計

**推薦類型** (通用領域):
- `person` - 人物
- `organization` - 組織/公司
- `location` / `geo` - 地理位置
- `event` - 事件
- `product` - 產品
- `concept` - 概念/術語

**領域特定類型** (醫療):
- `disease` - 疾病
- `drug` - 藥物
- `symptom` - 症狀
- `treatment` - 治療方法

#### 10.2.3 查詢參數調優

```python
# 高精度查詢 (犧牲召回率)
kg.retrieval(
    question=question,
    ent_topn=10,                    # 更多實體
    rel_topn=10,                    # 更多關係
    comm_topn=3,                    # 多個社區報告
    ent_sim_threshold=0.5,          # 高閾值
    rel_sim_threshold=0.5,
    max_token=16384                 # 大上下文
)

# 高效查詢 (快速響應)
kg.retrieval(
    question=question,
    ent_topn=3,                     # 少量實體
    rel_topn=3,                     # 少量關係
    comm_topn=1,                    # 單個報告
    ent_sim_threshold=0.2,          # 低閾值
    rel_sim_threshold=0.2,
    max_token=4096                  # 小上下文
)
```

#### 10.2.4 監控與調試

**關鍵日志**:
```python
# 啟用詳細日志
import logging
logging.getLogger("graphrag").setLevel(logging.DEBUG)

# 監控的指標
- 提取的實體/關係數量
- PageRank 計算時間
- 實體解析合併數量
- 社區數量和大小
- 查詢響應時間
- LLM 調用次數和 token 消耗
```

**回調函數**:
```python
def progress_callback(msg):
    """實時進度監控"""
    print(f"[{datetime.now()}] {msg}")
    # 可以發送到監控系統或前端
```

#### 10.2.5 故障處理

1. **LLM 調用失敗**: 自動重試 (已內置緩存)
2. **圖合併衝突**: 使用分佈式鎖 (RedisDistributedLock)
3. **Elasticsearch 超時**: 增加 `es_bulk_size` 或減少並發
4. **內存不足**: 減少圖規模或增加服務器內存

### 10.3 故障排查

#### 問題 1: 提取不到實體

**原因**:
- LLM 模型能力不足
- 提示詞不適合目標語言
- 實體類型定義不清晰

**解決**:
- 使用更強大的模型 (如 GPT-4)
- 自定義提示詞 (修改 `graph_prompt.py`)
- 添加示例實體到提示詞

#### 問題 2: 查詢無結果

**原因**:
- 相似度閾值過高
- 嵌入模型質量差
- 問題表述與圖譜內容不匹配

**解決**:
- 降低 `ent_sim_threshold` 和 `rel_sim_threshold`
- 使用更好的嵌入模型
- 檢查圖譜是否正確構建

#### 問題 3: 性能慢

**原因**:
- 圖規模過大
- 使用 General 模式
- 未啟用緩存

**解決**:
- 切換到 Light 模式
- 禁用實體解析和社區報告
- 確保 Redis 緩存正常工作
- 分區知識庫

---

## 附錄 A: 完整示例

### A.1 端到端示例

```python
import asyncio
from api import settings
from api.db.services.llm_service import LLMBundle
from api.db import LLMType
from graphrag.general.index import run_graphrag
from graphrag.search import KGSearch

async def main():
    # 初始化
    settings.init_settings()

    # 準備模型
    chat_model = LLMBundle("tenant_id", LLMType.CHAT, "llm_id")
    embedding_model = LLMBundle("tenant_id", LLMType.EMBEDDING, "embd_id")

    # 構建圖譜
    task = {
        "tenant_id": "tenant_123",
        "kb_id": "kb_456",
        "doc_id": "doc_789",
        "kb_parser_config": {
            "graphrag": {
                "method": "light",
                "entity_types": ["person", "organization", "location"],
                "resolution": True,
                "community": True
            }
        }
    }

    await run_graphrag(
        row=task,
        language="English",
        with_resolution=True,
        with_community=True,
        chat_model=chat_model,
        embedding_model=embedding_model,
        callback=lambda msg: print(msg)
    )

    # 查詢圖譜
    kg = KGSearch(settings.docStoreConn)
    result = kg.retrieval(
        question="Who is the CEO of Apple?",
        tenant_ids="tenant_123",
        kb_ids=["kb_456"],
        emb_mdl=embedding_model,
        llm=chat_model,
        ent_topn=5,
        rel_topn=5,
        comm_topn=1,
    )

    print("=== Knowledge Graph Result ===")
    print(result["content_with_weight"])

if __name__ == "__main__":
    asyncio.run(main())
```

### A.2 自定義提取器示例

```python
from graphrag.general.extractor import Extractor

class CustomExtractor(Extractor):
    """自定義領域特定提取器"""

    def __init__(self, llm_invoker, language="English"):
        super().__init__(llm_invoker)
        self.language = language

    async def __call__(self, doc_id, chunks, callback=None):
        entities = []
        relations = []

        for chunk in chunks:
            # 自定義提取邏輯
            # 例如: 使用 NER 模型 + LLM
            ents, rels = await self.extract_from_chunk(chunk)
            entities.extend(ents)
            relations.extend(rels)

        return entities, relations

    async def extract_from_chunk(self, chunk):
        # 實現提取邏輯
        pass
```

### A.3 自定義查詢分析器

```python
from graphrag.search import KGSearch

class EnhancedKGSearch(KGSearch):
    """增強的查詢分析器"""

    def query_rewrite(self, llm, question, idxnms, kb_ids):
        # 先調用基礎方法
        type_keywords, entities = super().query_rewrite(
            llm, question, idxnms, kb_ids
        )

        # 添加自定義邏輯
        # 例如: 同義詞擴展、拼寫糾正
        enhanced_entities = self.expand_synonyms(entities)

        return type_keywords, enhanced_entities

    def expand_synonyms(self, entities):
        # 實現同義詞擴展
        synonym_map = {
            "Apple": ["Apple Inc.", "Apple Computer"],
            "CEO": ["Chief Executive Officer", "President"]
        }

        expanded = []
        for ent in entities:
            expanded.append(ent)
            if ent in synonym_map:
                expanded.extend(synonym_map[ent])

        return expanded
```

---

## 附錄 B: API 參考速查

### B.1 主要函數簽名

```python
# 構建 API
async def run_graphrag(
    row: dict, language: str, with_resolution: bool,
    with_community: bool, chat_model, embedding_model, callback
) -> None

async def generate_subgraph(
    extractor: Extractor, tenant_id: str, kb_id: str, doc_id: str,
    chunks: list[str], language: str, entity_types: list[str],
    llm_bdl, embed_bdl, callback
) -> nx.Graph | None

async def merge_subgraph(
    tenant_id: str, kb_id: str, doc_id: str,
    subgraph: nx.Graph, embedding_model, callback
) -> nx.Graph

# 查詢 API
def retrieval(
    self, question: str, tenant_ids: str | list[str], kb_ids: list[str],
    emb_mdl, llm, max_token: int = 8196, ent_topn: int = 6,
    rel_topn: int = 6, comm_topn: int = 1,
    ent_sim_threshold: float = 0.3, rel_sim_threshold: float = 0.3
) -> dict

def query_rewrite(
    self, llm, question: str, idxnms: list[str], kb_ids: list[str]
) -> tuple[list[str], list[str]]

# 工具 API
def graph_merge(g1: nx.Graph, g2: nx.Graph, change: GraphChange) -> nx.Graph
async def get_graph(tenant_id: str, kb_id: str, exclude_rebuild=None) -> nx.Graph | None
async def set_graph(tenant_id: str, kb_id: str, embd_mdl, graph: nx.Graph, change: GraphChange, callback) -> None
def get_llm_cache(llmnm: str, txt: str, history: list, genconf: dict) -> str | None
def set_llm_cache(llmnm: str, txt: str, v: str, history: list, genconf: dict) -> None
```

### B.2 數據類型

```python
# GraphChange
@dataclass
class GraphChange:
    removed_nodes: Set[str]
    added_updated_nodes: Set[str]
    removed_edges: Set[Tuple[str, str]]
    added_updated_edges: Set[Tuple[str, str]]

# GraphragConfig
class GraphragConfig(BaseModel):
    use_graphrag: bool = False
    entity_types: list[str] = ["organization", "person", "geo", "event", "category"]
    method: GraphragMethodEnum = GraphragMethodEnum.light
    community: bool = False
    resolution: bool = False

# 節點屬性
node_attrs = {
    "entity_name": str,
    "entity_type": str,
    "description": str,
    "source_id": list[str],
    "pagerank": float
}

# 邊屬性
edge_attrs = {
    "description": str,
    "weight": int,
    "source_id": list[str],
    "src_id": str,
    "tgt_id": str
}
```

---

## 附錄 C: 提示詞模板

### C.1 實體提取提示詞（Light）

```python
# graphrag/light/graph_prompt.py
PROMPTS = {
    "entity_extraction": """
-Goal-
Given a text document, identify all entities and their relationships.

-Steps-
1. Identify all entities mentioned in the text
2. Classify each entity by type: {entity_types}
3. For each pair of related entities, describe their relationship

-Output Format-
("entity"<|>entity_name<|>entity_type<|>entity_description##
("relationship"<|>source_entity<|>target_entity<|>relationship_description<|>strength)

Where strength is 1-10.

-Examples-
("entity"<|>Apple Inc.<|>organization<|>A technology company##
("relationship"<|>Apple Inc.<|>iPhone<|>manufactures<|>9)

-Text-
{input_text}

Output:
"""
}
```

### C.2 實體解析提示詞

```python
# graphrag/entity_resolution_prompt.py
PROMPTS = {
    "entity_resolution": """
You are a helpful assistant that determines if two entity mentions refer to the same real-world entity.

Entity 1: {entity1_name}
Description 1: {entity1_desc}

Entity 2: {entity2_name}
Description 2: {entity2_desc}

Question: Are these two entities the same?
Answer with only YES or NO.

Answer:
"""
}
```

### C.3 社區報告提示詞

```python
# graphrag/general/community_report_prompt.py
PROMPTS = {
    "community_report": """
Generate a comprehensive report for the following community of entities and relationships.

Entities:
{entities}

Relationships:
{relationships}

Generate a JSON report with:
- title: A descriptive title (5-10 words)
- summary: A comprehensive summary (150 words)
- rating: Importance rating 0-10
- rating_explanation: Why this rating
- findings: List of key findings, each with summary and explanation

Output JSON:
"""
}
```

---

## 附錄 D: 版本歷史

| 版本 | 日期 | 變更說明 |
|------|------|---------|
| 1.0 | 2025-01-14 | 初始版本，完整的 GraphRAG 規格文檔 |

---

## 附錄 E: 參考文獻

1. Microsoft GraphRAG: https://github.com/microsoft/graphrag
2. LightRAG: https://github.com/HKUDS/LightRAG
3. MiniRAG: https://github.com/HKUDS/MiniRAG
4. NetworkX Documentation: https://networkx.org/
5. Leiden Algorithm: https://www.nature.com/articles/s41598-019-41695-z
6. PageRank Algorithm: http://ilpubs.stanford.edu:8090/422/

---

**文檔維護**: RAGFlow 開發團隊
**聯繫方式**: https://github.com/infiniflow/ragflow
**許可證**: Apache License 2.0

---

*本規格文檔基於 RAGFlow 項目的實際實現編寫，旨在為其他專案提供集成和使用指南。*
