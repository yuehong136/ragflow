# RAGFlow 多数据源同步功能实现深度解析

## 🎯 架构概览

RAGFlow的多数据源同步功能采用**插件化架构 + 异步任务调度**的设计模式，能够从10+个第三方平台自动同步数据到知识库。这是一个企业级的、可扩展的数据集成解决方案。

### 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                    前端UI层 (Web)                            │
│         创建连接器 → 配置凭证 → 关联知识库                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   API层 (Flask)                              │
│  connector_app.py → connector_service.py                     │
│  • 连接器CRUD管理                                            │
│  • 任务调度和状态管理                                         │
│  • 凭证验证                                                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              任务调度层 (sync_data_source.py)                 │
│  • 基于Trio异步框架                                          │
│  • 并发任务控制 (Semaphore)                                  │
│  • 定时轮询和增量同步                                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│           连接器层 (common/data_source/)                      │
│  ┌──────────────┬──────────────┬──────────────┐             │
│  │ Confluence   │   Slack      │   Notion     │             │
│  │ Connector    │   Connector  │   Connector  │ ...         │
│  └──────────────┴──────────────┴──────────────┘             │
│  • 实现统一接口                                              │
│  • 处理API认证                                               │
│  • 数据获取和转换                                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              文档处理层 (RAG Pipeline)                        │
│  • 去重检测                                                  │
│  • 文件解析 (PDF, Markdown, HTML等)                          │
│  • 分块 (Chunking)                                           │
│  • 向量化 (Embedding)                                        │
│  • 存储到向量数据库                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 📐 核心接口设计

### 1. 统一连接器接口

所有数据源连接器必须实现以下接口之一：

```python
# common/data_source/interfaces.py

class CheckpointedConnector(BaseConnector[CT]):
    """基于检查点的连接器 - 支持增量同步"""

    @abstractmethod
    def load_from_checkpoint(
        self,
        start: SecondsSinceUnixEpoch,      # 开始时间戳
        end: SecondsSinceUnixEpoch,        # 结束时间戳
        checkpoint: CT,                     # 上次同步的检查点
    ) -> CheckpointOutput[CT]:
        """
        从检查点加载文档，返回生成器
        - yield Document: 成功获取的文档
        - yield ConnectorFailure: 失败的文档
        - return CT: 新的检查点
        """
        raise NotImplementedError

    @abstractmethod
    def build_dummy_checkpoint(self) -> CT:
        """构建初始检查点（首次同步时）"""
        raise NotImplementedError
```

**关键设计理念**：
- **增量同步**：通过checkpoint记录上次同步状态，只获取新增/修改的数据
- **失败容错**：可以返回`ConnectorFailure`而不中断整个同步流程
- **断点续传**：checkpoint存储在数据库中，支持中断后继续

### 2. 凭证管理接口

```python
class CredentialsProviderInterface(abc.ABC):
    """凭证提供者接口 - 支持动态凭证刷新"""

    @abstractmethod
    def get_credentials(self) -> dict[str, Any]:
        """获取认证凭证（可能包含OAuth token等）"""
        raise NotImplementedError

    @abstractmethod
    def set_credentials(self, credential_json: dict[str, Any]) -> None:
        """更新凭证（如刷新OAuth token）"""
        raise NotImplementedError

    @abstractmethod
    def is_dynamic(self) -> bool:
        """
        是否为动态凭证
        - True: 需要定期刷新（OAuth）
        - False: 静态凭证（API Key）
        """
        raise NotImplementedError
```

**支持的认证方式**：
- OAuth 2.0 (Confluence Cloud, Slack, Google Drive等)
- API Token (Confluence Server, Notion等)
- Service Account (Gmail, Google Drive等)

---

## 🔄 同步流程详解

### 第1步：连接器配置与初始化

**用户操作**（前端）：
```javascript
// 1. 创建连接器
POST /api/connector
{
  "name": "公司Confluence知识库",
  "source": "confluence",
  "config": {
    "wiki_base": "https://company.atlassian.net",
    "username": "user@company.com",
    "access_token": "xxxxxxx",
    "space": "TECH",
    "is_cloud": true
  },
  "refresh_freq": 60,  // 每60分钟同步一次
  "timeout_secs": 3600
}

// 2. 关联到知识库
POST /api/connector/{connector_id}/kb/{kb_id}
```

