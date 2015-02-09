# 扫码支付模式一

用户使用微信客户端的扫一扫功能，扫描商户的商品二维码，完成支付

# 使用场景

把商品信息按照微信支付要求的规则生成一个二维码，可以把该二维码放在需要的地方，例如：自助购买的商品包装盒上，或者是地铁的广告栏上，然后用户就可以扫码获得该商品的详细信息并支付

# 二维码有效期

长期有效

# 业务流程

-	商户根据特殊的规则，生成商品二维码	
-	用户使用微信“扫一扫”扫描二维码
-	微信后台收到用户的扫描到的二维码信息，
-	微信后台将收到的商品信息发送给在公众平台上设置的商户服务器的回调地址
-	商户服务器的回调地址根据收到的商品信息，调用统一下单接口，获取prepay_id
-	商户服务器的回调地址将统一下单的prepay_id结果发送给微信支付后台
-	微信支付后台收到prepay_id结果后，向微信客户端发送指令，掉起微信支付

# 扫码支付模式一接口

-	使用微信支付规定的格式生成二维码，规则如下，xxx为我们要填写的内容，详情见下面的开发实现
> `weixin://wxpay/bizpayurl?sign=XXXXX&appid=XXXXX&mch_id=XXXXX&product_id=XXXXXX&time_stamp=XXXXXX&nonce_str=XXXXX`
    
-	使用统一统一下单接口发起支付


# 开发实现

## 创建nativeone包

创建文件夹`nativeone`

## 创建商品二维码

创建`qrcode.go`文件，实现

-	`Qrcode`结构体:存放要二维码中要添加的字段
-	`func (v *Qrcode) Signmd5()`函数:对`Qrcode`结构体中的字段进行名md5签名
-	`func (v *Qrcode) Qrcodeurl()`函数：将`Qrcode`结构体里面的字段，按照微信要求的规则组成特定url，该url就是我们的二维码，可以利用工具把它转化为二维码团

代码实现:

	package nativeone

	import (
	"wechat/config"
	"wechat/wlog"
	"wechat/wsign"
	)

	//1.存储二维码信息的结构体
	type Qrcode struct {
	Appid      string `xml:"appid"`  //公众帐号ID
	Mch_id     string `xml:"mch_id"` //商户号
	Nonce_str  string `xml:"nonce_str"` //随机字符串
	Product_id string `xml:"product_id"`//商品id，商户自定义，在用户扫码的时候会传给商户设置的回调url，商户根据该信息在自身的系统里面查找商品，然后来发起支付
	Sign       string `xml:"sign"`       //md5签名
	Time_stamp string `xml:"time_stamp"`  //时间戳,毫秒级
	}

	//2.对Qrcode里面的有效信息进行md5签名，并存储该签名
	func (v *Qrcode) Signmd5() bool {
	sign, _ := wsign.SignMD5(*v, config.API_KEY)
	v.Sign = sign
	wlog.Println(sign)
	return true
	}

	//3.对Qrcode里面的字段进行组装，按照微信支付要求的规则生成特定的url
	func (v *Qrcode) Qrcodeurl() string {
	qrcodeurl := "weixin://wxpay/bizpayurl?sign=" + v.Sign + "&appid=" + v.Appid + "&mch_id=" + v.Mch_id + "&product_id=" + v.Product_id + "&time_stamp=" + v.Time_stamp + "&nonce_str=" + v.Nonce_str
	return qrcodeurl
	}

当我们扫码我们生成的二维码时候，微信支付就会把该二维码的信息发送到商户填写的回调url，在该url里，我们要实现微信支付的调起

## 回调url实现

创建`nativecallback.go`文件,实现:

-	`Callback`结构体:存储用户扫码时候，微信支付后台发送给回调url的参数
-	`func Nativeonecallback(w http.ResponseWriter, r *http.Request)`函数:回调地址对应的处理函数，用户扫码时候，微信支付将二维码信息传到该函数,该函数根据得到的二维码信息，获得对应的商品信息，调用统一下单接口，获取prepay_id，并白统一下单的接口的结果发送给微信支付后台，然后微信支付后台会去调起用户手机端的微信支付操作

代码实现：

	package nativeone

	import (
	"encoding/xml"
	"fmt"
	"io/ioutil"
	"net/http"
	"wechat/config"
	"wechat/rand"
	"wechat/unifiedorder"
	"wechat/wxml"
	)

	//1.存储用户扫码二维码时候，微信支付发送给回调url的数据
	type Callback struct {
	XMLName      xml.Name `xml:"xml"`
	Appid        string   `xml:"appid"`        //公众号ID
	Is_subscribe string   `xml:"is_subscribe"` //标识用户是否关注该公众号
	Mch_id       string   `xml:"mch_id"`       //商户号
	Nonce_str    string   `xml:"nonce_str"`    //随机字符串
	Openid       string   `xml:"openid"`       //用户openid
	Product_id   string   `xml:"product_id"`   //商品id
	Sign         string   `xml:"sign"`         //签名
	}

	//1.回调url，实现统一下单操作，并把下单结果发送给微信支付后台
	func Nativeonecallback(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()

	//将微信支付后台发送的二维码相关信息解析存储到Callback结构体
	result, _ := ioutil.ReadAll(r.Body)
	vw := &Callback{}
	wxml.Decodebytes(result, vw)

	//进行统一下单
	v := &unifiedorder.Unifieldrequest{Appid: config.APP_ID, Mch_id: config.MCH_ID}
	v.Total_fee = "1"
	v.Body = "NATIVE1支付测试一分钱"
	v.Nonce_str = rand.Getnoncestr(32)
	v.Out_trade_no = rand.Getnoncestr(32)
	v.Spbill_create_ip = "127.0.0.1"
	v.Notify_url = config.URL_UNIFIEDORDER_NOTIFYURL
	v.Trade_type = "NATIVE"
	v.Product_id = vw.Product_id
	v.Signmd5()
	v.Xml()
	response := v.Dorequest()

	//将统一下单的结果发送给微信支付后台
	xmlresult, _ := wxml.Endoestruct(response)
	fmt.Fprintf(w, xmlresult)

	fmt.Println("========================================")
	}

我们到这一步实现了扫码支付模式1的二维码生成，以及对应的回调url对扫码回调的处理，那么要让微信后台把这二维码的信息发送的回调的url，我们需要去公众平台设置添加这个回调url

## 设置回调url

-	登录mp.weixin.qq.com
-	进入左边:微信支付->开发配置菜单
-	点击修改，填写:Native原生支付回调URL，在这个例子里面我们填写为：http://xxx/Nativeonecallback



## 测试

   










