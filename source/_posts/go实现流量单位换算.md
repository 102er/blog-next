---
title: go实现流量单位换算
date: 2021-04-05 13:31:02
tags:
   - GO
---

B字节换算成存储单位，通过位运算实现

1024 = 1 << 10

<!-- more -->

```go
//返回换算后的值 换算单位 换算基准
func humanBytesLoaded(s int64) (string, string, int64) {
   suffix := ""
   var n int64 = 1
   units := []string{"B", "KB", "MB", "GB", "TB", "PB"}
   for k, v := range units {
      suffix = v
      b := (k + 1) * 10
      n = 1 << (k * 10)
      if k == len(units)-1 {
         break
      }
      if s < (1 << b) {
         break
      }
   }
   r := round(float64(s)/float64(n), 1)
   return fmt.Sprintf("%v%s", r, suffix), suffix, n
}

//保留1位小数
func round(value float64, precision int) float64 {
	p := math.Pow10(precision)
	return math.Trunc((value+0.5/p)*p) / p
}
```

测试用例：

```go
func TestFormatByteUnit(t *testing.T) {
   assertions := assert.New(t)
   var tests = []struct {
      Name     string
      s        int64
      expected string
   }{
      {
         "test B", 1023, "1023B",
      },
      {
         "test kB", 1 * 1024, "1KB",
      },
      {
         "test MB", 1025 * 1024, "1MB",
      },
      {
         "test GB", 50 * 1024 * 1024 * 1024, "50GB",
      },
     {
         "test TB", 1024 * 1024 * 1024, "1GB",
      },
      {
         "test TB", 50 * 1024 * 1024 * 1024 * 1024, "50TB",
      },
     {
         "test PB", 50 * 1024 * 1024 * 1024 * 1024* 1024, "50PB",
      },
   }

   for _, tt := range tests {
      t.Run(tt.Name, func(t *testing.T) {
         res := FormatByteUnitAsInt64(tt.s)
         assertions.Equal(tt.expected, res)
         t.Log(res)
      })
   }
}
```