**数据库记录**：
```sql
-- connector表
INSERT INTO connector VALUES (
  id='conn_123',
  tenant_id='tenant_456',
  name='公司Confluence知识库',
  source='confluence',
  input_type='poll',  -- 轮询模式
  config='{...}',     -- 加密存储的配置
  refresh_freq=60,
  status='schedule'
);

-- connector2kb表（多对多关系）
INSERT INTO connector2kb VALUES (
  connector_id='conn_123',
  kb_id='kb_789'
);

-- sync_logs表（同步任务记录）
INSERT INTO sync_logs VALUES (
  id='sync_001',
  connector_id='conn_123',
  kb_id='kb_789',
  status='schedule',       -- 等待调度
  from_beginning='1',      -- 首次同步，全量拉取
  poll_range_start=NULL
);
```

### 第2步：任务调度系统

**核心代码**：`rag/svr/sync_data_source.py`

```python
# 异步任务调度器（基于Trio框架）
async def dispatch_tasks():
    async with trio.open_nursery() as nursery:
        # 查询所有待执行的同步任务
        for task in SyncLogsService.list_sync_tasks():
            # 获取对应的连接器工厂
            func = func_factory[task["source"]](task["config"])
            # 并发启动任务（受信号量限制）
            nursery.start_soon(func, task)
    await trio.sleep(1)

# 主循环
async def main():
    while not stop_event.is_set():
        await dispatch_tasks()
```

**任务查询SQL**：
```sql
SELECT
    sl.id, sl.connector_id, sl.kb_id,
    sl.poll_range_start, sl.poll_range_end,
    c.name, c.source, c.tenant_id, c.timeout_secs, c.config,
    kb.name as kb_name
FROM sync_logs sl
JOIN connector c ON sl.connector_id = c.id
JOIN connector2kb c2k ON sl.kb_id = c2k.kb_id
JOIN knowledgebase kb ON sl.kb_id = kb.id
WHERE
    c.input_type = 'poll'
    AND c.status = 'schedule'
    AND sl.status = 'schedule'
    AND sl.update_date < (NOW() - INTERVAL c.refresh_freq MINUTE)
ORDER BY sl.update_time DESC;
```

**调度逻辑**：
- 每个连接器根据`refresh_freq`定期触发
- 使用`Semaphore`限制并发数（默认5个）
- 支持超时控制（`timeout_secs`）

### 第3步：Confluence连接器实现（示例）

**代码位置**：`common/data_source/confluence_connector.py`

```python
class ConfluenceConnector(CheckpointedConnector):
    def __init__(self, wiki_base: str, space: str, is_cloud: bool):
        self.wiki_base = wiki_base
        self.space = space
        self.is_cloud = is_cloud
        self._credentials_provider = None

    def set_credentials_provider(self, provider):
        self._credentials_provider = provider

    def load_from_checkpoint(
        self,
        start: float,  # Unix时间戳
        end: float,
        checkpoint: ConnectorCheckpoint
    ):
        """从Confluence获取文档"""

        # 1. 获取凭证
        with self._credentials_provider as cred:
            username = cred.get_credentials()["confluence_username"]
            token = cred.get_credentials()["confluence_access_token"]

        # 2. 构建API客户端
        auth = (username, token)
        base_url = f"{self.wiki_base}/rest/api/content"

        # 3. 构建CQL查询（Confluence Query Language）
        # 只查询在时间范围内更新的页面
        cql = f"space={self.space} AND lastModified >= {start_date}"
        if not checkpoint.has_more:
            cql += f" AND lastModified <= {end_date}"

        # 4. 分页获取页面列表
        params = {"cql": cql, "limit": 100, "start": 0}

        while True:
            response = requests.get(base_url, auth=auth, params=params)
            response.raise_for_status()
            data = response.json()

            # 5. 处理每个页面
            for page in data["results"]:
                try:
                    # 获取页面完整内容
                    page_detail = self._get_page_content(page["id"], auth)

                    # 提取文本内容（HTML -> 纯文本）
                    html_content = page_detail["body"]["storage"]["value"]
                    text_content = html_to_text(html_content)

                    # 处理附件
                    attachments = self._get_attachments(page["id"], auth)

                    # 构建Document对象
                    doc = Document(
                        id=build_confluence_document_id(
                            self.wiki_base,
                            page["_links"]["webui"],
                            self.is_cloud
                        ),
                        source="confluence",
                        semantic_identifier=page["title"],
                        extension=".html",
                        blob=text_content.encode('utf-8'),
                        doc_updated_at=datetime.fromisoformat(
                            page["version"]["when"]
                        ),
                        size_bytes=len(text_content)
                    )

                    yield doc

                except Exception as e:
                    # 返回失败记录而不中断流程
                    yield ConnectorFailure(
                        failed_document=DocumentFailure(
                            document_id=page["id"],
                            document_link=page["_links"]["webui"]
                        ),
                        failure_message=str(e)
                    )

            # 6. 检查是否还有更多页面
            if "next" not in data["_links"]:
                break
            params["start"] += 100

        # 7. 返回新的检查点
        new_checkpoint = ConnectorCheckpoint(has_more=False)
        return new_checkpoint
```

