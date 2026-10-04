# slang 集成到 code-review-graph 的开发手册

## 1. 背景

`code-review-graph` 当前对 Verilog / SystemVerilog 的支持建立在 `tree_sitter_language_pack` 之上，入口集中在 `code_review_graph/parser.py`。这条链路已经具备基础可用性，能够识别：

- 文件语言归类：`.sv`、`.svh`、`.v`、`.vh`
- 结构节点：`module_declaration`、`interface_declaration`、`class_declaration`
- 过程节点：`task_declaration`、`function_declaration`、`always_construct`
- 基础依赖：`package_import_declaration`
- 基础调用：`module_instantiation`、`function_subroutine_call`、`subroutine_call`、`system_tf_call`

现状的问题也很明确：这是一套“按单文件做浅层语法抽取”的通用框架，对真实 RTL 工程中的多文件编译单元、`include` / `define`、包与库配置、参数化实例、top 解析、elaboration 后实例层级与语义诊断支持不足。

`slang` 的价值不在于“再换一个 AST 解析器”，而在于补上 `code-review-graph` 目前缺失的工程级 SystemVerilog 语义前端能力：预处理、编译、诊断、类型检查、实例化、分析。这意味着首版集成不应该粗暴替换现有 `parser.py` 主链，而应该把 `slang` 设计成 Verilog / SystemVerilog 的可选语义增强器。

## 2. slang 关键能力

### 2.1 语法与语义能力

结合 `slang` 官方文档、`pyslang` 包说明和公开 API，可以把对本项目最关键的能力归纳为以下几类：

- 语法树能力：`slang::syntax::SyntaxTree` / `pyslang.SyntaxTree` 可以从文件或内存文本构建 CST，且容错能力强，适合 IDE / 编辑中场景。
- 语法遍历与改写：`SyntaxVisitor`、`SyntaxRewriter` 支持遍历与重写语法节点，这对后续做 refactor、定位原始 span、保留源码文本很有价值。
- 语义编译能力：`slang::ast::Compilation` / `pyslang.ast.Compilation` 是类型检查、绑定、elaboration 的核心入口，可将多个 `SyntaxTree` 组合为一个设计级编译对象。
- 诊断能力：`SourceManager`、`DiagnosticEngine`、文本诊断输出等能力可以追踪源位置、宏展开位置并产出较高质量错误信息。
- 实例化与层级能力：编译后的 elaborated symbol hierarchy 可以看到模块、接口、包、实例、作用域等语义对象，适合映射到图谱中的 `NodeInfo` / `EdgeInfo`。
- 静态分析能力：`AnalysisManager` 位于 AST 之后，可对已构建的设计做分析并产出结构化结果，适合未来扩展 richer edges 或硬件特定检查。
- 工程级编译能力：支持 include 路径、宏定义、library、top、多个编译单元等工程配置，这正是当前 `tree-sitter` 方案的短板。

### 2.2 三种接入方式对比

#### A. `pyslang`

优点：

- 与当前仓库同属 Python 生态，最容易在 `incremental.py` 后处理链路中挂接
- 能直接把 `slang` 的 CST / AST / diagnostics / symbols 映射到现有 `NodeInfo` / `EdgeInfo`
- 易于做可选依赖和降级处理，未安装时不影响主链
- 适合做 repo 级语义增强器，而无需改造 CLI / MCP 外部协议

缺点：

- 需要额外 Python 轮子或本地构建环境
- Python 绑定暴露面虽大，但并非所有 C++ 内部 API 都保证同等易用
- 在大仓库上需要关注编译对象生命周期、内存与增量策略

#### B. C++ 直连

优点：

- 理论上能力最完整，性能和 API 可控性最高
- 可直接访问完整 `slang` C++ 生态与分析能力

缺点：

- 与当前 Python 主体仓库技术栈不一致
- 引入构建系统、打包、跨平台分发、CI 复杂度显著上升
- 首版成本远高于收益，不适合作为当前仓库的第一阶段接入方式

#### C. `slang` CLI + JSON 导出

优点：

- 集成路径看似简单，可以把 `slang` 当外部工具调用
- 对 Python 侧实现改动较少

缺点：

