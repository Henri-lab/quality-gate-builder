# quality-gate-builder 🛡️ 企业级质量门禁构建器

> A **Codex skill** that builds an executable quality-gate pipeline — test layers,
> load-testing with baselines and thresholds, plus three simulators (payment callbacks,
> concurrent users, business order flows) — reusing the MSP/ASP conformance vectors.

<p align="center">
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-blue.svg"></a>
  <img alt="Version" src="https://img.shields.io/badge/version-v0.1.0-informational">
</p>

核心立场：**质量是门禁，不是口号**——每一条保障都必须能在 CI 里自动执行、不可跳过、超阈值即红。

## 快速开始

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/Henri-lab/quality-gate-builder.git ~/.codex/skills/quality-gate-builder
```

之后在任意 Codex 会话中按 skill 名（`quality-gate-builder`）触发即可。

## 目录

```text
SKILL.md                     主入口：访谈 → 门禁档位 Q0~Q3 → 按档位生成流水线技能
agents/openai.yaml           agent 元数据
references/
├── solution-tiers.md        访谈与档位矩阵 Q0~Q3
├── pipeline-blueprint.md    流水线阶段、门禁规则、GitHub Actions 骨架
├── test-pyramid.md          测试分层与数据纪律
├── load-testing.md          压测工具/流量模型/指标/基线阈值
├── simulators.md            三个模拟器：支付 / 并发用户 / 订单流程
├── protection-checklist.md  项目保护措施总表
├── lessons-log.md           经验账本（L-001 起）
└── maintenance.md           技能持续进化纪律
```

## 三个不寻常之处

1. **门禁可执行**：任何措施必须能回答"怎么自动执行、怎么阻断、证据在哪"；
2. **三个模拟器**：支付回调（含异常剧本）、并发用户（真实会话）、订单流程（断言账本/权益不变量）；
3. **复用协议向量**：直接消费 membership-system-builder 的 MSP T 向量与 identity-system-builder 的 ASP S 向量。

## 起源与署名

2026-10-15 整理为公开仓库。作者清单见 [AUTHORS.md](AUTHORS.md)；二次分发须保留 [LICENSE](LICENSE) 版权声明。

```text
quality-gate-builder（v0.1.0）
作者：henrifox
仓库：https://github.com/Henri-lab/quality-gate-builder
首发：2026-10-15｜许可：MIT（保留署名）
```

## License

[MIT](LICENSE) © 2026 henrifox
