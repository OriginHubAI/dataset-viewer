# Dataset Viewer 架构与 API 文档

## 1. 项目概述

Dataset Viewer 是 Hugging Face 提供的数据集查看服务，用于访问 Hugging Face Hub 上数据集的内容、元数据和基本统计信息。该项目采用微服务架构，以 monorepo 形式组织代码。

### 1.1 核心功能

- **数据集预览**: 查看数据集的前 100 行数据
- **数据浏览**: 分页查看任意切片的数据
- **全文搜索**: 在文本列中进行全文搜索
- **数据过滤**: 使用 SQL-like 语法过滤数据
- **元数据获取**: 获取数据集的结构信息、特征类型等
- **Parquet 导出**: 获取数据集的 Parquet 文件列表
- **统计信息**: 获取数值列的描述性统计信息

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              外部客户端                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ Hugging Face │  │   Admin UI   │  │   CLI/API    │                       │
│  │     Hub      │  │              │  │   客户端      │                       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                       │
└─────────┼────────────────┼────────────────┼─────────────────────────────────┘
          │                │                │
          ▼                │                │
┌──────────────────────┐   │                │
│   Reverse Proxy      │   │                │
│   (Nginx)            │   │                │
└──────┬───────────────┘   │                │
       │                   │                │
       ▼                   ▼                ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│     API      │    │    Admin     │    │   Webhook    │
