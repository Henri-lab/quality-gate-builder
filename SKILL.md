---
name: quality-gate-builder
description: 为企业级后端质量门禁设计并生成可执行的流水线构建技能：测试流水（lint/单测/集成/契约/E2E/覆盖率/安全扫描）+ 压测流水（流量模型/基线/阈值门禁），以及三个模拟器——模拟支付回调、模拟并发用户、模拟业务订单流程；直接复用 MSP/ASP 的符合性向量当测试素材。适用于 Go/Node/Java 后端 + CI 平台的项目；不适用于只接现成 SaaS 测试平台或纯前端项目的小改动。
---

# 企业级质量门禁构建器

## 产出物

一次运行产出：

1. 一份**门禁方案结论**（档位 + 理由，写进产出技能的「分支决策」章节）；
2. 一份针对该项目的**流水线构建技能目录**：CI workflow 骨架、压测脚本与阈值、三个模拟器的契约与接入点。

核心立场：**质量是门禁，不是口号**——每一条保障都必须能在 CI 里自动执行、不可跳过、超阈值即红。
本技能直接复用 [membership-system-builder](https://github.com/Henri-lab/membership-system-builder)（MSP T-01~T-64）
与 [identity-system-builder](https://github.com/Henri-lab/identity-system-builder)（ASP S-01~S-20）的符合性向量当测试素材。

## 第一步：选型访谈 → 门禁档位（必做，禁止静默默认）

按 [references/solution-tiers.md](references/solution-tiers.md) 一次问清三件事——
**目的**（新项目起步 / 已有项目补门禁 / 上线前压测 / 长期质量体系）、
**规模**（团队人数、目标并发量级、发布频率）、**硬约束**（CI 平台、技术栈、预算、数据合规）——
然后选档并说明理由；未回答的项按最保守假设处理并在产出中标注"未确认"。

| 档位 | 目的 × 规模 | 典型配置 |
| --- | --- | --- |
| Q0 最小门禁 | 个人/起步，低风险 | lint + 单测 + 冒烟（MSP/ASP 关键向量） |
| Q1 标准门禁 | 小团队，正常发布 | + 集成/契约测试、覆盖率门禁、密钥与依赖漏洞扫描 |
| Q2 压测流水线 | 有真实流量预期 | + 负载基线、阈值门禁、三个模拟器 |
| Q3 企业级 | 多人/高并发/多环境 | + 持续性能回归、故障注入、环境晋升、观测联动 |

## 第二步：按档位读取参考

- [references/solution-tiers.md](references/solution-tiers.md) —— 访谈与档位矩阵（第一步必读）
- [references/design-judgments.md](references/design-judgments.md) —— **设计判据 D-1~D-5 的唯一权威定义**（跨技能共用；各域表述与通过证据）
- [references/pipeline-blueprint.md](references/pipeline-blueprint.md) —— 流水线阶段、门禁规则与 CI 骨架
- [references/test-pyramid.md](references/test-pyramid.md) —— 测试分层（单元/集成/契约/E2E）与迁移测试
- [references/load-testing.md](references/load-testing.md) —— 压测工具、流量模型、指标与阈值门禁
- [references/simulators.md](references/simulators.md) —— **三个模拟器**：支付回调 / 并发用户 / 业务订单流程
- [references/protection-checklist.md](references/protection-checklist.md) —— 项目保护措施总表（密钥/依赖/部署/告警）
- [references/lessons-log.md](references/lessons-log.md) —— 经验账本（使用前扫最近条目）
- [references/maintenance.md](references/maintenance.md) —— 技能维护手册

## 三个模拟器（本技能的核心资产）

1. **模拟支付**：签名回调 + 异常路径（重复投递、金额不符、时间出窗、拒付），直接覆盖 MSP 的 T-02~T-08/T-27/T-36/T-44~T-46；
2. **模拟并发用户**：真实会话令牌池 + 混合行为分布，目标并发与限流验证（MSP INV-31/32）；
3. **模拟业务订单流程**：浏览 → 下单 → 支付 → 权益 → 退款 → 争议的全链路剧本，断言账本与权益不变量。

细节与契约见 [references/simulators.md](references/simulators.md)；压测侧用同一套流量模型（见 load-testing.md）。

## 不可妥协项

1. **门禁在 CI 执行且不可跳过**：本地能跑不算数；`skip` 按钮要么不存在，要么有审批记录。
2. **压测有基线和阈值**：p95/p99、错误率、吞吐任一超阈值即红，不凭感觉"看着还行"。
3. **模拟支付必须覆盖异常路径**：重复投递/金额不符/超时/拒付是必测项，不是加分项。
4. **模拟用户用真实会话令牌**：走完整认证（ASP），不打生产环境，数据可清理。
5. **测试数据隔离**：共享测试库要串行化，各用例自带清理（对齐 engineering-pitfalls 的共享库教训）。
6. **安全扫描常驻**：密钥扫描（gitleaks）+ 依赖漏洞（trivy/snyk）+ 静态分析在每次合并前跑。
7. **环境晋升要过门禁**：staging 冒烟通过才允许发 prod；发布必须带回滚预案（expand → migrate → contract）。
8. **一切断言落在可观测数据上**：账本 Σ delta、权益到期时间、发放事实条数——不只看接口返回 200。

## 必须显式禁止的反面模式

- 只在本地跑测试，CI 无门禁或可跳过；
- 压测只做 happy path、没有基线，把"能压 1 万 RPS"当结论却无视 p99 与错误率；
- 模拟支付只测成功回调，不测重复/伪造/超时；
- 并发模拟共用同一账号/令牌，打生产环境；
- 测试后不清理数据，共享库互相污染；
- 用"编译通过"代替端到端冒烟；
- 发布无回滚预案，迁移不可逆。

## 完成后

- 把本轮新经验按 [references/lessons-log.md](references/lessons-log.md) 格式追加（唯一进化入口）；
- 与迁移/部署联动时用 `vps-migrator`；与安全加固用 `server-hardening`，不在本技能内重复。

## 持续进化（每次使用都必须执行）

1. **使用前**：读 lessons-log 最近条目；
2. **使用中/后**：新经验按"日期｜来源｜档位 / 结论 / 为什么＋落地"追加，注明真实项目或推理；
3. **提升与归档**：同主题 ≥2 条或触及不可妥协项 → 提升进对应 reference，保留溯源。
