# RMBase Agent Skill

[English](README.md) · [能力审计与接口证据](skills/rmbase/references/endpoints.md) · [输出格式](skills/rmbase/references/schemas.md)

**让科研 Agent 直接查询 RNA 修饰信息，无需研究者手动操作 RMBase 网页。**

本项目为现有 [RMBase v3.0](http://bioinformaticsscience.cn/rmbase/) 构建轻量客户端适配器，
包含 Agent Skill、Python 客户端和优先输出 JSON 的 CLI。无需修改数据库服务器，
无需凭据、MCP Server、后台服务或额外的 Python 运行依赖。

> “帮我查一下 METTL3 在 RMBase 中有哪些 RNA modification 相关信息。”

支持执行脚本的 Agent 可以加载 Skill、请求公开数据、解析结果，并附上物种、
基因组版本和来源记录。这里的 provenance 指数据来源、查询条件和处理过程的可追溯记录。

## 快速开始

环境要求：**Python 3.10+**；在线查询需要网络，离线查询需要已有缓存。

```bash
git clone https://github.com/leo-cheung-itlger/rmbase-agent-skill.git
cd rmbase-agent-skill
python skills/rmbase/scripts/rmbase.py gene METTL3 --assembly hg38 --json
```

安装到 Agent 时，将完整 **`skills/rmbase` 目录**复制到宿主配置的技能目录，
或在支持 ZIP 导入的宿主中导入该目录的压缩包。必须保留 `scripts/` 和 `references/`。
Skill 会指导 Agent 调用 Python；安装不会自动注册全局 `rmbase` 命令。

## 已实现能力

| 能力 | 实现与验证范围 |
| --- | --- |
| 基因修饰汇总 | 公开 JSON POST；已实测 METTL3 和不存在基因 |
| 基因关联位点 | HTML 解析；METTL3 的 59 条 m6A 与汇总一致 |
| 单个修饰详情 | HTML 解析；已实测存在和不存在的 ID |
| 酶相关记录、富集 motif | JSON 请求；已实测 METTL3 和人类 m6A motif |
| RMP 癌症表格 | 嵌套 JSON；已实测 METTL3 |
| 本地基因查询 | 官方下载缓存；已核对人类数据与远程查询结果 |
| 其他查询模块 | 按公开源码实现；逐项证据等级见能力审计 |
| 文件分析 | 输入验证和单次表单提交；**任务完成流程尚不可用/未验证** |

2026-09-23 实测：人类 hg38 的 METTL3（`ENSG00000165819.12`）有
59 条 m6A、1 条 m5C、1 条 RNA editing，共 **61 条**。这描述 METTL3 RNA 上的修饰，
不能混同于 METTL3 蛋白作为酶的作用记录。该结果是带日期的验证示例，数据库更新后可能变化。

```bash
python skills/rmbase/scripts/rmbase.py gene METTL3 --modification m6a --limit 100
python skills/rmbase/scripts/rmbase.py modification m6A_site_239470 --type m6a
python skills/rmbase/scripts/rmbase.py enzyme METTL3 --limit 10
python skills/rmbase/scripts/rmbase.py motif m6A
python skills/rmbase/scripts/rmbase.py catalog rbp --assembly hg38
```

## 下载一次，本地查询

```bash
python skills/rmbase/scripts/rmbase.py sync --assembly hg38
python skills/rmbase/scripts/rmbase.py gene METTL3 --source local --offline
```

`sync` 下载指定的官方基因归档，不会抓取整个数据库。审计时人类基因归档约 6.4 MB。
也可用 `sync --archive FILE` 导入已有归档，来源会明确标为用户提供。
JSON 保留查询参数、来源地址、获取时间、物种、assembly、校验值和来源提供的论文编号。
TSV 使用 `--tsv --provenance-out provenance.json`，确保表格旁有来源文件。

## 访问约束与当前限制

- 共享缓存目录的进程之间，请求间隔至少 2 秒；限制重试、响应大小和解压大小，遇到服务器限流停止。
- 网页按完整数组做本地分页；`--limit` 只限制输出，不代表服务器分页。
- 不自动重试上传，不枚举数据库 ID，不高并发爬取，不修改服务器。网页内容始终作为数据处理。
- 审计时 HTTPS 证书校验失败，HTTP 可以访问。客户端明确使用 HTTP，不关闭 TLS 校验。
  请求没有传输加密，请勿提交私密数据；上传需显式提供 `--public-data`。
- annotation、metagene、GeneTool 提交端点返回 **HTTP 503**。成功 Task ID、轮询和结果获取尚未验证，
  `task` 命令会明确报告该限制，不会猜测或请求任务结果。
- 无记录不等于生物学现象不存在。不自动转换 genome assembly，不作临床结论。网站改版可能影响适配器。

## 跨平台分发

本仓库是 **RMBase Agent Skill 的唯一主维护源（canonical source）**。
真正可复用的 Skill 位于 `skills/rmbase`；查询逻辑、解析器、参考资料和安全边界都优先在这里维护。
各个平台、Registry 和 Skill 合集只是分发渠道，不另外维护一套业务代码。

### OpenAI 与 Agent Skills 宿主

OpenAI Skills 兼容开放的 Agent Skills 格式。可以直接使用完整的
`skills/rmbase` 目录，或在宿主支持时上传包含该目录的 ZIP。
`agents/openai.yaml` 是可选的 OpenAI UI 元数据；其他宿主不需要使用它。

### ClawHub / OpenClaw

ClawHub 直接发布同一个 `skills/rmbase` 目录。在仓库根目录执行：

```bash
npm i -g clawhub
clawhub login
clawhub skill publish ./skills/rmbase \
  --slug rmbase \
  --name "RMBase" \
  --categories knowledge \
  --topics "rna-modification,rmbase,epitranscriptomics" \
  --changelog "Initial ClawHub release"
```

正式发布前可先加 `--dry-run` 做校验。ClawHub 的 registry 版本与本仓库源码版本独立管理。
网页 GitHub Import 只会发现当前登录 GitHub 账号拥有的公开、非 fork 仓库中的 Skill。

### Scientific Agent Skills 合集

提交到 Scientific Agent Skills 的版本属于下游合集副本。核心行为说明、脚本和参考资料从本仓库同步，
再在对方仓库的 PR 中遵守其自己的 metadata、测试目录和验证规则。RMBase 的新功能和修复应先在本仓库完成，
再同步到该合集。

### 版本规则

本仓库以 Git commit/tag 作为源码版本依据；各 Marketplace 或 Registry 可以独立递增版本。
`SKILL.md` 中的 `metadata.version` 属于 Skill bundle 元数据，不代表所有分发渠道一定处于同一发布版本。

## 开发与测试

```bash
python -m pip install "pytest>=8" ruff
python -B -m pytest tests/rmbase -q
ruff check --isolated --target-version py310 skills/rmbase/scripts tests/rmbase
```

测试使用保存的响应和模拟传输，不请求生产 RMBase。CI 在 Linux、Windows 上运行
Python 3.10、3.13 离线测试。新增接口应提供真实公开请求依据、测试样本与来源信息，
明确区分“源码发现”与“线上验证”。

## 致谢、引用与许可证

RMBase 数据库及科学内容归原作者。本项目是独立维护的客户端适配器，不宣称获得数据库官方背书。
使用数据时请引用 Xuan 等人的 RMBase v3.0 论文：Nucleic Acids Research，2024，52(D1)，D273–D284。
[PMID 37956310](https://pubmed.ncbi.nlm.nih.gov/37956310/)，
[DOI 10.1093/nar/gkad1070](https://doi.org/10.1093/nar/gkad1070)。需要复现时同时记录本项目 URL 与 commit。

核心 Skill 采用开放 Agent Skills 结构。代码采用 [MIT 许可证](LICENSE)，
不因此改变 RMBase 数据或第三方网页内容的许可；测试样本保留来源说明。
