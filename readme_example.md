# GoPay 完整接入实战示例

本文档提供基于 GoPay SDK 的**生产级**接入示例，涵盖客户端初始化、多场景下单、回调处理、退款、主动查单、对账的完整闭环。

---

## 1. 项目目录结构

```text
payment-service/
├── config/
│   └── config.yaml             # 配置：商户号、证书路径、MQ 地址等
├── internal/
│   ├── pkg/payclient/          # 支付客户端单例初始化
│   ├── handler/                # HTTP 路由层：下单接口、回调接口
│   ├── service/                # 业务逻辑层：下单、退款、查单
│   ├── consumer/               # MQ 消费者：异步处理支付结果
│   └── repository/             # 数据访问层 (GORM)
├── cert/                       # 证书目录（生产环境建议外部挂载）
│   ├── wechat/apiclient_key.pem
│   └── alipay/
├── go.mod
└── main.go
```

---

## 2. 配置文件 (config/config.yaml)

```yaml
server:
  port: 8080

wechat:
  mch_id: "1600000000"
  serial_no: "7132CECE9xxxxxx"
  api_v3_key: "your-api-v3-key-32chars"
  private_key_path: "cert/wechat/apiclient_key.pem"
  app_id: "wx1234567890"
  notify_url: "https://api.yourdomain.com/v1/pay/wechat/notify"

alipay:
  app_id: "20210000000000"
  private_key_path: "cert/alipay/app_private_key.pem"
  app_cert_path: "cert/alipay/appPublicCert.crt"
  root_cert_path: "cert/alipay/alipayRootCert.crt"
  public_cert_path: "cert/alipay/alipayPublicCert.crt"
  is_prod: true
  notify_url: "https://api.yourdomain.com/v1/pay/alipay/notify"

rocketmq:
  name_server: "127.0.0.1:9876"
  topic: "PAY_NOTIFY"
  consumer_group: "PAY_CONSUMER_GROUP"
```

---

## 3. 支付客户端初始化 (internal/pkg/payclient/client.go)

```go
package payclient

import (
	"log"
	"os"

	"github.com/go-pay/gopay"
	alipayv3 "github.com/go-pay/gopay/alipay/v3"
	wechatv3 "github.com/go-pay/gopay/wechat/v3"
)

var (
	WechatClient *wechatv3.ClientV3
	AlipayClient *alipayv3.ClientV3
)

func InitClients(cfg *Config) {
	initWechat(cfg.Wechat)
	initAlipay(cfg.Alipay)
}

func initWechat(cfg WechatConfig) {
	privateKey, err := os.ReadFile(cfg.PrivateKeyPath)
	if err != nil {
		log.Fatalf("[Wechat] 读取私钥失败: %v", err)
	}

	client, err := wechatv3.NewClientV3(cfg.MchId, cfg.SerialNo, cfg.ApiV3Key, string(privateKey))
	if err != nil {
		log.Fatalf("[Wechat] 初始化失败: %v", err)
	}

	// 开启自动验签：后台 goroutine 每 12 小时自动拉取最新平台证书
	if err = client.AutoVerifySign(); err != nil {
		log.Fatalf("[Wechat] 开启自动验签失败: %v", err)
	}

	client.DebugSwitch = gopay.DebugOff // 生产环境关闭
	WechatClient = client
	log.Println("[Wechat] 客户端初始化成功")
}

func initAlipay(cfg AlipayConfig) {
	privateKey, err := os.ReadFile(cfg.PrivateKeyPath)
	if err != nil {
		log.Fatalf("[Alipay] 读取私钥失败: %v", err)
	}

	client, err := alipayv3.NewClientV3(cfg.AppId, string(privateKey), cfg.IsProd)
	if err != nil {
		log.Fatalf("[Alipay] 初始化失败: %v", err)
	}

	// 设置证书（推荐证书签名模式）
	appCert, _ := os.ReadFile(cfg.AppCertPath)
	rootCert, _ := os.ReadFile(cfg.RootCertPath)
	publicCert, _ := os.ReadFile(cfg.PublicCertPath)
	if err = client.SetCert(appCert, rootCert, publicCert); err != nil {
		log.Fatalf("[Alipay] 设置证书失败: %v", err)
	}

	AlipayClient = client
	log.Println("[Alipay] 客户端初始化成功")
}
```

