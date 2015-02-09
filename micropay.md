
# 刷卡支付

刷卡支付是用户展示钱包内的“刷微信卡条码/二维码”给商户系统扫描后直接完成支付的模式。主要应用线下面对面收银的场景，需要有扫描设备的支撑，最常见的莫过于扫码枪，但注意不能是激光扫码枪，因为这种枪因为反光的原因无法扫码手机上的条形码


# 使用场景


-	用户打开微信，进入"我"->“钱包”->“刷卡”，展示条形码的界面         

-	收银台在商户系统操作生成支付订单，用户确认支付金额

-	收银员使用扫码枪扫码用户展示的条形码/二维码,获得条形码/二维码信息，商户系统使用刷卡支付接口提交支付

-	机端会弹出密码输入框要求用户输入密码。支付成功或者失败都会弹出对应的提示


###### 验密规则


-	支付金额>1000的交易需要输入密码
	
-	用户账户每天最多允许5笔免密交易，超过需要输入密码

-	微信支付后台判定用户行为有异常，符合免密规则的也要求输入密码
	
## 被扫支付接口

	https://api.mch.weixin.qq.com/pay/micropay

-	数据提交方式为post方式

-	数据格式为xml格式



#	 业务流程

-	使用扫码备读设取微信用户刷卡授权码，
-	将订单信息和授权码组成特定的xml数据格式，以post的方式提交的到接口地址（建议先传到商户自己的服务器，然后在商户的服务器端向接口以post的方式来提交支付，当然也可以在收银台发起，但不建议这样做，网络，安全在收银台都无法有效保障）

在上面向接口提交支付的时候，会返回对应的结果，分2种情况，一种是直接扣款，称为免密；一种是微信支付要求用户输入密码，称为验密

-	免密：返回支付结果，告知商户支付结果，成功或者失败。成功的话同时用户也会收到支付的通知，通过微信，短信的方式
-	验密：返回支付结果，返回的结果为USERPAYING状态，表示用户正在输入密码，此时商户可以每隔5秒，使用查询订单来查询该笔支付的结果




#	开发实现
## 创建micropay包 

   创建文件夹micropay 

## 支付请求
 
创建文件 `micropayrequest.go`，实现:

-	`Mircopayrequest`结构体：存储被扫支付需要提交的参数
-	`func (v *Mircopayrequest) Signmd5()`:对`Mircopayrequest`里面的非空字段进行签名，并存储签名
-	`func (v *Mircopayrequest) Xml()`:根据`Mircopayrequest`里面的非空字段，生成对应的xml格式数据，并存储该数据
-	`func (v Mircopayrequest) Dorequest() `:向被扫支付接口以post的请求方式发送生成的xml数据，并返回结果



代码实现

	package micropay

	import (
	"encoding/xml"
	//配置文件
	"wechat/config"
	//http请求工具
	"wechat/whttp"
	//签名工具wsign
	"wechat/wsign"
	//xml格式转换工具
	"wechat/wxml"
	)

	//1.创建Mircopayrequest结构体，存放我们的请求字段，带有 omitempty 表示该字段选填，其他的字段属于必填
	type Mircopayrequest struct {
	XMLName          xml.Name `xml:"xml"`                   //表示xml数据格式
	Appid            string   `xml:"appid"`                 //公众账号ID
	Attach           string   `xml:"attach,omitempty"`      //附加数据
	Auth_code        string   `xml:"auth_code"`             //授权码
	Body             string   `xml:"body"`                  //商品描述
	Detail           string   `xml:"detail,omitempty"`      //商品详情
	Device_info      string   `xml:"device_info,omitempty"` //设备号
	Fee_type         string   `xml:"fee_type,omitempty"`    //货币类型
	Goods_tag        string   `xml:"goods_tag,omitempty"`   //商品标记
	Mch_id           string   `xml:"mch_id"`                //商户号
	Nonce_str        string   `xml:"nonce_str"`             //随机字符串
	Out_trade_no     string   `xml:"out_trade_no"`          //商户订单号
	Sign             string   `xml:"sign"`                  //签名
	Spbill_create_ip string   `xml:"spbill_create_ip"`      //终端IP
	Time_expite      string   `xml:"time_expite,omitempty"` //交易失效时间
	Time_start       string   `xml:"time_start,omitempty"`  //交易起始时间
	Total_fee        string   `xml:"total_fee"`             //总金额

	RequestXML string `xml:"-"` //存放最终生成的xml请求数据,不参与签名以及xml的组成
	}

	//2.对Mircopayrequest里面的字段进行md5签名，存储到Mircopayrequest里面的Sign变量
	func (v *Mircopayrequest) Signmd5() bool {
	sign, _ := wsign.SignMD5(*v, config.API_KEY)
	v.Sign = sign
	return true
	}

	//3.生成Mircopayrequest对应的xml数据，存储到Mircopayrequest里面的RequestXML变量
	func (v *Mircopayrequest) Xml() error {
	xmlresult, err := wxml.Endoestruct(v)
	v.RequestXML = xmlresult
	return err
	}

	//4.向刷卡接口发送post的请求，请求的数据为我们的生成存储在RequestXML变量的xml数据
	//  并且将得到的数据解析到Mircopayresponse结构体
	func (v Mircopayrequest) Dorequest() Mircopayresponse {
	//发送post请求,f返回数据为data
	data := whttp.Post(config.URL_MICROPAY, v.RequestXML)

	//解析data到Mircopayresponse结构体
	mircopayresponse := Mircopayresponse{}
	wxml.Decodebytes(data, &mircopayresponse)
	mircopayresponse.ReponseXML = string(data)
	//返回结果
	return mircopayresponse

	}



