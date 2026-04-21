# GoPay: 企业级多渠道支付网关 SDK

<p align="center">
  <img width="180" height="180" src="logo.png" alt="GoPay Logo">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/golang-1.24.0-brightgreen.svg" alt="Golang">
  <img src="https://img.shields.io/badge/version-v1.5.118-blue.svg" alt="Version">
  <img src="https://img.shields.io/badge/license-Apache%202.0-blue.svg" alt="License">
</p>

`gopay` 是一款生产级 Golang 支付 SDK，统一封装微信、支付宝、PayPal、QQ、通联、拉卡拉、扫呗、Apple 等 **8 大支付渠道**，屏蔽底层接口差异，让开发者用一套代码完成多渠道接入。

项目核心价值：**统一参数构造 (BodyMap) + 统一调用模式 (Context + BodyMap) + 统一回调验签**，配合异步消息队列架构可轻松支撑高并发交易场景。

---

## 项目架构与目录结构

```text
gopay/
├── alipay/           # 支付宝 (V2 + V3，推荐 V3)
├── wechat/           # 微信支付 (V2 + V3，推荐 V3)
├── apple/            # Apple IAP 支付校验
├── paypal/           # PayPal 国际支付
├── qq/               # QQ 支付
├── allinpay/         # 通联支付
├── lakala/           # 拉卡拉支付
├── saobei/           # 扫呗支付
├── pkg/              # 公共工具：xhttp 请求封装、JWT 等
├── doc/              # 各渠道 API 文档
├── examples/         # 微信 & 支付宝接入示例
├── body_map.go       # 核心：动态参数构造器 BodyMap
├── constant.go       # 全局常量与版本号
└── release_note.md   # 版本变更记录
```

---

## 核心能力总览

| 渠道 | 下单支付 | 订单查询 | 退款 | 退款查询 | 回调验签 | 证书管理 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **微信 V3** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ 自动验签 |
| **支付宝 V3** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **PayPal** | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| **QQ 支付** | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| **通联支付** | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| **拉卡拉** | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| **扫呗** | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| **Apple** | — | ✅ 校验 | — | — | — | — |

---

## 一笔支付的完整生命周期

```text
┌─────────────────────────────────────────────────────────────────────┐
│                      支付全流程架构图                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户下单          支付网关              RocketMQ        消费者       │
│    │                 │                     │              │          │
│    │  1.创建订单     │                     │              │          │
│    │────────────────>│                     │              │          │
│    │                 │ 2.参数签名+调用渠道   │              │          │
│    │                 │  (gopay SDK)        │              │          │
│    │  3.返回支付凭证  │                     │              │          │
│    │<────────────────│                     │              │          │
│    │                 │                     │              │          │
│    │  4.用户完成支付   │                     │              │          │
│    │                 │                     │              │          │
│    │        5.渠道回调通知                   │              │          │
│    │                 │<──── 微信/支付宝      │              │          │
│    │                 │                     │              │          │
│    │                 │ 6.验签+解密(<20ms)   │              │          │
│    │                 │ 7.投递MQ            │              │          │
│    │                 │────────────────────>│              │          │
│    │                 │ 8.立即返回200 OK     │              │          │
│    │                 │──── 响应渠道         │              │          │
│    │                 │                     │  9.异步消费    │          │
│    │                 │                     │─────────────>│          │
│    │                 │                     │              │ 10.乐观锁 │
│    │                 │                     │              │   更新DB  │
│    │                 │                     │  11.手动ACK   │          │
│    │                 │                     │<─────────────│          │
│    │                 │                     │              │          │
│    │  ── T+1 对账 ── │ ─── 下载渠道账单 ─── │ ── 差异比对 ──│          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 快速接入

### 1. 安装
```bash
go get github.com/go-pay/gopay
```

### 2. 核心概念：BodyMap 参数构造

所有支付接口统一使用 `gopay.BodyMap`（底层为 `map[string]any`）来构造请求参数，告别为每个接口定义 struct 的繁琐：

```go
bm := make(gopay.BodyMap)
bm.Set("out_trade_no", "GoPay202604210001").
    Set("description", "用户钱包充值").
    Set("notify_url", "https://your-domain.com/wechat/notify")

