---
layout: post
title: "告警引擎表达式引擎从词法分析到AST求值"
date: 2026-01-04
categories: [golang, 架构]
---

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

## 二、语法解析（Parser）

有了 Token 序列，Parser 通过递归下降算法将 Token 流转换为 AST。递归下降解析器的核心思想是：为每种语法结构定义对应的解析函数，函数之间相互递归调用。

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
```

### 核心语法解析

#### 1. 选择器解析

选择器是 Prometheus 风格的指标查询，例如 `cpu_usage{env="production", zone="cn-north-1"}`。解析流程：

1. 读取指标名（标识符）
2. 如果下一个 Token 是 `{`，解析标签匹配器
3. 如果下一个 Token 是 `[`，解析范围（如 `[5m]`）

#### 2. 函数调用解析

函数调用的语法是 `func_name(arg1, arg2) [by (...) | without (...)]`。

当 Parser 识别到 `name(` 模式时，调用 `parseFunctionCall`。该函数：

1. 解析参数列表（递归调用 `parseLogical`）
2. 解析分组修饰符（by/without）
3. 如果函数名是聚合函数（avg, sum, max, min, count, stddev, stdvar, topk, bottomk, quantile），则创建 `AggregationNode`
4. 否则创建 `FunctionNode`

#### 3. 标签匹配器解析

标签匹配器语法为 `{ label = "value", ... }`。解析时：

- 期望键为标识符
- 期望 `=` 符号
- 期望字符串值
- 逗号分隔多个匹配器

## 三、AST 节点定义

AST（抽象语法树）由一组 Node 接口的实现类构成。每个 Node 实现两个方法：

- `Type() NodeType`：返回节点类型
- `String() string`：返回可调试的字符串表示

### 节点类型

| 节点类型 | 说明 | ��例 |
|----------|------|------|
| `SelectorNode` | 选择器 | `cpu_usage{env="prod"}` |
| `AggregationNode` | 聚合操作 | `sum() by (service)` |
| `FunctionNode` | 函数调用 | `rate(cpu[5m])` |
| `BinaryOpNode` | 二元算术运算 | `a + b` |
| `ComparisonNode` | 比较运算 | `x > 80` |
| `LogicalOpNode` | 逻辑运算 | `a and b` |
| `NumberNode` | 数字常量 | `80` |
| `ParenNode` | 括号表达式 | `(a + b)` |

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

选择器是表达式求值的关键节点。它需要将 AST 中的选择器转换为 VM 的 PromQL 查询请求。关键设计点是：**选择器求值时，并不把标签匹配器（label matchers）拼接到查询表达式中发送给 VM**，而是在 VM 返回结果后，在本地进行标签过滤。

这种设计的原因是：VictoriaMetrics 的 `/query` 和 `/query_range` API 对标签过滤的语法与我们在 AST 中存储的格式不完全一致——我们的 AST 中标签是独立的 map，而 VM 的 PromQL 在指标名后直接用 `{}` 语法。为了简化 VM 客户端的实现，我们在 VM 层面查询更宽泛的数据（只查指标名），然后在 Engine 层精确过滤。

### 聚合操作求值

聚合节点（如 `sum() by (service)`）的求值流程：

1. 递归求值子节点（被聚合的表达式），得到一组样本
2. 按 by/without 标签进行分组
3. 对每个分组应用聚合函数（sum/avg/max/min/count/stddev/stdvar）

### 二元运算符求值

二元运算符（+, -, *, /）的求值需要处理 **标签对齐**。PromQL 的二元运算遵循"标签集合相等才能参与运算"的规则。

### 比较运算符求值

比较运算符（>, <, >=, <=, ==, !=）与二元运算符类似，也需要标签对齐。但区别在于：**比较运算只返回满足条件的样本**。

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

## 总结

表达式引擎是 alert-engine 的"大脑"。它将用户编写的类 PromQL 表达式经过词法分析、语法解析，转换为 AST，再通过递归求值，最终产出用于告警判断的样本数据。

核心设计要点：

- **递归下降解析器**：简洁直观，优先级的处理通过多层解析函数实现
- **标签对齐语义**：二元运算和比较运算遵循 PromQL 的标签对齐规则
- **VM 查询代理**：选择器节点不拼接复杂标签，而是查询后本地过滤，降低 VM 客户端复杂度
- **函数注册表模式**：通过 FunctionRegistry 支持运行时函数注册，便于扩展