---

## 4. 业务逻辑层 (internal/service/pay_service.go)

### 4.1 微信 Native 扫码下单

```go
package service

import (
	"context"
	"fmt"
	"time"

	"github.com/go-pay/gopay"
	"yourproject/internal/pkg/payclient"
)

// CreateWechatNativeOrder 创建微信 Native 扫码支付订单
func CreateWechatNativeOrder(ctx context.Context, orderNo string, amountFen int, desc string) (string, error) {
	bm := make(gopay.BodyMap)
	bm.Set("appid", "wx1234567890").
		Set("description", desc).
		Set("out_trade_no", orderNo).
		Set("notify_url", "https://api.yourdomain.com/v1/pay/wechat/notify").
		SetBodyMap("amount", func(b gopay.BodyMap) {
			b.Set("total", amountFen). // 单位：分
				Set("currency", "CNY")
		})

	// ⚠️ 生产必须带超时控制，防止渠道卡死拖垮服务
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	rsp, err := payclient.WechatClient.V3TransactionNative(ctx, bm)
	if err != nil {
		return "", fmt.Errorf("微信Native下单网络异常: %w", err)
	}
	if rsp.Code != 0 {
		return "", fmt.Errorf("微信返回业务错误: code=%d, err=%s", rsp.Code, rsp.Error)
	}
	return rsp.Response.CodeUrl, nil // 返回支付二维码链接
}
```

### 4.2 微信 JSAPI 下单 (小程序/公众号)

```go
// CreateWechatJsapiOrder JSAPI 下单，需要用户 openid
func CreateWechatJsapiOrder(ctx context.Context, orderNo string, amountFen int, desc, openid string) (string, error) {
	bm := make(gopay.BodyMap)
	bm.Set("appid", "wx1234567890").
		Set("description", desc).
		Set("out_trade_no", orderNo).
		Set("notify_url", "https://api.yourdomain.com/v1/pay/wechat/notify").
		SetBodyMap("amount", func(b gopay.BodyMap) {
			b.Set("total", amountFen)
		}).
		SetBodyMap("payer", func(b gopay.BodyMap) {
			b.Set("openid", openid) // JSAPI 必须传 openid
		})

	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	rsp, err := payclient.WechatClient.V3TransactionJsapi(ctx, bm)
	if err != nil {
		return "", fmt.Errorf("微信JSAPI下单失败: %w", err)
	}
	if rsp.Code != 0 {
		return "", fmt.Errorf("微信返回错误: %s", rsp.Error)
	}
	return rsp.Response.PrepayId, nil // 前端用 PrepayId 调起支付
}
```

### 4.3 支付宝网页支付

```go
// CreateAlipayPageOrder 支付宝电脑网站支付（返回支付页面 URL）
func CreateAlipayPageOrder(ctx context.Context, orderNo string, amountYuan float64, desc string) (string, error) {
	bm := make(gopay.BodyMap)
	bm.Set("subject", desc).
		Set("out_trade_no", orderNo).
		Set("total_amount", fmt.Sprintf("%.2f", amountYuan)). // ⚠️ 支付宝单位是元(string)
		Set("product_code", "FAST_INSTANT_TRADE_PAY").
		Set("notify_url", "https://api.yourdomain.com/v1/pay/alipay/notify").
		Set("return_url", "https://www.yourdomain.com/pay/success")

	payUrl, err := payclient.AlipayClient.TradePagePay(ctx, bm)
	if err != nil {
		return "", fmt.Errorf("支付宝PagePay下单失败: %w", err)
	}
	return payUrl, nil // 浏览器直接重定向到此 URL
}
```

### 4.4 退款接口

