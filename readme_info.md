# 🚀 GoPay: 企业级高并发支付网关核心引擎

<p align="center">
  <img width="180" height="180" src="logo.png" alt="GoPay Logo">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/golang-1.24.0-brightgreen.svg" alt="Golang">
  <img src="https://img.shields.io/badge/license-Apache%202.0-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/Architecture-High%20Concurrency-orange.svg" alt="Architecture">
</p>

`gopay` 是一款久经生产环境考验的 Golang 支付集成利器。我们不仅提供基础的 SDK 封装，更致力于输出**现代、高可用、可扩展的支付网关架构规范**。通过彻底抽象底层接口差异，`gopay` 让构建千万级交易体量的充值、退款链路变得轻而易举。

---

## 🏗 项目架构与目录索引

基于良好的开闭原则（OCP）与高内聚设计，项目结构一目了然：

```text
gopay/
├── alipay/           // 📦 支付宝 API (支持 V3 与老版本)
├── wechat/           // 📦 微信支付 API (包含 V2/V3)
├── apple/            // 📦 苹果应用内支付 (IAP) 校验
├── paypal/           // 📦 PayPal 全球支付支持
├── pkg/              // 🧰 公共依赖：工具类、加解密算法核心
├── body_map.go       // 🌟 核心结构：独创的动态参数构造器
├── client_test.go    // 🧪 极其详尽的真实调用用例 (即时文档)
└── examples/         // 📚 各大框架下的实战接入示例
```

---

## 🌟 核心能力矩阵

| 渠道特性 | 支付 (Pay) | 查询 (Query) | 退款 (Refund) | 回调验签 (Notify) | 动态证书管理 |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **微信 V3** | ✅ | ✅ | ✅ | ✅ | ✅ (平台自动更新) |
| **支付宝 V3** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **PayPal** | ✅ | ✅ | ✅ | ✅ | N/A |

> **架构亮点**：本 SDK 的请求链路经过极致精简，**完全不持有阻塞状态**。这使得其可以无缝接入任何基于 **RocketMQ / Kafka** 的异步削峰架构中，助力系统完成吞吐量（QPS）的大幅跃升。

---

## 🛠 最佳实践接入指南

### 1. 快速安装
```bash
go get github.com/go-pay/gopay
```

### 2. 核心利器：BodyMap 的优雅构造
摒弃拼凑繁杂的 `struct` 结构体，利用 `BodyMap` 实现丝滑的动态参数装载：
```go
bm := make(gopay.BodyMap)
bm.Set("out_trade_no", "202604210001").
   Set("total_fee", 1).
   // 优雅支持深度嵌套
   SetBodyMap("detail", func(b gopay.BodyMap) {
       b.Set("cost_price", 600).
         Set("receipt_id", "sx124")
   })
```

---

### 3. 实战演练：微信 V3 充值与异步回调

构建可靠支付链路的核心在于**解耦**。以下展示带有良好架构风格的代码示例：

**【网关层】初始化与下单**
```go
import "github.com/go-pay/gopay/wechat/v3"

// 1. 初始化客户端 (从配置中心加载秘钥)
client, err := wechat.NewClientV3("MchId", "SerialNo", "APIv3Key", "PK_Content")
if err != nil {
    panic("初始化致命错误: " + err.Error())
}

// 2. 开启自动公钥验签同步防黑客攻击
client.AutoVerifySignByPublicKey([]byte("微信支付公钥内容"), "PUB_KEY_ID_XXX")

// 3. 构建充值参数并发起请求
bm := make(gopay.BodyMap)
bm.Set("appid", "wx12345678").Set("description", "用户钱包充值") // ...省去其他参数
wxRsp, err := client.V3TransactionNative(ctx, bm)

// 4. 严谨的错误处理机制
if err != nil {
    xlog.Error("网络层或系统层异常:", err)
    return
}
if wxRsp.Code != wechat.Success {
    xlog.Errorf("业务层失败, 错误码:%s, 信息:%s", wxRsp.Error, wxRsp.Error)
    return
}
```

**【网关层】回调削峰 (极速响应)**
```go
// 解析并进行签名强校验
notifyReq, err := wechat.V3ParseNotify(request)
if err != nil { return err }

// 解密核心报文
result, err := notifyReq.DecryptCipherText("APIv3Key")
if err == nil {
    // ⚠️ 警告：千万不要在这里查库和修改订单状态！
    // 正确做法：立即序列化 result 发入 RocketMQ，然后响应 HTTP 200。
    // MQProducer.Send(result) 
    xlog.Infof("回调校验成功并已投递 MQ，交易号: %s", result.TransactionId)
}
```

---

## 🛡️ 企业级排坑与监控防线 (Debug Tips)

作为支付网关，不容许哪怕万分之一的偏差。请务必检查以下防线：

1. **绝对拦截机制 (Redis SetNX)**：
   在处理回调、处理退款动作前，**必须且一定**要用 Redis 等分布式锁进行拦截处理，防范渠道方的突发重试风暴。
2. **底层单位红线**：
   千万牢记：**微信支付金额单位是【分】（Int），支付宝金额单位是【元】（String）**。建议在您自己的代码中做一层强类型的金额转换拦截器。
3. **退款原路返回防并发**：
   调用退款接口 `client.V3Refund()` 时，请确保同一笔订单不要在 1 秒内并发调用，否则极易触发第三方风控告警。

---

## 💬 社区与技术支持

* 详尽的 API 定义，尽在各大模块的 `*_test.go` 中，那是与代码时刻同步的最佳活文档。
* 我们鼓励以 Issue 形式提交 Bug，也欢迎 PR 为建设更健壮的 Go 支付生态添砖加瓦！
