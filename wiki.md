# OpenTelemetry 转换语言 (OTTL) 中文翻译

## OpenTelemetry 转换语言 (OTTL)

OpenTelemetry 转换语言 (OTTL) 提供了一种领域特定语言 (DSL)，用于在收集器内处理 OpenTelemetry 数据，能够在各种信号(如日志、指标、跟踪和配置文件)上进行遥测数据的变异和生成。该语言设计灵活，允许用户定义精确的规则来操纵遥测数据，而无需修改底层组件代码。

OTTL 语句作为执行的基本单位，将函数调用与可选的条件逻辑相结合。这些语句可以应用编辑器(Editor)，直接修改遥测负载，或利用转换器(Converter)，将数据格式转换为供参数或条件使用。例如，OTTL 语句可能仅在特定跨度属性尚不存在时才设置它，或根据日志内容过滤日志。该语言支持各种数据类型和结构，包括对遥测字段的直接引用(路径)、列表、映射、字面量(字符串、整数、浮点数、布尔值、nil、字节切片)、枚举和数学表达式。这些用于与转换上下文(TransformContext)内的遥测数据交互，转换上下文为读取和写入数据提供 Getter、Setter 和 GetSetter 接口。

OTTL 的设计强调可扩展性和解耦解释。OTTL 没有内置的编辑器或转换器，而是依赖用户提供字符串标识符与其相应实现之间的映射。类似地，路径(对遥测字段的引用)和枚举的解释被委托给用户提供的解析器。这种方法允许 OTTL 的核心语法保持独立于特定的遥测数据模型，便于其跨 OpenTelemetry 收集器的不同组件(如转换处理器、过滤处理器、尾部采样处理器和路由连接器)进行集成。错误处理可配置为忽略、传播或静默处理错误，同时支持逻辑操作(如 AND 和 OR)用于条件处理。

有关核心语言和其执行的更多详细信息，请参阅 OTTL 核心语言和执行。有关 OTTL 如何与特定数据模型集成的信息，请参阅 OTTL 上下文和数据模型集成。可用函数及其用法在 OTTL 标准库：函数和编辑器中有详细说明。有关综合测试策略，请参阅 OTTL 端到端测试。OTTL 的贡献指南在 pkg/ottl/CONTRIBUTING.md 中概述。

---

## OTTL 核心语言和执行

OpenTelemetry 转换语言 (OTTL) 提供了一种用于修改和生成遥测数据的领域特定语言 (DSL)。在其核心，OTTL 语句由一个转换遥测的函数和一个可选的 where 子句组成，该子句指定函数何时应执行。这使得条件处理和针对性的数据操纵成为可能。OTTL 设计用于跨越各种 OpenTelemetry 信号(包括日志、指标、跟踪和配置文件)进行操作，每个都有特定的 TransformContext 实现。

OTTL 的语法和结构在 pkg/ottl/grammar.go 中定义，它构造一个抽象语法树 (AST) 来表示语句、表达式和字面量。这个基础支持多种类型的表达式评估：

**布尔表达式**处理 AND、OR 和 NOT 等逻辑操作，也包括比较操作。该系统采用了短路求值和字面布尔值预评估等优化方法，提高了效率，详见 pkg/ottl/boolean_value.go。

**比较操作**支持跨多种数据类型(包括原始类型、字节切片和与时间相关的类型)的相等性、不相等性和顺序比较(<、>、<=、>=)。数值比较时，如果类型不同，整数被提升为浮点数。对于映射和切片等复杂类型，比较通常限于相等性检查。pkg/ottl/compare.go 中的 ValueComparator 接口提供了这些操作的机制。

**数学表达式**支持算术操作(+、-、*、/)并遵循标准操作符优先级。这些表达式支持 int64、float64、time.Time 和 time.Duration 类型，有特定的处理方式防止除以零等问题，并确保混合类型操作时的类型一致性。

动态数据访问和修改是 OTTL 功能的核心。pkg/ottl/expression.go 中的 Expr、Getter、Setter 和 GetSetter 接口便于灵活地检索和操纵遥测数据中的值。PathExpressionParser 将 OTTL 路径(例如 metric.name)转换为具体方法，用于访问 TransformContext 内的数据，从而抽象了底层数据模型。该系统还包括通过 pkg/ottl/literal_expr.go 中的 literalExpr 和 pkg/ottl/literal_getter.go 中的 literalGetter 将字面量值作为表达式处理的机制。

