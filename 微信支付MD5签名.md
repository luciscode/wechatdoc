# 微信支付md5签名

微信所有的支付接口都要传送一个签名字段，签名使用md5算法签名

# 签名key和api秘钥

在微信支付申请下来的审批邮件里面，有一项指引我们去设置`api秘钥`，这个api秘钥就是我们签名时候使用的签名key,必须要商户去[商户平台](https://pay.weixin.qq.com/index.php/home/login?return_url=https%3A%2F%2Fpay.weixin.qq.com%2Findex.php%2Faccount%2Fapi_cert)设置，否则会在使用支付接口的时候会报：签名错误

# 签名算法

-	第一步：首先设所有的数据为集合M，将集合M内非空参数值的参数按照参数名ASCII码从小到大排序（字典序），使用URL键值对的格式（即key1=value1&key2=value2…）拼接成字符串stringA。

> 参数名ASCII码从小到大排序（字典序）
> 
> 如果参数的值为空不参与签名
> 
> 参数名区分大小写
> 
> 验证调用返回或微信主动通知签名时，传送的sign参数不参与签名，将生成的签名与该sign值作校验

-	第二步：stringA最后拼接上 签名key=(API密钥的值)得到stringSignTemp字符串，并对stringSignTemp进行MD5运算，再将得到的字符串所有字符转换为大写，得到sign值signValue。

# 举例如下

假设传送的参数如下：

	appid：	wxd930ea5d5a258f4f
	mch_id：	10000100
	device_info：	1000
	body：	test
	nonce_str：	ibuaiVcKdpRxkhJA

第一步：对参数按照key=value的格式，并按照参数名ASCII字典序排序如下：

	stringA="appid=wxd930ea5d5a258f4f&body=test&device_info=1000&mch_id=10000100&nonce_str=ibuaiVcKdpRxkhJA";

第二步：拼接API密钥：

	stringSignTemp="stringA&key=192006250b4c09247ec02edce69f6a2d"
	sign=MD5(stringSignTemp).toUpperCase()="9A0A8659F005D6984697E2CA0A9CF3B7"

# 代码实现

## 创建wsign包

创建文件夹`wsign`

## 实现md5签名

创建文件`sign.go`文件，实现

-	`func SignMD5(v interface{}, apikey string) (string, bool)`方法：实现对传入的v结构体里面的所有参数的签名，并返回签名得到MD5。要保证参数是按照ASCII码从小到大排序，所以v里面的字段在定义的时候就按照ASCII的排序来定义，

代码实现:


	package wsign

	import (
	"bytes"
	"crypto/md5"
	"crypto/sha1"
	"encoding/hex"
	"fmt"
	"reflect"
	"strings"
	)

	//1.得到v里面的字段的md5签名，v在创建的时候就要保证里面的字段是按照ascii从小到大的顺序填写
	func SignMD5(v interface{}, apikey string) (string, bool) {
	var signstr bytes.Buffer

	vt := reflect.TypeOf(v)
	vv := reflect.ValueOf(v)
	for i := 0; i < vt.NumField(); i++ {
		field := vt.Field(i)
		name := field.Name

		keytemp := field.Tag.Get("xml")
		keymap := strings.Split(keytemp, ",")
		key := keymap[0]

		value := (reflect.Indirect(vv).FieldByName(name)).String()
		if value != "" && key != "xml" {
			signstr.WriteString(key + "=" + value + "&")

		}
	}
	signstr.WriteString("key=" + apikey)
	hasher := md5.New()
	hasher.Write([]byte(signstr.String()))
	sign := hex.EncodeToString(hasher.Sum(nil))
	sign = strings.ToUpper(sign)
	return sign, true

	}
	func SignSHA1(v interface{}) (string, bool) {
	var signstr bytes.Buffer

	vt := reflect.TypeOf(v)
	vv := reflect.ValueOf(v)
	for i := 0; i < vt.NumField(); i++ {
		field := vt.Field(i)
		name := field.Name

		keytemp := field.Tag.Get("json")
		keymap := strings.Split(keytemp, ",")
		key := keymap[0]

		value := (reflect.Indirect(vv).FieldByName(name)).String()
		if value != "" {
			signstr.WriteString(key + "=" + value)

		}

		if i < (vt.NumField() - 1) {
			signstr.WriteString("&")
		}
	}

	fmt.Println(signstr.String())
	hasher := sha1.New()
	hasher.Write([]byte(signstr.String()))
	sign := hex.EncodeToString(hasher.Sum(nil))
	return sign, true

	}