// 支持嵌套结构
bm.SetBodyMap("amount", func(b gopay.BodyMap) {
    b.Set("total", 100).       // 微信单位：分
      Set("currency", "CNY")
})

// 参数校验：检查必填字段是否为空
err := bm.CheckEmptyError("out_trade_no", "description")
```

### 3. 统一调用模式

所有渠道的 API 调用遵循相同范式：`client.Method(ctx, bm) → (typedResponse, error)`

---

## 微信支付 V3 实战

### 客户端初始化
```go
import "github.com/go-pay/gopay/wechat/v3"

// 参数：商户ID、证书序列号、APIv3Key、私钥内容(apiclient_key.pem)
client, err := wechat.NewClientV3("MchId", "SerialNo", "APIv3Key", "PrivateKeyContent")
if err != nil {
    panic(err)
}

// 开启自动验签（推荐）
// 方式一：微信支付公钥验签
err = client.AutoVerifySignByPublicKey([]byte("微信支付公钥内容"), "PUB_KEY_ID_xxx")
// 方式二：微信平台证书验签
// err = client.AutoVerifySign()

// 可选：开启 Debug 日志
client.DebugSwitch = gopay.DebugOn
```

### Native 下单（充值场景 - 扫码支付）
```go
bm := make(gopay.BodyMap)
bm.Set("appid", "wx12345678").
    Set("description", "钱包充值100元").
    Set("out_trade_no", "RECHARGE_202604210001").
    Set("notify_url", "https://your-domain.com/wechat/notify").
    SetBodyMap("amount", func(b gopay.BodyMap) {
        b.Set("total", 10000).    // 100元 = 10000分
          Set("currency", "CNY")
    })

wxRsp, err := client.V3TransactionNative(ctx, bm)
if err != nil {
    xlog.Errorf("请求异常: %v", err)
    return
}
// wxRsp.Response.CodeUrl 即为支付二维码链接
xlog.Infof("Native 下单成功, code_url: %s", wxRsp.Response.CodeUrl)
```

### JSAPI 下单（小程序/公众号场景）
```go
bm.Set("appid", "wx12345678").
    Set("description", "订单支付").
    Set("out_trade_no", "ORDER_202604210001").
    Set("notify_url", "https://your-domain.com/wechat/notify").
    SetBodyMap("amount", func(b gopay.BodyMap) {
        b.Set("total", 100)
    }).
    SetBodyMap("payer", func(b gopay.BodyMap) {
        b.Set("openid", "user_openid_xxx")  // JSAPI 必须传 openid
    })

wxRsp, err := client.V3TransactionJsapi(ctx, bm)
// wxRsp.Response.PrepayId 用于前端调起支付
```

### 其他下单方式
```go
// APP 支付
wxRsp, err := client.V3TransactionApp(ctx, bm)

// H5 支付（手机浏览器）
wxRsp, err := client.V3TransactionH5(ctx, bm)

// 付款码支付（被扫）
wxRsp, err := client.V3TransactionCodePay(ctx, bm)

// 订单查询
wxRsp, err := client.V3TransactionQueryOrder(ctx, wechat.OutTradeNo, "ORDER_202604210001")

// 关闭订单
wxRsp, err := client.V3TransactionCloseOrder(ctx, "ORDER_202604210001")
```

### 申请退款
```go
bm := make(gopay.BodyMap)
bm.Set("out_trade_no", "RECHARGE_202604210001").
    Set("out_refund_no", "REFUND_202604210001").
    Set("notify_url", "https://your-domain.com/wechat/refund-notify").
    SetBodyMap("amount", func(b gopay.BodyMap) {
        b.Set("refund", 10000).   // 退款金额（分）
          Set("total", 10000).    // 原订单金额（分）
          Set("currency", "CNY")
    })

refundRsp, err := client.V3Refund(ctx, bm)
if err != nil {
    xlog.Errorf("退款请求异常: %v", err)
    return
}
xlog.Infof("退款发起成功, 微信退款号: %s", refundRsp.Response.RefundId)

// 退款查询
queryRsp, err := client.V3RefundQuery(ctx, "REFUND_202604210001", nil)
```

### 支付回调处理
```go
import "net/http"

