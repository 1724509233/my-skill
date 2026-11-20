---
name: docs-archive
description: "Archive project docs to the vault mirror (~/md-vault/1-项目/<project>): optional spec-format rename, mirror copy, frontmatter, dedup, reference rewrites, git rm, test + MCP smoke verification. Use when the user asks to archive/migrate project docs to the vault（归档/迁移文档到 vault）。On-demand pipeline only — no cron."
disable-model-invocation: true
metadata:
  opencode/autoinvoke: false
---

# 文档归档 pipeline（项目 → vault 镜像）

把项目文档归档到 `~/md-vault/1-项目/<project>/` 镜像，减少仓库内文档的上下文消耗；检索/召回改走 md-vault MCP（search_docs / read_doc / get_related_docs）。vault 在 git 外：归档文档失去 git diff 能力（历史留 git log）；变更追踪靠 vault ledger（外部写入归 source=watchdog，预期内）。

## 参数解析（按项目现场确定，不写死）

- **project** = 当前仓库名（或用户指定）；frontmatter tag = 项目名
- **vault 目标** = `1-项目/<project>/`，子目录结构跟随项目既有布局（vault 里已有则沿用；没有则按仓库 docs/ 结构镜像）
- **迁移池** = 项目 docs/ 树 + 研究笔记（范围不确定时问用户）
- **留仓库（永不动）** = 根级活文档（AGENTS.md / GLOSSARY.md / README.md / PRD.md 等）、代码、docs/ 下 HTML（vault 只索引 md，HTML 放进去检索不到）
- **本地文件**（gitignore、未入 git）：复制 + 补 FM + 正文校验一致后删源，**不走 git rm**
- **测试** = 项目自己的测试命令（见其 AGENTS.md）；改动前跑一次建基线，失败集合不得变

## 步骤

0. **快照**：`git ls-files` 生成迁移清单 + 每文件 sha256 manifest（临时目录 T/opencode，不入仓库）；记录 vault 既有文件清单 + 正文哈希
1. **（用户要求时）spec 格式重命名**：`YYYY-MM-DD-<slug>.md`，日期 = 文件创建日（git 首提交，非最后修改日）；`git mv` 后同步改写仓库内全部**全路径**引用为新文件名。代码注释里的**裸 stem 简写**（如 `m1-plan item 9`）不改——stem 保留在新文件名中，经 search_docs 可召回
2. **镜像复制**：保持相对目录结构（plan↔spec 内部互引存活，零链接重写）；mkdir -p；排除 `.DS_Store`；逐文件 sha256 校验副本 = 源
3. **frontmatter**（幂等：已有 FM 的文件跳过；加在 vault 副本上，仓库源文件直接删）：
   ```
   ---
   title: "<首个围栏外 H1 文本，必须双引号；无 H1 则人工定>"
   tags: [<project>, <kind>]    # kind ∈ plan|spec|adr|benchmark|issue|note；实施契约 = spec，任务分解 = plan，流程梳理/调研笔记 = note
   date: YYYY-MM-DD             # 仅文件名含日期时
   ---
   ```
   已知坑：标题内 ` #` 不加引号会被 YAML 当注释截断；H1 提取必须跳过 ``` 围栏（否则取到代码注释行，如 `# main.py`）。
4. **去重**（仅对 vault 既有副本目录）：正文哈希 = 剥离 FM 块 + 去行尾空白后 sha256。哈希一致 → 删旧副本；stem 相同但哈希异（改名/异版）→ 移 `legacy/`；无 stem 对应（用户笔记）→ 不动
5. **仓库引用改写**：指向被迁路径的全部**全路径**引用（根级文档 / docstring / README 等）→ `~/md-vault/1-项目/<project>/...`。**若目标文件有未提交改动**：`git show HEAD:<file>` 取 HEAD 版 → 只改引用行 → `git update-index --cacheinfo 100644,$(git hash-object -w <tmp>),<file>` 仅暂存引用改写；工作区文件做同样改写——用户的未提交改动不混进归档提交，且提交后其 pending diff 只剩自己的改动
6. **git rm**：逐条核对 staged 审计——不含留仓项、HTML、本地忽略文件、**他人未跟踪文件**（如并行进行中的新源码）；index 与工作区内容不一致时（如改名后又在工作区改了引用）`git rm -f`（vault 副本已校验为权威）
7. **提交**：`docs: archive <what> to vault mirror (1-项目/<project>)`，一批一提交；manager/fixer 改完不提交，验证后提交
8. **验证**：
   - 测试套件全量 → 失败集合不得变。若工作区有并行未提交的源码改动导致编译失败，确认错误文件不在本提交内即可，基线以改动前运行结果为准
   - 全部新 FM `yaml.safe_load` 通过 + 抽 3 份人工核对 title 与 H1 一致
   - MCP 冒烟（md-vault 工具）：`search_docs` 抽 2-3 个已知词命中归档文档；`read_doc` 抽 1 篇全文可读；`get_related_docs` 抽 1 份 → 返回同项目其他文档（tag 枢纽生效）。已知 byproduct：kind tag 跨项目共享，related 列表会混入其他项目同 kind 文档——如需项目内隔离，kind tag 改 `<project>-<kind>`（需用户裁决）
9. **报告**：迁移 N 文件 / FM M / 去重 删X / legacy Y / 留 Z + commit hash + MCP 证据