- 中间 JSON 协议稳定性、字段覆盖率、版本兼容性都需要额外治理
- 很难高效做增量、缓存、span 回溯与 richer diagnostics
- 调用链变长，错误处理、性能和调试体验较差

### 2.3 推荐结论

首版推荐：`pyslang`。

不推荐作为首版主路：C++ 直连、CLI JSON 导出。

原因很简单：本仓库已经有成熟的 Python 构建、增量更新、图存储、CLI / MCP 接口，最佳策略是把 `slang` 压缩成一个“Python 内部语义增强模块”，而不是重新发明一条外部工具链或二次语言运行时。

## 3. slang 源码结构

从 `slang` 公开仓库结构与文档看，首版集成最需要理解的目录如下：

- `include/slang/`
  - 对外公开头文件
  - 其中最关键的是 `syntax/`、`ast/`、`analysis/`、`diagnostics/`、`text/`
- `source/`
  - 核心实现代码
  - 包括 `analysis/*.cpp`、`diagnostics/*.cpp` 以及语法 / AST / 数值等实现
- `bindings/python/`
  - `pyslang` 的 pybind11 绑定层
  - 关键入口是 `bindings/python/pyslang.cpp`
  - `SyntaxBindings.cpp`、`StatementBindings.cpp`、`ExpressionBindings.cpp` 等文件按能力拆分绑定
- `pyslang/`
  - Python 包封装、示例、测试与文档
  - 适合直接参考 Python 层用法与能力暴露情况
- `scripts/`
  - 代码生成定义，例如 `syntax.txt`、`diagnostics.txt`、`tokenkinds.txt`、`systemnames.txt`
  - 说明 `slang` 的一部分语法与诊断类型是生成式维护
- `tools/`
  - 官方 CLI / 辅助工具

对本项目尤其重要的源码启示有三点：

- `slang` 的核心不是单一 AST 类，而是“预处理 + 语法树 + 编译 + 诊断 + 分析”整条前端流水线。
- `pyslang` 不是轻量包装，而是对 `slang` 核心能力的系统性暴露，足够支持 Python 内集成。
- 源码与生成脚本结构说明：如果后续需要更深能力，优先依赖稳定公开 API，不要把集成建立在内部生成文件或私有布局之上。

## 4. 当前仓库改造分析

### 4.1 当前 Verilog / SystemVerilog 实现现状

`code_review_graph/parser.py` 中已经把 Verilog 视为内建语言：

- 扩展名映射：`EXTENSION_TO_LANGUAGE`
- 结构节点映射：`_CLASS_TYPES["verilog"]`
- 函数节点映射：`_FUNCTION_TYPES["verilog"]`
- import 节点映射：`_IMPORT_TYPES["verilog"]`
- call 节点映射：`_CALL_TYPES["verilog"]`
- 名字提取：`_get_name()` 中对 `module_declaration`、`interface_declaration`、`task_declaration`、`function_declaration` 的专门分支
- import 抽取：`_collect_import_names()` 中对 `package_import_declaration` 的专门分支
- 调用目标提取：`_get_callee_name()` 中对 `module_instantiation` 的专门分支

测试也已经覆盖了基础能力，集中在 `tests/test_multilang.py`：

- 识别 `.sv` / `.svh` / `.v` / `.vh`
- 识别 module / interface
- 识别 task / function
- 识别 package import
- 将 `module_instantiation` 归为 `CALLS`
- 保证实例化 caller 归属到 enclosing module

这说明当前仓库不是“完全不支持 Verilog”，而是“已经有可复用基线，但缺失工程级语义增强”。

### 4.2 增量构建与挂点分析

`code_review_graph/incremental.py` 是最适合挂 `slang` 的位置，原因如下：

- `full_build()` 先做统一文件解析与落库，之后已经会追加语言专用 resolver：
  - `_run_rescript_resolver(store)`
  - `_run_spring_resolver(store)`
  - `_run_temporal_resolver(store)`
- `incremental_update()` 也已经具备“仅当相关语言文件变化时再跑 resolver”的模式
- `_parse_single_file()` 与 `CodeParser.parse_bytes()` 仍是按文件解析模型，不适合直接塞进多文件 elaboration 逻辑
- `watch()` 也复用了 `incremental_update()` 的检测能力，这意味着新增 repo 级 resolver 能天然进入 watch / CLI / MCP 主流程