**关键技术点**：
1. **CQL查询**：使用Confluence的查询语言过滤时间范围
2. **分页处理**：每次获取100个页面，避免内存溢出
3. **HTML清洗**：将HTML内容转换为纯文本（`html_to_text`）
4. **附件处理**：下载并提取PDF、Word等附件内容
5. **错误处理**：单个页面失败不影响整体流程

### 第4步：文档处理与去重

**代码位置**：`api/db/services/connector_service.py`

```python
class SyncLogsService:
    @classmethod
    def duplicate_and_parse(cls, kb, docs, tenant_id, source_prefix):
        """
        文档去重和解析

        Args:
            kb: 知识库对象
            docs: 文档列表
            tenant_id: 租户ID
            source_prefix: 来源前缀（如 "confluence/conn_123"）
        """
        errors = []
        document_ids = []

        for doc in docs:
            try:
                # 1. 计算文档指纹（MD5）
                doc_id = doc["id"]
                blob_hash = hashlib.md5(doc["blob"]).hexdigest()

                # 2. 查询数据库是否已存在
                existing = DocumentService.query(
                    kb_id=kb.id,
                    connector_id=doc["connector_id"],
                    source_id=doc_id
                )

                # 3. 去重逻辑
                if existing:
                    # 检查内容是否变化
                    if existing[0].blob_hash == blob_hash:
                        logging.info(f"文档未变化，跳过: {doc_id}")
                        continue
                    else:
                        # 内容有更新，删除旧版本
                        DocumentService.delete_by_id(existing[0].id)
                        logging.info(f"检测到更新，重新解析: {doc_id}")

                # 4. 保存到文件系统
                file_path = f"{source_prefix}/{doc_id}{doc['extension']}"
                FileService.save(
                    tenant_id=tenant_id,
                    file_path=file_path,
                    blob=doc["blob"]
                )

                # 5. 创建文档记录
                doc_record = DocumentService.insert({
                    "id": get_uuid(),
                    "kb_id": kb.id,
                    "connector_id": doc["connector_id"],
                    "source_id": doc_id,
                    "name": doc["semantic_identifier"],
                    "location": file_path,
                    "size": doc["size_bytes"],
                    "blob_hash": blob_hash,
                    "parser_id": kb.parser_id,  # 使用知识库的默认解析器
                    "status": "pending"
                })

                # 6. 触发异步解析任务
                TaskService.create_parsing_task(doc_record.id)

                document_ids.append(doc_record.id)

            except Exception as e:
                errors.append(f"{doc_id}: {str(e)}")

        return errors, document_ids
```

**去重策略**：
- **基于MD5哈希**：计算文档内容的指纹
- **基于source_id**：同一连接器的同一文档ID视为同一份文档
- **更新检测**：如果内容哈希不同，删除旧版本并重新解析

### 第5步：异步解析流程

