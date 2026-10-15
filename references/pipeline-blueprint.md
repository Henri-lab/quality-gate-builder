# 流水线蓝本（CI quality gates）

> 目标：把"保护措施"变成 CI 里一段不可跳过的自动检查。示例以 GitHub Actions 为主，原则通用。

## 1. 阶段图（失败即停，从左到右）

```
lint/静态分析 → 单元测试 → 集成/契约测试 → E2E 冒烟 → 安全扫描 → 压测（Q2+） → 构建镜像
      │             │            │              │           │           │
      └──────────── 任一失败 → 阻断合并/发布 ──────────────────────────────┘
```

每阶段要求：

| 阶段 | 工具（示例） | 门禁 |
| --- | --- | --- |
| lint/静态 | golangci-lint / eslint / ruff | 0 warning 或按存量豁免清单（只减不增） |
| 单元测试 | go test / vitest / pytest | 必须通过；覆盖率 ≥ 阈值且**环比不降** |
| 集成/契约 | Testcontainers + 真实 Postgres/Redis | 真库冒烟；迁移 up/down 可重复 |
| E2E 冒烟 | 模拟器驱动（simulators.md） | MSP/ASP 关键向量全绿 |
| 安全扫描 | gitleaks + trivy | 密钥 0 命中；高危 CVE 0 |
| 压测（Q2+） | k6 / Locust | p95/p99、错误率、吞吐对基线，超阈值即红 |

## 2. 门禁规则（必须写进 CI 配置与团队约定）

1. 合并到主分支必须 CI 全绿；`skip ci` 只允许紧急修复且事后补跑；
2. 覆盖率：绝对值做底线（如 70%），**diff 覆盖率**做硬门槛（新增代码必须 ≥ 80%）——防止存量低覆盖率吞掉新代码；
3. 压测门禁用**基线与容差**：p95 ≤ 基线×1.2、错误率 ≤ 基线+0.5pp、吞吐 ≥ 基线×0.9；
4. 迁移测试：每个 migration 必须能 up→冒烟→down→up（或按 expand→migrate→contract 证明可回滚）；
5. 密钥与漏洞：gitleaks 0 命中、trivy 高危 0——命中即红，不允许"标记忽略"绕过。

## 3. GitHub Actions 骨架（Q1 最小示例）

```yaml
name: quality-gate
on:
  pull_request:
  push:
    branches: [main]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [checkout@v4, setup-go@v5, run: golangci-lint run]
  test:
    runs-on: ubuntu-latest
    services:
      postgres: {image: postgres:16, env: {POSTGRES_PASSWORD: test}, ports: ["5432:5432"]}
    steps:
      - {uses: actions/checkout@v4}
      - run: go test ./... -coverprofile=cover.out -race
      - run: bash scripts/smoke-conformance.sh   # 复用 MSP/ASP 冒烟骨架
  security:
    runs-on: ubuntu-latest
    steps:
      - {uses: actions/checkout@v4, with: {fetch-depth: 0}}
      - uses: gitleaks/gitleaks-action@v2
        with: {GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"}
  perf:
    if: inputs.run_perf == 'true' || github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: docker compose -f docker-compose.perf.yml up -d --build
      - run: k6 run --summary-export=perf.json scripts/perf/order-flow.js
      - run: python3 scripts/perf/check_thresholds.py perf.json
```

## 4. 分支保护与发布

- 主分支保护：要求 CI 通过 + 至少 1 人评审（solo 阶段可先只要求 CI）；
- 环境晋升：staging 冒烟通过 → 打 tag → prod 部署 → 部署后冒烟 → 失败自动回滚；
- 每个环境都有独立的密钥集合（CI Secrets / 服务器 600 权限文件），绝不通配共享。