func WechatNotifyHandler(req *http.Request) {
    // 1. 解析回调请求
    notifyReq, err := wechat.V3ParseNotify(req)
    if err != nil {
        xlog.Errorf("回调解析失败: %v", err)
        return
    }

    // 2. 解密支付结果（支付回调用 DecryptPayCipherText）
    result, err := notifyReq.DecryptPayCipherText("APIv3Key")
    if err != nil {
        xlog.Errorf("解密失败: %v", err)
        return
    }

    // 3. result 包含完整支付信息
    // result.OutTradeNo     - 商户订单号
    // result.TransactionId  - 微信支付订单号
    // result.TradeState     - 交易状态 (SUCCESS/REFUND/NOTPAY/CLOSED...)
    // result.Amount.Total   - 订单总金额（分）
    // result.Payer.Openid   - 支付者 openid
    // result.SuccessTime    - 支付完成时间

    xlog.Infof("支付成功: 订单号=%s, 微信交易号=%s, 金额=%d分",
        result.OutTradeNo, result.TransactionId, result.Amount.Total)

    // ===== 生产环境最佳实践 =====
    // 不要在回调中直接执行耗时的数据库操作！
    // 推荐：将 result 投递到 RocketMQ/Kafka，由消费者异步处理
    // 这样可以极速响应微信 200 OK，避免超时重试
}

// 退款回调解密：notifyReq.DecryptRefundCipherText("APIv3Key")
// 合单回调解密：notifyReq.DecryptCombineCipherText("APIv3Key")
// 分账回调解密：notifyReq.DecryptProfitShareCipherText("APIv3Key")
```

---

## 支付宝 V3 实战

### 客户端初始化
```go
import "github.com/go-pay/gopay/alipay/v3"

// 参数：应用ID、应用私钥、是否正式环境
client, err := alipay.NewClientV3("AppId", "PrivateKey", true)
if err != nil {
    panic(err)
}

// 设置证书（推荐证书模式）
err = client.SetCert(
    appCertContent,          // 应用公钥证书内容
    alipayRootCertContent,   // 支付宝根证书内容
    alipayPublicCertContent, // 支付宝公钥证书内容
)

client.DebugSwitch = gopay.DebugOn
```

### 统一收单 - 当面付（扫码支付）
```go
bm := make(gopay.BodyMap)
bm.Set("subject", "钱包充值100元").
    Set("out_trade_no", "ALI_RECHARGE_202604210001").
    Set("total_amount", "100.00")   // 支付宝单位：元，string 类型

aliRsp, err := client.TradePay(ctx, bm)
```

### 统一收单 - 退款
```go
bm := make(gopay.BodyMap)
bm.Set("out_trade_no", "ALI_RECHARGE_202604210001").
    Set("refund_amount", "100.00").         // 退款金额（元）
    Set("out_request_no", "ALI_REFUND_001") // 退款请求号（同一笔订单多次部分退款时必须不同）