OTTL 语句的执行流程涉及通过工厂进行函数管理，工厂创建并组织 OTTL 函数。这些由 pkg/ottl/factory.go 管理的工厂能够通过解析参数和解决路径来动态创建可执行函数。错误处理行为可通过 ErrorMode 设置进行配置，允许灵活地响应处理期间的问题，如忽略错误、传播错误或静默处理错误，如 pkg/ottl/config.go 中所定义。整体设计强调可扩展性，依赖用户提供的编辑器和转换器映射，而非内置实现。

---

## OTTL 上下文和数据模型集成

OpenTelemetry 转换语言 (OTTL) 通过特定的 TransformContext 实现与各种 OpenTelemetry 数据模型(日志、指标、跟踪和配置文件)进行集成。每个 TransformContext 充当适配器，允许 OTTL 表达式访问和操纵这些数据结构中的字段。这种模块化使得 OTTL 能够跨不同的遥测信号均匀地应用转换。

TransformContext 管理正在处理的数据模型(如日志记录、指标、跨度或配置文件)，并通常包括转换期间的临时数据缓存。将 OTTL 表达式映射到具体数据结构的过程依赖于两个关键组件：PathExpressionParser 和 EnumParser。

**PathExpressionParser** 解释 OTTL 路径表达式，如 metric.name。对于每个数据模型，该解析器的特定实现了解如何导航该模型的内部结构并返回 ottl.GetSetter。这个 ottl.GetSetter 提供了检索或修改指定字段值的机制。例如，数据点上下文使用 PathGetSetter 来访问 datapoint.value_double 或 datapoint.attributes 等字段，如 pkg/ottl/contexts/ottldatapoint 文档所述。类似地，日志上下文的 PathGetSetter 支持与 log.timestamp、log.severity_number 和 log.body 的交互，如 pkg/ottl/contexts/ottllog 中详细说明。指标上下文通过其 PathGetSetter 提供对 metric.name 和 metric.aggregation_temporality 等属性的访问，如 pkg/ottl/contexts/ottlmetric 文档所述。配置文件上下文(如 pkg/ottl/contexts/ottlprofile 所述)使用其自己的 PathGetSetter 来处理 profile.sample_type 和 profile.profile_id 等字段。

**EnumParser** 将 OTTL 枚举符号(表示枚举值的字符串，如"SEVERITY_NUMBER_INFO")转换为数据模型的相应内部枚举值。这允许 OTTL 表达式以人类可读的格式引用预定义常量，如日志严重性级别或指标聚合时间。例如，日志上下文定义了一个 SymbolTable 来映射严重性数字，而指标上下文则映射指标类型和聚合时间。

每个 TransformContext 通常使用对象池进行管理，如 pkg/ottl/contexts/ottldatapoint 和 pkg/ottl/contexts/ottllog 中 sync.Pool 对 tcPool 的使用所示。这种池化机制(使用 NewTransformContextPtr 和 Close 等函数)通过重用上下文实例和减少内存分配开销来优化性能。

OpenTelemetry 数据模型的各种 TransformContext 实现位于 pkg/ottl/contexts 下。这个目录结构，包括其内部子目录如 pkg/ottl/contexts/internal，集中了 OTTL 上下文管理和跨不同遥测类型的数据操纵的核心功能。

---

## OTTL 标准库：函数和编辑器

OpenTelemetry 转换语言 (OTTL) 提供了一套完整的内置函数，用于在 OpenTelemetry 收集器内操纵和转换遥测数据。这些函数主要分为两类："编辑器(Editors)"和"转换器(Converters)"，反映了它们不同的操作特性。编辑器修改遥测数据的过程中可能产生副作用，而转换器是不会改变输入遥测的纯函数。这些函数的设计原则强调安全性、稳定性和确定性，避免文件系统或网络访问，并确保可预测的行为。

### 编辑器函数