```python
# rag/svr/task_executor.py

def parse_document(doc_id):
    """异步文档解析任务"""
    doc = DocumentService.get_by_id(doc_id)
    kb = KnowledgebaseService.get_by_id(doc.kb_id)

    # 1. 根据文件类型选择解析器
    parser = get_parser(doc.parser_id, doc.extension)
    # 可能的解析器: PDFParser, MarkdownParser, HTMLParser等

    # 2. 解析文档提取文本
    chunks = parser.parse(doc.location)

    # 3. 分块处理
    tokenizer = get_tokenizer(kb.chunk_method)
    text_chunks = tokenizer.split(chunks, kb.chunk_size)

    # 4. 生成嵌入向量
    embedding_model = get_embedding_model(kb.embd_id)
    for chunk in text_chunks:
        chunk["embedding"] = embedding_model.encode(chunk["text"])

    # 5. 存储到向量数据库
    vector_db = get_vector_db()  # Elasticsearch/Infinity
    vector_db.bulk_insert(kb.id, text_chunks)

    # 6. 更新文档状态
    DocumentService.update_by_id(doc_id, {"status": "success"})
```

---

## 🔍 Slack连接器实现要点

**代码位置**：`common/data_source/slack_connector.py`

```python
class SlackConnector(CheckpointedConnector):
    def load_from_checkpoint(self, start, end, checkpoint):
        client = WebClient(token=self.bot_token)

        # 1. 获取所有频道
        channels = get_channels(
            client,
            get_public=True,
            get_private=True
        )

        for channel in channels:
            # 2. 获取频道内的消息
            for messages in get_channel_messages(
                client, channel,
                oldest=str(start),
                latest=str(end)
            ):
                # 3. 过滤消息（排除bot消息等）
                for msg in messages:
                    if default_msg_filter(msg):
                        continue

                    # 4. 如果是线程，获取完整对话
                    if "thread_ts" in msg:
                        thread = get_thread(
                            client,
                            channel["id"],
                            msg["thread_ts"]
                        )
                        # 将线程转换为文档
                        doc = thread_to_doc(channel, thread, ...)
                        yield doc
                    else:
                        # 单条消息
                        doc = message_to_doc(channel, msg, ...)
                        yield doc

        return ConnectorCheckpoint(has_more=False)
```

**特殊处理**：
- **线程聚合**：将Slack的thread合并为一个文档
- **Bot过滤**：自动排除机器人消息
- **频道权限**：自动加入频道以获取历史消息

---

## 📊 数据模型

### 核心表结构

```sql
-- 连接器表
CREATE TABLE connector (
    id VARCHAR(32) PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    name VARCHAR(128) NOT NULL,
    source VARCHAR(128) NOT NULL,        -- confluence/slack/notion等
    input_type VARCHAR(128) NOT NULL,    -- poll/event
    config JSON NOT NULL,                 -- 加密的配置（凭证、URL等）
    refresh_freq INT DEFAULT 0,           -- 同步频率（分钟）
    timeout_secs INT DEFAULT 3600,
    status VARCHAR(16) DEFAULT 'schedule',
    INDEX idx_tenant (tenant_id),
    INDEX idx_source (source)
);

-- 连接器-知识库关联表（多对多）
CREATE TABLE connector2kb (
    id VARCHAR(32) PRIMARY KEY,
    connector_id VARCHAR(32) NOT NULL,
    kb_id VARCHAR(32) NOT NULL,
    INDEX idx_connector (connector_id),
    INDEX idx_kb (kb_id)
);

-- 同步日志表（任务记录）
CREATE TABLE sync_logs (
    id VARCHAR(32) PRIMARY KEY,
    connector_id VARCHAR(32) NOT NULL,
    kb_id VARCHAR(32) NOT NULL,
    status VARCHAR(128) NOT NULL,         -- schedule/running/done/fail
    from_beginning CHAR(1) DEFAULT '0',   -- 是否全量同步
    new_docs_indexed INT DEFAULT 0,       -- 本次新增文档数
    total_docs_indexed INT DEFAULT 0,     -- 累计文档数
    error_msg TEXT,
    error_count INT DEFAULT 0,
    time_started DATETIME,
    poll_range_start VARCHAR(255),        -- 同步开始时间（ISO格式）
    poll_range_end VARCHAR(255),          -- 同步结束时间
    INDEX idx_connector (connector_id),
    INDEX idx_kb (kb_id),
    INDEX idx_status (status)
);

-- 文档表（扩展）
ALTER TABLE document ADD COLUMN connector_id VARCHAR(32);
ALTER TABLE document ADD COLUMN source_id VARCHAR(512);  -- 外部系统的文档ID
ALTER TABLE document ADD COLUMN blob_hash VARCHAR(32);   -- 用于去重
```

