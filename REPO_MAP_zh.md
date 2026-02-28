# REPO_MAP — otel-collector-contrib / pkg/ottl

> 由 repo-explorer 生成于 2026-02-28。范围：pkg/ottl。
> 目标背景：GSoC 贡献就绪。

<!-- GENERATED:overview -->
## 概述

**OpenTelemetry 转换语言 (OTTL)** 是一种领域特定语言 (DSL)，设计用于让运维人员通过 Collector YAML 配置表达针对 OpenTelemetry 遥测信号（跨度、日志、指标和配置文件）的数据变化和过滤。它作为自包含的 Go 模块存在于 `opentelemetry-collector-contrib` 单一仓库中的 `pkg/ottl` 内。

OTTL 的核心设计刻意极简：引擎本身提供解析、语法、类型安全的参数绑定和执行循环，但它**没有内置函数和内置字段路径**。两者都由嵌入 OTTL 的组件提供。这意味着相同的解析引擎可以为转换处理器、过滤处理器、尾部采样处理器、路由连接器以及任何未来的组件服务，每个都可以接入自己的函数集和自己的 `PathExpressionParser`，该解析器知道如何将路径令牌映射到 pdata 字段。

稳定性取决于信号类型：跨度、指标和日志为 **beta**；配置文件为 **development**。代码所有者是 @TylerHelmuth、@evan-bradley、@edmocosta 和 @bogdandrutu。

<!-- /GENERATED:overview -->

<!-- GENERATED:component-map -->
## 组件映射