编辑器用于直接修改遥测属性，如追加、删除或替换值。例如，append 函数允许向目标字段添加单个或多个值，处理各种原始和 pcommon 类型，必要时将标量字段转换为数组。此功能得到 pkg/ottl/ottlfuncs/func_append.go 中代码的支持。类似地，函数如 delete_index(定义于 pkg/ottl/ottlfuncs/func_delete_index.go)、delete_key(定义于 pkg/ottl/ottlfuncs/func_delete_key.go)和 delete_matching_keys(定义于 pkg/ottl/ottlfuncs/func_delete_matching_keys.go)能够基于索引、特定键或正则表达式模式精确地从类似映射的结构或切片中移除元素或键。pkg/ottl/ottlfuncs/README.md 中描述的其他编辑器函数(未在提供的上下文中详细介绍)包括 keep_matching_keys、flatten、keep_keys、limit、merge_maps、replace_all_matches、replace_all_patterns、replace_match、replace_pattern、set 和 truncate_all。

### 转换器函数

转换器提供实用操作，不修改遥测数据。它们用于类型转换、字符串操纵、哈希、时间操作和解析各种数据格式。例如，Base64Decode(定义于 pkg/ottl/ottlfuncs/func_base64decode.go)解码 Base64 编码的字符串(现已被 Decode 弃用)，Bool(定义于 pkg/ottl/ottlfuncs/func_bool.go)从各种输入类型提取布尔值。一个值得注意的转换器是 CommunityID(定义于 pkg/ottl/ottlfuncs/func_community_id.go)，它根据源/目标 IP 地址、端口和协议为网络流生成唯一标识符。该函数规范化流方向并使用 SHA-1 哈希和 Base64 编码来生成一致的标识符，如 pkg/ottl/ottlfuncs/func_community_id.go 中所述。pkg/ottl/ottlfuncs/README.md 中列出的其他转换器函数涵盖了广泛的操作，如字符串连接(Concat)、检查切片中值的存在(ContainsValue)、案例转换(ConvertCase)、XML 操纵(ConvertAttributesToElementsXML、ConvertTextToElementsXML)、提取时间分量(Day、Hour、Minute、Month、Nanosecond、Second、Weekday、Year)和数值转换(Double、Int)。哈希函数(如 FNV、MD5、Murmur3Hash、Murmur3Hash128、SHA1、SHA256、SHA512、XXH3、XXH128)可用于数据完整性检查。格式化函数(Format、FormatTime)、字符串实用函数(HasPrefix、HasSuffix、Hex、Index、Split、Substring、Trim、TrimPrefix、TrimSuffix)、类型检查(IsBool、IsDouble、IsInt、IsList、IsMap、IsString)以及 CSV、JSON、键值对、XML 和用户代理的解析函数也是标准库的一部分。

---

## OTTL 端到端测试

OpenTelemetry 转换语言 (OTTL) 的端到端测试框架验证了其在各种遥测信号(包括日志、跨度、跨度事件和配置文件)上的功能。该框架确保 OTTL 语句正确转换遥测数据，涵盖从属性操纵和类型转换到高级功能(如 where 子句和动态索引)的广泛操作。

### 测试类别

**编辑器函数测试** 验证修改日志记录、跨度、跨度事件和配置文件中属性的操作的正确行为，如插入、删除、替换或展平键值对。

**转换器函数测试** 涵盖设计用于转换数据类型或格式的函数，此类别包括字符串操纵、类型转换、哈希、解析和数据验证函数。

**高级功能测试** 检查各种 OTTL 功能的集成，包括条件语句(where 子句)、动态属性索引、枚举用法、字符串实用函数(HasPrefix、HasSuffix)、十六进制字面量解释和组合多个函数。

**语句序列化测试** 确保按顺序执行的多个 OTTL 语句产生预期的累积效果，包括验证后续操作如何与先前修改的数据交互。

**值表达式测试** 关注 OTTL 内各种值表达式的评估，包括验证字符串字面量、属性值、枚举值和转换器函数结果的正确解释。

### 辅助函数

辅助函数通过创建带有示例遥测数据的初始化转换上下文并将转换后的输出与预期状态进行比较来简化测试过程。这些比较确保实际转换的遥测与所需的结果相匹配。测试框架还包括 Goroutine 泄漏检测，以确保测试执行期间的适当资源管理，如 pkg/ottl/e2e/package_test.go 和 pkg/ottl/ottltest/package_test.go 中所配置。pkg/ottl/ottltest/ottltest.go 中提供的原始类型指针创建实用工具简化了测试用例的构建。