### Document数据模型

```python
# common/data_source/models.py

class Document(BaseModel):
    """标准化文档模型"""
    id: str                    # 文档唯一标识（URL或组合ID）
    source: str                # 来源系统（confluence/slack等）
    semantic_identifier: str   # 人类可读的标识（标题/频道名等）
    extension: str             # 文件扩展名
    blob: bytes                # 文档内容（二进制）
    doc_updated_at: datetime   # 文档更新时间
    size_bytes: int            # 文档大小
```

---

## 🛠️ 关键技术实现

### 1. 增量同步（Checkpoint机制）

```python
# 首次同步
checkpoint = connector.build_dummy_checkpoint()
# ConnectorCheckpoint(has_more=True)

# 后续增量同步
last_sync_time = task["poll_range_start"].timestamp()
current_time = datetime.now(timezone.utc).timestamp()

for doc in connector.load_from_checkpoint(
    start=last_sync_time,
    end=current_time,
    checkpoint=checkpoint
):
    # 处理文档...
    pass

# 保存新的时间点
SyncLogsService.update(task["id"], {
    "poll_range_start": current_time
})
```

### 2. 并发控制

```python
# 使用Trio的Semaphore限制并发
MAX_CONCURRENT_TASKS = 5
task_limiter = trio.Semaphore(MAX_CONCURRENT_TASKS)

async def run_sync_task(task):
    async with task_limiter:
        # 最多同时运行5个同步任务
        with trio.fail_after(task["timeout_secs"]):
            await sync_logic(task)
```

### 3. OAuth Token刷新

```python
class OAuthCredentialsProvider(CredentialsProviderInterface):
    def get_credentials(self):
        # 检查token是否过期
        if self._is_token_expired():
            self._refresh_token()
        return self._credentials

    def _refresh_token(self):
        # 使用refresh_token获取新的access_token
        response = requests.post(
            "https://auth.atlassian.com/oauth/token",
            data={
                "grant_type": "refresh_token",
                "refresh_token": self._refresh_token,
                "client_id": OAUTH_CLIENT_ID,
                "client_secret": OAUTH_CLIENT_SECRET
            }
        )
        new_tokens = response.json()
        self._credentials["access_token"] = new_tokens["access_token"]
        # 保存到数据库
        self._save_to_db()
```

### 4. 错误处理与重试

```python
# 速率限制处理
def _handle_http_error(e: requests.HTTPError, attempt: int):
    if e.response.status_code == 429:
        # 从响应头读取重试时间
        retry_after = int(e.response.headers.get("Retry-After", 60))
        logging.warning(f"速率限制，等待 {retry_after} 秒...")
        time.sleep(retry_after)
        return retry_after

    # 403错误（Confluence Server的速率限制）
    if e.response.status_code == 403:
        time.sleep(10)
        return 10

    raise e

# 装饰器应用
@retry_with_backoff(max_attempts=7)
def fetch_confluence_page(page_id):
    response = requests.get(url, auth=auth)
    response.raise_for_status()
    return response.json()
```

### 5. HTML内容清洗

```python
# common/data_source/html_utils.py

def html_to_text(html_content: str) -> str:
    """将HTML转换为纯文本"""
    # 1. 使用BeautifulSoup解析
    soup = BeautifulSoup(html_content, "html.parser")

    # 2. 移除script和style标签
    for tag in soup(["script", "style"]):
        tag.decompose()

    # 3. 处理链接
    if TRANSFORM_LINKS_STRATEGY == "markdown":
        for link in soup.find_all("a"):
            link.string = f"[{link.text}]({link.get('href')})"
    elif TRANSFORM_LINKS_STRATEGY == "strip":
        for link in soup.find_all("a"):
            link.unwrap()

    # 4. 提取纯文本
    text = soup.get_text(separator="\n", strip=True)

    # 5. 清理多余空行
    text = re.sub(r'\n{3,}', '\n\n', text)

    return text
```

