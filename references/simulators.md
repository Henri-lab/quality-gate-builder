# 三个模拟器（simulators）

> 本技能的核心资产：把"模拟支付、模拟并发用户、模拟订单流程"做成可复用、可断言、可清理的组件。
> 共同纪律：**幂等、数据隔离、不打生产、以事件与账本为断言依据**。

## 1. 模拟支付（PaymentSimulator）

职责：扮演支付渠道，向服务端上报结算与争议，**并故意制造异常**。

接口（与真实 Provider 同形，业务代码无感知）：

```text
reportSettlement(order, {amount, paid_at, provider_ref, signature})
replay(count)          # 重复投递同一通知
misreport(amount)      # 金额不符
lateReport()           # paid_at 出窗
forge()                # 无效签名
openDispute(order) / resolveDispute(win|lose)
refund(order, lines?)  # 全退/部分退款
```

必测剧本（映射 MSP 向量）：

| 剧本 | 期望 | 向量 |
| --- | --- | --- |
| 正常支付 | 订单 paid→fulfilled，权益 active | T-02 |
| 重复投递 ×3 | duplicate，权益/账本不重复 | T-03/T-27 |
| 金额不符 | unmatched，无副作用 | T-04 |
| 伪造签名 | INVALID_SIGNATURE，无副作用 | T-07 |
| 支付窗口外 | 拒绝/补单路径 | T-05/T-38 |
| 指纹退款 | 金额 = payable，不含尾数 | T-36 |
| 部分退款×2 | 段回收两次，Σ≤paid | T-39/T-40 |
| 拒付开/赢/输 | 段冻结/恢复/追索 + 账本负向分录 | T-44~T-46 |

断言落在可观测事实：订单状态、权益到期时间、账本 `balance == Σ delta`、事件条数。

## 2. 模拟并发用户（UserSimulator）

职责：用**真实会话**（走 ASP 注册/登录，非后门 token）模拟用户池与混合行为。

```text
UserPool(size, behavior_distribution)
  # behavior: browse 60% / order 20% / pay 10% / query 5% / refund 3% / abuse 2%
abuse()  # 连点下单、高频撞码、撞限流——验证 INV-31/32 与限速
```

- 会话生命周期完整（登录→使用→过期/登出），覆盖 S-03/S-06/S-10；
- abuse 剧本用于验证限流/熔断是"按设计生效"而不是"被测出 bug"；
- 每个用户数据带 `run_id` 前缀，结束即清理。

## 3. 模拟业务订单流程（OrderFlowSimulator）

职责：按剧本状态机跑全链路，每步断言不变量。

```text
browse → createOrder → createPaymentIntent → (pay | 截图凭证审核 | 指纹自动匹配)
      → dispatchOnPaid → checkAccess → refund(全退/部分) → dispute(开/赢/输)
      → 账户合并(段并集 + 账本 transfer)
```

每步断言：

- 下单：金额/条目快照（INV-2）；重复支付意图复用（INV-31）；
- 支付：唯一写入口（INV-6）；两层幂等防线；
- 履约：到期时间 = `max(原到期, now)+周期`（INV-9）；发放事实条数不增；
- 退款：段回收只收本单贡献段（INV-16）；礼物已送出则 MANUAL_REQUIRED；
- 争议：只冻结本单段（INV-29）；合并：段并集 + 账本 transfer（INV-36）。

## 4. 与压测流水的衔接

同一套模拟器代码，两处用：契约/E2E 测试（断言正确性）、k6/Locust 压测（断言容量与阈值）。
压测脚本复用剧本，只是把"断言每步结果"换成"统计延迟/错误率/吞吐"——避免两套流量模型漂移。