结论：`slang` 首版应模仿现有 resolver 体系，作为 Verilog / SystemVerilog 的 repo 级后处理 / 语义增强步骤接入，而不是强行重写文件级解析函数。

### 4.3 图模型与上层接口复用分析

`code_review_graph/graph.py` 当前模型足以容纳 `slang` 首版接入：

- `NodeInfo` / `EdgeInfo` 已支持统一节点和边抽象
- `GraphStore.store_file_nodes_edges()` 负责原子落库
- `nodes.extra` / `edges.extra` 是天然元数据承载位
- `confidence` / `confidence_tier` 已存在于边模型中，适合未来区分 tree-sitter 抽取与 slang 语义推断

`main.py` 与 `tools/` 侧基本不需要为首版做接口级修改：

- `build_or_update_graph_tool`
- `query_graph_tool`
- `detect_changes_tool`
- `semantic_search_nodes_tool`
- `get_impact_radius_tool`

这些工具消费的是图谱结果，不关心底层 edge / node 来自 tree-sitter 还是 slang。只要图模型稳定，上层接口可以复用。

### 4.4 `custom_languages.py` 为什么不适合

`custom_languages.py` 的目标是“为 tree-sitter grammar 提供配置式接入”，核心假设是：

- 输入仍然是 tree-sitter grammar
- 抽取仍然依赖 generic AST walker
- 不处理工程级多文件语义编译

而 `slang` 的价值恰恰在于突破这三个限制。因此：

- 不应把 `slang` 伪装成一个新的 `languages.toml` 语言项
- 不应把 repo 级编译、诊断、实例化能力硬塞进 generic node type 配置
- `slang` 应走独立模块边界，而不是 `custom_languages.py` 通道

## 5. 需要重写 / 扩展 / 复用的文件与函数

### 5.1 需要重写或新增的部分

建议新增：

- `code_review_graph/slang_config.py`
- `code_review_graph/slang_resolver.py`
- `tests/test_slang_resolver.py`

建议在新增模块中重写的核心职责：

- 读取 repo 级 SystemVerilog 工程配置
- 构建 `pyslang` 编译对象
- 收集 diagnostics
- 从 elaborated design / syntax tree / symbols 生成 `NodeInfo` / `EdgeInfo`
- 将语义增强结果与现有图谱合并

建议新增或重写的函数原型可以包括：

- `load_slang_config(repo_root: Path) -> SlangConfig`
- `is_slang_enabled(repo_root: Path) -> bool`
- `resolve_slang_semantics(store: GraphStore, repo_root: Path, changed_files: list[str] | None = None) -> dict`
- `build_compilation(config: SlangConfig, repo_root: Path) -> Compilation`
- `extract_slang_nodes_edges(compilation, repo_root: Path) -> tuple[list[NodeInfo], list[EdgeInfo]]`
- `map_diagnostics_to_graph(...)`
- `merge_slang_edges(...)`

### 5.2 需要扩展的部分

`code_review_graph/incremental.py`

- `full_build()`：在现有 resolver 链后增加 `_run_slang_resolver(...)`
- `incremental_update()`：仅当 `.sv` / `.svh` / `.v` / `.vh` 或 `slang` 配置文件变更时运行 `slang`
- `_parse_single_file()`：原则上不重写主逻辑，但可以为后续缓存或标记保留扩展点

`code_review_graph/graph.py`

- 主要复用现有结构
- 必要时仅补充对 `extra` 的约定，例如：
  - `origin = "tree-sitter" | "slang"`
  - `sv_kind = "module" | "interface" | "package" | "instance" | "parameter"`
  - `diagnostic_severity`
  - `diagnostic_code`

`tests/`

- `tests/test_multilang.py`：保留现有 Verilog 基线测试，不应回归
- `tests/test_parser.py`：必要时补充 parser 级兼容断言
- 新增 `tests/test_slang_resolver.py`：覆盖语义增强路径与降级路径

`pyproject.toml`

- 增加可选依赖组，例如 `slang` / `rtl` extra
- 不要把 `pyslang` 直接变成强依赖

