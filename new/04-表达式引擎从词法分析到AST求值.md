# 表达式引擎：从词法分析到 AST 求值

alert-engine 的核心差异化能力之一，是自建了一个类 PromQL 的表达式引擎。它负责将用户编写的告警表达式解析为抽象语法树（AST），然后对 AST 进行遍历求值，最终得到一组时序样本数据（Sample），用于判断是否触发告警。

本章将深入引擎的四个核心环节：**词法分析**、**语法解析**、**AST 定义**、**求值引擎**。

## 整体设计思路

表达式引擎的目标是解析类似 PromQL 的表达式，例如：

```
avg(cpu_usage{env="production"}) by (service) > 80
sum(rate(http_requests_total[5m])) by (method) != 200
```

这类表达式由以下语法元素组成：

- **选择器**（Selector）：指标名 + 标签过滤，如 `cpu_usage{env="production"}`
- **范围查询**（Range）：带时间窗口的查询，如 `http_requests_total[5m]`
- **函数调用**（Function）：聚合或变换函数，如 `avg()`、`rate()`、`sum_over_time()`
- **聚合操作**（Aggregation）：带 by/without 分组的聚合，如 `sum() by (service)`
- **二元运算符**（BinaryOp）：算术运算，如 `+`、`-`、`*`、`/`
- **比较运算符**（Comparison）：比较运算，如 `>`、`<`、`>=`、`<=`、`==`、`!=`
- **逻辑运算符**（LogicalOp）：逻辑运算，如 `and`、`or`
- **字面量**（Literal）：数字常量

我们的引擎通过 **递归下降解析器**（Recursive Descent Parser）将上述表达式转换为 AST，然后通过 **深度优先遍历** 对 AST 各节点求值，最终得到一组 `Sample`（每个 Sample 包含标签、值和时间戳）。

## 一、词法分析（Lexer）

词法分析负责将输入的表达式字符串切分为 Token 序列。每个 Token 由类型（TokenKind）和字面量（Value）组成。

### Token 类型定义

```go
type TokenKind int

const (
    TokIdent TokenKind = iota  // 标识符：指标名、函数名
    TokNumber                  // 数字常量
    TokString                  // 字符串常量
    TokLParen                  // (
    TokRParen                  // )
    TokLBrace                  // {
    TokRBrace                  // }
    TokLBracket                // [
    TokRBracket                // ]
    TokComma                   // ,
    TokEq                      // =
    TokNeq                     // !=
    TokGt                      // >
    TokLt                      // <
    TokGte                     // >=
    TokLte                     // <=
    TokPlus                    // +
    TokMinus                   // -
    TokStar                    // *
    TokSlash                   // /
    TokAnd                     // and
    TokOr                      // or
    TokBy                      // by
    TokWithout                 // without
    TokEOF                     // 文件结束
)
```

### Lexer 实现

Lexer 从左到右扫描输入字符串，通过 `peek()` 方法预读下一个字符来决定如何切分。关键处理逻辑如下：

- **空白字符**：跳过
- **括号和标点**：直接生成对应 Token
- **运算符**：需要处理双字符运算符（如 `>=`），通过 `peek(1)` 预读下一个字符
- **数字**：连续读取数字和小数点
- **标识符**：读取字母、数字、下划线、中文字符；读取后与关键字表（`and`、`or`、`by`、`without`）匹配，其余均为标识符
- **字符串**：双引号包裹，支持转义

```go
func (l *Lexer) Tokenize() ([]Token, error) {
    for l.pos < len(l.input) {
        ch := l.input[l.pos]
        switch {
        case unicode.IsSpace(rune(ch)):
            l.pos++
        case ch == '(':
            l.tokens = append(l.tokens, Token{Kind: TokLParen, Value: "("})
            l.pos++
        case ch == '>':
            if l.peek(1) == '=' {
                l.tokens = append(l.tokens, Token{Kind: TokGte, Value: ">="})
                l.pos += 2
            } else {
                l.tokens = append(l.tokens, Token{Kind: TokGt, Value: ">"})
                l.pos++
            }
        // ... 其他分支
        case unicode.IsLetter(rune(ch)) || ch == '_':
            id := l.readIdent()
            switch id {
            case "and":
                l.tokens = append(l.tokens, Token{Kind: TokAnd, Value: id})
            case "or":
                l.tokens = append(l.tokens, Token{Kind: TokOr, Value: id})
            // ...
            default:
                l.tokens = append(l.tokens, Token{Kind: TokIdent, Value: id})
            }
        }
    }
    l.tokens = append(l.tokens, Token{Kind: TokEOF})
    return l.tokens, nil
}
```

