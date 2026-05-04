# 心悦搜索短剧资源抓取程序计划

## 一、目标

开发一个 Python 命令行程序，用户输入一个或多个短剧关键词后，程序自动访问 `xinyueso.com` 的搜索接口，获取搜索结果，并输出：

- 关键词
- 短剧名称
- 资源地址
- 短剧对应日期
- 网盘来源类型，例如夸克网盘、百度网盘、UC 网盘

程序需要保留搜索到的所有短剧结果，但每个短剧结果只保存一个最终资源链接。选择规则：

1. 对搜索结果按短剧名称归并，认为同一个短剧可能有多个网盘来源。
2. 如果同一个短剧有夸克网盘链接，优先保存夸克网盘链接。
3. 如果同一个短剧没有夸克网盘链接，但有百度网盘链接，保存百度网盘链接。
4. 如果同一个短剧夸克和百度都没有，保存其他来源中排序最靠前的一条。
5. 对未被选中的候选结果，不再调用二次解析接口，减少无效请求。

保存前必须通过 MySQL 连接池检查数据库：

1. 如果本地数据库中已经存在同名短剧资源，则跳过保存。
2. 如果本地数据库中不存在该短剧资源，才写入数据库。
3. 数据库检查应发生在解析并准备保存最终优先链接之后、实际插入之前。
4. 跳过保存时应在输出中标记状态，方便用户知道该资源已存在。

程序应支持三种输出格式：

- 普通文本
- JSON
- CSV

同时支持通过 MySQL 8.0 连接池写入数据库，用于去重和持久化保存。

## 二、调研结论

`xinyueso.com` 的搜索流程不是普通 HTML 表单提交，而是前端通过 SSE 接口获取搜索结果。

搜索接口：

```text
GET https://www.xinyueso.com/api/other/web_search?title=<关键词>&is_type=0
```

接口返回内容中会混有进度文本，例如：

```text
线路：自定义线路1
```

真正的数据行以 `data:` 开头，例如：

```text
data: {"title":"黑夜告白（更至16）夸克","url":"token123","is_type":0}
```

结束行是：

```text
data: [DONE]
```

搜索结果中的 `url` 有两种情况：

1. 已经是 `http://` 或 `https://` 开头，可以直接作为资源地址输出。
2. 是一段 token 字符串，需要调用二次解析接口获取真实网盘链接。

结果选择优先级：

```text
夸克网盘 source_type=0 > 百度网盘 source_type=2 > 其他来源
```

程序先收集当前关键词的全部候选搜索结果，再按短剧名称分组；每组根据上述优先级选出一条，最后只解析并输出每组被选中的资源地址。

二次解析接口：

```text
POST https://www.xinyueso.com/api/other/save_url
```

请求体：

```json
{
  "url": "<URL 编码后的 token>",
  "title": "<短剧名称>"
}
```

成功返回后可获得真实资源地址，例如：

```json
{
  "code": 200,
  "message": "临时资源获取成功",
  "data": {
    "title": "黑夜告白（更至16）夸克",
    "url": "https://pan.quark.cn/s/..."
  }
}
```

## 三、项目文件结构

计划创建以下文件：

```text
D:\code
├── requirements.txt
├── README.md
├── xinyueso_search
│   ├── __init__.py
│   ├── models.py
│   ├── client.py
│   ├── database.py
│   └── cli.py
└── tests
    ├── test_client.py
    ├── test_database.py
    └── test_cli.py
```

各文件职责：

