---
title: go查询Elastic
date: 2021-07-16 00:18:28
categories:
    - GO
tags:
    - go开发
---

go操作elasticsearch有两个常用的库：

- github.com/elastic/go-elasticsearch 官方提供的操作库
- github.com/olivere/elastic 第三方包，封装了更多的高级api

两个库都可以使用，出于我们操作场景不复杂，用来查询数据，我们使用官方库封装了一个api，:(官方库说是性能比较好

<!-- more -->

### 初始化客户端

```go
import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	es "github.com/elastic/go-elasticsearch/v7"
	"github.com/tidwall/gjson"
	"io/ioutil"
	"time"
)
// Elastic 基于官方库封装
type Elastic struct {
   es  *es.Client
}

// M+A构建复杂的查询条件
type M = map[string]interface{}
type A = []interface{}

func NewElasticClient(conf *conf.Data, logger log.Logger) (*Elastic, error) {
   var err error
   client, err := es.NewClient(es.Config{
      Addresses: conf.Es.Clusters, //es集群 支持多个host
      Username:  conf.Es.User, //用户名
      Password:  conf.Es.Password, //密码
   })
   ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
   defer cancel()
  //es信息
   res, err := client.Info(client.Info.WithContext(ctx))
   if err != nil {
      return nil, err
   }
   defer res.Body.Close()
   if res.IsError() {
      return nil, err
   }

   b, err := ioutil.ReadAll(res.Body)
   if err != nil {
      return nil, err
   }
   	fmt.Println("es version: ", gjson.ParseBytes(b).Get("version.number").String())
   return &Elastic{
      es:  client,
      log: l,
   }, err
}
```

### 封装API

查询函数

```go
func (c *Elastic) Search(ctx context.Context, index []string, query M) ([]byte, error) {
   var buf bytes.Buffer
   if err := json.NewEncoder(&buf).Encode(query); err != nil {
      return nil, err
   }
   // b, _ := json.MarshalIndent(query, " ", " ")
   //fmt.Println(string(b))
   res, err := c.es.Search(
      c.es.Search.WithContext(ctx),
      c.es.Search.WithIndex(index...),
      c.es.Search.WithBody(&buf),
      c.es.Search.WithTrackTotalHits(true),
      c.es.Search.WithPretty(),
   )
   if err != nil {
      return nil, err
   }

   defer res.Body.Close()
   if res.IsError() {
      var e map[string]interface{}
      if err := json.NewDecoder(res.Body).Decode(&e); err != nil {
         return nil, fmt.Errorf("error parsing the response body: %s", err)
      } else {
         errB, _ := json.Marshal(e)
         fmt.Println(string(errB))
         return nil, fmt.Errorf("[%s] %s: %s", res.Status(), e["error"].(map[string]interface{})["type"],
            e["error"].(map[string]interface{})["reason"])
      }
   } else {
      b, err := ioutil.ReadAll(res.Body)
      if err != nil {
         return nil, err
      }
      return b, nil
   }
}
```

测试用例

```go
func TestNewEsClientOld(t *testing.T) {
   info, err := NewElasticClient(&conf.Data{
      Es: &conf.ES{
         Clusters: []string{"xxxxx"},
         User:     "xxxxx",
         Password: "xxxxxx",
      },
   }, l)
   if err != nil {
      t.Log(err)
   }
   var query = M{
      "size": 0,
      "from": 1,
      "sort": A{
         M{"timestamp": M{"order": "desc"}},
      },
      "query": M{
         "bool": M{
            "filter": M{
               "bool": M{
                  "must": A{
                     M{"term": M{"xxx": "407904ce-c279-1d0e-fbeb-d12ccfafa827"}},
                     M{"term": M{"xxx": "775e6349-dbd2-6d8e-6b9c-58818f770574"}},
                     M{"range": M{"timestamp": M{
                        "gte": time.Now().AddDate(0, 0, -1),
                        "lte": time.Now(),
                     }}},
                  },
               },
            },
         },
      },
   }
   search, err := info.Search(context.Background(),[]string{"xxxx"},query)
   if err != nil {
      return
   }
   if err != nil {
      return
   }
   fmt.Println(search)
}
```
