# xml格式数据的解析

支付接口传送的数据都是要xml格式的，代码里面我们定义的字段都存储在结构体里面，要实现xml字符串和结构体的互相转换

# 代码实现

## 创建wxml包

创建文件夹`wxml`

## struct结构体转换为xml字符串

创建`encode.go`文件，实现:

-	`func Endoestruct(v interface{}) (xmlresult string, err error)`函数：传入struct结构体，将结构体转化为xml字符串并返回

代码实现:

	package wxml

	import (
	"encoding/xml"
	)

	//把结构体v 转化为xml字符串，并返回
	func Endoestruct(v interface{}) (xmlresult string, err error) {
	output, err := xml.MarshalIndent(v, "  ", "    ")
	xmlresult = string(output)
	return xmlresult, err
	}
	
## [Github地址](https://github.com/luciscode/wechat/tree/master/src/wechat/wxml)