```go
// RefundWechatOrder 发起微信退款
func RefundWechatOrder(ctx context.Context, outTradeNo, outRefundNo string, refundFen, totalFen int, reason string) (string, error) {
	bm := make(gopay.BodyMap)
	bm.Set("out_trade_no", outTradeNo).
		Set("out_refund_no", outRefundNo).
		Set("reason", reason).
		Set("notify_url", "https://api.yourdomain.com/v1/pay/wechat/refund-notify").
		SetBodyMap("amount", func(b gopay.BodyMap) {
			b.Set("refund", refundFen). // 本次退款金额(分)
				Set("total", totalFen). // 原订单金额(分)
				Set("currency", "CNY")
		})

	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	rsp, err := payclient.WechatClient.V3Refund(ctx, bm)
	if err != nil {
		return "", fmt.Errorf("退款请求异常: %w", err)
	}
	if rsp.Code != 0 {
		return "", fmt.Errorf("退款失败: %s", rsp.Error)
	}
	return rsp.Response.RefundId, nil
}
```

### 4.5 主动查单（定时任务用）

```go
import wechatv3 "github.com/go-pay/gopay/wechat/v3"

// QueryWechatOrder 主动调用微信查单 API，用于定时任务兜底
func QueryWechatOrder(ctx context.Context, outTradeNo string) (string, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	rsp, err := payclient.WechatClient.V3TransactionQueryOrder(ctx, wechatv3.OutTradeNo, outTradeNo)
	if err != nil {
		return "", fmt.Errorf("查单网络异常: %w", err)
	}
	if rsp.Code != 0 {
		return "", fmt.Errorf("查单失败: %s", rsp.Error)
	}
	// rsp.Response.TradeState: SUCCESS / NOTPAY / CLOSED / REFUND 等
	return rsp.Response.TradeState, nil
}
```

---

## 5. 回调处理层 (internal/handler/pay_handler.go)

```go
package handler

import (
	"encoding/json"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
	wechatv3 "github.com/go-pay/gopay/wechat/v3"
	"yourproject/internal/pkg/payclient"
)

// WechatPayNotify 微信支付成功回调
func WechatPayNotify(c *gin.Context) {
	// Step 1: 解析请求（自动读取 Body + Header 签名信息）
	notifyReq, err := wechatv3.V3ParseNotify(c.Request)
	if err != nil {
		log.Printf("解析回调失败: %v", err)
		c.JSON(http.StatusBadRequest, gin.H{"code": "FAIL", "message": "parse error"})
		return
	}

	// Step 2: 验签（使用 SDK 内部维护的微信平台证书公钥）
	err = notifyReq.VerifySignByPKMap(payclient.WechatClient.SnCertMap.Map())
	if err != nil {
		log.Printf("验签失败: %v", err)
		c.JSON(http.StatusUnauthorized, gin.H{"code": "FAIL", "message": "verify sign error"})
		return
	}

	// Step 3: 解密支付结果明文
	result, err := notifyReq.DecryptPayCipherText("your-api-v3-key")
	if err != nil {
		log.Printf("解密失败: %v", err)
		c.JSON(http.StatusInternalServerError, gin.H{"code": "FAIL", "message": "decrypt error"})
		return
	}

	// Step 4: 投递到 MQ 异步处理（不要在回调中直接操作数据库！）
	if result.TradeState == "SUCCESS" {
		msgBody, _ := json.Marshal(result)
		log.Printf("支付成功 -> 投递MQ: orderNo=%s, amount=%d", result.OutTradeNo, result.Amount.Total)
		// TODO: mq.Publish("PAY_NOTIFY", result.OutTradeNo, msgBody)
		_ = msgBody
	}

	// Step 5: 极速响应 200 OK（< 20ms），防止微信判定超时触发重试
	c.JSON(http.StatusOK, gin.H{"code": "SUCCESS", "message": "OK"})
}

// WechatRefundNotify 微信退款结果回调
func WechatRefundNotify(c *gin.Context) {
	notifyReq, err := wechatv3.V3ParseNotify(c.Request)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"code": "FAIL", "message": "parse error"})
		return
	}

	// 退款回调用 DecryptRefundCipherText 解密
	refundResult, err := notifyReq.DecryptRefundCipherText("your-api-v3-key")
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"code": "FAIL", "message": "decrypt error"})
		return
	}

	log.Printf("退款通知: orderNo=%s, refundNo=%s, status=%s",
		refundResult.OutTradeNo, refundResult.OutRefundNo, refundResult.RefundStatus)

	// TODO: 投递 MQ 异步更新退款单状态
	c.JSON(http.StatusOK, gin.H{"code": "SUCCESS", "message": "OK"})
}
```