```
pkg/ottl/
├── grammar.go              # 词法分析器 + 完整 participle 语法 AST (parsedStatement, value, path, editor, converter, mathExpression, booleanExpression, …)
├── parser.go               # Parser[K]、Statement[K]、Condition[K]、ValueExpression[K]、StatementSequence[K]、ConditionSequence[K]
├── expression.go           # Expr[K]、Getter/Setter/GetSetter 接口、所有类型化的 Standard*Getter 包装器、listGetter、mapGetter
├── functions.go            # PathExpressionParser、Path[K]/Key[K] 接口、Optional[T]、反射驱动的参数绑定 (buildArgs/buildArg)
├── factory.go              # Factory[K] 接口、NewFactory()、CreateFactoryMap()、FunctionContext、Arguments
├── config.go               # ErrorMode (ignore/propagate/silent)、LogicOperation (and/or)
├── boolean_value.go        # boolExpr[K] 求值 — 在运行时遍历 booleanExpression AST
├── compare.go              # 跨类型比较表 (int64, float64, string, bool, time, pcommon.Map/Slice, []byte, nil)
├── math.go                 # 数学表达式求值 (int64/float64/time.Time/time.Duration，+−×÷ 带优先级)
├── paths.go                # basePath[K] 构造、上下文名称验证、insertContextIntoPathsOffsets
├── literal_getter.go       # 编译期常量字面量 getter、isLiteralGetter()
├── context_inferrer.go     # contextInferrer 接口、priorityContextInferrer (从路径名自动选择上下文)
├── parser_collection.go    # ParserCollection[R] — 多上下文分发器，由转换处理器使用
├── transform_context_encoder.go  # newTransformContextField() 用于通过 zap 的结构化调试日志
│
├── contexts/               # 每个 OTel 信号一个包；每个都提供一个 TransformContext + PathExpressionParser
│   ├── ottllog/            # log.body、log.severity_number、log.attributes、log.timestamp、…
│   ├── ottlspan/           # span.name、span.kind、span.attributes、span.start_time_unix_nano、…
│   ├── ottlspanevent/      # spanevent.name、spanevent.attributes、…
│   ├── ottlmetric/         # metric.name、metric.description、metric.unit、metric.type、…
│   ├── ottldatapoint/      # datapoint.value_double/int/histogram/…、datapoint.attributes、…
│   ├── ottlresource/       # resource.attributes、resource.dropped_attributes_count
│   ├── ottlscope/          # instrumentation_scope.name、.version、.attributes、…
│   ├── ottlprofile/        # (development) profile.profile_id、profile.sample_type、…
│   ├── ottlprofilesample/  # (development) 配置文件样本上下文
│   └── internal/           # 按信号类型划分的共享字段访问器
│       ├── ctxlog/         # 核心日志字段读写
│       ├── ctxspan/        # 核心跨度字段读写
│       ├── ctxspanevent/
│       ├── ctxmetric/
│       ├── ctxdatapoint/
│       ├── ctxresource/    # 在日志/跨度/指标上下文中共享的 resource.attributes
│       ├── ctxscope/       # instrumentation_scope 共享访问器
│       ├── ctxprofile/ ctxprofilesample/ ctxprofilecommon/
│       ├── ctxcache/       # 每语句暂存空间 (pcommon.Map、池化)
│       ├── ctxcommon/      # 共享辅助程序 (通用字段的 GetSetter 包装器)
│       ├── ctxerror/       # 路径未找到错误格式化
│       ├── ctxutil/        # 跨上下文共享的实用函数
│       ├── logging/        # 用于调试输出的 zap ObjectMarshaler 辅助程序
│       ├── logprofile/     # 日志/配置文件互操作辅助程序
│       └── pathtest/       # 路径验证的测试辅助程序
│
├── ottlfuncs/              # 标准函数库 (编辑器 + 转换器)
│   ├── functions.go        # StandardFuncs[K]() 和 StandardConverters[K]() — 所有函数的注册表
│   ├── func_set.go         # set(target, value) — 最常见的编辑器
│   ├── func_delete_key.go / func_delete_matching_keys.go / func_keep_keys.go / …
│   ├── func_replace_pattern.go / func_replace_match.go / func_replace_all_*.go
│   ├── func_merge_maps.go / func_flatten.go / func_limit.go / func_truncate_all.go
│   ├── func_append.go      # append() — 切片编辑器
│   ├── func_parse_json.go / func_parse_csv.go / func_parse_key_value.go
│   ├── func_parse_xml.go / func_get_xml.go / func_insert_xml.go / func_remove_xml.go
│   ├── func_extract_patterns.go / func_extract_grok_patterns.go
│   ├── func_url.go / func_useragent.go / func_community_id.go
│   ├── func_sha256.go / func_sha512.go / func_sha1.go / func_md5.go
│   ├── func_fnv.go / func_murmur3_hash.go / func_xxh3.go / func_xxh128.go
│   ├── func_base64decode.go / func_base64encode.go / func_decode.go / func_hex.go
│   ├── func_concat.go / func_split.go / func_substring.go / func_format.go
│   ├── func_is_match.go / func_is_string.go / func_is_int.go / func_is_bool.go / func_is_map.go / …
│   ├── func_time.go / func_formattime.go / func_truncate_time.go / func_duration.go / func_now.go
│   ├── func_unix_nano.go / func_unix_milli.go / … (时间单位转换器)
│   ├── func_year.go / func_month.go / func_day.go / … (时间分量提取器)
│   ├── func_uuid.go / func_uuidv7.go / func_span_id.go / func_trace_id.go / func_profile_id.go
│   ├── func_slice_to_map.go / func_sort.go / func_len.go / func_index.go / func_keys.go / func_values.go
│   └── func_log.go / func_parse_severity.go / func_parse_int.go / func_luhn_valid.go
│
├── ottltest/               # 测试实用程序：Strp() 用于创建字符串指针等
├── e2e/                    # 端到端测试，使用真实的 TransformContext (ottllog, ottlspan)
└── internal/ottlcommon/    # 低级 pdata 值辅助程序 (GetValue 等)
```

### 关键类型与接口