### 5.3 可以直接复用的部分

可以直接复用，不建议首版重写：

- `NodeInfo`
- `EdgeInfo`
- `GraphStore`
- `GraphStore.store_file_nodes_edges()`
- `full_build()` / `incremental_update()` 主流程框架
- `main.py` 中的 CLI / MCP tool 暴露层
- 查询、review、impact、search 等上层图分析工具

### 5.4 当前仓库的关键函数分类结论

#### 需要重写或替代

- `parser.py` 中 Verilog 专属的名字提取逻辑
- `parser.py` 中 Verilog 专属的 import 提取逻辑
- `parser.py` 中 `module_instantiation` 调用目标推断逻辑

这里的“重写”不是马上删除旧代码，而是由 `slang_resolver.py` 产出更高置信度的 repo 级结果，并逐步替代这些浅层启发式。

#### 需要扩展挂点

- `incremental.full_build()`
- `incremental.incremental_update()`
- 可能需要一个 `_run_slang_resolver()` 包装函数

#### 可以继续复用

- `GraphStore.store_file_nodes_edges()`
- `query_graph_tool` / `detect_changes_tool` / `get_impact_radius_tool`
- 现有 FTS、community、flow、review 链路

## 6. 推荐架构

### 6.1 总体方案

推荐架构：保留现有 Tree-sitter 基线解析，新增 repo 级 `slang` 语义增强器。

数据流建议如下：

1. `CodeParser` 按现有逻辑完成所有文件基础抽取
2. 结果落入 SQLite 图谱
3. 若启用 `slang` 且环境具备 `pyslang`，运行 `slang_resolver`
4. `slang_resolver` 按 repo 配置收集 RTL 文件、include 路径、宏、top、library 等
5. 构建 `Compilation`
6. 从 elaborated design 中提取更准确的包、实例、参数化关系、诊断与跨文件依赖
7. 将增强节点 / 边以覆盖或补充形式写回图谱
8. 上层 CLI / MCP / review 工具继续消费统一图谱

### 6.2 新增模块职责

#### `code_review_graph/slang_config.py`

职责：

- 读取 repo 内 `slang` 配置
- 规范 include 路径、defines、library、top、source roots
- 提供配置默认值与校验
- 判断是否启用 `slang`

建议配置来源优先级：

1. 显式配置文件
2. 环境变量
3. 默认扫描规则

可选配置文件建议：

- `.code-review-graph/slang.toml`

建议字段：

- `enabled`
- `top_modules`
- `source_patterns`
- `include_dirs`
- `defines`
- `lib_dirs`
- `allow_missing_includes`
- `fallback_to_tree_sitter`

#### `code_review_graph/slang_resolver.py`

职责：

- 安全导入 `pyslang`
- 构建 compilation
- 抽取包、模块、接口、实例、参数化实例、跨文件 import / include
- 产出 diagnostics 与置信度更高的边
- 在失败时优雅降级

### 6.3 为什么不直接替换 `parser.py` 主通路

不建议首版替换的原因：

- `parser.py` 是按文件并行解析模型，`slang` 更适合 repo 级编译
- 直接替换会改变并行策略、缓存策略、失败模式与依赖模型
- `watch`、增量更新、文件哈希跳过逻辑都建立在“单文件 parser”假设上
- 首版最大目标是语义增强，而不是重写整条构建管线

因此更合理的迁移路径是：

- 第一步：后处理增强
- 第二步：对部分 Verilog 关系做优先覆盖
- 第三步：验证稳定后，再讨论是否提升为主解析路径

## 7. 分阶段实施计划

### Phase 0: 方案固化

目标：

- 固化能力边界、接口边界与文档结论

交付：

- 本开发手册
- 勾选后的 spec task / checklist

### Phase 1: 配置与依赖接入

目标：

- 新增 `slang_config.py`
- 让仓库可选感知 `pyslang`

交付：

- 可解析的 `.code-review-graph/slang.toml`
- `pyproject.toml` 可选依赖
- 未安装 `pyslang` 时零中断降级

### Phase 2: POC 级 resolver

目标：

- 新增 `slang_resolver.py`
- 能对一个小型多文件 SystemVerilog 工程完成编译并抽取核心语义关系

