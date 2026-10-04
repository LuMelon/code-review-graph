# Tasks
- [ ] Task 1: 形成 `slang` 能力与集成边界的基础调研结论，并沉淀为可评审的阶段输出。
  - [ ] SubTask 1.1: 汇总 `slang` 的官方能力边界，包括语法树、语义编译、诊断、实例化与分析能力
  - [ ] SubTask 1.2: 明确 `pyslang`、C++ 直连、CLI JSON 导出三种接入方式的优缺点
  - [ ] SubTask 1.3: 给出首版推荐接入方式与不推荐方案
  - [ ] SubTask 1.4: 将第一阶段输出组织为“能力摘要 + 接入路线对比 + 首版建议”

- [ ] Task 2: 梳理 `slang` 源码与公开接口，识别适合本仓库复用的 API 与潜在限制。
  - [ ] SubTask 2.1: 识别 `SyntaxTree`、`Compilation`、`AnalysisManager`、诊断相关接口在集成中的作用
  - [ ] SubTask 2.2: 分析 `slang` 对多文件、include、define、library、top 等工程配置的依赖
  - [ ] SubTask 2.3: 形成对本项目最关键的源码结论，明确哪些能力适合作为首版目标
  - [ ] SubTask 2.4: 将第二阶段输出组织为“关键 API / 工程限制 / 集成启示”

- [ ] Task 3: 深入分析 `code-review-graph` 的 Verilog/SystemVerilog 现状，明确改造落点与复用边界。
  - [ ] SubTask 3.1: 盘点 `parser.py` 中与 Verilog 相关的节点映射、名字提取、import 抽取、实例化抽取逻辑
  - [ ] SubTask 3.2: 盘点 `incremental.py` 中全量构建、增量构建与语言专用后处理挂点
  - [ ] SubTask 3.3: 盘点 `graph.py`、`main.py`、`tools/`、`tests/` 中可直接复用的接口与需要扩展的测试
  - [ ] SubTask 3.4: 输出“需要重写 / 需要扩展 / 可以复用”的文件与函数清单

- [ ] Task 4: 设计 `slang` 与当前仓库的集成方案，并落实到目录、文件、函数级别的改造计划。
  - [ ] SubTask 4.1: 给出推荐的模块边界，包括新增 `slang_config.py` 与 `slang_resolver.py` 的职责
  - [ ] SubTask 4.2: 明确哪些函数需要重写、哪些函数只需补挂点、哪些接口完全复用
  - [ ] SubTask 4.3: 给出从 POC 到正式集成的分阶段实施路径与降级策略
  - [ ] SubTask 4.4: 说明为何首版不建议直接替换 `parser.py` 主通路，也不建议走 `custom_languages.py`

- [ ] Task 5: 制定验证方案与最终交付结构，保证后续实现与评审可操作。
  - [ ] SubTask 5.1: 设计测试矩阵，覆盖基础兼容、语义增强与降级路径
  - [ ] SubTask 5.2: 设计实施完成后的验收标准与风险清单
  - [ ] SubTask 5.3: 整理适合用户中途介入评审的阶段性输出结构
  - [ ] SubTask 5.4: 确保阶段性输出可映射为后续实现阶段的 issue / 子任务

# Task Dependencies
- Task 2 depends on Task 1
- Task 3 depends on Task 1
- Task 4 depends on Task 2
- Task 4 depends on Task 3
- Task 5 depends on Task 4