- `requirements.txt`：记录依赖：`requests`、`pytest`、`mysql-connector-python`
- `xinyueso_search/__init__.py`：Python 包初始化文件，记录版本号
- `xinyueso_search/models.py`：定义搜索结果数据结构 `SearchResult` 和网盘类型映射
- `xinyueso_search/client.py`：负责访问网站接口、解析 SSE 数据、解析 token 地址和短剧日期
- `xinyueso_search/database.py`：负责 MySQL 8.0 连接池、资源存在性检查、保存搜索结果
- `xinyueso_search/cli.py`：负责命令行参数解析、调用搜索客户端、格式化输出结果
- `tests/test_client.py`：测试模型、SSE 解析、搜索客户端、URL 解析逻辑
- `tests/test_database.py`：测试数据库建表、存在性检查、重复资源跳过保存
- `tests/test_cli.py`：测试文本、JSON、CSV 三种输出格式
- `README.md`：写明安装方式、使用示例和注意事项

## 四、实施任务

### 任务 1：创建项目骨架

创建依赖文件：

```txt
requests>=2.31.0
pytest>=8.0.0
mysql-connector-python>=8.4.0
```

创建包初始化文件：

```python
"""Utilities for searching xinyueso.com short-drama resources."""

__version__ = "0.1.0"
```

创建搜索结果模型：

```python
from __future__ import annotations

from dataclasses import dataclass


SOURCE_TYPES = {
    0: "夸克网盘",
    1: "阿里云盘",
    2: "百度网盘",
    3: "UC网盘",
    4: "迅雷网盘",
}


@dataclass(frozen=True)
class SearchResult:
    keyword: str
    title: str
    url: str
    resource_date: str | None = None
    source_type: int | None = None
    save_status: str = "pending"

    @property
    def source_name(self) -> str:
        if self.source_type is None:
            return "未知来源"
        return SOURCE_TYPES.get(self.source_type, f"未知来源({self.source_type})")
```

添加基础测试，验证 `source_type=0` 时返回“夸克网盘”。

验证命令：

```powershell
python -m pytest tests/test_client.py -v
```

预期结果：

```text
1 passed
```

当前状态：已完成。

说明：任务 1 已经完成过。新增的 `resource_date`、`save_status` 字段应在后续客户端和数据库任务中补充，避免回改时遗漏测试。

### 任务 2：实现 SSE 解析

创建 `parse_sse_events(raw_text)` 函数。

解析规则：

- 只处理 `data:` 开头的行
- 忽略普通进度行，例如 `线路：自定义线路1`
- 忽略空行
- 忽略 `data: [DONE]`
- 遇到坏 JSON 时跳过，不让程序崩溃
- 只返回 JSON 对象类型的数据

核心实现：

```python
def parse_sse_events(raw_text: str) -> list[dict[str, Any]]:
    rows: list[dict[str, Any]] = []
    for line in raw_text.splitlines():
        line = line.strip()
        if not line.startswith("data:"):
            continue

        payload = line.removeprefix("data:").strip()
        if not payload or payload == "[DONE]":
            continue

        try:
            value = json.loads(payload)
        except json.JSONDecodeError:
            continue

        if isinstance(value, dict):
            rows.append(value)
    return rows
```

添加测试：

- 能正确跳过进度行和 `[DONE]`
- 能跳过坏 JSON
- 能解析有效搜索结果

验证命令：

```powershell
python -m pytest tests/test_client.py -v
```

预期结果：

```text
3 passed
```

当前状态：已完成。

### 任务 3：实现搜索客户端

创建 `XinyuesoClient`。

主要职责：

1. 创建 HTTP session
2. 设置合理请求头
3. 调用搜索接口
4. 解析 SSE 返回结果
5. 提取每条短剧资源对应日期
6. 判断资源地址是否需要二次解析
7. 对候选结果按短剧名称分组
8. 每个短剧分组按“夸克优先、百度次优先、其他兜底”的规则选出一个链接
9. 返回所有短剧分组的最终 `SearchResult` 列表

搜索方法：

```python
client.search_keyword("黑夜告白", limit=20)
```

搜索接口：

```text
GET /api/other/web_search?title=<关键词>&is_type=<来源类型>
```

URL 解析逻辑：

- 如果 `url` 以 `http://` 或 `https://` 开头，直接返回
- 否则调用：

```text
POST /api/other/save_url
```

返回真实网盘地址。

需要添加测试：