| 类型/接口 | 文件 | 目的 |
|---|---|---|
| `Parser[K any]` | `parser.go:66` | 入口点。持有函数注册表、路径解析器、枚举解析器。K = TransformContext 类型 |
| `Statement[K]` | `parser.go:22` | 已解析语句 = 函数 Expr + 可选布尔条件 |
| `Condition[K]` | `parser.go:54` | 独立布尔表达式 |
| `ValueExpression[K]` | `parser.go:509` | 独立值提取器 (无函数调用) |
| `StatementSequence[K]` | `parser.go:363` | 有序语句切片；处理 ErrorMode |
| `ConditionSequence[K]` | `parser.go:420` | 有序条件切片；处理 AND/OR 逻辑和 ErrorMode |
| `Factory[K]` | `factory.go:29` | 从已解析参数创建函数实例 |
| `ExprFunc[K]` | `expression.go:25` | `func(ctx, tCtx K) (any, error)` — 实际函数闭包 |
| `Getter[K]` | `expression.go:38` | 在运行时从 tCtx 读取任何值 |
| `Setter[K]` | `expression.go:43` | 在运行时将值写入 tCtx |
| `GetSetter[K]` | `expression.go:49` | 组合 Getter + Setter (用于路径目标) |
| `PathExpressionParser[K]` | `functions.go:18` | `func(Path[K]) (GetSetter[K], error)` — 由每个上下文提供 |
| `Path[K]` | `functions.go:153` | 具有上下文、名称、键的路径段链表 |
| `Key[K]` | `functions.go:267` | 路径段内的字符串或整数索引 |
| `Optional[T]` | `functions.go:736` | 包装可选函数参数；通过 `IsEmpty()`/`Get()`/`GetOr()` 暴露 |
| `ErrorMode` | `config.go:12` | `ignore` / `propagate` / `silent` |
| `ParserCollection[R]` | `parser_collection.go:77` | 多上下文解析器，从路径自动推断上下文 |
| `TransformContext` (每个上下文包) | `contexts/ottl*/` | 持有 pdata 指针 + 暂存缓存；实现信号特定的 PathExpressionParser |

<!-- /GENERATED:component-map -->

<!-- GENERATED:data-flow -->
## 主要数据流

### 流程 A：解析语句

```
1. 组件调用 NewParser[K](functions, pathParser, settings, opts...)  [parser.go:75]
   - K 是具体的 TransformContext 类型 (例如 ottllog.TransformContext)
   - functions 是 map[string]Factory[K] (例如来自 ottlfuncs.StandardFuncs[K]())
   - pathParser 是上下文的 PathExpressionParser[K]

2. Parser.ParseStatement(raw string)  [parser.go:150]
   └─ parseStatement(raw)  [parser.go:282]
      └─ participle.Parser.ParseString()  使用来自 buildLexer() 的词法分析器 [grammar.go:502]
         - 标记化：Bytes、Float、Int、String、Nil、OpNot、OpOr、OpAnd、OpComparison、
           OpAddSub、OpMultDiv、Boolean、Uppercase、Lowercase、…
         - 构建以 parsedStatement{Editor, WhereClause} 为根的 AST  [grammar.go:15]
      └─ parsedStatement.checkForCustomError()  [grammar.go:22]
         - 验证命名约定 (编辑器小写开头，转换器大写开头)

3. Parser.newFunctionCall(parsed.Editor)  [functions.go:317]
   - 通过编辑器.Function 名称在 p.functions 中查找 Factory[K]
   - Factory.CreateDefaultArguments() → 通过反射的空 *ArgsStruct
   - Parser.buildArgs(ed, argsVal)  [functions.go:350]
     - 对于结构体中的每个参数字段 (按位置或名称)：
       - Setter/GetSetter → buildGetSetterFromPath → pathParser(basePath)
       - Getter、StringGetter、IntGetter 等 → newGetter(argVal) → 包装在 Standard*Getter
       - Enum → enumParser(*EnumSymbol) → int64
       - 字面量 (string/float64/int64/bool) → 直接设置
       - Optional[T] → 仅当参数存在时设置；否则保留零值
   - Factory.CreateFunction(FunctionContext, args) → ExprFunc[K]
   - 返回 Expr[K]{exprFunc}

4. Parser.newBoolExpr(parsed.WhereClause)  [boolean_value.go]
   - 从 booleanExpression AST 递归构建 boolExpr[K]
   - 处理 NOT、AND (更高优先级)、OR
   - 每个叶子 Comparison 构建两个 Getter[K]s 和一个 compareOp

5. 返回 &Statement[K]{function: Expr[K], condition: boolExpr[K], origText: raw}
```

### 流程 B：对实时遥测执行语句

```
6. StatementSequence[K].Execute(ctx, tCtx)  [parser.go:398]
   └─ 对于每个 Statement[K]：
      └─ Statement[K].Execute(ctx, tCtx)  [parser.go:33]

7. condition.Eval(ctx, tCtx)  [boolean_value.go]
   - 求值 booleanExpression 树
   - 每个 Comparison 在两侧调用 Getter[K].Get() → any 值
   - compare(left, right, op)  [compare.go] → bool

8. 如果条件为真 → Expr[K].Eval(ctx, tCtx)  [expression.go:33]
   - 调用函数闭包 (ExprFunc[K])
   - 在函数内，作为 Getter[K] 的参数调用 Get(ctx, tCtx)
     以通过上下文的 PathExpressionParser 读取实时 pdata 值
   - 作为 Setter[K] 的参数调用 Set(ctx, tCtx, val) 写回

9. 调试日志记录 (如果启用)：  [parser.go:37]
   - 通过 zap 记录每个语句前后的 TransformContext 状态
```