---

## 🔐 安全性设计

### 1. 凭证加密存储

```python
# 配置存储时加密
config_json = {
    "wiki_base": "https://company.atlassian.net",
    "username": "user@company.com",
    "access_token": "secret_token"
}

# 使用AES加密
encrypted_config = encrypt_config(config_json, tenant_key)

# 存入数据库
connector.config = encrypted_config
connector.save()
```

### 2. 租户隔离

```python
# 所有查询都必须带上tenant_id
connectors = Connector.select().where(
    Connector.tenant_id == current_tenant_id,
    Connector.id == connector_id
)

# 防止跨租户访问
if not connectors:
    raise PermissionError("无权访问此连接器")
```

### 3. 最小权限原则

```markdown
**Confluence所需权限**：
- `read:page:confluence` - 读取页面
- `read:space:confluence` - 读取空间

**Slack所需权限**：
- `channels:read` - 读取公开频道
- `channels:history` - 读取频道历史
- `groups:read` - 读取私有频道（可选）

**避免请求的权限**：
- ❌ `write` 权限
- ❌ `admin` 权限
```

---

## 📈 性能优化

### 1. 分页获取

```python
# 避免一次性加载所有数据
def get_all_pages(client, space):
    start = 0
    limit = 100

    while True:
        response = client.get_pages(
            space=space,
            start=start,
            limit=limit
        )

        yield response["results"]

        if len(response["results"]) < limit:
            break
        start += limit
```

### 2. 批量处理

```python
# 批量插入向量数据库
BATCH_SIZE = 100
chunk_buffer = []

for chunk in all_chunks:
    chunk_buffer.append(chunk)

    if len(chunk_buffer) >= BATCH_SIZE:
        vector_db.bulk_insert(chunk_buffer)
        chunk_buffer.clear()

# 处理剩余
if chunk_buffer:
    vector_db.bulk_insert(chunk_buffer)
```

### 3. 缓存用户信息

```python
# Slack连接器缓存用户信息
user_cache = {}  # user_id -> BasicExpertInfo

def expert_info_from_slack_id(user_id, client, user_cache):
    if user_id in user_cache:
        return user_cache[user_id]

    # 调用Slack API获取用户信息
    user_info = client.users_info(user=user_id)
    expert = BasicExpertInfo(
        email=user_info["profile"]["email"],
        display_name=user_info["real_name"]
    )

    user_cache[user_id] = expert
    return expert
```

---

## 🎛️ 配置项

### 环境变量

```bash
# 连接器配置
MAX_CONCURRENT_TASKS=5                      # 最大并发同步任务数
CONTINUE_ON_CONNECTOR_FAILURE=true          # 单个文档失败是否继续

# Confluence配置
CONFLUENCE_CONNECTOR_LABELS_TO_SKIP=draft,archive
CONFLUENCE_CONNECTOR_INDEX_ARCHIVED_PAGES=false
CONFLUENCE_CONNECTOR_ATTACHMENT_SIZE_THRESHOLD=10485760  # 10MB
CONFLUENCE_TIMEZONE_OFFSET=8                # 时区偏移（小时）

# Notion配置
NOTION_CONNECTOR_DISABLE_RECURSIVE_PAGE_LOOKUP=false

# OAuth配置
OAUTH_CONFLUENCE_CLOUD_CLIENT_ID=xxx
OAUTH_CONFLUENCE_CLOUD_CLIENT_SECRET=xxx
OAUTH_SLACK_CLIENT_ID=xxx
OAUTH_SLACK_CLIENT_SECRET=xxx
```

---

## 🧪 测试与监控

### 同步状态监控

```python
# 查询同步历史
GET /api/connector/{connector_id}/sync_logs

# 响应示例
{
  "logs": [
    {
      "id": "sync_001",
      "status": "done",
      "new_docs_indexed": 25,
      "total_docs_indexed": 125,
      "error_count": 2,
      "error_msg": "Page 123: 403 Forbidden\nPage 456: Timeout",
      "time_started": "2025-11-05T10:00:00Z",
      "poll_range_start": "2025-11-04T10:00:00Z",
      "poll_range_end": "2025-11-05T10:00:00Z"
    }
  ]
}
```