- token 型 URL 会调用 `save_url` 解析
- 直链 URL 不调用 `save_url`
- 返回结果会正确转换成 `SearchResult`
- 返回结果会保存短剧对应日期到 `resource_date`
- `is_type` 会转成整数类型
- 接口返回错误时给出清晰异常
- 同一个短剧同时出现夸克和百度时，只保存夸克链接
- 同一个短剧没有夸克但有百度时，只保存百度链接
- 同一个短剧既没有夸克也没有百度时，保存原始排序中最靠前的其他来源
- 不同短剧都应保留，不能因为同一个关键词只输出一条
- 未被选中的候选结果不调用 `save_url`

当前状态：待执行。

### 任务 3.5：实现 MySQL 8.0 连接池、去重和保存

创建 `xinyueso_search/database.py`。

数据库使用 MySQL 8.0，所有数据库操作必须通过连接池获取连接。默认通过环境变量读取连接配置：

```text
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=
MYSQL_DATABASE=xinyueso
MYSQL_POOL_NAME=xinyueso_pool
MYSQL_POOL_SIZE=5
```

建议表结构：

```sql
CREATE TABLE IF NOT EXISTS resources (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    title VARCHAR(512) NOT NULL,
    normalized_title VARCHAR(512) NOT NULL,
    keyword VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    resource_date DATE NULL,
    source_type INT NULL,
    source_name VARCHAR(64) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_resources_normalized_title (normalized_title),
    KEY idx_resources_keyword (keyword),
    KEY idx_resources_source_type (source_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

核心规则：

- `normalized_title` 用于去重，建议先实现为：去除首尾空白并转小写。
- 保存前先根据 `normalized_title` 查询。
- 如果已存在，跳过插入，并将结果状态标记为 `skipped_existing`。
- 如果不存在，插入数据库，并将结果状态标记为 `saved`。
- 保存资源时必须一起保存短剧对应日期 `resource_date`。
- 数据库层只负责去重和保存，不负责调用网络接口。
- 插入时仍依赖 `UNIQUE KEY uq_resources_normalized_title` 做并发兜底；如果插入时遇到重复键错误，也应返回 `skipped_existing`。
- 每次数据库操作从连接池获取连接，使用完成后必须关闭 connection，使其归还连接池。
- cursor 和 connection 必须用 `try/finally` 或上下文管理方式确保释放。
- CLI 启动时创建一个 `ResourceDatabase` 实例并复用其连接池，不要每保存一条结果就新建连接池。
- 默认连接池大小为 5，可通过 `MYSQL_POOL_SIZE` 配置。

建议函数：

```python
def normalize_title(title: str) -> str:
    return title.strip().lower()


class ResourceDatabase:
    def __init__(
        self,
        host: str,
        port: int,
        user: str,
        password: str,
        database: str,
        pool_name: str = "xinyueso_pool",
        pool_size: int = 5,
    ) -> None:
        ...

    @classmethod
    def from_env(cls) -> "ResourceDatabase":
        ...

    def initialize(self) -> None:
        ...

    def exists(self, title: str) -> bool:
        ...

    def save_result(self, result: SearchResult) -> SearchResult:
        ...

    def save_results(self, results: list[SearchResult]) -> list[SearchResult]:
        ...