### 流程 C：路径解析 (在步骤 3 / 步骤 8 内部)

```
10. pathParser(basePath[K])  [functions.go:305]
    - 调用 p.pathParser (上下文提供的 PathExpressionParser[K])
    - 例如 ottllog 上下文映射 "log.body" → 包装 plog.LogRecord.Body() 的 StandardGetSetter
    - basePath.isComplete() 验证没有未使用的路径段保留
    - 返回由实际 pdata 访问器支持的 GetSetter[K]

11. 在运行时，路径支持的 GetSetter 上的 Getter[K].Get(ctx, tCtx)
    - 直接读取 pdata 字段 (例如 lr.Body().Str())
    - 如果存在键 (例如 attributes["foo"]) → exprGetter 遍历 pcommon.Map 或 pcommon.Slice
```

<!-- /GENERATED:data-flow -->

<!-- GENERATED:test-structure -->
## 测试结构

| 类别 | 位置 | 运行命令 |
|---|---|---|
| 单元 — 核心引擎 | `pkg/ottl/*_test.go` | `cd pkg/ottl && go test ./...` |
| 单元 — 上下文 | `pkg/ottl/contexts/ottl*/*_test.go` | `cd pkg/ottl && go test ./contexts/...` |
| 单元 — 函数 | `pkg/ottl/ottlfuncs/*_test.go` | `cd pkg/ottl && go test ./ottlfuncs/...` |
| 端到端 | `pkg/ottl/e2e/e2e_test.go` | `cd pkg/ottl && go test ./e2e/...` |
| 基准测试 | `pkg/ottl/performance_bench_test.go` | `cd pkg/ottl && go test -bench=. -benchmem ./...` |
| 模糊 | 不清楚 — 见各个测试文件 | — |

**测试约定：**

- `ottlfuncs/` 中的每个 `func_*.go` 都有一个匹配的 `func_*_test.go`，包含表驱动测试。
- 上下文测试 (`ottllog/log_test.go`、`ottlspan/span_test.go`) 针对真实 pdata 对象测试路径解析。
- `e2e/` 中的端到端测试将真实 Parser + 真实上下文 + 真实函数组合在一起，并针对构造的 pdata 执行语句/条件。
- `ottltest/ottltest.go` 提供 `Strp(s string) *string` 和类似的指针辅助程序。
- `contexts/internal/pathtest/` 包含用于测试 PathExpressionParser 实现的共享辅助程序。
- `id_test_helpers.go` 在 ottlfuncs 中提供用于跟踪/跨度/配置文件 ID 的测试工厂。

<!-- /GENERATED:test-structure -->

<!-- GENERATED:entry-points -->
## 贡献入口点 (GSoC)

### 首先要查看的地方

1. **语法 (标记化器 + 解析器 AST)**：`pkg/ottl/grammar.go` — 基于 participle 的语法。任何新语法 (新运算符、新字面量类型、新语句形式) 都从这里开始。
2. **表达式求值**：`pkg/ottl/expression.go` 和 `pkg/ottl/boolean_value.go` — 值和条件的运行时求值。
3. **数学求值**：`pkg/ottl/math.go` — 算术支持；目前处理 `int64`、`float64`、`time.Time`、`time.Duration`。
4. **函数绑定**：`pkg/ottl/functions.go` — 基于反射的参数绑定。新参数类型 (例如新的 getter 变体) 需要在这里更改。
5. **标准函数**：`pkg/ottl/ottlfuncs/` — 添加新函数是最常见的贡献。遵循任何现有 `func_*.go` 的模式。
6. **上下文路径**：`pkg/ottl/contexts/internal/ctx*/` — 添加新路径 (例如新的 pdata 字段) 需要在相关 `ctx*` 包的 `PathExpressionParser` 中添加一个 case。

### 添加新的 OTTL 函数 (编辑器或转换器)