## 支付返回

创建文件 `micropayresponse.go`，存储发起支付请求后的返回结果，实现代码如下

	package micropay

	import (
	"encoding/xml"
	)

	//1.创建Mircopayresponse结构体，存放我们的请求字段，带有 omitempty 表示该字段选填，其他的字段属于必填,支付请求的结果会被解析到该结构体

	type Mircopayresponse struct {
	XMLName        xml.Name `xml:"xml"`
	Return_code    string   `xml:"return_code"`           //返回状态码
	Return_msg     string   `xml:"return_msg"`            //返回信息
	Appid          string   `xml:"appid"`                 //公众账号ID
	Mch_id         string   `xml:"mch_id"`                //商户号
	Device_info    string   `xml:"device_info,omitempty"` //设备号
	Nonce_str      string   `xml:"nonce_str"`             //随机字符串
	Sign           string   `xml:"sign"`                  //签名
	Result_code    string   `xml:"result_code"`           //业务结果
	Err_code       string   `xml:"err_code"`              //错误代码
	Err_code_des   string   `xml:"err_code_des"`          //错误代码描述
	Is_subscribe   string   `xml:"is_subscribe"`          //是否关注公众账号
	Trade_type     string   `xml:"trade_type"`            //交易类型
	Bank_type      string   `xml:"bank_type"`             //付款银行
	Fee_type       string   `xml:"fee_type"`              //货币类型
	Total_fee      string   `xml:"total_fee"`             //总金额
	Cash_fee_type  string   `xml:"cash_fee_type"`         //现金支付货币类型
	Cash_fee       string   `xml:"cash_fee"`              //现金支付金额
	Coupon_fee     string   `xml:"coupon_fee"`            //代金券或立减优惠金额
	Transaction_id string   `xml:"transaction_id"`        //微信支付订单号
	Out_trade_no   string   `xml:"out_trade_no"`          //商户订单号
	Attach         string   `xml:"attach"`                //商家数据包
	Time_end       string   `xml:"time_end"`              //支付完成时间

	ReponseXML string `xml:"-"` //结果xml串

	}

## 执行支付

创建文件`miropay.go`,执行被扫支付操作,代码如下
package micropay

import (
	"fmt"
	"net/http"
	"wechat/config"
	"wechat/rand"
)

//1.执行刷卡支付功能入口

	func Micropay(w http.ResponseWriter, r *http.Request) {

	r.ParseForm()

	//获得要支付的金额
	fee := r.FormValue("fee")

	//获取用户的刷卡二维码
	authcode := r.FormValue("authcode")

	//创建Mircopayrequest请求参数
	v := &Mircopayrequest{Appid: config.APP_ID, Mch_id: config.MCH_ID}
	v.Auth_code = authcode
	if fee != "" {
		v.Total_fee = fee
	} else {
		v.Total_fee = "1"
	}
	v.Body = "微信刷卡支付"
	v.Nonce_str = rand.Getnoncestr(32)
	v.Out_trade_no = rand.Getnoncestr(32)
	v.Spbill_create_ip = "127.0.0.1"

	//对请求参数进行MD5签名，得到签名字串
	v.Signmd5()

	//把请求参数转换为xml格式的数据
	v.Xml()

	//发起支付请求，并得到返回结果
	response := v.Dorequest()

	//输出支付结果信息
	fmt.Printf("返回状态：%s \n", response.Return_code)
	fmt.Printf("返回信息: %s \n", response.Return_msg)
	fmt.Printf("业务结果: %s \n", response.Result_code)
	fmt.Printf("错误代码: %s \n", response.Err_code)
	fmt.Printf("错误代码描述: %s\n", response.Err_code_des)
	fmt.Println("========================================")

	//发送到网页
	w.Header().Set("Access-Control-Allow-Origin", "*") //允许访问所有域,解决跨域问题
	fmt.Fprintf(w, response.ReponseXML)

	}