```

需要添加测试：

- 初始化 `ResourceDatabase` 时会创建 MySQL connection pool，而不是单个长连接
- `exists()` 每次从连接池获取连接，并在查询后归还连接
- `save_result()` 每次从连接池获取连接，并在插入或跳过后归还连接
- 初始化数据库会创建 `resources` 表
- 保存不存在的资源会插入，并返回 `save_status="saved"`
- 保存资源时会把 `resource_date` 写入 MySQL 的 `resource_date` 字段
- 再次保存同名资源会跳过，并返回 `save_status="skipped_existing"`
- 同名资源即使 URL 不同，也应跳过
- 不同名称资源都能保存
- 重复键错误应被安全处理为 `skipped_existing`

测试策略：

- 单元测试用 mock 模拟 `mysql.connector.pooling.MySQLConnectionPool`、connection、cursor，验证连接池创建参数、SQL 参数、分支逻辑和连接归还行为。
- 如果本地提供 MySQL 8.0 测试库，再额外运行集成测试；集成测试默认跳过，只有检测到 `MYSQL_TEST_DSN` 或完整测试环境变量时才执行。

验证命令：

```powershell
python -m pytest tests/test_database.py tests/test_client.py -v
```

预期结果：

```text
全部通过
```

验证命令：

```powershell
python -m pytest tests/test_client.py -v
```

预期结果：

```text
5 passed
```

当前状态：待执行。

### 任务 4：实现命令行工具

创建 `xinyueso_search/cli.py`。

命令行支持：

```powershell
python -m xinyueso_search.cli 黑夜告白
```

支持多个关键词：

```powershell
python -m xinyueso_search.cli 黑夜告白 良陈美锦
```

支持限制每个关键词候选扫描数量：

```powershell
python -m xinyueso_search.cli 黑夜告白 --limit 3
```

说明：`--limit` 表示最多扫描多少条候选结果。程序会保留扫描范围内出现的所有不同短剧，但每个短剧只输出一个最高优先级资源链接。

支持输出 JSON：

```powershell
python -m xinyueso_search.cli 黑夜告白 --limit 3 --format json
```

支持输出 CSV：

```powershell
python -m xinyueso_search.cli 黑夜告白 --limit 10 --format csv
```

支持指定 MySQL 连接配置。默认从环境变量读取：

```powershell
$env:MYSQL_HOST="127.0.0.1"
$env:MYSQL_PORT="3306"
$env:MYSQL_USER="root"
$env:MYSQL_PASSWORD="你的密码"
$env:MYSQL_DATABASE="xinyueso"
$env:MYSQL_POOL_SIZE="5"
python -m xinyueso_search.cli 黑夜告白 --limit 10
```

默认行为：搜索、按短剧名称归并、选择最高优先级链接、检查数据库、保存不存在的资源、跳过已存在资源，并输出每条结果的保存状态。

CLI 数据库生命周期：

- 程序启动后创建一个 `ResourceDatabase.from_env()` 实例。
- `ResourceDatabase` 内部创建并持有一个 MySQL 连接池。
- 保存所有结果时复用同一个连接池。
- 每次查询或插入只短暂借用连接，用完立即归还连接池。

如果只想预览不写入数据库，可以支持 `--dry-run`：

```powershell
python -m xinyueso_search.cli 黑夜告白 --limit 10 --dry-run
```

普通文本输出示例：

```text
1. 黑夜告白（更至16）夸克
   关键词: 黑夜告白
   来源: 夸克网盘
   日期: 2026-04-28
   地址: https://pan.quark.cn/s/...
   状态: saved
```

JSON 输出示例：

```json
[
  {
    "keyword": "黑夜告白",
    "title": "黑夜告白（更至16）夸克",
    "url": "https://pan.quark.cn/s/...",
    "resource_date": "2026-04-28",
    "source_type": 0,
    "source_name": "夸克网盘",
    "save_status": "saved"
  }
]
```

CSV 输出示例：

```csv
keyword,title,url,resource_date,source_type,source_name,save_status
黑夜告白,黑夜告白（更至16）夸克,https://pan.quark.cn/s/...,2026-04-28,0,夸克网盘,saved
```

需要添加测试：

- 文本格式包含标题、来源和地址
- JSON 格式可以被 `json.loads` 正确解析
- CSV 格式可以被 `csv.DictReader` 正确读取
- 三种输出格式都包含短剧对应日期 `resource_date`
- 多关键词输入时，每个关键词可以输出多个不同短剧结果
- 同一个短剧存在多个来源时，输出格式中只出现被选中的最高优先级链接
- 不同短剧即使来源相同，也应分别输出
- 已存在数据库中的短剧，输出状态为 `skipped_existing`
- 不存在数据库中的短剧，保存后输出状态为 `saved`
- `--dry-run` 不写入数据库，输出状态为 `pending`

验证命令：

```powershell
python -m pytest tests/test_cli.py tests/test_client.py -v
```

预期结果：

```text
8 passed
```

当前状态：待执行。

### 任务 5：编写 README 和最终验证

创建 `README.md`，内容包括：

- 项目说明
- 安装依赖方式
- 命令行使用示例
- JSON 输出示例
- CSV 输出示例
- 数据库保存和去重说明
- MySQL 8.0 建表 SQL 和环境变量配置
- 合规和使用频率提醒

安装示例：

```powershell
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