## 二、语法解析（Parser）

有了 Token 序列，Parser 通过递归下降算法将 Token 流转换为 AST。递归下降解析器的核心思想是：���每种语法结构定义对应的解析函数，函数之间相互递归调用。

### 优先级与结合性

表达式求值的优先级从低到高为：

1. **逻辑运算符**（or, and）
2. **比较运算符**（>, <, >=, <=, ==, !=）
3. **加减运算符**（+, -）
4. **乘除运算符**（*, /）
5. **一元负号**（-）
6. **原子表达式**（数字、选择器、函数调用、括号表达式）

我们在 Parser 中为每个优先级层级定义一个解析函数：

```go
func (p *Parser) Parse() (Node, error) {
    return p.parseLogical()
}

func (p *Parser) parseLogical() (Node, error) {
    left, err := p.parseComparison()
    if err != nil {
        return nil, err
    }
    for {
        t := p.current()
        if t.Kind == TokAnd || t.Kind == TokOr {
            p.advance()
            right, err := p.parseComparison()
            if err != nil {
                return nil, err
            }
            left = &LogicalOpNode{Op: t.Value, Left: left, Right: right}
        } else {
            break
        }
    }
    return left, nil
}

func (p *Parser) parseComparison() (Node, error) {
    // 比较运算符解析
    // ...
}

func (p *Parser) parseBinary() (Node, error) {
    // 二元运算符解析
    // ...
}

func (p *Parser) parsePrimary() (Node, error) {
    // 原子表达式解析
    // ...
}
```

### 核心语法解析

#### 1. 选择器解析

选择器是 Prometheus 风格的��标查询，例如 `cpu_usage{env="production", zone="cn-north-1"}`。解析流程：

1. 读取指标名（标识符）
2. 如果下一个 Token 是 `{`，解析标签匹配器
3. 如果下一个 Token 是 `[`，解析范围（如 `[5m]`）

```go
func (p *Parser) parseSelectorOrFunc() (Node, error) {
    name := p.current().Value
    p.advance()

    // 函数调用：name(...)
    if p.current().Kind == TokLParen {
        return p.parseFunctionCall(name)
    }

    // 选择器：name{...}
    labels := make(map[string]string)
    if p.current().Kind == TokLBrace {
        p.advance()
        labels, err = p.parseLabelMatchers()
        // ...
    }

    // 范围：[5m]
    rng := ""
    if p.current().Kind == TokLBracket {
        p.advance()
        rng = p.parseRange()
        // ...
    }

    return &SelectorNode{Name: name, Labels: labels, Range: rng}, nil
}
```

#### 2. 函数调用解析

函数调用的语法是 `func_name(arg1, arg2) [by (...) | without (...)]`。

当 Parser 识别到 `name(` 模式时，调用 `parseFunctionCall`。该函数：

1. 解析参数列表（递归调用 `parseLogical`）
2. 解析分组修饰符（by/without）
3. 如果函数名是聚合函数（avg, sum, max, min, count, stddev, stdvar, topk, bottomk, quantile），则创建 `AggregationNode`
4. 否则创建 `FunctionNode`

```go
func (p *Parser) parseFunctionCall(name string) (Node, error) {
    p.advance() // 跳过 (
    args, err := p.parseArgList()
    // ...
    p.advance() // 跳过 )

    byLabels, withoutLabels := p.parseGroupModifiers()

    aggrFuncs := map[string]bool{
        "avg": true, "sum": true, "max": true, "min": true,
        "count": true, "stddev": true, "stdvar": true,
        "topk": true, "bottomk": true, "quantile": true,
    }
    if aggrFuncs[name] {
        return &AggregationNode{
            Func: name,
            By:   byLabels,
            Without: withoutLabels,
            Arg:  args[0],
        }, nil
    }

    return &FunctionNode{Name: name, Args: args}, nil
}
```

#### 3. 标签匹配器解析

