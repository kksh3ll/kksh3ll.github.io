---
layout: post
title: "告警引擎调度器与告警状态管理"
date: 2026-01-05
categories: [golang, 架构]
---

调度器是 alert-engine 的"心脏"——它驱动着规则的周期性评估、告警状态的流转、以及告警事件的触发与恢复。本章将深入调度器的设计、评估流程、状态管理以及与各组件的协作。

## 调度器职责

调度器（Scheduler）承担以下核心职责：

1. **规则生命周期管理**：启动、停止、暂停、恢复规则
2. **周期性评估**：按规则定义的间隔（Interval）触发规则求值
3. **告警状态管理**：跟踪每条规则的告警状态，区分 pending / firing / resolved
4. **通知触发**：当告警状态发生变更（新触发、新恢复）时，调用分发器发送通知
5. **规则同步**：定期同步活跃规则列表，处理外部变更（如管理员暂停规则）

## 调度器核心结构

```go
type Scheduler struct {
    ruleMgr     *rule.Manager      // 规则管理器
    engine      *engine.Engine     // 表达式引擎
    vmClient    VMClient           // VM 查询客户端
    dispatcher  Dispatcher         // 告警分发器
    workers     int                // 评估并发度（当前未使用，预留）
    evalTimeout time.Duration      // 评估超时

    mu       sync.RWMutex          // 保护以下共享状态
    states   map[string]*alertState  // 规则 -> 告警状态
    runners  map[string]*runner      // 规则 ID -> 评估 goroutine
    runnerWg sync.WaitGroup         // goroutine 退出等待
    stopCh   chan struct{}          // 停止信号
}
```

### 告警状态结构

```go
type alertState struct {
    ruleID       string
    pendingSince time.Time    // 进入 pending 状态的时间
    firedSince   time.Time    // 进入 firing 状态的时间
    alerts       []model.Alert // 当前 firing 的告警列表
}
```

每个规则对应一个 `alertState`，用于记录该规则在最近一次评估中产生的告警。状态对比正是通过对比本次告警列表与上次告警列表来实现的。

### Runner 结构

```go
type runner struct {
    ruleID   string
    interval time.Duration  // 评估间隔
    lastRun  time.Time
    ticker   *time.Ticker   // 定时器
    stop     chan struct{}  // 停止信号
}
```

每个活跃规则对应一个 `runner` goroutine，通过 `time.NewTicker(interval)` 周期性触发评估。

## 规则启动与评估循环

### 启动流程

```go
func (s *Scheduler) Start(ctx context.Context) error {
    // 1. 加载所有活跃规则
    rules, err := s.ruleMgr.ListActiveRules(ctx)

    // 2. 为每条规则启动 goroutine
    for _, r := range rules {
        if err := s.StartRule(ctx, r.ID); err != nil {
            return err
        }
    }

    // 3. 启动规则变更监听 goroutine
    go s.ruleWatcher(ctx)
    return nil
}
```

启动时，调度器首先获取所有状态为 "active" 的规则，为每条规则创建一个独立的评估 goroutine（runner）。

### StartRule：启动单条规则

```go
func (s *Scheduler) StartRule(ctx context.Context, ruleID string) error {
    r, err := s.ruleMgr.GetRule(ctx, ruleID)
    // ...

    s.mu.Lock()
    defer s.mu.Unlock()

    // 防止重复启动
    if _, ok := s.runners[ruleID]; ok {
        return nil
    }

    // 创建 runner
    runner := &runner{
        ruleID:   ruleID,
        interval: r.Interval,
        ticker:   time.NewTicker(r.Interval),
        stop:     make(chan struct{}),
    }
    s.runners[ruleID] = runner

    // 启动评估 goroutine
    s.runnerWg.Add(1)
    go func() {
        defer s.runnerWg.Done()
        s.runLoop(runner)
    }()

    return nil
}
```

### runLoop：评估循环

```go
func (s *Scheduler) runLoop(r *runner) {
    for {
        select {
        case <-r.stop:
            r.ticker.Stop()
            return
        case <-r.ticker.C:
            s.evaluateRule(r.ruleID)
        }
    }
}
```

评估 goroutine 通过 `select` 监听两个 channel：
- `r.stop`：接收停止信号，优雅退出
- `r.ticker.C`：接收定时 tick，触发评估

## 核心评估流程

### evaluateRule：规则评估

