---
title: go参数校验
date: 2021-12-15 00:17:53
categories:
    - GO
tags:
    - GO
---

go常用 [validator](https://github.com/go-playground/validator) 进行字段参数校验，其内置了很多常用的字段参数校验方法，同时支持注册自定义方案。v10版本有支持验证结果的国际化。内置的验证tag翻阅文档：[内置tag](https://github.com/go-playground/validator/blob/master/README.md) 

<!-- more -->

### 基础功能

注册器初始化：

```go
package validator

import (
	"fmt"
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/zh"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	en_translations "github.com/go-playground/validator/v10/translations/en"
	zh_translations "github.com/go-playground/validator/v10/translations/zh"
	"reflect"
	"strings"
)

var (
	valid            *validator.Validate
	uni              *ut.UniversalTranslator
	enTrans, zhTrans ut.Translator
)

func init() {
	var found bool
	valid = validator.New() //验证器初始化
	ent := en.New()
	uni = ut.New(ent, ent)
	enTrans, found = uni.GetTranslator("en")
	if !found {
		panic("en translation not found")
	}
	zht := zh.New()
	uni = ut.New(zht, zht)
	zhTrans, found = uni.GetTranslator("zh")
	if !found {
		panic("zh translation not found")
	}
	//注册英文翻译
	err := en_translations.RegisterDefaultTranslations(valid, enTrans)
	if err != nil {
		panic(err)
	}
	//注册中文翻译
	err = zh_translations.RegisterDefaultTranslations(valid, zhTrans)
	if err != nil {
		panic(err)
	}
	//自定义校验函数
	// ....
}

```

验证方法：

```go
// ValidateStructData 验证方法
func ValidateStructData(l string, dataStruct interface{}) error {
   trans := enTrans
   //前端传的字段 通过tag json获取
   valid.RegisterTagNameFunc(func(fld reflect.StructField) string {
      name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
      if name == "-" {
         return ""
      }
      return name
   })
   if l == "zh" {
      trans = zhTrans
      // 中文需要字段国际化 注册字段标签
      valid.RegisterTagNameFunc(func(fld reflect.StructField) string {
         name := fld.Tag.Get("label")
         return name
      })
   }
   err := valid.Struct(dataStruct)
   if err != nil {
      if errs, ok := err.(validator.ValidationErrors); ok {
         for _, fe := range errs {
            errStr := fe.Translate(trans)
            //可以根据平台的业务错误码进行封装业务错误
            fmt.Println(errStr)
         }
      }
      return err
   }
   return nil
}
//验证单个字段
func ValidateVarData(f interface{}, tag, filed string) error {
	trans := enTrans
	err := valid.Var(f, tag)
	if err != nil {
		if errs, ok := err.(validator.ValidationErrors); ok {
			for _, fe := range errs {
				errStr := fe.Translate(trans)
				//可以根据平台的业务错误码进行封装业务错误
				fmt.Println(filed, errStr)
			}
		}
	}
	return err
}
```

测试用例：

```go
func TestValidateData(t *testing.T) {
	type testStruct struct {
		Name      string `json:"name" validate:"required" label:"函数名"`
		StartTime int    `json:"start_time" validate:"required,gt=0" label:"开始时间"`
		EndTime   int    `json:"end_time"`
		Interval  string `json:"interval" validate:"required" label:"时间间隔"`
	}
	d:=testStruct{
		Name: "1",
	}
	err := ValidateStructData("en", d)
	if err != nil {
		return
	}
}
```

### 通过字段tag自定义函数