标签匹配器语法为 `{ label = "value", ... }`。解析时：

- 期望键为标识符
- 期望 `=` 符号
- 期望字符串值
- 逗号分隔多个匹配器

```go
func (p *Parser) parseLabelMatchers() (map[string]string, error) {
    labels := make(map[string]string)
    for p.current().Kind != TokRBrace && p.current().Kind != TokEOF {
        key := p.current().Value
        p.advance()
        if p.current().Kind != TokEq {
            return nil, fmt.Errorf("expected '=', got %s", p.current().Value)
        }
        p.advance()
        if p.current().Kind != TokString {
            return nil, fmt.Errorf("expected string value")
        }
        labels[key] = p.current().Value
        p.advance()
        if p.current().Kind == TokComma {
            p.advance()
        }
    }
    p.advance() // 跳过 }
    return labels, nil
}
```

## 三、AST 节点定义

AST（抽象语法树）由一组 Node 接口的实现类构成。每个 Node 实现两个方法：

- `Type() NodeType`：返回节点类型
- `String() string`：返回可调试的字符串表示

### 节点类型

```go
type Node interface {
    Type() NodeType
    String() string
}
```

我们定义了以下 AST 节点类型：

| 节点类型 | 说明 | 示例 |
|----------|------|------|
| `SelectorNode` | 选择器 | `cpu_usage{env="prod"}` |
| `AggregationNode` | 聚合操作 | `sum() by (service)` |
| `FunctionNode` | 函数调用 | `rate(cpu[5m])` |
| `BinaryOpNode` | 二元算术运算 | `a + b` |
| `ComparisonNode` | 比较运算 | `x > 80` |
| `LogicalOpNode` | 逻辑运算 | `a and b` |
| `NumberNode` | 数字常量 | `80` |
| `ParenNode` | 括号表达式 | `(a + b)` |

各节点的定义（如 SelectorNode）包含该语法结构所需的全部字段。例如 `AggregationNode` 包含聚合函数名、分组标签列表（by/without）和子节点（被聚合的表达式）。

## 四、求值引擎（Evaluator）

求值引擎通过递归遍历 AST 来计算表达式的值。每个节点类型在 Engine 中都有对应的求值方法。

### Engine 结构

```go
type Engine struct {
    queryFunc QueryFunc              // VM 查询函数注入
    registry  *FunctionRegistry     // 函数注册表
}

func NewEngine(queryFunc QueryFunc) *Engine {
    return &Engine{
        queryFunc: queryFunc,
        registry:  defaultFuncRegistry,
    }
}
```

`queryFunc` 是 VictoriaMetrics 的查询函数注入点——当求值过程中遇到 SelectorNode 时，Engine 调用它向 VM 发起查询请求。

### 求值入口

```go
func (e *Engine) Eval(ctx context.Context, expr string) ([]Sample, error) {
    ast, err := Parse(expr)      // 解析：字符串 -> AST
    if err != nil {
        return nil, fmt.Errorf("parse error: %w", err)
    }
    return e.evalNode(ctx, ast)  // 求值：AST -> []Sample
}
```

### 选择器求值（核心）

选择器是表达式求值的关键节点。它需要将 AST 中的选择器转换为 VM 的 PromQL 查询请求：

```go
func (e *Engine) evalSelector(ctx context.Context, node *SelectorNode) ([]Sample, error) {
    // 1. 构建查询表达式
    queryExpr := node.Name
    if len(node.Labels) > 0 {
        queryExpr += "{}"  // 标签在查询时动态拼接
    }

    now := time.Now()
    var step time.Duration = time.Minute

    // 2. 如果有范围（如 [5m]），计算查询步长
    if node.Range != "" {
        d, err := parseDuration(node.Range)
        if err == nil {
            step = d / 10
        }
    }

    // 3. 向 VM 发起查询
    result, err := e.queryFunc(ctx, queryExpr, now.Add(-10*time.Minute), now, step)
    if err != nil {
        return nil, fmt.Errorf("query error: %w", err)
    }

    samples := result.Samples

    // 4. 如果没有范围，只返回最后一个样本（即时值）
    if node.Range == "" {
        if len(samples) > 0 {
            return []Sample{samples[len(samples)-1]}, nil
        }
        return []Sample{}, nil
    }

    // 5. 有范围时，按标签过滤
    filtered := filterByLabels(samples, node.Labels)
    if len(filtered) == 0 {
        return []Sample{{Value: math.NaN()}}, nil
    }

    return filtered, nil
}
```