按照任何现有 `func_set.go` (编辑器) 或 `func_concat.go` (转换器) 中的模式：

1. 创建 `pkg/ottl/ottlfuncs/func_<name>.go`：
   - 定义包含参数字段的 `Arguments` 结构体。
   - 实现函数体 (`ExprFunc[K]` 闭包)。
   - 实现 `New<Name>Factory[K]()` 返回 `ottl.NewFactory(...)`。
2. 在 `pkg/ottl/ottlfuncs/functions.go` 内的 `StandardFuncs` 或 `converters` 中注册它。
3. 创建 `pkg/ottl/ottlfuncs/func_<name>_test.go`，包含表驱动测试。
4. 运行 `go test ./ottlfuncs/...` 和 `go test ./e2e/...`。

### 添加新的遥测路径到现有上下文

1. 找到匹配的 `pkg/ottl/contexts/internal/ctx<signal>/` 包。
2. 在 `PathExpressionParser` 函数中添加匹配新字段名的新 `case`。
3. 返回 `StandardGetSetter[K]`，其中 `Getter` 读取 pdata 字段，`Setter` 写入它。
4. 在 `pkg/ottl/contexts/ottl<signal>/<signal>_test.go` 中添加测试。

### 添加新的语法 / 语言功能

1. 编辑 `grammar.go` 以添加新的 AST 节点或扩展现有节点 (例如 `value`、`mathExpression`)。
2. 更新 `expression.go` / `math.go` / `boolean_value.go` 以求值新 AST 节点。
3. 如果需要新的参数类型，更新 `functions.go`。
4. 在 `parser_test.go`、`expression_test.go`、`math_test.go` 中添加或扩展测试。
5. 使用新语法更新 `LANGUAGE.md`。

### 要遵循的关键模式

- **泛型**：所有核心类型对 `K any` (TransformContext) 都是泛型。函数也必须是泛型。
- **工厂模式**：每个函数都通过 `Factory[K]` 创建；`Arguments` 结构体在解析时被检查。
- **可选参数**：使用 `ottl.Optional[T]` 作为字段类型；必须在必需参数之后。
- **字面量优化**：如果函数的所有输入都是字面量，结果会在解析时通过 `isLiteralGetter()` / `newLiteral()` 预计算。
- **错误处理**：尊重 `ErrorMode`。函数应该返回错误而不是 panic。
- **测试**：期望正面和负面 (错误) 情况。使用表驱动测试。

### 重要的开放问题 / GSoC 机会领域

- 在 GitHub 上搜索标签为 `pkg/ottl` 的问题以获取开放功能请求。
- 上下文感知路径完成和改进的错误消息是经常性需求。
- 配置文件上下文 (`ottlprofile`) 正在积极开发中，是增长领域。
- 性能改进 (字面量预计算、内存分配减少) 总是欢迎的。

<!-- /GENERATED:entry-points -->