## 支付结果判断##


    reponse.Return_code为SUCCESS，response.Result_code为SUCCESS,表示支付成功 
   
    reponse.Return_code为SUCCESS，reponse.Result_code为FAIL,reponse.Err_code为USERPAYING 表示正在输入密码

## 更多`reponse.Err_code`以及对应的处理办法##





|名称|描述 |原因|解决方案|



| SYSTEMERROR	  	|  接口返回错误|  系统超时    | 请立即调用被扫订单结果查询API，查询当前订单状态，并根据订单的状态决定下一步的操作|

| ORDERPAID	  	| 	   订单已支付复    | 请确认该订单号是否重复支付，如果是新单，请使用新订单号提交|
限。请联系
| NOAUTH	  	| 	   商户无权限|  商户没有开通被扫支付权限    | 请开通商户号权产品或商务申请|

| AUTHCODEEXPIRE	  	| 	   二维码已过期，请用户在微信上刷新后再试|  用户的条码已经过期    | 	请收银员提示用户，请用户在微信上刷新条码，然后请收银员重新扫码。 直接将错误展示给收银员|

| NOTENOUGH	  	| 	   余额不足|  用户的零钱余额不足    |请收银员提示用户更换当前支付的卡，然后请收银员重新扫码。建议：商户系统返回给收银台的提示为“用户余额不足.提示用户换卡支付”|

| NOTSUPORTCARD	  	| 	   不支持卡类型|  用户使用卡种不支持当前支付形式,请用户重新选择卡种    | 建议：商户系统返回给收银台的提示为“该卡不支持当前支付，提示用户换卡支付或绑新卡支付”|

| ORDERCLOSED	  	| 	   订单已关闭|  该订单已关    | 商户订单号异常，请重新下单支付|

| ORDERREVERSED	  	| 	   订单已撤销|  当前订单已经被撤销    | 当前订单状态为“订单已撤销”，请提示用户重新支付|

| BANKERROR	  	| 	   银行系统异常|  银行端超时    | 请立即调用被扫订单结果查询API，查询当前订单的不同状态，决定下一步的操作|

| USERPAYING	  	| 	   用户支付中，需要输入密码|  该笔交易因为业务规则要求，需要用户输入支付密码    | 等待5秒，然后调用被扫订单结果查询API，查询当前订单的不同状态，决定下一步的操作|

| AUTH_CODE_ERROR	  	| 	   授权码参数错误|  请求参数未按指引进行填写    | 每个二维码仅限使用一次，请刷新再试|

| AUTH_CODE_INVALID	  	| 	   授权码检验错误|  收银员扫描的不是微信支付的条码    | 请扫描微信支付被扫条码/二维码|

| XML_FORMAT_ERROR	  	| 	   XML格式错误|  XML格式错误    | 请检查XML参数格式是否正确|

| REQUIRE_POST_METHOD	  	| 	   请使用post方法|  未使用post传递参数    | 请检查请求参数是否通过post方法提交|

| SIGNERROR	  	| 	   签名错误|  参数签名结果不正确    | 请检查签名参数和方法是否都符合签名算法要求|

| LACK_PARAMS	  	| 	   缺少参数|  缺少必要的请求参数    | 请检查参数是否齐全|

| NOT_UTF8	  	| 	   编码格式错误|  未使用指定编码格式    | 请使用NOT_UTF8编码格式|

| BUYER_MISMATCH	  	| 	   支付帐号错误|  暂不支持同一笔订单更换支付方    | 请确认支付方是否相同|

| APPID_NOT_EXIST	  	| 	   APPID不存在|  参数中缺少APPID    | 请检查APPID是否正确|

| MCHID_NOT_EXIST	  	| 	   MCHID不存在|  参数中缺少MCHID    | 请检查MCHID是否正确|

| OUT_TRADE_NO_USED	  	| 	   商户订单号重复|  同一笔交易不能多次提交    | 请核实商户订单号是否重复提交|

| APPID_MCHID_NOT_MATCH	  	| 	 appid和mch_id不匹配|  appid和mch_id不匹配    | 请确认appid和mch_id是否匹配|


## [签名校验工具](http://mch.weixin.qq.com/wiki/tools/signverify/)

