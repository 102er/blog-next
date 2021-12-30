---
title: go国际化实现
date: 2021-09-15 21:08:02
categories:
    - GO
tags:
    - GO
---

用go-i18n实现国际化

```go
type languageKey struct{}
	
//设置上下文参数
func NewI18nContext(ctx context.Context, locale string) context.Context {
	var lang = "en"
	switch locale {
	case "zh", "zh-CN":
		lang = "zh"
	}
	return context.WithValue(ctx, languageKey{}, lang)
}

func FromI18nContext(ctx context.Context) (string, bool) {
	lang, ok := ctx.Value(languageKey{}).(string)
	return lang, ok
}

type LocalizeOptions struct {
   MsgParams map[string]interface{} //自定义替换字段参数
   RawMsg    string                 //某找到翻译时 返回的信息 兼容php版本
}

type LocalizeOptionsFunc func(option *LocalizeOptions)

func Localize(ctx context.Context, ID string, opts ...LocalizeOptionsFunc) string {
   option := &LocalizeOptions{
      MsgParams: make(map[string]interface{}),
      RawMsg:    "",
   }
   //设置参数
   for _, f := range opts {
      f(option)
   }
   var lang string
   var ok bool
   tag := language.English
   //从上下文中获取语言类型
   lang, ok = FromI18nContext(ctx)
   if ok && lang == "zh" {
      tag = language.Chinese
   }
  //获取语言包
   template, ok := templates(lang, ID)
   if !ok {
      if option.RawMsg != "" {
         return option.RawMsg
      }
      return ID
   }
   return i18n.NewLocalizer(i18n.NewBundle(tag), tag.String()).MustLocalize(&i18n.LocalizeConfig{
      DefaultMessage: &i18n.Message{
         ID:    ID,
         Other: template,
      },
      TemplateData: option.MsgParams, //模板中的参数
   })
}

//用map存储的语言包
//可以支持文件
func templates(lang, key string) (string, bool) {
   data := en.TranslateToml //这是自己定义的英文语言包map
   if lang == "zh" {
      data = zh.TranslateToml //这是自己定义的中文语言包map
   }
   v, ok := data[key]
   return v, ok
}
```

测试用例：

```go
func TestLocalize(t *testing.T) {
   ctx := context.TODO()
   ctx = NewI18nContext(ctx, "zh")
   assertions := assert.New(t)
   var tests = []struct {
      Name     string
      ID       string
      f        []LocalizeOptionsFunc
      expected string
   }{
      {
         Name:     "hello world no set name",
         ID:       "hello_world",
         f:        []LocalizeOptionsFunc{},
         expected: "你好 <no value>",
      },
      {
         Name: "hello set name",
         ID:   "hello_world",
         f: []LocalizeOptionsFunc{func(option *LocalizeOptions) {
            option.MsgParams = map[string]interface{}{
               "name": "apiserver",
            }
         }},
         expected: "你好 apiserver",
      },
      {
         Name: "set msg param",
         ID:   "time_duration",
         f: []LocalizeOptionsFunc{func(option *LocalizeOptions) {
            option.MsgParams = map[string]interface{}{
               "d": 11,
               "h": 11,
               "m": 11,
               "s": 11,
            }
         }},
         expected: "11天11时11分11秒",
      },
   }

   for _, tt := range tests {
      t.Run(tt.Name, func(t *testing.T) {
         res := Localize(ctx, tt.ID, tt.f...)
         assertions.Equal(tt.expected, res)
         t.Log(res)
      })
   }
}
```

完整代码示例参数：[102er-apiserver](https://github.com/102er/apiserver/tree/main/pkg/i18n)
