# 扫码支付模式二

用户使用微信客户端扫码收银员生成的支付二维码，完成付款

# 使用场景

用户确认要购买的商品，收银员进行下单操作，生成一个用来支付的二维码，用户用扫一扫功能扫码二维码，调起微信支付，完成支付
	
# 二维码有效期 

有效期为2个小时

# 业务流程

-	用户确认要购买的商品信息
-	商户系统使用统一下单接口生成支付二维码
-	用户扫描该二维码，调起微信支付

# 扫码支付模式二接口

使用统一下单接口生成要支付二维码

# 开发实现

## 创建nativetwo包

创建文件夹`nativetwo`

## 生成支付二维码

创建文件`native2.go`,实现：
-	`func Native2(w http.ResponseWriter, r *http.Request)`函数:调用统一支付接口，生成支付二维码

代码如下:

	package nativetwo

	import (
	"fmt"
	"net/http"
	"wechat/config"
	"wechat/rand"
	"wechat/unifiedorder"
	)

	//使用统一下单接口生成支付二维码，并打印到浏览器
	func Native2(w http.ResponseWriter, r *http.Request) {

	//进行统一下单
	v := &unifiedorder.Unifieldrequest{Appid: config.APP_ID, Mch_id: config.MCH_ID}
	v.Total_fee = "1"
	v.Body = "NATIVE2支付测试一分钱"
	v.Nonce_str = rand.Getnoncestr(32)
	v.Out_trade_no = rand.Getnoncestr(32)
	fmt.Println(v.Out_trade_no)
	v.Spbill_create_ip = "127.0.0.1"
	v.Notify_url = config.URL_UNIFIEDORDER_NOTIFYURL
	//支付模式一定要设置为NATIVE
	v.Trade_type = "NATIVE"
	v.Product_id = "native2"
	v.Signmd5()
	v.Xml()
	fmt.Fprintf(w, v.RequestXML)
	response := v.Dorequest()

	//打印得到二维码url，然后可以利用第三方工具生成二维码图标，就可以进行扫码支付了
	fmt.Fprintf(w, response.Code_url)
	}

# 测试