│   Service    │    │   Service    │    │   Service    │
│  (公共API)    │    │  (管理API)    │    │ (Hub回调)    │
└──────┬───────┘    └──────────────┘    └──────┬───────┘
       │                                       │
       │    ┌──────────────┐    ┌──────────────┐
       │    │    Rows      │    │    Search    │
       └───►│   Service    │◄──►│   Service    │
            │  (行数据服务)  │    │ (搜索/过滤)   │
            └──────┬───────┘    └──────────────┘
                   │
                   ▼
            ┌──────────────┐
            │    Worker    │
            │   Service    │
            │ (后台任务处理) │
            └──────┬───────┘
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│  MongoDB │ │    S3    │ │   EFS    │
│ (Cache & │ │ (Assets &│ │(临时文件) │
│  Queue)  │ │Parquet)  │ │          │
└──────────┘ └──────────┘ └──────────┘
```

### 2.2 核心组件说明

#### 2.2.1 数据存储层

| 组件 | 用途 | 说明 |
|------|------|------|
| **MongoDB** | 缓存和队列 | 三个数据库：cache(缓存结果)、queue(任务队列)、maintenance(维护) |
| **S3** | 对象存储 | 存储生成的资源文件(assets)、Parquet 文件、DuckDB 索引 |
| **EFS** | 共享文件系统 | Worker 临时文件存储、Parquet 元数据缓存 |

#### 2.2.2 服务层

| 服务 | 端口 | 职责 | 核心端点 |
|------|------|------|----------|
| **API** | 8080 | 公共 API 服务，提供预计算的缓存响应 | `/splits`, `/first-rows`, `/parquet`, `/info`, `/size`, `/is-valid`, `/statistics`, `/opt-in-out-urls`, `/presidio-entities` |
| **Rows** | 8081 | 动态行数据查询服务 | `/rows` |
| **Search** | 8082 | 全文搜索和过滤服务 | `/search`, `/filter` |
| **Admin** | 8083 | 管理接口，用于监控和运维 | `/dataset-status`, `/pending-jobs`, `/force-refresh/*`, `/cache-reports`, `/blocked-datasets` |
| **Webhook** | 8084 | 接收 Hugging Face Hub 的 webhook 通知 | `/webhook` |
| **Worker** | - | 异步处理任务队列 | 消费 queue 数据库中的 job，处理数据后写入 cache |

#### 2.2.3 共享库

| 库 | 路径 | 职责 |
|----|------|------|
| **libcommon** | `libs/libcommon/` | 通用代码：配置、队列操作、缓存操作、存储客户端、处理图定义 |
| **libapi** | `libs/libapi/` | API 相关：认证、异常、请求/响应处理、JWT 验证 |
| **libviewer** | `libs/libviewer/` | Rust 实现的数据集处理库，用于高性能数据操作 |

---

## 3. 数据流与处理流程

### 3.1 数据预热流程（Webhook 触发）

```
Hub Webhook ──► Webhook Service ──► Queue (MongoDB)
                                        │
                                        ▼
                              ┌──────────────────┐
                              │   Worker Pool    │
                              │ (N 个实例并行处理) │
                              └────────┬─────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
              ┌──────────┐      ┌──────────┐      ┌──────────┐
              │ 解析数据集 │  →   │ 生成Parquet│  →   │ 写入缓存  │
              │ 配置/分割 │      │ 文件/索引 │      │ (MongoDB)│
              └──────────┘      └──────────┘      └──────────┘
```

### 3.2 API 请求处理流程

```
Client Request
      │
      ▼
┌─────────────┐
│  Auth Check │  (JWT / HF Token / 外部认证)
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│  Cache Hit? │────►│  直接返回缓存结果   │
└──────┬──────┘     └──────────────────┘
       │ No
       ▼
┌─────────────┐
│  Trigger    │
│ Backfill Job│
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 返回 503    │
│ 提示稍后重试 │
└─────────────┘
```

### 3.3 任务队列处理流程

```
┌─────────────────────────────────────────────────────────────┐
│                         Worker Loop                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. 从 Queue 获取一个 Pending Job                    │    │
│  │ 2. 更新状态为 Started                               │    │
│  │ 3. 执行对应的 Processing Step                       │    │
│  │ 4. 将结果写入 Cache (MongoDB)                        │    │
│  │ 5. 更新状态为 Finished / Failed                     │    │
│  │ 6. 将依赖的下游 Job 加入 Queue                       │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 处理图（Processing Graph）

系统使用有向无环图（DAG）定义数据处理步骤之间的依赖关系：

```
Dataset Level (数据集级别):
─────────────────────────
dataset-config-names
    │
    ├──► dataset-info
    │
    ├──► config-parquet-and-info ──► config-parquet-metadata
    │                                    │
    │                                    ├──► split-first-rows
    │                                    │
    │                                    ├──► split-descriptive-statistics
    │                                    │
    │                                    └──► (用于 rows/search 服务)
    │
    ├──► config-size ──► dataset-size
    │
    └──► config-split-names ──► dataset-split-names

Split Level (分割级别):
───────────────────────
split-first-rows (预计算前100行)
split-descriptive-statistics (描述性统计)

Dataset Scan Level (扫描级别):
─────────────────────────────
dataset-opt-in-out-urls (URL 合规检查)
dataset-presidio-entities (PII 实体检测)
```

---

## 5. API 接口详细说明

### 5.1 公共 API（API Service）

基础 URL: `https://datasets-server.huggingface.co`

#### 5.1.1 健康检查

| 端点 | 方法 | 描述 |
|------|------|------|
| `/healthcheck` | GET | 服务健康状态检查 |
| `/metrics` | GET | Prometheus 监控指标 |

#### 5.1.2 数据集信息

##### GET `/splits`
获取数据集的分割（splits）列表。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称，如 `nyu-mll/glue` |
| `config` | string | ✗ | 配置/子集名称 |

**响应示例:**
```json
{
  "splits": [
    {"dataset": "ibm/duorc", "config": "ParaphraseRC", "split": "train"},
    {"dataset": "ibm/duorc", "config": "ParaphraseRC", "split": "validation"},
    {"dataset": "ibm/duorc", "config": "ParaphraseRC", "split": "test"}
  ],
  "pending": [],
  "failed": []
}
```

##### GET `/first-rows`
获取分割的前 100 行数据（预览）。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |
| `config` | string | ✓ | 配置名称 |
| `split` | string | ✓ | 分割名称 |

**响应示例:**
```json
{
  "dataset": "stanfordnlp/imdb",
  "config": "plain_text",
  "split": "train",
  "features": [
    {"feature_idx": 0, "name": "text", "type": {"dtype": "string", "_type": "Value"}},
    {"feature_idx": 1, "name": "label", "type": {"names": ["neg", "pos"], "_type": "ClassLabel"}}
  ],
  "rows": [...],
  "truncated": false
}
```

##### GET `/parquet`
获取数据集的 Parquet 文件列表。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |
| `config` | string | ✗ | 配置名称 |

**响应示例:**
```json
{
  "parquet_files": [
    {
      "dataset": "ibm/duorc",
      "config": "ParaphraseRC",
      "split": "train",
      "url": "https://huggingface.co/datasets/ibm/duorc/resolve/refs%2Fconvert%2Fparquet/ParaphraseRC/train/0000.parquet",
      "filename": "duorc-train.parquet",
      "size": 26005668
    }
  ],
  "partial": false
}
```

##### GET `/info`
获取数据集的元数据信息。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |
| `config` | string | ✗ | 配置名称 |

##### GET `/size`
获取数据集的大小信息。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |
| `config` | string | ✗ | 配置名称 |

##### GET `/is-valid`
检查数据集的有效性（支持的功能）。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |
| `config` | string | ✗ | 配置名称 |
| `split` | string | ✗ | 分割名称 |

**响应示例:**
```json
{
  "preview": true,
  "viewer": true,
  "search": true,
  "filter": true,
  "statistics": true
}
```

##### GET `/statistics`
获取分割列的描述性统计信息。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |
| `config` | string | ✓ | 配置名称 |
| `split` | string | ✓ | 分割名称 |

---

### 5.2 行数据 API（Rows Service）

基础 URL: `http://rows-service:8081`

#### GET `/rows`
获取指定范围的行数据（支持任意偏移量和长度）。

**参数:**
| 参数 | 类型 | 必需 | 默认值 | 描述 |
|------|------|------|--------|------|
| `dataset` | string | ✓ | - | 数据集名称 |
| `config` | string | ✓ | - | 配置名称 |
| `split` | string | ✓ | - | 分割名称 |
| `offset` | integer | ✗ | 0 | 起始行索引 |
| `length` | integer | ✗ | 100 | 返回行数 (最大 100) |

**响应示例:**
```json
{
  "features": [...],
  "rows": [
    {
      "row_idx": 234,
      "row": {"text": "...", "label": 0},
      "truncated_cells": []
    }
  ],
  "num_rows_total": 25000,
  "num_rows_per_page": 100,
  "partial": false
}
```

---

### 5.3 搜索 API（Search Service）

基础 URL: `http://search-service:8082`

#### GET `/search`
在文本列中执行全文搜索。

**参数:**
| 参数 | 类型 | 必需 | 默认值 | 描述 |
|------|------|------|--------|------|
| `dataset` | string | ✓ | - | 数据集名称 |
| `config` | string | ✓ | - | 配置名称 |
| `split` | string | ✓ | - | 分割名称 |
| `query` | string | ✓ | - | 搜索查询词 |
| `offset` | integer | ✗ | 0 | 结果偏移量 |
| `length` | integer | ✗ | 100 | 返回行数 (最大 100) |

**技术实现:**
- 使用 DuckDB 的 FTS（Full Text Search）扩展
- 在查询时动态构建 DuckDB 索引文件
- 支持 BM25 评分排序

#### GET `/filter`
使用 SQL-like 条件过滤数据。

**参数:**
| 参数 | 类型 | 必需 | 默认值 | 描述 |
|------|------|------|--------|------|
| `dataset` | string | ✓ | - | 数据集名称 |
| `config` | string | ✓ | - | 配置名称 |
| `split` | string | ✓ | - | 分割名称 |
| `where` | string | ✓ | - | 过滤条件 |
| `offset` | integer | ✗ | 0 | 结果偏移量 |
| `length` | integer | ✗ | 100 | 返回行数 (最大 100) |

**过滤条件示例:**
```
Age = 30
Sex = 'female'
Fare > 50
Pclass = 2 AND "Siblings/Spouses Aboard" > 0
```

---

### 5.4 管理 API（Admin Service）

基础 URL: `http://admin-service:8083`

#### GET `/healthcheck`
服务健康检查。

#### GET `/dataset-status`
获取数据集的处理状态。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |

**响应:**
```json
{
  "config-names": {
    "cached_responses": [...],
    "jobs": [...]
  },
  "config-parquet-and-info": {...},
  ...
}
```

#### GET `/pending-jobs`
获取所有待处理任务的统计信息。

**响应:**
```json
{
  "config-names": {"waiting": 10, "started": 2, ...},
  "split-first-rows": {...}
}
```

#### POST `/force-refresh/{job_type}`
强制刷新指定类型的任务。

**参数:**
| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `dataset` | string | ✓ | 数据集名称 |
| `config` | string | ✗ | 配置名称 |
| `split` | string | ✗ | 分割名称 |
| `priority` | string | ✗ | 优先级 (low/normal/high) |
| `difficulty` | integer | ✗ | 任务难度 (0-100) |

#### GET `/cache-reports`
获取缓存报告统计。

#### GET `/blocked-datasets`
获取被阻止的数据集列表。

---

## 6. 认证机制

### 6.1 支持的认证方式

1. **Hugging Face Token** (推荐)
   - Header: `Authorization: Bearer hf_xxxxxxxxxxxxxxxx`
   - 支持 User Access Token 和 Organization API Token

2. **JWT Token** (Hub 内部使用)
   - Header: `Authorization: Bearer jwt:xxxxxxxx`
   - 由 Hugging Face Hub 签名

3. **外部认证** (可选配置)
   - 通过 `EXTERNAL_AUTH_URL` 配置外部认证服务

### 6.2 访问权限

| 数据集类型 | 公开访问 | 认证访问 |
|------------|----------|----------|
| 公开数据集 | ✓ | ✓ |
| Gated 数据集 | ✗ | ✓ (需接受条款) |
| 私有数据集 | ✗ | ✓ (PRO 用户或 Enterprise Hub) |

---

## 7. 数据类型支持

### 7.1 特征类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `Value` | 标量值（字符串、数值、布尔等） | `{"dtype": "string", "_type": "Value"}` |
| `ClassLabel` | 分类标签 | `{"names": ["neg", "pos"], "_type": "ClassLabel"}` |
| `Image` | 图像数据 | `{"_type": "Image"}` |
| `Audio` | 音频数据 | `{"sampling_rate": 16000, "_type": "Audio"}` |
| `Video` | 视频数据 | `{"_type": "Video"}` |
| `Pdf` | PDF 文档 | `{"_type": "Pdf"}` |
| `Array2D/3D/4D/5D` | 多维数组 | `{"shape": [28, 28], "_type": "Array2D"}` |
| `Sequence` | 序列 | `{"feature": {...}, "_type": "Sequence"}` |
| `List` | 列表 | `{"feature": {...}, "_type": "List"}` |
| `LargeList` | 大型列表 | `{"feature": {...}, "_type": "LargeList"}` |
| `Dict` | 字典 | `{"key": {...}}` |
| `Translation` | 翻译 | `{"languages": ["en", "fr"], "_type": "Translation"}` |
| `TranslationVariableLanguages` | 可变语言翻译 | `{"_type": "TranslationVariableLanguages"}` |

### 7.2 统计类型

| 类型 | 支持统计 |
|------|----------|
| `float` | min, max, mean, median, std, histogram |
| `int` | min, max, mean, median, std, histogram |
| `class_label` | frequencies, n_unique |
| `string_label` | frequencies, n_unique |
| `string_text` | length histogram |
| `bool` | frequencies |
| `list` | length histogram |
| `audio` | duration histogram |
| `image` | width histogram |
| `datetime` | min, max, mean, median, std, histogram |

### 7.3 代码架构

#### 7.3.1 特征处理架构

特征类型的处理主要位于 `libs/libcommon/src/libcommon/viewer_utils/features.py`：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Feature Processing                           │
│              (libs/libcommon/src/libcommon/viewer_utils/features.py) │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐    ┌─────────────┐    ┌─────────────────────┐   │
│  │  Value Types    │    │ Media Types │    │  Container Types    │   │
│  │  - Value        │    │ - Image     │    │  - List             │   │
│  │  - ClassLabel   │    │ - Audio     │    │  - LargeList        │   │
│  │  - ArrayXD      │    │ - Video     │    │  - Sequence         │   │
│  │  - Translation  │    │ - Pdf       │    │  - Dict             │   │
│  └─────────────────┘    └──────┬──────┘    └─────────────────────┘   │
│                                 │                                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    get_cell_value()                          │    │
│  │         Main dispatch function for type processing          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                 │                                   │
│           ┌─────────────────────┴─────────────────────┐             │
│           ▼                                         ▼             │
│  ┌───────────────────────┐              ┌───────────────────────┐  │
│  │  Media Asset Creation │              │  Nested Processing    │  │
│  │  - create_image_file()│              │  - Recursive list/dict│  │
│  │  - create_audio_file()│              │    handling           │  │
│  │  - create_video_file()│              │  - json_path tracking │  │
│  │  - create_pdf_file()  │              │    for nested items   │  │
│  └───────────────────────┘              └───────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**核心函数说明：**

| 函数 | 路径 | 职责 |
|------|------|------|
| `get_cell_value()` | `features.py:384` | 特征类型分发处理主函数 |
| `image()` | `features.py:82` | 处理 Image 类型，生成缩略图并上传到 S3 |
| `audio()` | `features.py:134` | 处理 Audio 类型，支持格式转换（统一转为 wav/mp3/opus） |
| `video()` | `features.py:255` | 处理 Video 类型，直接存储原始字节或路径 |
| `pdf()` | `features.py:339` | 处理 Pdf 类型，生成缩略图并上传 |
| `to_features_list()` | `features.py:551` | 将 Features 对象转换为列表格式 |

**媒体文件资产结构（`libs/libcommon/src/libcommon/viewer_utils/asset.py`）：**

```python
# 图像资产返回格式
ImageSource: {
    "src": str,      # CDN URL
    "height": int,   # 像素高度
    "width": int     # 像素宽度
}

# 音频资产返回格式
AudioSource: {
    "src": str,      # CDN URL
    "type": str      # MIME type (audio/wav, audio/mpeg, audio/ogg)
}

# 视频资产返回格式
VideoSource: {
    "src": str       # CDN URL or original path
}

# PDF资产返回格式
PDFSource: {
    "src": str,           # CDN URL
    "size_bytes": int,    # 文件大小
    "thumbnail": ImageSource  # 缩略图
}
```

#### 7.3.2 统计计算架构

统计类型处理位于 `libs/libcommon/src/libcommon/statistics_utils.py`，采用面向对象设计：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Statistics Computation                          │
│           (libs/libcommon/src/libcommon/statistics_utils.py)        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      Column (ABC)                             │   │
│  │                   Abstract Base Class                       │   │
│  │  - compute_statistics()  → SupportedStatistics               │   │
│  │  - compute_and_prepare_response() → StatisticsPerColumnItem│   │
│  └────────────────┬────────────────────────────────────────────┘   │
│                   │                                                 │
│    ┌──────────────┼──────────────┬──────────────┬─────────────┐  │
│    │              │              │              │             │  │
│    ▼              ▼              ▼              ▼             ▼  │
│ ┌───────┐    ┌───────┐    ┌───────────┐   ┌───────┐    ┌────────┐│
│ │Float  │    │ Int   │    │ String    │   │ Bool  │    │ List   ││
│ │Column │    │Column │    │ Column    │   │Column │    │Column  ││
│ └───────┘    └───────┘    └───────┬───┘   └───────┘    └────────┘│
│                                   │                                │
│              ┌────────────────────┼────────────────────┐          │
│              │                    │                    │          │
│              ▼                    ▼                    ▼          │
│       ┌──────────┐        ┌──────────┐        ┌──────────┐       │
│       │ ClassLabel│        │ StringLabel│       │ StringText │       │
│       │ Column   │        │ (分类标签)  │        │ (长文本)   │       │
│       └──────────┘        └──────────┘        └──────────┘       │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    MediaColumn (ABC)                        │  │
│  │              媒体类型统计基类（处理Parquet文件）            │  │
│  └────────────────┬────────────────────────────────────────────┘  │
│                   │                                              │
│        ┌──────────┴──────────┐                                   │
│        ▼                     ▼                                   │
│  ┌─────────────┐      ┌─────────────┐                           │
│  │ AudioColumn │      │ ImageColumn │                           │
│  │ (duration)  │      │ (width)     │                           │
│  └─────────────┘      └─────────────┘                           │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**统计类型枚举定义：**

```python
class ColumnType(str, enum.Enum):
    FLOAT = "float"
    INT = "int"
    BOOL = "bool"
    LIST = "list"
    CLASS_LABEL = "class_label"
    STRING_LABEL = "string_label"
    STRING_TEXT = "string_text"
    AUDIO = "audio"
    IMAGE = "image"
    DATETIME = "datetime"
```

**统计项数据结构：**

```python
# 数值统计
NumericalStatisticsItem: {
    "nan_count": int,
    "nan_proportion": float,
    "min": float | int | None,
    "max": float | int | None,
    "mean": float | None,
    "median": float | None,
    "std": float | None,
    "histogram": Histogram | None
}

# 分类统计
CategoricalStatisticsItem: {
    "nan_count": int,
    "nan_proportion": float,
    "no_label_count": int,
    "no_label_proportion": float,
    "n_unique": int,
    "frequencies": dict[str, int]
}

# 布尔统计
BoolStatisticsItem: {
    "nan_count": int,
    "nan_proportion": float,
    "frequencies": dict[str, int]
}

# 日期时间统计
DatetimeStatisticsItem: {
    "nan_count": int,
    "nan_proportion": float,
    "min": str | None,
    "max": str | None,
    "mean": str | None,
    "median": str | None,
    "std": str | None,  # timedelta string
    "histogram": DatetimeHistogram | None
}
```

#### 7.3.3 DTO 定义

数据传输对象定义位于 `libs/libcommon/src/libcommon/dtos.py`：

```python
# 特征项定义
FeatureItem: {
    "feature_idx": int,    # 特征索引
    "name": str,           # 特征名称
    "type": dict[str, Any] # 特征类型定义
}

# 行项定义
RowItem: {
    "row_idx": int,        # 行索引
    "row": dict[str, Any], # 行数据
    "truncated_cells": list[str] # 被截断的列名
}

# 分页响应
PaginatedResponse: {
    "features": list[FeatureItem],
    "rows": list[RowItem],
    "num_rows_total": int,
    "num_rows_per_page": int,
    "partial": bool
}

# 前100行响应
SplitFirstRowsResponse: {
    "dataset": str,
    "config": str,
    "split": str,
    "features": list[FeatureItem],
    "rows": list[RowItem],
    "truncated": bool
}
```

### 7.4 使用接口文档

#### 7.4.1 OpenAPI Schema 定义

完整的数据类型 Schema 定义位于 `docs/source/openapi.json`。

**特征类型 Schema 结构：**

```json
{
  "Feature": {
    "oneOf": [
      {"$ref": "#/components/schemas/ValueFeature"},
      {"$ref": "#/components/schemas/ClassLabelFeature"},
      {"$ref": "#/components/schemas/ArrayXDFeature"},
      {"$ref": "#/components/schemas/TranslationFeature"},
      {"$ref": "#/components/schemas/SequenceFeature"},
      {"$ref": "#/components/schemas/ListFeature"},
      {"$ref": "#/components/schemas/LargeListFeature"},
      {"$ref": "#/components/schemas/DictFeature"},
      {"$ref": "#/components/schemas/AudioFeature"},
      {"$ref": "#/components/schemas/ImageFeature"},
      {"$ref": "#/components/schemas/VideoFeature"},
      {"$ref": "#/components/schemas/PdfFeature"}
    ]
  }
}
```

**单元格值 Schema 结构：**

```json
{
  "Cell": {
    "oneOf": [
      {"$ref": "#/components/schemas/ValueCell"},           // 标量值
      {"$ref": "#/components/schemas/ClassLabelCell"},    // 整数标签
      {"$ref": "#/components/schemas/Array2DCell"},         // 2D数组
      {"$ref": "#/components/schemas/AudioCell"},           // 音频URL列表
      {"$ref": "#/components/schemas/ImageCell"},            // 图像URL+尺寸
      {"$ref": "#/components/schemas/VideoCell"},           // 视频URL
      {"$ref": "#/components/schemas/PdfCell"},             // PDF URL+缩略图
      {"$ref": "#/components/schemas/ListCell"},             // 嵌套列表
      {"$ref": "#/components/schemas/DictCell"}             // 嵌套字典
    ]
  }
}
```

#### 7.4.2 API 响应示例

**Value 类型响应：**

```json
{
  "features": [
    {
      "feature_idx": 0,
      "name": "text",
      "type": {"dtype": "string", "_type": "Value"}
    },
    {
      "feature_idx": 1,
      "name": "label",
      "type": {"dtype": "int64", "_type": "Value"}
    }
  ],
  "rows": [
    {
      "row_idx": 0,
      "row": {
        "text": "Sample text data",
        "label": 1
      },
      "truncated_cells": []
    }
  ]
}
```

**Image 类型响应：**

```json
{
  "features": [
    {
      "feature_idx": 0,
      "name": "image",
      "type": {"_type": "Image"}
    }
  ],
  "rows": [
    {
      "row_idx": 0,
      "row": {
        "image": {
          "src": "https://datasets-server.huggingface.co/assets/.../image.jpg",
          "height": 256,
          "width": 256
        }
      },
      "truncated_cells": []
    }
  ]
}
```

**Audio 类型响应：**

```json
{
  "features": [
    {
      "feature_idx": 0,
      "name": "audio",
      "type": {"sampling_rate": 16000, "_type": "Audio"}
    }
  ],
  "rows": [
    {
      "row_idx": 0,
      "row": {
        "audio": [
          {
            "src": "https://datasets-server.huggingface.co/assets/.../audio.wav",
            "type": "audio/wav"
          }
        ]
      },
      "truncated_cells": []
    }
  ]
}
```

**Nested 类型响应（List/Dict）：**

```json
{
  "features": [
    {
      "feature_idx": 0,
      "name": "answers",
      "type": {
        "feature": {"dtype": "string", "_type": "Value"},
        "_type": "List"
      }
    }
  ],
  "rows": [
    {
      "row_idx": 0,
      "row": {
        "answers": ["Answer 1", "Answer 2", "Answer 3"]
      },
      "truncated_cells": []
    }
  ]
}
```

#### 7.4.3 统计 API 响应示例

**数值列统计：**

```json
{
  "column_name": "alcohol",
  "column_type": "float",
  "column_statistics": {
    "nan_count": 0,
    "nan_proportion": 0.0,
    "min": 8.0,
    "max": 14.9,
    "mean": 10.4918,
    "median": 10.3,
    "std": 1.19271,
    "histogram": {
      "hist": [40, 1133, 1662, 1156, 1092, 628, 569, 175, 41, 1],
      "bin_edges": [8.0, 8.69, 9.38, 10.07, 10.76, 11.45, 12.14, 12.83, 13.52, 14.21, 14.9]
    }
  }
}
```

**分类列统计：**

```json
{
  "column_name": "label",
  "column_type": "class_label",
  "column_statistics": {
    "nan_count": 0,
    "nan_proportion": 0.0,
    "no_label_count": 0,
    "no_label_proportion": 0.0,
    "n_unique": 2,
    "frequencies": {"red": 1599, "white": 4898}
  }
}
```

**字符串列统计（长文本）：**

```json
{
  "column_name": "text",
  "column_type": "string_text",
  "column_statistics": {
    "nan_count": 0,
    "nan_proportion": 0.0,
    "min": 11,
    "max": 296,
    "mean": 97.46649,
    "median": 88.0,
    "std": 55.82714,
    "histogram": {
      "hist": [171, 224, 235, 180, 102, 99, 53, 28, 10, 2],
      "bin_edges": [11, 40, 69, 98, 127, 156, 185, 214, 243, 272, 296]
    }
  }
}
```

**日期时间列统计：**

```json
{
  "column_name": "charttime",
  "column_type": "datetime",
  "column_statistics": {
    "nan_count": 0,
    "nan_proportion": 0.0,
    "min": "2110-01-13 09:39:00",
    "max": "2214-07-26 08:00:00",
    "mean": "2153-03-20 23:15:24",
    "median": "2153-01-19 04:19:30",
    "std": "8691 days, 20:22:21",
    "histogram": {
      "hist": [644662, 824869, 883173, 884980, 861445, 863916, 838647, 664347, 156213, 30922],
      "bin_edges": ["2110-01-13 09:39:00", "2120-06-27 07:05:07", ...]
    }
  }
}
```

---

## 8. 错误处理

### 8.1 HTTP 状态码

| 状态码 | 含义 | 场景 |
|--------|------|------|
| 200 | 成功 | 请求成功完成 |
| 401 | 未认证 | 需要认证才能访问 |
| 404 | 未找到 | 数据集/配置/分割不存在 |
| 422 | 参数错误 | 缺少必需参数或参数无效 |
| 500 | 服务器错误 | 内部错误或响应尚未准备好 |
| 501 | 未实现 | 数据集被阻止或功能不支持 |

### 8.2 错误响应格式

```json
{
  "error": "错误描述",
  "cause_exception": "原始异常类型",
  "cause_message": "原始错误信息",
  "cause_traceback": ["堆栈跟踪..."]
}
```

### 8.3 错误码 Header

响应中包含 `X-Error-Code` header，可能的值：
- `ExternalUnauthenticatedError` - 外部认证失败（未认证用户）
- `ExternalAuthenticatedError` - 外部认证失败（已认证用户）
- `ResponseNotFound` - 响应未找到
- `MissingRequiredParameter` - 缺少必需参数
- `ResponseNotReadyError` - 响应尚未准备好
- `DatasetInBlockListError` - 数据集在阻止列表中
- `DatasetWithTooManyConfigsError` - 数据集配置过多
- `UnexpectedError` - 意外错误

---

## 9. 性能优化

### 9.1 缓存策略

| 缓存层级 | 存储 | TTL | 说明 |
|----------|------|-----|------|
| CDN | CloudFront | 长期 | 静态资源和资产文件 |
| MongoDB | Cache 集合 | 永久 | 预计算的处理结果 |
| DuckDB 索引 | EFS | 配置过期时间 | 搜索和过滤的索引文件 |
| Parquet 元数据 | EFS | 会话级 | 行数据服务的元数据索引 |

### 9.2 限流策略

- 特定大数据集（如 `HuggingFaceFW/fineweb-edu-score-2`）的偏移量限制
- 每页最大行数：100
- 搜索/过滤服务的索引文件自动清理

---

## 10. 部署架构

### 10.1 环境

| 环境 | URL | 部署方式 |
|------|-----|----------|
| Production | https://datasets-server.huggingface.co | Helm / Kubernetes |
| Development | https://datasets-server.us.dev.moon.huggingface.tech | Helm / Kubernetes |
| Local | http://localhost:8100 | Docker Compose |

### 10.2 Docker Compose 服务

```yaml
services:
  - mongodb          # 数据库
  - reverse-proxy    # Nginx 反向代理
  - api              # 公共 API 服务
  - rows             # 行数据服务
  - search           # 搜索服务
  - admin            # 管理服务
  - webhook          # Webhook 服务
  - worker           # Worker 服务（多个实例）
```

---

## 11. 开发与测试

### 11.1 本地开发

```bash
# 安装依赖
make install

# 启动服务
make start

# 开发模式（热重载）
make dev-start

# 运行测试
make test

# 运行 E2E 测试
make e2e
```

### 11.2 项目结构

```
.
├── services/
│   ├── api/           # 公共 API 服务
│   ├── rows/          # 行数据服务
│   ├── search/        # 搜索服务
│   ├── admin/         # 管理服务
│   ├── webhook/       # Webhook 服务
│   ├── worker/        # 后台任务处理
│   ├── sse-api/       # Server-Sent Events API
│   └── reverse-proxy/ # Nginx 配置
├── libs/
│   ├── libcommon/     # 通用库
│   ├── libapi/        # API 工具库
│   └── libviewer/     # Rust 数据集处理库
├── jobs/
│   ├── cache_maintenance/      # 缓存维护任务
│   └── mongodb_migration/      # 数据库迁移任务
├── e2e/               # 端到端测试
├── front/
│   └── admin_ui/      # 管理界面
└── docs/              # 文档
```

---

## 12. 参考资料

- [Hugging Face Dataset Viewer 文档](https://huggingface.co/docs/dataset-viewer)
- [OpenAPI 规范](docs/source/openapi.json)
- [开发者指南](DEVELOPER_GUIDE.md)
- [架构图](architecture.png)

---

*文档生成时间: 2026-03-04*
*版本: 1.0*