### 手动触发同步

```python
# API端点
POST /api/connector/{connector_id}/sync
{
  "kb_id": "kb_789",
  "reindex": false  # true=全量同步，false=增量同步
}

# 后端实现
def trigger_sync(connector_id, kb_id, reindex=False):
    SyncLogsService.schedule(
        connector_id=connector_id,
        kb_id=kb_id,
        poll_range_start=None if reindex else last_sync_time,
        reindex=reindex
    )
```

---

## 🚀 扩展新数据源指南

### 步骤1：创建连接器类

```python
# common/data_source/your_connector.py

from common.data_source.interfaces import CheckpointedConnector
from common.data_source.models import Document, ConnectorCheckpoint

class YourConnector(CheckpointedConnector):
    def __init__(self, api_url: str, api_key: str):
        self.api_url = api_url
        self.api_key = api_key

    def load_credentials(self, credentials):
        # 加载凭证
        self.api_key = credentials["api_key"]
        return credentials

    def load_from_checkpoint(self, start, end, checkpoint):
        # 实现数据获取逻辑
        # 1. 调用API获取数据
        # 2. 转换为Document对象
        # 3. yield documents
        # 4. return new checkpoint
        pass

    def build_dummy_checkpoint(self):
        return ConnectorCheckpoint(has_more=True)
```

### 步骤2：注册到工厂

```python
# rag/svr/sync_data_source.py

class YourSource(SyncBase):
    async def _run(self, task: dict):
        connector = YourConnector(
            api_url=self.conf["api_url"],
            api_key=self.conf["api_key"]
        )
        # ... 实现同步逻辑
        pass

# 注册
func_factory = {
    FileSource.YOUR_SOURCE: YourSource,
    # ... 其他连接器
}
```

### 步骤3：添加前端UI

```typescript
// web/src/pages/connector/your-source-form.tsx

export const YourSourceForm = () => {
  return (
    <Form>
      <FormItem label="API URL">
        <Input name="api_url" />
      </FormItem>
      <FormItem label="API Key">
        <Input.Password name="api_key" />
      </FormItem>
    </Form>
  );
};
```

---

## 💡 最佳实践

### 1. 错误处理

```python
# ✅ 推荐：单个文档失败不影响整体
try:
    doc = process_page(page)
    yield doc
except Exception as e:
    yield ConnectorFailure(
        failed_document=DocumentFailure(...),
        failure_message=str(e)
    )

# ❌ 避免：直接抛出异常中断同步
doc = process_page(page)  # 可能导致整个同步失败
yield doc
```

### 2. 时间范围查询

```python
# ✅ 推荐：使用外部系统的时间过滤
cql = f"lastModified >= {start_date} AND lastModified < {end_date}"
pages = confluence.search(cql)

# ❌ 避免：获取所有数据后在本地过滤
all_pages = confluence.get_all_pages()
filtered = [p for p in all_pages if start <= p.updated_at < end]
```

### 3. 大文件处理

```python
# ✅ 推荐：流式下载大文件
with requests.get(url, stream=True) as r:
    with open(file_path, 'wb') as f:
        for chunk in r.iter_content(chunk_size=8192):
            f.write(chunk)

# ❌ 避免：一次性加载到内存
content = requests.get(url).content  # 可能OOM
```

---

## 📚 总结

RAGFlow的多数据源同步功能是一个**企业级的、可扩展的、容错性强**的数据集成系统，核心特点：

1. **插件化架构**：统一接口，轻松扩展新数据源
2. **增量同步**：基于checkpoint机制，只同步变化的数据
3. **异步并发**：Trio框架 + Semaphore并发控制
4. **容错机制**：单个文档失败不影响整体，详细的错误日志
5. **安全性**：凭证加密、租户隔离、最小权限
6. **性能优化**：分页获取、批量处理、智能缓存

这套架构已成功支持：
- Confluence (2,030行代码)
- Slack (670行代码)
- Notion, Gmail, Google Drive, Jira等

为企业构建知识库提供了强大的数据来源支持！