---

## 6. MQ 消费者 (internal/consumer/pay_consumer.go)

```go
package consumer

import (
	"database/sql"
	"encoding/json"
	"log"

	wechatv3 "github.com/go-pay/gopay/wechat/v3"
)

// HandlePaySuccess 消费支付成功消息（核心：乐观锁幂等处理）
func HandlePaySuccess(msgBody []byte, db *sql.DB) error {
	var result wechatv3.V3DecryptPayResult
	if err := json.Unmarshal(msgBody, &result); err != nil {
		return err
	}

	// 核心：乐观锁更新，WHERE status='PAYING' 保证幂等
	res, err := db.Exec(
		`UPDATE t_pay_order 
		 SET status='SUCCESS', transaction_id=?, pay_time=NOW(), updated_at=NOW() 
		 WHERE out_trade_no=? AND status='PAYING'`,
		result.TransactionId, result.OutTradeNo,
	)
	if err != nil {
		return err // 返回 error → MQ 不 ACK → 触发重试
	}

	affected, _ := res.RowsAffected()
	if affected == 0 {
		// 已被处理过（幂等），直接跳过
		log.Printf("订单 %s 已处理，跳过", result.OutTradeNo)
	} else {
		log.Printf("订单 %s 状态更新为 SUCCESS", result.OutTradeNo)
		// TODO: 后续业务（如充值加余额、发通知等）
	}

	return nil // 返回 nil → 提交 ACK
}
```

---

## 7. 程序入口 (main.go)

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
	"yourproject/internal/handler"
	"yourproject/internal/pkg/payclient"
)

func main() {
	// 1. 加载配置
	// cfg := config.Load("config/config.yaml")

	// 2. 初始化支付客户端（全局单例）
	payclient.InitClients(nil)

	// 3. 初始化路由
	r := gin.Default()
	v1 := r.Group("/v1/pay")
	{
		v1.POST("/wechat/notify", handler.WechatPayNotify)
		v1.POST("/wechat/refund-notify", handler.WechatRefundNotify)
		v1.POST("/alipay/notify", handler.AlipayNotify)
	}

	// 4. 优雅启停
	srv := &http.Server{Addr: ":8080", Handler: r}
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("启动失败: %v", err)
		}
	}()
	log.Println("支付服务已启动，监听 :8080")

	// 5. 等待退出信号
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("正在优雅关闭...")

	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatalf("强制关闭: %v", err)
	}
	// TODO: consumer.Shutdown() / db.Close()
	log.Println("服务已安全退出")
}
```

---

## 8. 常见问题排查

| 问题现象 | 排查方向 |
| :--- | :--- |
| `crypto/rsa: verification error` | ① Body 被中间件提前读取消费；② APIv3Key 填错；③ 平台证书过期未刷新 |
| `cert not match error` | 商户号和证书序列号不匹配，或服务器时钟不准 |
| 回调收不到 | ① notify_url 不是公网 HTTPS；② 防火墙拦截了微信 IP；③ 微信商户后台未配置回调域名 |
| 支付宝签名报错 | 回调 Form 参数不能做任何 URL Decode 或过滤操作，必须用原始值验签 |
| 订单一直 PAYING | 主动查单定时任务未启动，或查单频率太低 |

## 9. 安全清单

- ✅ 私钥、APIv3Key 绝不硬编码，通过环境变量或 Vault 注入
- ✅ 回调解密后**必须校验金额**与本地订单一致
- ✅ Docker 镜像不打包证书文件，用 K8s Secret 挂载
- ✅ 回调接口不做鉴权（微信/支付宝需要能访问），但必须验签
- ✅ 生产环境关闭 `DebugSwitch`，避免泄露敏感数据到日志
