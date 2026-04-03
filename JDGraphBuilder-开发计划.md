# JDGraphBuilder 开发计划

> **核心系统 — 数据清洗、图建模、入库**
>
> 本文档基于对 [Reports](https://github.com/WHUKE/Reports)（项目总体计划）、[JDParser](https://github.com/WHUKE/JDParser)（知识抽取工具，已基本完成）的分析，制定 JDGraphBuilder 的详细开发计划。

---

## 一、项目定位与目标

JDGraphBuilder 是"职业规划知识图谱"项目的**核心中枢系统**，承接 JDParser 输出的结构化 JSON 数据，完成以下核心功能：

1. **数据清洗与校验** — 对 JDParser 输出进行质量检查、缺失值处理、字段规范化
2. **图建模** — 设计并实现 Neo4j 图数据库的节点/关系 Schema
3. **数据入库** — 将清洗后的数据批量导入 Neo4j，构建完整的知识图谱
4. **图谱查询 API** — 提供面向上层应用（职位推荐、技能路径规划等）的查询接口
5. **数据统计与可视化支持** — 输出图谱统计信息，支持前端可视化展示

---

## 二、上游数据分析（JDParser 输出）

### 2.1 数据格式

JDParser 输出 `data/parsed/_all.json`（全量汇总）和每条 JD 的独立 JSON 文件，数据模型如下：

```json
{
  "source_file": "xyh_01.txt",
  "job_title": "全栈开发工程师",
  "location": "广州市",
  "education": "本科",
  "experience": "3-5年",
  "job_category": "前端开发",
  "responsibilities": ["职责1", "职责2"],
  "required_skills": [
    {"name": "JavaScript", "proficiency": "精通", "category": "编程语言", "parent": null},
    {"name": "Vue.js", "proficiency": "精通", "category": "前端框架", "parent": null},
    {"name": "MySQL", "proficiency": "熟悉", "category": "数据库", "parent": null}
  ],
  "preferred_skills": [
    {"name": "Docker", "proficiency": "熟悉", "category": "DevOps工具", "parent": null}
  ]
}
```

### 2.2 数据规模

- 当前：**120 条 JD**，共约 **3217 个技能项**
- 后续：爬虫扩展后预计可达数千条 JD

### 2.3 JDParser 提供的 Python API

```python
from src.loader import load_all, load_file

jds = load_all()          # 加载全量 _all.json
jds = load_file("xx.json")  # 加载指定文件

# JobDescription 对象字段
jd.source_file        # str: 原始文件名
jd.job_title           # Optional[str]: 职位名称
jd.location            # Optional[str]: 工作地点
jd.education           # Optional[str]: 学历要求
jd.experience          # Optional[str]: 工作年限
jd.job_category        # Optional[str]: 职位类别
jd.responsibilities    # list[str]: 工作职责
jd.required_skills     # list[Skill]: 必需技能
jd.preferred_skills    # list[Skill]: 加分技能

# Skill 对象字段
skill.name             # str: 归一化技能名称
skill.proficiency      # Optional[str]: 熟练度 (了解/熟悉/熟练/精通/不限)
skill.category         # Optional[str]: 技能分类
skill.parent           # Optional[str]: 父技能名称
```

### 2.4 已知数据质量问题（需清洗处理）

| 问题类型 | 说明 | 处理策略 |
|----------|------|----------|
| 字段缺失 | `job_title`、`location` 等可能为 `null` | 设置默认值或标记为"未知" |
| 地点不一致 | "广州市" vs "广州" vs "广州·天河区" | 地点名称归一化 |
| 学历表述差异 | "本科及以上" vs "本科" vs "Bachelor" | 统一为标准枚举值 |
| 经验格式不统一 | "3-5年" vs "3年以上" vs "不限" | 正则提取结构化经验范围 |
| 职位类别不一致 | "后端" vs "后端开发" vs "服务端开发" | 类别归一化映射表 |
| 技能 proficiency 缺失 | 部分技能 proficiency 为 null | 默认设为 "不限" |
| 技能 category 缺失 | 部分技能 category 为 null | 基于技能名称推断或标记为"其他" |

---

## 三、知识图谱 Schema 设计

### 3.1 节点类型（Labels）

| 节点标签 | 属性 | 说明 |
|----------|------|------|
| `Job` | `title`, `source_file`, `experience_min`, `experience_max`, `responsibilities` | 职位节点 |
| `Skill` | `name`, `category`, `description` | 技能节点（全局唯一，被多个 Job 共享） |
| `Location` | `name`, `province`, `city` | 工作地点节点 |
| `Education` | `level`, `rank` | 学历节点（本科=1, 硕士=2, 博士=3，用于排序比较） |
| `Category` | `name` | 职位类别节点（如 后端开发、前端开发、AI/机器学习） |

### 3.2 关系类型（Relationship Types）

| 关系 | 起点 → 终点 | 属性 | 说明 |
|------|-------------|------|------|
| `REQUIRES_SKILL` | `Job` → `Skill` | `proficiency` | 职位要求的必需技能 |
| `PREFERS_SKILL` | `Job` → `Skill` | `proficiency` | 职位的加分技能 |
| `REQUIRES_EDUCATION` | `Job` → `Education` | — | 职位的学历要求 |
| `LOCATED_IN` | `Job` → `Location` | — | 职位的工作地点 |
| `BELONGS_TO` | `Job` → `Category` | — | 职位所属类别 |
| `PARENT_OF` | `Skill` → `Skill` | — | 技能层级关系（如 Spring → Spring Boot） |
| `CO_OCCURS_WITH` | `Skill` → `Skill` | `weight`, `job_count` | 技能共现关系（同一 JD 中同时出现的技能） |

### 3.3 约束与索引

```cypher
-- 唯一性约束（同时自动创建索引）
CREATE CONSTRAINT skill_name IF NOT EXISTS FOR (s:Skill) REQUIRE s.name IS UNIQUE;
CREATE CONSTRAINT location_name IF NOT EXISTS FOR (l:Location) REQUIRE l.name IS UNIQUE;
CREATE CONSTRAINT education_level IF NOT EXISTS FOR (e:Education) REQUIRE e.level IS UNIQUE;
CREATE CONSTRAINT category_name IF NOT EXISTS FOR (c:Category) REQUIRE c.name IS UNIQUE;
CREATE CONSTRAINT job_source IF NOT EXISTS FOR (j:Job) REQUIRE j.source_file IS UNIQUE;

-- 全文索引（支持模糊搜索）
CREATE FULLTEXT INDEX skill_search IF NOT EXISTS FOR (s:Skill) ON EACH [s.name];
CREATE FULLTEXT INDEX job_search IF NOT EXISTS FOR (j:Job) ON EACH [j.title];
```

---

## 四、项目结构设计

```
JDGraphBuilder/
├── README.md
├── requirements.txt
├── .env.example                    # 环境变量模板（Neo4j 连接信息等）
├── .gitignore
│
├── data/
│   ├── input/                      # JDParser 输出的 JSON 文件（输入数据）
│   │   └── _all.json               # 从 JDParser/data/parsed/ 复制过来
│   ├── cleaned/                    # 清洗后的标准化数据
│   │   └── _all_cleaned.json
│   └── mappings/                   # 归一化映射表
│       ├── location_mapping.json   # 地点归一化映射
│       ├── education_mapping.json  # 学历归一化映射
│       └── category_mapping.json   # 职位类别归一化映射
│
├── src/
│   ├── __init__.py
│   │
│   ├── config.py                   # 全局配置（路径、Neo4j 连接、常量）
│   │
│   ├── cleaner/                    # 数据清洗模块
│   │   ├── __init__.py
│   │   ├── base.py                 # 清洗器抽象基类
│   │   ├── field_cleaner.py        # 字段级清洗（地点/学历/经验归一化）
│   │   ├── skill_cleaner.py        # 技能数据增强与清洗
│   │   ├── validator.py            # 数据校验（必填项检查、格式验证）
│   │   └── pipeline.py             # 清洗流水线编排
│   │
│   ├── modeler/                    # 图建模模块
│   │   ├── __init__.py
│   │   ├── schema.py               # 图谱 Schema 定义（节点/关系/约束）
│   │   ├── node_builder.py         # 节点构造器（从清洗数据生成节点）
│   │   ├── relation_builder.py     # 关系构造器（生成显式关系 + 推断关系）
│   │   └── co_occurrence.py        # 技能共现关系计算
│   │
│   ├── loader/                     # 数据入库模块
│   │   ├── __init__.py
│   │   ├── neo4j_client.py         # Neo4j 数据库连接管理
│   │   ├── batch_importer.py       # 批量导入器（高效写入 Neo4j）
│   │   └── schema_initializer.py   # Schema 初始化（创建约束/索引）
│   │
│   ├── query/                      # 查询接口模块
│   │   ├── __init__.py
│   │   ├── job_query.py            # 职位查询（按技能/地点/学历查找）
│   │   ├── skill_query.py          # 技能查询（关联职位/共现技能/技能树）
│   │   ├── path_query.py           # 路径查询（技能成长路径/职位跳转路径）
│   │   └── stats_query.py          # 统计查询（热门技能/地域分布等）
│   │
│   └── cli/                        # 命令行入口
│       ├── __init__.py
│       ├── clean.py                # 数据清洗命令
│       ├── build.py                # 图谱构建命令（清洗+建模+入库一站式）
│       ├── query.py                # 交互式查询命令
│       └── stats.py                # 图谱统计命令
│
├── scripts/
│   ├── init_neo4j.cypher           # Neo4j Schema 初始化脚本
│   └── sample_queries.cypher       # 常用查询示例
│
├── tests/
│   ├── __init__.py
│   ├── test_cleaner.py             # 清洗模块单元测试
│   ├── test_modeler.py             # 建模模块单元测试
│   ├── test_loader.py              # 入库模块单元测试
│   ├── test_query.py               # 查询模块单元测试
│   └── conftest.py                 # pytest fixtures（Mock Neo4j 等）
│
└── docs/
    ├── schema.md                   # 图谱 Schema 详细文档
    └── api.md                      # 查询 API 文档
```

---

## 五、模块详细设计

### 5.1 数据清洗模块 (`src/cleaner/`)

#### 5.1.1 字段级清洗 (`field_cleaner.py`)

| 清洗任务 | 输入示例 | 输出示例 | 实现方式 |
|----------|----------|----------|----------|
| 地点归一化 | "广州市" / "广州·天河区" | "广州" | 映射表 + 正则提取城市名 |
| 学历归一化 | "本科及以上" / "Bachelor" | "本科" | 枚举映射 |
| 经验结构化 | "3-5年" | `{"min": 3, "max": 5}` | 正则提取数字范围 |
| 经验结构化 | "3年以上" | `{"min": 3, "max": null}` | 正则提取数字范围 |
| 经验结构化 | "不限" / null | `{"min": 0, "max": null}` | 默认值处理 |
| 职位类别归一化 | "后端" / "服务端" | "后端开发" | 映射表 |
| 职位名称清洗 | "【急招】高级Java开发" | "高级Java开发" | 正则去除标记前缀 |

#### 5.1.2 技能数据增强 (`skill_cleaner.py`)

- **proficiency 缺失补全**：未标注熟练度的技能默认为 "不限"
- **category 缺失推断**：基于技能名称在 JDParser 的归一化映射表中查找分类，或标记为 "其他"
- **parent 关系补全**：检查 parent 字段指向的父技能是否作为独立节点存在，不存在则自动创建
- **去重校验**：同一 JD 内 required_skills 与 preferred_skills 之间的重复检测

#### 5.1.3 数据校验 (`validator.py`)

```python
# 校验规则
REQUIRED_FIELDS = ["source_file"]  # 必填字段
WARN_IF_MISSING = ["job_title", "location", "education", "job_category"]  # 缺失则警告
VALID_EDUCATION = ["博士", "硕士", "本科", "大专", "不限"]
VALID_PROFICIENCY = ["了解", "熟悉", "熟练", "精通", "不限"]
```

- 输出校验报告：记录每条 JD 的清洗操作和警告信息
- 异常数据单独存放，不阻塞正常数据入库

#### 5.1.4 清洗流水线 (`pipeline.py`)

```
输入 JSON → 字段校验 → 地点归一化 → 学历归一化 → 经验结构化 → 类别归一化 → 技能清洗/增强 → 输出清洗后 JSON
```

### 5.2 图建模模块 (`src/modeler/`)

#### 5.2.1 节点构造 (`node_builder.py`)

从清洗后的 JSON 中提取并去重所有节点：

```python
def build_nodes(jds: list[dict]) -> dict:
    """从清洗后的 JD 数据中构造所有节点
    
    Returns:
        {
            "jobs": [{"title": ..., "source_file": ..., ...}],
            "skills": [{"name": ..., "category": ...}],  # 全局去重
            "locations": [{"name": ...}],                  # 全局去重
            "educations": [{"level": ..., "rank": ...}],   # 全局去重
            "categories": [{"name": ...}],                 # 全局去重
        }
    """
```

#### 5.2.2 关系构造 (`relation_builder.py`)

从清洗后的 JSON 中提取所有显式关系：

- `REQUIRES_SKILL`: 遍历每条 JD 的 `required_skills`，创建 `Job → Skill` 关系，附带 `proficiency` 属性
- `PREFERS_SKILL`: 遍历 `preferred_skills`，同上
- `REQUIRES_EDUCATION`: `Job → Education`
- `LOCATED_IN`: `Job → Location`
- `BELONGS_TO`: `Job → Category`
- `PARENT_OF`: 遍历所有 Skill 的 `parent` 字段，创建 `Skill → Skill` 层级关系

#### 5.2.3 技能共现计算 (`co_occurrence.py`)

```python
def compute_co_occurrence(jds: list[dict], min_count: int = 2) -> list[dict]:
    """计算技能共现关系
    
    对于同一 JD 中同时出现的技能对 (A, B)，统计：
    - job_count: 共同出现在多少个 JD 中
    - weight: 归一化后的共现强度 (0-1)
    
    仅保留 job_count >= min_count 的共现关系，避免噪声。
    
    Returns:
        [{"skill_a": "Python", "skill_b": "Django", "job_count": 15, "weight": 0.83}, ...]
    """
```

### 5.3 数据入库模块 (`src/loader/`)

#### 5.3.1 Neo4j 客户端 (`neo4j_client.py`)

```python
class Neo4jClient:
    """Neo4j 数据库连接管理"""
    
    def __init__(self, uri: str, user: str, password: str, database: str = "neo4j"):
        ...
    
    def run_query(self, cypher: str, parameters: dict = None) -> list[dict]:
        """执行单条 Cypher 查询"""
    
    def run_batch(self, cypher: str, batch_data: list[dict], batch_size: int = 500):
        """批量执行参数化 Cypher（使用 UNWIND 批量写入）"""
    
    def close(self):
        """关闭连接"""
```

#### 5.3.2 批量导入器 (`batch_importer.py`)

使用 **UNWIND + MERGE** 模式高效批量导入，避免逐条写入：

```cypher
-- 批量创建 Skill 节点示例
UNWIND $skills AS s
MERGE (skill:Skill {name: s.name})
SET skill.category = s.category

-- 批量创建 REQUIRES_SKILL 关系示例
UNWIND $relations AS r
MATCH (j:Job {source_file: r.source_file})
MATCH (s:Skill {name: r.skill_name})
MERGE (j)-[rel:REQUIRES_SKILL]->(s)
SET rel.proficiency = r.proficiency
```

导入顺序（保证引用完整性）：

1. 创建约束和索引
2. 导入 `Skill` 节点
3. 导入 `Location` 节点
4. 导入 `Education` 节点
5. 导入 `Category` 节点
6. 导入 `Job` 节点
7. 创建 `Job → Skill` 关系（REQUIRES_SKILL / PREFERS_SKILL）
8. 创建 `Job → Location / Education / Category` 关系
9. 创建 `Skill → Skill` 关系（PARENT_OF）
10. 创建 `Skill → Skill` 关系（CO_OCCURS_WITH）

#### 5.3.3 Schema 初始化 (`schema_initializer.py`)

- 执行 `scripts/init_neo4j.cypher` 中定义的约束和索引
- 支持 `--reset` 参数清空数据库重新导入
- 支持增量导入（MERGE 语义保证幂等性）

### 5.4 查询接口模块 (`src/query/`)

#### 5.4.1 职位查询 (`job_query.py`)

| 查询功能 | 说明 | 示例 Cypher |
|----------|------|-------------|
| 按技能匹配职位 | 给定用户技能集，返回匹配度最高的职位 | `MATCH (j:Job)-[r:REQUIRES_SKILL]->(s:Skill) WHERE s.name IN $skills ...` |
| 按地点筛选 | 查询指定城市的职位 | `MATCH (j:Job)-[:LOCATED_IN]->(l:Location {name: $city})` |
| 按学历筛选 | 查询指定学历要求的职位 | `MATCH (j:Job)-[:REQUIRES_EDUCATION]->(e:Education) WHERE e.rank <= $rank` |
| 职位详情 | 获取职位的完整信息（技能/地点/学历/类别） | 多 MATCH + OPTIONAL MATCH |

#### 5.4.2 技能查询 (`skill_query.py`)

| 查询功能 | 说明 |
|----------|------|
| 热门技能排行 | 统计被最多职位要求的技能 |
| 技能关联技能 | 查询与指定技能共现频率最高的技能 |
| 技能树 | 查询指定技能的子技能层级关系 |
| 按分类查技能 | 查询某分类下的所有技能（如所有"编程语言"） |

#### 5.4.3 路径查询 (`path_query.py`)

| 查询功能 | 说明 |
|----------|------|
| 技能成长路径 | 从当前技能集到目标职位所需技能，找出需要学习的技能差集 |
| 职位跳转路径 | 从职位 A 到职位 B，分析技能重叠度和需要补充的技能 |
| 技能升级路径 | 从"了解"到"精通"的学习路径建议 |

#### 5.4.4 统计查询 (`stats_query.py`)

| 查询功能 | 说明 |
|----------|------|
| 图谱概览 | 节点总数、关系总数、各类型分布 |
| 技能需求热力图 | 各技能被要求的频次统计 |
| 地域分布 | 各城市的职位数量统计 |
| 学历要求分布 | 各学历层次的职位数量统计 |
| 技能类别分布 | 各技能类别的技能数量和职位关联数 |

---

## 六、技术选型

| 组件 | 选型 | 说明 |
|------|------|------|
| 语言 | Python 3.12+ | 与 JDParser 保持一致，便于数据衔接 |
| 图数据库 | Neo4j 5.x (Community Edition) | 免费、成熟、Cypher 查询语言强大 |
| Neo4j 驱动 | `neo4j` (官方 Python 驱动) | 支持连接池、事务管理、参数化查询 |
| 配置管理 | `python-dotenv` | 通过 `.env` 文件管理数据库连接等敏感配置 |
| 数据处理 | `pandas` (可选) | 用于数据清洗中的批量处理和统计分析 |
| CLI | `argparse` | 标准库，无需额外依赖 |
| 测试 | `pytest` | 与 JDParser 保持一致 |
| 日志 | `logging` (标准库) | 与 JDParser 保持一致 |

### `requirements.txt` 预计内容

```
neo4j>=5.0.0
python-dotenv>=1.0.0
pytest>=7.0.0
```

---

## 七、开发阶段与里程碑

### 阶段一：项目初始化与数据清洗（第 2 周前半）

**目标**：搭建项目框架，完成数据清洗模块

- [ ] 初始化项目结构（目录、`.gitignore`、`requirements.txt`、`.env.example`）
- [ ] 创建 `src/config.py`（路径配置、Neo4j 连接配置）
- [ ] 准备归一化映射表（`data/mappings/` 下的 JSON 文件）
- [ ] 实现 `field_cleaner.py`（地点/学历/经验/类别归一化）
- [ ] 实现 `skill_cleaner.py`（技能数据增强与清洗）
- [ ] 实现 `validator.py`（数据校验与报告生成）
- [ ] 实现 `cleaner/pipeline.py`（清洗流水线编排）
- [ ] 实现 `cli/clean.py`（命令行入口）
- [ ] 编写清洗模块单元测试 (`tests/test_cleaner.py`)
- [ ] 将 JDParser 输出数据导入 `data/input/` 并执行清洗

**交付物**：`data/cleaned/_all_cleaned.json` — 清洗后的标准化数据

### 阶段二：图谱 Schema 设计与建模（第 2 周后半）

**目标**：完成图谱 Schema 设计和节点/关系构造逻辑

- [ ] 编写 `schema.py`（Neo4j Schema 定义，包含约束和索引的 Cypher）
- [ ] 编写 `scripts/init_neo4j.cypher`（初始化脚本）
- [ ] 实现 `node_builder.py`（从清洗数据构造节点列表）
- [ ] 实现 `relation_builder.py`（构造显式关系列表）
- [ ] 实现 `co_occurrence.py`（技能共现关系计算）
- [ ] 编写建模模块单元测试 (`tests/test_modeler.py`)
- [ ] 编写 `docs/schema.md` Schema 文档

**交付物**：完整的节点/关系数据结构、Schema 文档

### 阶段三：数据入库（第 3 周前半）

**目标**：将数据批量导入 Neo4j，构建完整知识图谱

- [ ] 实现 `neo4j_client.py`（连接管理、查询执行、批量写入）
- [ ] 实现 `schema_initializer.py`（Schema 初始化逻辑）
- [ ] 实现 `batch_importer.py`（UNWIND 批量导入）
- [ ] 实现 `cli/build.py`（一站式构建命令：清洗→建模→入库）
- [ ] 编写入库模块单元测试 (`tests/test_loader.py`)
- [ ] 部署 Neo4j 并执行首次全量导入
- [ ] 在 Neo4j Browser 中验证图谱正确性

**交付物**：Neo4j 中完整的知识图谱

### 阶段四：查询接口开发（第 3 周后半）

**目标**：实现面向应用层的查询 API

- [ ] 实现 `job_query.py`（职位匹配/筛选/详情查询）
- [ ] 实现 `skill_query.py`（技能排行/关联/技能树查询）
- [ ] 实现 `path_query.py`（技能成长路径/职位跳转路径）
- [ ] 实现 `stats_query.py`（图谱统计查询）
- [ ] 实现 `cli/query.py`（交互式查询命令）
- [ ] 实现 `cli/stats.py`（统计命令）
- [ ] 编写查询模块单元测试 (`tests/test_query.py`)
- [ ] 编写 `scripts/sample_queries.cypher`（常用查询示例）
- [ ] 编写 `docs/api.md` 查询 API 文档

**交付物**：可用的查询接口和 CLI 工具

### 阶段五：集成测试与优化（第 4-5 周，配合可视化/应用开发）

**目标**：端到端验证，性能优化，支持前端可视化

- [ ] 端到端集成测试（从原始 JSON 到图谱查询）
- [ ] 性能优化（批量导入速度、查询性能）
- [ ] 支持增量导入（新增 JD 不需要全量重建）
- [ ] 提供图谱可视化所需的 JSON 输出接口（前端友好的节点/边格式）
- [ ] 编写完整 README 文档
- [ ] 代码审查和重构

---

## 八、命令行使用预览

```bash
# 1. 清洗数据
python -m src.cli.clean --input data/input/_all.json --output data/cleaned/

# 2. 一站式构建图谱（清洗 + 建模 + 入库）
python -m src.cli.build --input data/input/_all.json --reset  # 全量重建
python -m src.cli.build --input data/input/_all.json          # 增量导入

# 3. 查看图谱统计
python -m src.cli.stats

# 4. 交互式查询
python -m src.cli.query --skill "Python" "Django"       # 按技能匹配职位
python -m src.cli.query --location "北京"                # 按地点筛选
python -m src.cli.query --path "前端开发" "全栈开发"      # 职位跳转路径
python -m src.cli.query --trending                       # 热门技能
```

---

## 九、环境配置

### 9.1 Neo4j 安装

```bash
# Docker 方式（推荐）
docker run -d \
  --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/your-password \
  -v neo4j-data:/data \
  neo4j:5
```

### 9.2 环境变量 (`.env`)

```env
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your-password
NEO4J_DATABASE=neo4j
```

---

## 十、风险与应对

| 风险 | 影响 | 应对措施 |
|------|------|----------|
| Neo4j 性能不足 | 大数据量时查询慢 | 合理设计索引；使用 UNWIND 批量写入；考虑 APOC 插件 |
| 数据质量差 | 图谱中存在垃圾节点/关系 | 严格的数据校验；清洗报告人工审核 |
| 技能归一化不完善 | 同一技能有多个节点 | 持续维护归一化映射表；定期检测重复节点 |
| JDParser 输出格式变更 | 清洗模块解析失败 | 清洗模块对输入格式做兼容性处理；版本化接口约定 |
| 共现关系噪声大 | 无意义的技能关联 | 设置最小共现阈值 `min_count`；引入 PMI/TF-IDF 加权 |

---

## 十一、与其他仓库的协作关系

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Reports (项目管理)                           │
│              进度跟踪 · 课程报告 · 分工协调 · 项目看板                    │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
    ┌──────────────────────────┐    ┌──────────────────────────────┐
    │     JDParser (已完成)      │    │  JDGraphBuilder (本仓库)       │
    │                          │    │                              │
    │  原始 JD → 结构化 JSON     │───►│  JSON → 清洗 → 建模 → Neo4j   │
    │                          │    │                              │
    │  输出: _all.json          │    │  输出: 知识图谱 + 查询 API      │
    └──────────────────────────┘    └──────────────────────────────┘
                                                    │
                                                    ▼
                                    ┌──────────────────────────────┐
                                    │     上层应用 (第4周开发)         │
                                    │                              │
                                    │  · 图谱可视化界面               │
                                    │  · 职位推荐系统                 │
                                    │  · 技能成长路径规划             │
                                    │  · 就业决策支持                 │
                                    └──────────────────────────────┘
```

---

## 十二、分工建议

| 模块 | 建议负责人 | 预计工作量 |
|------|-----------|-----------|
| 数据清洗 (`cleaner/`) | 1-2 人 | 2-3 天 |
| 图建模 (`modeler/`) | 1-2 人 | 2-3 天 |
| 数据入库 (`loader/`) | 1 人 | 2 天 |
| 查询接口 (`query/`) | 1-2 人 | 3-4 天 |
| 测试与文档 | 全员 | 贯穿全程 |

> 注：具体分工以 [GitHub Projects 看板](https://github.com/orgs/WHUKE/projects/1) 为准。