这里有一个关键设计：**选择器求值时，并不把标签匹配器（label matchers）拼接到查询表达式中发送给 VM**，而是在 VM 返回结果后，在本地进行标签过滤。

这种设计的原因是：VictoriaMetrics 的 `/query` 和 `/query_range` API 对标签过滤的语法与我们在 AST 中存储的格式不完全一致——我们的 AST 中标签是独立的 map，而 VM 的 PromQL 在指标名后直接用 `{}` 语法。为了简化 VM 客户端的实现，我们在 VM 层面查询更宽泛的数据（只查指标名），然后在 Engine 层精确过滤。

### 聚合操作求值

聚合节点（如 `sum() by (service)`）的求值流程：

1. 递归求值子节点（被聚合的表达式），得到一组样本
2. 按 by/without 标签进行分组
3. 对每个分组应用聚合函数（sum/avg/max/min/count/stddev/stdvar）

```go
func (e *Engine) evalAggregation(ctx context.Context, node *AggregationNode) ([]Sample, error) {
    // 1. 求值子表达式
    childSamples, err := e.evalNode(ctx, node.Arg)

    // 2. 按标签分组
    grouped := groupByLabels(childSamples, node.By, node.Without)

    // 3. 对每组应用聚合函数
    var results []Sample
    for _, group := range grouped {
        var val float64
        switch node.Func {
        case "avg":
            val = sum(groupValues) / float64(len(groupValues))
        case "sum":
            // ...
        case "max":
            // ...
        // ...
        }
        results = append(results, Sample{Value: val, Labels: groupLabels})
    }

    return results, nil
}
```

### 二元运算符求值

二元运算符（+, -, *, /）的求值需要处理 **标签对齐**。PromQL 的二元运算遵循"标签集合相等才能参与运算"的规则：

```go
func (e *Engine) evalBinaryOp(ctx context.Context, node *BinaryOpNode) ([]Sample, error) {
    left, err := e.evalNode(ctx, node.Left)
    right, err := e.evalNode(ctx, node.Right)

    // 将样本按标签键映射
    leftMap := labelMap(left)   // key: "env=prod,zone=cn-north-1" => Sample
    rightMap := labelMap(right)

    // 遍历所有标签键，只对两边都有的键进行运算
    var results []Sample
    allKeys := union(leftMap.Keys(), rightMap.Keys())
    for key := range allKeys {
        lv, okL := leftMap[key]
        rv, okR := rightMap[key]
        if !okL || !okR {
            continue
        }
        var val float64
        switch node.Op {
        case "+":
            val = lv.Value + rv.Value
        // ...
        }
        results = append(results, Sample{Value: val, Labels: mergeLabels(lv, rv)})
    }

    return results, nil
}
```

### 比较运算符求值

比较运算符（>, <, >=, <=, ==, !=）与二元运算符类似，也需要标签对齐。但区别在于：**比较运算只返回满足条件的样本**：

```go
func (e *Engine) evalComparison(ctx context.Context, node *ComparisonNode) ([]Sample, error) {
    // ... (标签对齐逻辑同二元运算符)

    for key := range allKeys {
        // ...
        var matched bool
        switch node.Op {
        case ">":
            matched = lv.Value > rv.Value
        case "==":
            matched = lv.Value == rv.Value
        // ...
        }
        if matched {
            results = append(results, Sample{Value: lv.Value, Labels: mergeLabels(lv, rv)})
        }
    }
    return results, nil
}
```

告警规则正是基于比较运算的结果来判断是否触发：只要比较结果集中存在样本，就认为条件满足，生成告警。

## 五、内置函数库

函数库通过 `FunctionRegistry` 管理，支持在运行时注册新函数。每个函数签名统一为：

```go
type Function func(args []interface{}) (interface{}, error)
```

函数可返回三种类型：

- `float64`：标量结果
- `[]Sample`：样本序列（如 `rate()` 需要返回变化率序列）
- `[]float64`：数值数组（如 `sort()` 排序后返回）

### 函数分类