aliRsp, err := client.TradeRefund(ctx, bm)
```

### 订单查询
```go
bm := make(gopay.BodyMap)
bm.Set("out_trade_no", "ALI_RECHARGE_202604210001")
aliRsp, err := client.TradeQuery(ctx, bm)
```

---

## 核心数据库表设计（生产参考）

### 支付订单表 t_pay_order

```sql
CREATE TABLE t_pay_order (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    out_trade_no    VARCHAR(64)  NOT NULL COMMENT '商户订单号（全局唯一）',
    channel         VARCHAR(16)  NOT NULL COMMENT '支付渠道: WECHAT/ALIPAY/PAYPAL',
    trade_type      VARCHAR(16)  NOT NULL COMMENT '交易类型: NATIVE/JSAPI/APP/H5',
    transaction_id  VARCHAR(64)  DEFAULT '' COMMENT '渠道交易号（微信/支付宝返回）',
    amount          INT UNSIGNED NOT NULL COMMENT '订单金额（统一用分）',
    status          VARCHAR(16)  NOT NULL DEFAULT 'INIT' COMMENT 'INIT/PAYING/SUCCESS/FAILED/CLOSED',
    notify_url      VARCHAR(256) NOT NULL COMMENT '回调通知地址',
    user_id         BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    description     VARCHAR(128) DEFAULT '' COMMENT '商品描述',
    pay_time        DATETIME     DEFAULT NULL COMMENT '支付成功时间',
    expire_time     DATETIME     NOT NULL COMMENT '订单过期时间',
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_out_trade_no (out_trade_no),
    INDEX idx_user_id (user_id),
    INDEX idx_status_created (status, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付订单表';
```

### 退款单表 t_refund_order

```sql
CREATE TABLE t_refund_order (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    out_refund_no   VARCHAR(64)  NOT NULL COMMENT '商户退款单号',
    out_trade_no    VARCHAR(64)  NOT NULL COMMENT '原支付订单号',
    refund_id       VARCHAR(64)  DEFAULT '' COMMENT '渠道退款号',
    channel         VARCHAR(16)  NOT NULL COMMENT '支付渠道',
    refund_amount   INT UNSIGNED NOT NULL COMMENT '退款金额（分）',
    total_amount    INT UNSIGNED NOT NULL COMMENT '原订单金额（分）',
    status          VARCHAR(16)  NOT NULL DEFAULT 'REFUNDING' COMMENT 'REFUNDING/REFUND_SUCCESS/REFUND_FAILED',
    reason          VARCHAR(256) DEFAULT '' COMMENT '退款原因',
    refund_time     DATETIME     DEFAULT NULL COMMENT '退款到账时间',
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_out_refund_no (out_refund_no),
    INDEX idx_out_trade_no (out_trade_no)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='退款单表';
```

### 本地消息表 t_message_fallback（可靠投递兜底）

```sql
CREATE TABLE t_message_fallback (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    topic       VARCHAR(64)  NOT NULL COMMENT 'MQ topic',
    tag         VARCHAR(64)  DEFAULT '' COMMENT 'MQ tag',
    msg_key     VARCHAR(64)  NOT NULL COMMENT '消息唯一键（通常为订单号）',
    msg_body    TEXT         NOT NULL COMMENT '消息体 JSON',
    status      TINYINT      NOT NULL DEFAULT 0 COMMENT '0=待发送 1=已发送 2=发送失败',
    retry_count INT          NOT NULL DEFAULT 0 COMMENT '重试次数',
    next_retry  DATETIME     NOT NULL COMMENT '下次重试时间',
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_status_retry (status, next_retry)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='本地消息兜底表';
```

### 关键设计说明

| 设计点 | 说明 |
| :--- | :--- |
| 金额统一用分 (int) | 消除微信/支付宝单位差异，入库前统一转换 |
| out_trade_no 唯一索引 | 保证订单号全局唯一，防止重复下单 |
| status 字段 + 乐观锁 | `UPDATE ... WHERE status='旧状态'` 实现幂等更新 |
| 退款与支付分表 | 退款单独建表，一笔支付可多次部分退款 |
| idx_status_created 联合索引 | 定时任务扫描超时未支付订单用（关单/查单） |

---

## 异步架构最佳实践（RocketMQ 削峰）

### 架构全景

```text
                    ┌──────────────────────────────────────────────┐
                    │               支付网关 (Go)                   │
 微信/支付宝回调 ──>│  V3ParseNotify → 验签解密 → 投递 MQ → 200 OK │
                    │  耗时 < 20ms，无 DB 操作，无阻塞              │
                    └───────────────────┬──────────────────────────┘
                                        │
                                        ▼
                    ┌──────────────────────────────────────────────┐
                    │         RocketMQ Broker (削峰缓冲)            │
                    │  Topic: PAY_NOTIFY / REFUND_NOTIFY           │
                    │  同步刷盘 + 主从同步复制 → 不丢消息           │
                    └───────────────────┬──────────────────────────┘
                                        │
                         ┌──────────────┼──────────────┐
                         ▼              ▼              ▼
                    ┌─────────┐  ┌─────────┐   ┌─────────────┐
                    │消费者实例1│  │消费者实例2│   │消费者实例N   │
                    │(充值处理)│  │(充值处理)│   │(退款处理)   │
                    │乐观锁更新│  │乐观锁更新│   │乐观锁更新   │
                    │手动ACK  │  │手动ACK  │   │手动ACK      │
                    └─────────┘  └─────────┘   └─────────────┘
                         │              │              │
                         ▼              ▼              ▼
                    ┌──────────────────────────────────────────────┐
                    │              MySQL (主从架构)                  │
                    │  t_pay_order / t_refund_order                │
                    │  乐观锁: WHERE status = '旧状态'              │
                    └──────────────────────────────────────────────┘

        ┌─────────────────────────────────────────┐
        │          兜底机制 (定时任务)              │
        │  1. 扫描 t_message_fallback 重发         │
        │  2. 扫描超时 PAYING 订单 → 主动查单      │
        │  3. T+1 对账 → 发现差异 → 补单/告警      │
        └─────────────────────────────────────────┘
```

### 可靠性三道防线

| 防线 | 机制 | 解决的问题 |
| :--- | :--- | :--- |
| 发端可靠 | 本地消息表 + 定时重发 | MQ 宕机/网络抖动时不丢消息 |
| 收端幂等 | 手动 ACK + 乐观锁 | 重复消费不重复更新 |
| 极端兜底 | DLQ 告警 + 主动查单 + T+1 对账 | 消费失败/网络分区的最终兜底 |

**最终效果**：消息丢失率 < 0.01%（通过日终对账差异率度量）

---

## 生产环境注意事项

### 金额单位差异（极易踩坑）

| 渠道 | 金额单位 | 类型 | 示例 | 数据库存储 |
| :--- | :--- | :--- | :--- | :--- |
| 微信支付 | **分** | int | 100 = 1元 | 直接存 |
| 支付宝 | **元** | string | "1.00" = 1元 | 转为分再存 |

建议在业务层做统一的金额转换拦截器，入库前统一转为**分（int）**，出参时按渠道转换。

### 回调幂等
支付渠道可能因超时等原因重复发送回调通知，务必保证回调处理的幂等性：
- 数据库更新带状态前置条件（`WHERE status = 'PAYING'`），这是最核心的防线
- 可选：Redis SetNX 做前置去重拦截，减少无效 DB 请求

### 退款防并发
同一笔订单避免在短时间内并发调用退款接口，否则可能触发渠道风控。建议用分布式锁控制：
```go
lockKey := fmt.Sprintf("refund:lock:%s", outTradeNo)
// Redis SetNX 加锁，TTL 30s
```

### 超时订单处理
- 创建订单时设置 `expire_time`（通常 30 分钟）
- 定时任务扫描超时的 PAYING 订单：先向渠道查单确认状态，再决定关单或补单
- 微信支持创建订单时传 `time_expire` 参数让渠道侧也自动过期

### 证书安全
- 私钥、APIv3Key 等敏感信息必须从配置中心/密钥管理服务加载，禁止硬编码
- 微信平台证书建议开启自动更新（`client.AutoVerifySign()`）
- 支付宝证书过期前需要及时更换（关注证书有效期）

---

## 监控与告警

生产环境必须建立的监控指标：

| 监控项 | 告警阈值建议 | 说明 |
| :--- | :--- | :--- |
| 支付成功率 | < 95% 告警 | 成功订单数 / 总下单数 |
| 回调处理延迟 | > 100ms 告警 | 网关层验签+投递 MQ 的耗时 |
| MQ 消费积压 | > 1000 条告警 | Consumer Offset 与 Broker Offset 差值 |
| 退款成功率 | < 99% 告警 | 关注退款失败原因分布 |
| 对账差异数 | > 0 告警 | 每日对账后的长款/短款数量 |
| DLQ 消息数 | > 0 告警 | 死信队列有消息说明消费端异常 |
| 回调重试率 | > 5% 告警 | 渠道重复回调比例，过高说明响应太慢 |

---

## 调试与日志

```go
// 开启 Debug 模式，打印完整的请求/响应日志
client.DebugSwitch = gopay.DebugOn

// 自定义 Logger（实现 xlog.XLogger 接口）
client.SetLogger(yourCustomLogger)
```

---

## 更多资源

- 各渠道 API 文档：`doc/` 目录
- 完整调用示例：各模块 `*_test.go` 文件（与代码同步的活文档）
- 接入示例工程：`examples/` 目录
- 版本变更记录：`release_note.md`
- 支付面试指南：`readme_mianshi.md`
