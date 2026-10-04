# 使用 slang 增强 Verilog/SystemVerilog 解析 Spec

## Why
当前仓库对 Verilog/SystemVerilog 的支持建立在通用 Tree-sitter 抽取之上，只能覆盖较浅的结构信息，难以支持多文件语义解析、参数化实例、包导入、宏与 include 等更真实的 RTL 工程场景。需要引入更成熟的 `slang` 前端，为二次开发提供一套明确、分阶段、可落地的集成方案。

## What Changes
- 明确 `slang` 在当前仓库中的推荐接入方式、依赖模式与降级策略
- 明确需要修改的目录、文件、关键函数，以及可复用的接口与子系统
- 定义从“资料调研”到“源码调研”再到“仓库改造方案”的三阶段分析输出
- 定义 `slang` 集成后的最小实现边界，包括配置、解析、图映射、测试与文档
- 定义建议新增的文件与模块职责，例如 `slang_config.py`、`slang_resolver.py`
- **BREAKING** 不在首版中把整个 Verilog 解析链完全替换为 `slang` 主通路，而采用“保留 Tree-sitter 基线 + 可选启用 slang 语义增强”的方案

## Impact
- Affected specs: Verilog/SystemVerilog 语言支持、增量构建、图构建、MCP review 工作流、测试策略
- Affected code:
  - `code_review_graph/parser.py`
  - `code_review_graph/incremental.py`
  - `code_review_graph/graph.py`
  - `code_review_graph/main.py`
  - `code_review_graph/tools/`
  - `tests/test_multilang.py`
  - `tests/test_parser.py`
  - `pyproject.toml`
  - 新增 `code_review_graph/slang_config.py`
  - 新增 `code_review_graph/slang_resolver.py`
  - 新增 `tests/test_slang_resolver.py`
- Affected functions / logical hotspots:
  - `CodeParser.parse_bytes()`
  - `CodeParser._extract_from_tree()`
  - `CodeParser._collect_file_scope()`
  - `CodeParser._get_name()` 中的 Verilog/SystemVerilog 分支
  - `CodeParser._collect_import_names()` 中的 Verilog/SystemVerilog 分支
  - Verilog `module_instantiation` 相关的调用目标抽取与 caller 归属逻辑
  - `incremental.full_build()`
  - `incremental.incremental_update()`
  - `_parse_single_file()` 及语言专用后处理挂点
  - `GraphStore.store_file_nodes_edges()` 及 `extra` 字段承载策略

## ADDED Requirements
### Requirement: 提供 slang 集成开发方案
系统 SHALL 产出一套针对 `code-review-graph` 引入 `slang` 的开发方案，明确说明接入模式、改造边界、阶段性任务与验证方法。

#### Scenario: 方案覆盖完整集成链路
- **WHEN** 用户查看该规格
- **THEN** 能看到 `slang` 的能力摘要、当前仓库 RTL 支持现状、推荐的集成方式，以及需要修改的目录、文件和函数
- **THEN** 能区分“需要重写”的实现与“可以继续复用”的接口

### Requirement: 分阶段输出分析结论
系统 SHALL 按“资料调研 -> slang 源码调研 -> 当前仓库改造分析”三个阶段组织输出，并使每个阶段都形成可评审的结论。

#### Scenario: 用户介入阶段性分析
- **WHEN** 用户希望在分析过程中介入评审
- **THEN** 规格与任务清单中应能体现三个阶段的独立目标、交付内容与依赖关系
- **THEN** 每个阶段都应有清晰的完成判据
- **THEN** 阶段输出至少应包含：
  - 第一阶段：`slang` 能力摘要、接入方式比较、首版推荐路线
  - 第二阶段：`slang` 关键 API / 源码能力与工程配置限制
  - 第三阶段：当前仓库中需改动的目录、文件、函数、复用接口与集成草图

### Requirement: 明确当前仓库中的改造落点
系统 SHALL 指出 `code-review-graph` 中与 Verilog/SystemVerilog 相关的关键改造落点，包括目录、文件、函数和可复用接口。

#### Scenario: 开发者评估代码改造范围
- **WHEN** 开发者根据规格制定实现计划
- **THEN** 能定位至少以下区域：
  - `parser.py` 中的 Verilog 语言映射、名字提取、import 抽取、实例化抽取相关分支
  - `incremental.py` 中的全量构建、增量构建与语言专用后处理挂点
  - `graph.py` 中可复用的节点/边与 `extra` 承载能力
  - `main.py` 与 `tools/` 中可继续复用的 CLI/MCP 工具接口
  - `tests/` 中需要新增或扩展的 Verilog/SystemVerilog 测试