首版能力建议：

- package / import
- module / interface / class 基础语义对象
- module instantiation
- parameterized instance
- diagnostics 汇总

### Phase 3: 接入 full build / incremental update

目标：

- 在 `full_build()` 后增加 `slang` resolver
- 在 `incremental_update()` 中按 Verilog 相关变更条件触发

注意事项：

- 先允许“全 repo 重跑 resolver”，再考虑真正的 `slang` 增量
- 触发条件应包含 source 文件与配置文件变化

### Phase 4: 结果合并与置信度治理

目标：

- 处理 tree-sitter 与 slang 结果冲突
- 为边和节点补齐 `origin`、`confidence`、`confidence_tier`

建议策略：

- 结构基线保留 tree-sitter 结果
- repo 级语义关系优先采用 `slang`
- 冲突时以 `slang` 覆盖特定 kind 的 edge，而不是粗暴清空全量 Verilog 图

### Phase 5: 强化测试与评审

目标：

- 扩展测试矩阵
- 形成可评审 issue 拆分

交付：

- `tests/test_slang_resolver.py`
- 中型 fixture 工程
- 性能与降级验证报告

## 8. 测试与验收

### 8.1 测试矩阵

#### 基础兼容

- 未安装 `pyslang` 时，现有 `tests/test_multilang.py` 的 Verilog 基线全部通过
- CLI / MCP / query / detect_changes 不因 `slang` 缺失而失败

#### 语义增强

- 多文件 `package import`
- `include` / `define`
- 参数化实例
- top 解析
- diagnostics 收集与定位

#### 增量路径

- 修改单个 `.sv` 文件时触发 `slang` resolver
- 非 RTL 文件变更时不触发 `slang` resolver
- 配置文件变化时重新触发 resolver

#### 冲突与降级

- `pyslang` 不存在
- `slang` 配置错误
- compilation 失败但 tree-sitter 仍可工作

### 8.2 验收标准

至少满足以下条件才算首版完成：

- 默认安装环境下，仓库行为与当前版本兼容
- 启用 `slang` 后，能对多文件 SystemVerilog fixture 产出比当前更准确的 import / instantiation 关系
- `detect_changes_tool`、`query_graph_tool` 等上层能力无需协议改动即可消费增强结果
- 失败时回退到 tree-sitter 基线，不阻塞 build / update / watch
- 测试能覆盖启用与未启用两条路径

## 9. 风险与降级策略

### 9.1 主要风险

- 依赖风险：`pyslang` 安装、平台 wheel、CI 构建复杂度
- 工程配置风险：真实 RTL 工程对 include / define / library / top 很敏感
- 性能风险：repo 级 compilation 可能慢于当前单文件解析
- 结果合并风险：tree-sitter 与 slang 双来源结果可能重复或冲突
- 增量风险：`slang` 的自然粒度更接近工程级而非文件级

### 9.2 降级策略

- `pyslang` import 失败：记录 warning，继续使用 tree-sitter 基线
- `slang` 配置缺失：默认不启用，避免误伤普通仓库
- compilation 失败：收集 diagnostics 到 `extra` 或日志，但不阻断图构建主流程
- 仅对高价值 edge 做增强：首版不要强求 100% 节点都由 `slang` 接管
- 先 repo 级重跑、后做真增量：优先保证正确性，再做性能优化

## 10. 核心结论

本次方案的核心判断如下：

- `slang` 首版应作为 Verilog / SystemVerilog 的可选语义增强器接入，而不是直接替换 `parser.py` 主通路。
- 推荐使用 `pyslang`，不推荐首版走 C++ 直连或 CLI JSON 导出。
- 最佳挂点是 `incremental.py` 中现有 resolver 后处理链，而不是 `custom_languages.py`。
- `parser.py` 中 Verilog 相关启发式逻辑应被视为“保底基线”，后续逐步由 `slang_resolver.py` 产出的高置信度结果覆盖。
- `graph.py`、`main.py`、`tools/` 与大部分图分析能力可继续复用，首版无需改协议。
- 实施顺序应是“配置接入 -> POC resolver -> 全量 / 增量挂接 -> 冲突治理 -> 性能优化”，而不是一开始就推翻现有解析架构。