```go
func (s *Scheduler) evaluateRule(ruleID string) {
    // 1. 设置评估超时
    ctx, cancel := context.WithTimeout(context.Background(), s.evalTimeout)
    defer cancel()

    // 2. 获取规则定义
    r, err := s.ruleMgr.GetRule(ctx, ruleID)
    if err != nil {
        return
    }

    // 3. 通过表达式引擎求值
    samples, err := s.engine.Eval(ctx, r.Expr)
    if err != nil {
        // 记录错误日志，返回
        return
    }

    // 4. 获取该规则当前的告警状态
    s.mu.Lock()
    state, ok := s.states[ruleID]
    if !ok {
        state = &alertState{ruleID: ruleID}
        s.states[ruleID] = state
    }
    s.mu.Unlock()

    // 5. 根据求值结果构造新告警列表
    var alerts []model.Alert
    now := time.Now()

    for _, sample := range samples {
        // 比较运算结果：非 NaN 且值 > 0 表示条件满足
        if !isFiring(sample.Value) {
            continue
        }

        alert := model.Alert{
            ID:          fmt.Sprintf("%s-%s", ruleID, labelsKey(sample.Labels)),
            RuleID:      ruleID,
            RuleName:    r.Name,
            State:       model.StateFiring,
            Severity:    r.Severity,
            Labels:      mergeLabels(r.Labels, sample.Labels),
            Annotations: r.Annotations,
            Value:       sample.Value,
            Expr:        r.Expr,
            EvaluatedAt: now,
        }
        alerts = append(alerts, alert)
    }

    // 6. 处理告警：状态对比 + 通知触发
    s.handleAlerts(ctx, r, state, alerts)
}
```

关键设计点：

1. **评估超时保护**：每次评估通过 `context.WithTimeout` 设置超时，避免某个规则长时间阻塞
2. **状态无关性**：`evaluateRule` 不修改状态，只构造本次评估产生的告警列表。真正的状态更新在 `handleAlerts` 中完成
3. **求值结果的筛选**：表达式引擎返回的是比较运算结果（即满足条件的样本），只要值非 NaN 且非零，就视为"条件满足"，生成告警

### 触发判断：isFiring

```go
func isFiring(value float64) bool {
    return !isNaN(value) && value > 0
}

func isNaN(f float64) bool {
    return f != f // NaN 不等于自身
}
```

这与 PromQL 的行为一致：比较运算返回 0/1（0 表示 false，1 表示 true），只要值为 1 即表示条件满足。

## 告警状态管理与通知触发

### handleAlerts：状态对比与通知

```go
func (s *Scheduler) handleAlerts(ctx context.Context, rule *model.Rule, state *alertState, newAlerts []model.Alert) {
    s.mu.Lock()
    defer s.mu.Unlock()

    // 1. 提取上次的 firing 告警映射
    oldFiring := make(map[string]model.Alert)
    for _, a := range state.alerts {
        oldFiring[a.ID] = a
    }

    // 2. 提取本次的 firing 告警映射
    newFiring := make(map[string]model.Alert)
    for _, a := range newAlerts {
        newFiring[a.ID] = a
    }

    // 3. 检测新触发的告警（上次没有，本次有）→ 发送 firing 通知
    for id, alert := range newFiring {
        if _, wasFiring := oldFiring[id]; !wasFiring {
            now := time.Now()
            alert.FiredAt = &now
            alert.State = model.StateFiring

            s.dispatcher.Notify(ctx, alert, rule.Channels)
        }
    }

    // 4. 检测恢复的告警（上次有，本次没有）→ 发送 resolved 通知
    for id, alert := range oldFiring {
        if _, isFiring := newFiring[id]; !isFiring {
            now := time.Now()
            alert.ResolvedAt = &now
            alert.State = model.StateResolved

            s.dispatcher.Notify(ctx, alert, rule.Channels)
        }
    }

    // 5. 更新状态
    state.alerts = newAlerts
}
```

核心逻辑是 **diff**：

- **新告警**：存在于 `newFiring` 但不在 `oldFiring` 中 → 触发 firing 通知
- **已恢复**：存在于 `oldFiring` 但不在 `newFiring` 中 → 触发 resolved 通知
- **持续告警**：两边都有，不重复通知

这种模型天然解决了告警风暴和重复通知问题——只有状态发生变化时才通知。

## 规则变更同步

调度器启动了一个后台 goroutine `ruleWatcher`，定期同步规则状态。这确保了当管理员通过 API 暂停或删除规则时，调度器能够及时感知并停止对应的评估 goroutine。

## 优雅关闭

`Scheduler.Stop()` 实现：

```go
func (s *Scheduler) Stop() {
    close(s.stopCh)              // 关闭停止信号 channel
    s.mu.Lock()
    for _, r := range s.runners {
        close(r.stop)           // 关闭每个 runner 的 stop channel
    }
    s.mu.Unlock()
    s.runnerWg.Wait()           // 等待所有 goroutine 退出
}
```

优雅关闭的关键点：

1. **关闭 stopCh**：这会让 `ruleWatcher` 退出循环
2. **关闭每个 runner 的 stop**：这会让每个评估 goroutine 退出 `runLoop`
3. **等待所有 goroutine**：通过 `WaitGroup` 确保所有 goroutine 退出后再返回，避免资源泄露

## 总结

调度器是 alert-engine 端到端链路的核心枢纽：

- **每规则独立 goroutine** 模型支持不同的评估间隔，实现简洁直观
- **状态 diff 通知模型** 天然解决重复告警和告警风暴问题
- **评估与状态管理分离**：求值在锁外执行，避免阻塞
- **优雅退出** 确保资源不泄露
- **API 驱动的启停控制** 支持运行时动态调整

下一章我们将深入告警通知分发的实现——告警触发后，如何通过 Webhook、邮件、钉钉等渠道将信息送达值班人员。