- **THEN** 应明确以下判断：
  - `parser.py` 中 Verilog 专属抽取逻辑需要重写或拆分
  - `incremental.py` 主要需要新增 `slang` 后处理挂点，而非整体重写
  - `graph.py`、`main.py`、`tools/` 以复用为主，只在必要处扩展元数据承载

### Requirement: 采用可选启用的 slang 语义增强方案
系统 SHALL 将 `slang` 集成为 Verilog/SystemVerilog 的可选语义增强器，而不是首版直接替换现有的 Tree-sitter 基线解析。

#### Scenario: 环境中未安装 pyslang
- **WHEN** 仓库环境未提供 `pyslang`
- **THEN** 现有 Verilog/SystemVerilog 基础解析能力仍应保留
- **THEN** 构建、CLI、MCP 与图查询主流程不应中断

#### Scenario: 环境中已安装 pyslang 且提供配置
- **WHEN** 用户启用 `slang` 配置并完成依赖安装
- **THEN** 系统应允许在现有图构建流程后追加 repo 级的 SystemVerilog 语义增强步骤
- **THEN** 该步骤应能补充 package/import、实例化、参数化、诊断等更准确的图信息

### Requirement: 新增 slang 专用配置与解析模块
系统 SHALL 为 `slang` 集成预留独立模块边界，以避免持续堆积到 `parser.py` 的巨型条件分支中。

#### Scenario: 定义新模块职责
- **WHEN** 开发者根据规格开始实现
- **THEN** 应具备明确的新模块建议：
  - `code_review_graph/slang_config.py` 负责读取 `slang` 相关仓库配置
  - `code_review_graph/slang_resolver.py` 负责调用 `pyslang` 并将结果映射回 `NodeInfo` / `EdgeInfo`
- **THEN** `incremental.py` 应作为语义增强器的主挂载点
- **THEN** `slang` 的首版实现不应强行并入 `custom_languages.py` / `languages.toml` 通道

### Requirement: 保持现有图模型与上层接口复用
系统 SHALL 尽量复用现有图存储、增量更新、CLI、MCP 与后处理能力，除非 `slang` 能力接入确实要求变更。

#### Scenario: 评估哪些接口可继续使用
- **WHEN** 开发者检查集成边界
- **THEN** 应确认以下能力原则上可继续复用：
  - `NodeInfo` / `EdgeInfo`
  - `GraphStore` 与图写入接口
  - `full_build` / `incremental_update`
  - `build_or_update_graph_tool`、`query_graph_tool`、`detect_changes_tool` 等上层工具
- **THEN** 如需新增 SystemVerilog 专有信息，优先通过 `extra` 字段承载
- **THEN** 不应在首版中要求大规模修改 blast radius、MCP 查询协议或 CLI 对外接口

### Requirement: 定义验证与测试策略
系统 SHALL 为 `slang` 集成定义明确的测试范围，使最终实现可以验证“兼容现有能力”与“新增语义能力”。

#### Scenario: 设计测试矩阵
- **WHEN** 开发者编写实现任务
- **THEN** 任务中应包含以下验证方向：
  - 保留当前 `tests/test_multilang.py` 的 Verilog/SystemVerilog 基线能力
  - 新增多文件 package import、module instantiation、parameterized instance、include/macro 的集成测试
  - 验证未安装 `pyslang` 时的降级路径
  - 验证启用 `pyslang` 时，图构建结果不会破坏现有 CLI/MCP 行为

## MODIFIED Requirements
### Requirement: Verilog/SystemVerilog 解析策略
系统当前通过 `parser.py` 中的通用 Tree-sitter 分支完成 Verilog/SystemVerilog 解析；修改后，该能力应调整为“通用基础抽取 + 可选语义增强”的双层策略。

#### Scenario: 基础模式
- **WHEN** 系统仅运行 Tree-sitter 基础抽取
- **THEN** 保持当前模块、接口、task/function、基础 import 与实例化抽取能力

#### Scenario: 增强模式
- **WHEN** 系统启用 `slang` 语义增强
- **THEN** 在不破坏现有图查询接口的前提下，提高多文件与语义级关系的准确性

## REMOVED Requirements
### Requirement: 首版直接全量替换 Verilog 解析主通路
**Reason**: 当前仓库的构建与图模型是围绕“按文件解析 + 统一图存储”设计的，首版直接切换为 `slang` 主通路会放大多文件 elaboration、配置管理与依赖安装的复杂度。
**Migration**: 首版采用可选启用的 `slang` 语义增强器；在验证稳定后，再评估是否进一步提升为主解析通路。
