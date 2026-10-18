# 设计判据 D-1~D-5（唯一权威定义）

> 跨技能共用的五条设计判据。**操作化定义只在这里**，其它技能引用、不复制；
> 判据出处与文献溯源见 membership-system-builder 的 `master-craft.md` §3。

## 1. 判据总表

| # | 判据 | 必检问题（通用） | 业务域（会员/支付） | 认证域（ASP） | 通过证据 |
| --- | --- | --- | --- | --- | --- |
| D-1 删表重建 | 派生结构可删可重建 | 删掉余额、计数器、索引后，钱与权益事实仍在账本/事件日志中且可重建 | 删掉 sessions 表：用户与身份事实仍在，重登即可恢复；审计事件不可删 | 重建步骤 + 前后 `balance == Σ delta` 输出 / 重登 + 审计核对 |
| D-2 一年后审计 | 每个事实可回溯 | 任一入账/退款/发放能指向不可变记录（actor+reason+occurred_at） | "这账号何时、为何被禁用/改密"能指向不可变审计记录 | 至少 1 笔退款 + 1 笔发放 / 1 次封禁 + 1 次改密的完整链 |
| D-3 并发同发 | 竞争恰一个赢 | 两个相同请求同时到：恰一个成功，失败方得到可解释错误，无部分副作用 | 同一验证码并发消费 2 次只 1 次成功；并发撤销会话幂等 | 并发压测输出 |
| D-4 事件重放 | 重放逐字节一致 | 同一条事件重放两次：到期时间、余额、发放事实条数一致 | 同一 `auth.login_succeeded` 重放不重复发会话/不重复审计 | 重放脚本 + 前后 diff |
| D-5 回滚演练 | 新版本可退回 | 部署新版本发现问题可退回：旧版本能读新数据、迁移可逆或安全降级 | 旧版本能读新版本写的会话/审计数据 | 回滚步骤 + 回滚后冒烟输出 |

## 2. 出处（文献溯源，全文见 membership-system-builder master-craft.md）

- D-1 → Kleppmann《Turning the database inside-out》：派生数据可删重建；
- D-2 → Helland《Immutability Changes Everything》：会计不用橡皮擦；审计链不可变；
- D-3 → Evans DDD 聚合边界 + ledger 原子扣减（条件更新/行锁）；
- D-4 → Helland《The I's Have It》/《Idempotence Is Not a Medical Condition》：幂等是业务层义务；
- D-5 → Amazon Builders Library《Ensuring rollback safety during deployments》：expand→migrate→contract。

## 3. 使用规则

- 任一判据不过 → 判定不符合；**不得通过降低声明等级豁免**（跨等级底线）；
- 测试向量（T-*/S-*）是行为验收，D-1~D-5 是设计审查，两者不互相替代；
- 新领域需要自己的"域表述"时，在 §1 表加一列，而不是另写一套判据。