<!-- MANUAL:notes -->
## 我的笔记

  ---
  OTTL 架构总览

  项目定位

  OTTL 是一个嵌入 OpenTelemetry Collector 的 DSL，让运维人员在 YAML
  配置里直接表达遥测数据的变换和过滤逻辑。它是 pkg/ottl 下的独立 Go 模块，被 Transform
  Processor、Filter Processor、Tail Sampling Processor、Routing Connector 等组件复用。

  ---
  核心设计原则

  - 无内置函数、无内置路径：引擎只负责解析、类型绑定和执行循环；函数和字段路径由使用方提供。
  - 泛型驱动：所有核心类型都对 K any（TransformContext 类型）泛型化。
  - 字面量预计算：若函数的所有参数都是字面量，解析阶段就预算出结果，运行时零开销。

  ---
  关键组件

  ┌─────────┬─────────────────────┬──────────────────────────────────────────────────────────────┐
  │  层次   │        文件         │                             职责                             │
  ├─────────┼─────────────────────┼──────────────────────────────────────────────────────────────┤
  │ 语法层  │ grammar.go          │ participle 定义的词法器 + AST（editor/converter/path/boolean │
  │         │                     │ Expression/mathExpression）                                  │
  ├─────────┼─────────────────────┼──────────────────────────────────────────────────────────────┤
  │ 解析层  │ parser.go           │ Parser[K] 将字符串解析为 Statement[K] / Condition[K] /       │
  │         │                     │ ValueExpression[K]                                           │
  ├─────────┼─────────────────────┼──────────────────────────────────────────────────────────────┤
  │ 表达式  │ expression.go       │ Getter/Setter/GetSetter 接口；各类型化                       │
  │ 层      │                     │ Getter（String/Int/Float/Bool/Time…）                        │
  ├─────────┼─────────────────────┼──────────────────────────────────────────────────────────────┤
  │ 函数绑  │ functions.go        │ PathExpressionParser、Path[K]/Key[K]、反射绑定参数、Optional │
  │ 定      │                     │ [T]                                                          │
  ├─────────┼─────────────────────┼──────────────────────────────────────────────────────────────┤
  │ 工厂    │ factory.go          │ Factory[K] 接口 + CreateFactoryMap()                         │
  ├─────────┼─────────────────────┼──────────────────────────────────────────────────────────────┤
  │ 配置    │ config.go           │ ErrorMode（ignore/propagate/silent）、LogicOperation（and/or │
  │         │                     │ ）                                                           │
  ├─────────┼─────────────────────┼──────────────────────────────────────────────────────────────┤
  │ 多上下  │ parser_collection.g │ ParserCollection[R] + contextInferrer（自动推断上下文）      │
  │ 文      │ o                   │                                                              │
  └─────────┴─────────────────────┴──────────────────────────────────────────────────────────────┘

  ---
  上下文（Contexts）

  每个 OTel 信号对应一个独立包，提供 TransformContext + PathExpressionParser：

  ┌──────────────────────┬───────────────┬─────────────┐
  │         信号         │      包       │   稳定性    │
  ├──────────────────────┼───────────────┼─────────────┤
  │ Log                  │ ottllog       │ Beta        │
  ├──────────────────────┼───────────────┼─────────────┤
  │ Span                 │ ottlspan      │ Beta        │
  ├──────────────────────┼───────────────┼─────────────┤
  │ SpanEvent            │ ottlspanevent │ Beta        │
  ├──────────────────────┼───────────────┼─────────────┤
  │ Metric               │ ottlmetric    │ Beta        │
  ├──────────────────────┼───────────────┼─────────────┤
  │ DataPoint            │ ottldatapoint │ Beta        │
  ├──────────────────────┼───────────────┼─────────────┤
  │ Resource             │ ottlresource  │ Beta        │
  ├──────────────────────┼───────────────┼─────────────┤
  │ InstrumentationScope │ ottlscope     │ Beta        │
  ├──────────────────────┼───────────────┼─────────────┤
  │ Profile              │ ottlprofile   │ Development │
  └──────────────────────┴───────────────┴─────────────┘

  ---
  内置函数库（ottlfuncs）

  Editors（就地修改）：set, delete_key, delete_matching_keys, keep_keys, keep_matching_keys, flatten,
   limit, merge_maps, replace_pattern, replace_match, replace_all_*, truncate_all, append, XML 系列

  Converters（返回新值，共 ~100 个）：类型转换、哈希（sha256/md5/fnv/xxh3/murmur3）、编码（base64/hex
  ）、字符串处理、时间处理、JSON/CSV/XML 解析、URL/UserAgent/CommunityID、UUID、集合操作、断言谓词等

  ---
  执行流程

  NewParser[K] → ParseStatement(raw)
    → participle 词法 + 解析 → parsedStatement AST
    → newFunctionCall → 反射绑定参数 → ExprFunc[K]
    → newBoolExpr → boolExpr[K]
    → Statement[K]

  Statement.Execute(ctx, tCtx)
    → condition.Eval → 比较值
    → 若 true → Expr.Eval → 调用函数
      → 函数内部通过 Getter.Get() 读 pdata 字段
      → 通过 Setter.Set() 写回 pdata 字段

  ---
  GSoC 贡献入口建议

  1. 新增函数：在 ottlfuncs/ 中按 func_set.go 模式添加，注册到 functions.go
  2. 新增路径：在 contexts/internal/ctx<signal>/ 中添加字段映射
  3. 语法扩展：改 grammar.go + 对应评估逻辑 + 更新 LANGUAGE.md
  4. Profile 上下文：ottlprofile 处于 Development 阶段，是活跃开发区域
  5. 性能优化：字面量预计算、减少内存分配等


<!-- /MANUAL:notes -->