运行示例：

```powershell
python -m xinyueso_search.cli 黑夜告白 --limit 3
```

配置 MySQL 8.0：

```powershell
$env:MYSQL_HOST="127.0.0.1"
$env:MYSQL_PORT="3306"
$env:MYSQL_USER="root"
$env:MYSQL_PASSWORD="你的密码"
$env:MYSQL_DATABASE="xinyueso"
$env:MYSQL_POOL_SIZE="5"
python -m xinyueso_search.cli 黑夜告白 --limit 20
```

最终测试：

```powershell
python -m pytest -v
```

预期结果：

```text
8 passed
```

真实联调测试：

```powershell
python -m xinyueso_search.cli 黑夜告白 --limit 1 --format json
```

预期结果形态：

```json
[
  {
    "keyword": "黑夜告白",
    "title": "黑夜告白（更至16）夸克",
    "url": "https://pan.quark.cn/s/...",
    "resource_date": "2026-04-28",
    "source_type": 0,
    "source_name": "夸克网盘",
    "save_status": "saved"
  }
]
```

优先级联调测试：

```powershell
python -m xinyueso_search.cli 黑夜告白 --limit 10 --format json
```

预期：

- 对同一个短剧，如果候选结果中存在夸克网盘，输出中 `source_type` 应为 `0`，`source_name` 应为 `夸克网盘`。
- 对同一个短剧，如果没有夸克网盘但存在百度网盘，输出中 `source_type` 应为 `2`，`source_name` 应为 `百度网盘`。
- 输出数组中可以包含多个不同短剧结果，但同一个短剧只应出现一次。
- 如果 MySQL 数据库里已存在某个短剧，真实联调输出中该短剧应显示 `save_status="skipped_existing"`，并且数据库不新增重复记录。
- 新保存的资源必须在 MySQL 中保存短剧对应日期 `resource_date`。

当前状态：待执行。

## 五、注意事项

- 当前目录 `D:\code` 不是 git 仓库，所以计划中的 `git commit` 步骤会跳过。
- 本程序应保持低频请求，避免对目标网站造成压力。
- 目标网站聚合第三方网盘资源，资源内容可能涉及版权限制；程序只做搜索和链接提取，请仅用于合法、个人用途。
- 因为网站接口可能变化，代码应把接口地址集中放在客户端模块中，方便后续维护。
- 由于该站点 JSON 响应头存在拼写问题，例如 `chartset=uft-8`，程序不能过度依赖响应头判断 JSON，而应直接解析响应体。
- 程序保留所有不同短剧结果；同一个短剧只保存一条资源，来源选择规则固定为：夸克优先、百度次优先、其他兜底。
- 保存前必须通过 MySQL 8.0 连接池查询数据库；已存在资源跳过保存，不写入重复记录。
- 所有数据库查询和写入都必须使用连接池，不允许每次操作新建独立数据库连接。

## 六、当前执行进度

- 任务 1：项目骨架与模型，已完成
- 任务 2：SSE 解析，已完成
- 任务 3：搜索客户端，待执行
- 任务 3.5：MySQL 8.0 连接池、数据库去重和保存，待执行
- 任务 4：命令行工具，待执行
- 任务 5：README 与最终验证，待执行