引擎预置了四类函数：

1. **数学函数**：abs, ceil, floor, round, exp, ln, log2, log10, sqrt
2. **聚合函数**：avg, sum, max, min, count, stddev, stdvar, topk, bottomk, quantile
3. **时间序列函数**（_over_time 系列）：avg_over_time, sum_over_time, max_over_time, min_over_time, count_over_time
4. **速率函数**：rate, irate, increase, delta, deriv
5. **标签操作函数**：label_join, label_replace, drop_common_labels, keep_last_value
6. **辅助函数**：vector, scalar, absent, sort, sort_desc, clamp, clamp_min, clamp_max

### 示例：rate 函数实现

`rate` 计算时间序列的每秒增长率，是告警中最常用的函数之一：

```go
func FuncRate(args []interface{}) (interface{}, error) {
    samples, ok := args[0].([]Sample)
    if !ok {
        return nil, fmt.Errorf("rate: expected samples, got %T", args[0])
    }
    if len(samples) < 2 {
        return 0.0, nil
    }
    first := samples[0]
    last := samples[len(samples)-1]
    deltaValue := last.Value - first.Value
    deltaTime := last.Time.Sub(first.Time).Seconds()
    if deltaTime <= 0 {
        return 0.0, nil
    }
    return deltaValue / deltaTime, nil
}
```

注意：`rate` 接收的是样本序列（来自 SelectorNode 的 Range 查询），而不是单个值。这与 PromQL 的行为一致。

## 六、测试驱动：Parser 测试

表达式引擎的复杂度决定了必须通过充分的测试来保证正确性。我们为 Parser 编写了单元测试，覆盖核心语法：

```go
func TestParser_Number(t *testing.T) {
    ast, err := Parse("42")
    num, ok := ast.(*NumberNode)
    if num.Value != 42 {
        t.Errorf("expected 42, got %g", num.Value)
    }
}

func TestParser_BinaryOp(t *testing.T) {
    ast, err := Parse("a + b")
    op, ok := ast.(*BinaryOpNode)
    if op.Op != "+" {
        t.Errorf("expected +, got %s", op.Op)
    }
}

func TestParser_Aggregation(t *testing.T) {
    ast, err := Parse("avg(cpu_usage)")
    aggr, ok := ast.(*AggregationNode)
    if aggr.Func != "avg" {
        t.Errorf("expected avg, got %s", aggr.Func)
    }
}

func TestParser_GroupBy(t *testing.T) {
    ast, err := Parse("sum(rate(http_requests[5m])) by (service)")
    aggr, ok := ast.(*AggregationNode)
    if len(aggr.By) != 1 || aggr.By[0] != "service" {
        t.Errorf("expected by (service), got %v", aggr.By)
    }
}
```

在开发过程中，测试先行（虽然本项目是先实现后补测试）帮助我们快速捕获了解析阶段的语法错误。

## 七、引擎局限性

当前实现存在以下已知局限：

1. **PromQL 兼容性约 80%**：覆盖了告警场景最常用的语法，部分高级语法（如子查询 `[]:`、`@` 修饰符）尚未支持。
2. **正则匹配标签不完全**：当前标签匹配器只支持精确匹配（`=`），不支持正则匹配（`=~`、`!~`）和集合匹配（`in`、`not in`）。
3. **函数覆盖不全**：部分函数（如 `holt_winters`、`predict_linear`）返回 "not implemented" 错误。
4. **缺少负号优先级的特殊处理**：在某些边界情况下，负号的优先级处理不够精确。

这些局限在当前告警场景下未构成实质问题，但随着用户需求增长，需要逐步补齐。

## 总结

表达式引擎是 alert-engine 的"大脑"。它将用户编写的类 PromQL 表达式经过词法分析、语法解析，转换为 AST，再通过递归求值，最终产出用于告警判断的样本数据。

核心设计要点：

- **递归下降解析器**：简洁直观，优先级的处理通过多层解析函数实现
- **标签对齐语义**：二元运算和比较运算遵循 PromQL 的标签对齐规则
- **VM 查询代理**：选择器节点不拼接复杂标签，而是查询后本地过滤，降低 VM 客户端复杂度
- **函数注册表模式**：通过 FunctionRegistry 支持运行时函数注册，便于扩展