# JSAPI支付

公众号支付是用户在微信中打开商户的H5页面，商户在H5页面通过调用微信支付提供的JSAPI接口调起微信支付模块完成支付

# 使用场景

-	用户在微信公众账号(必须是服务号)内进入商家公众号，打开某个H5页面，完成支付

-	用户的好友在朋友圈、聊天窗口等分享商家H5页面连接，用户点击链接打开商家H5页面，完成支付

-	将商户H5页面转换成二维码，用户扫描二维码后在微信浏览器中打开h5页面后完成支付


# JSAPI接口 

-	使用JS脚本：getBrandWCPayRequest 掉起微信支付


-	输入输出的参数格式为JSON
> getBrandWCPayRequest在其他浏览器中无效，必须在微信的浏览器中
	
> getBrandWCPayRequest使用的参数名区分大小

# 业务流程

-	商户下发图文消息或者通过自定义菜单吸引用户点击进入商户网页

-	进入商户网页，用户选择商品购买，完成选购流程

-	用户确认购买，商户使用统一下单接口获取prepay_id

-	商户网页使用getBrandWCPayRequest接口调起微信支付

#	开发实现

##	创建jsapipay包

创建文件夹jsapipay

## 创建一个我们要使用JSAPI支付的网页

创建`jsapi.html`文件，实现:

- 实现对getBrandWCPayRequest接口的调用，从而调起微信支付
- 打印出我们使用的参数

代码如下:

	<!DOCTYPE html>
	<html>
	<title>JSAPI支付一分钱</title>
	<head>

	 <script type="text/javascript">

    //实现微信支付JS脚本
    function pay() {
      
        WeixinJSBridge.invoke(
            'getBrandWCPayRequest', {
                "appId": "{{.AppId}}", 	 //公众号名称，由商户传入
                "timeStamp": "{{.TimeStamp}}",//时间戳，自1970年以来的秒数
                "nonceStr": "{{.NonceStr}}",//随机串
                "package": "{{.Package}}",
                "signType": "{{.SignType}}",//微信签名方式:
                "paySign": "{{.PaySign}}"   //微信签名
            },
            function (res) {
                if (res.err_msg == "get_brand_wcpay_request:ok") {
                    alert("支付成功");

                }else if (res.err_msg == "get_brand_wcpay_request:cancel")  {
                     alert("支付过程中用户取消");
                 }else{
                    //支付失败
                    alert(res.err_msg)
                 }
                
            }
        );
    }


 	</script>
	</head>

	<body>


	<!--打印出参数-->
	<h1>微信jsapi支付 1分钱</h1>
	<a>Appid:   {{.AppId}}</a><br>
	<a>TimeStamp:   {{.TimeStamp}}</a><br>
	<a>Nonce_str:   {{.NonceStr}}</a><br>
	<a>Package: {{.Package}}</a><br>
	<a>SignType:    {{.SignType}}</a><br>
	<a>PaySign: {{.PaySign}}</a><br>

	<button type="button"  onclick="pay()">点击微信支付</button>
	</body>
	</html>



##  getBrandWCPayRequest接口参数生成

创建文件`jsapirequest.go`,实现

-	Jsapirequest结构体:存储getCPayReBrandWquest需要的参数，传递给`jsapi.html`页面
-	Signmd5()函数:对Jsapirequest结构体中的非空参数进行签名，并存储签名结果

代码如下：

	package jsapipay

	import (
	"encoding/xml"
	"wechat/config"
	"wechat/wsign"
	)

	//1.定义Jsapirequest结构体，存储getBrandWCPayRequest接口要使用的参数
	type Jsapirequest struct {
	XMLName   xml.Name `xml:"xml"`
	AppId     string   `xml:"appId"`     //公众账号ID
	NonceStr  string   `xml:"nonceStr"`  //随机字符串
	Package   string   `xml:"package"`   //账单类型
	SignType  string   `xml:"signType"`  //商户号
	TimeStamp string   `xml:"timeStamp"` //对账单日起

	PaySign string `xml:"-"` //最终请求串

	}

	//2.对Jsapirequest的非空字段进行md5签名，并存储签名结构
	func (v *Jsapirequest) Signmd5() bool {
	sign, _ := wsign.SignMD5(*v, config.API_KEY)
	v.PaySign = sign
	return true
	}

## 发起jsapi支付

创建文件`jsapi.go`，实现

-	`func Jsapi(w http.ResponseWriter, r *http.Request)`：发起jsapi支付的入口,进行统一下单操作，然后打开`jsapi.html`支付页面，并把相关的参数填充到Jsapirequest结构体，传递到`jsapi.html`页面，调起支付

代码如下:

	package jsapipay

	import (
	"fmt"
	"html/template"
	"io/ioutil"
	"net/http"
	"wechat/config"
	"wechat/rand"
	"wechat/unifiedorder"
	"wechat/wlog"
	"wechat/wtime"
	)

	func Jsapi(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	// code := r.FormValue("code")
	// if code == "" {
	// 	http.Redirect(w, r, getjsapiurl(), http.StatusForbidden)
	// }

	//1.统一下单
	v := &unifiedorder.Unifieldrequest{Appid: config.APP_ID, Mch_id: config.MCH_ID}

	//支付金额，单位为分
	v.Total_fee = "1"
	//商品说明
	v.Body = "JSAPI支付测试一分钱"
	//32位随机字符串
	v.Nonce_str = rand.Getnoncestr(32)
	//商户订单号，此处也随机生成
	v.Out_trade_no = rand.Getnoncestr(32)
	//发起支付的机器IP
	v.Spbill_create_ip = "127.0.0.1"
	//设置openid,可以写死也可以根据code获取
	if r.FormValue("code") != "" {
		v.Openid = Getopenid(w, r)
	} else {
		v.Openid = "omL67jm0A1sKwystTq7WsU28MF_c"
	}
	//最终支付成功的通知地址
	v.Notify_url = config.URL_UNIFIEDORDER_NOTIFYURL
	//支付方式为JSAPI
	v.Trade_type = "JSAPI"
	//对上面设置的字段进行签名
	v.Signmd5()
	//把所有的有效字段组织为xml格式
	v.Xml()

	//向统一下单接口发起请求，并把返回请求结果
	res := v.Dorequest()
	fmt.Println("prepayid=", res.Prepay_id)
	fmt.Println("========================================")

	//打开支付网页,并传递参数,包括传递我们上面统一下单获取到的prepay_id
	if r.Method == "GET" {
		path, _ := os.Getwd()
		t, err := template.ParseFiles(path + "/jsapi.html")
		fmt.Println(err)
		da := Jsapirequest{AppId: config.APP_ID}
		da.Package = "prepay_id=" + res.Prepay_id
		da.TimeStamp = wtime.Time_stamp()
		da.NonceStr = rand.Getnoncestr(32)
		da.SignType = "MD5"
		da.Signmd5()

		t.Execute(w, da)
	}
	}


## 结果判定

在`jsapi.html`里面，根据判定结果，进行相应的处理

	返回值	描述
	get_brand_wcpay_request:ok	支付成功
	get_brand_wcpay_request:cancel	支付过程中用户取消
	get_brand_wcpay_request:fail	支付失败

## [Github地址](https://github.com/luciscode/wechat/tree/master/src/wechat/jsapipay)





