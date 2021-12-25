---
title: go获取ip地理信息
date: 2021-06-14 23:30:16
categories:
    - GO
tags:
    - GO
---

## 概述

MMDB即Maxmind DB，是一个设计用于存储IPv4和IPv6的数据信息的数据库，mmdb文件是一个二进制格式的文件，它使用一个[二分查找](https://so.csdn.net/so/search?from=pc_blog_highlight&q=二分查找)树加速IP信息的查询。ps：网上有一些免费但是更新不及时的ip库，如果用于公司商业化，建议购买ip库。

## GO解析mmdb文件

GO基于IP获取地理信息有两个库：

- github.com/oschwald/maxminddb-golang 可以解析标准的mmdb文件
- github.com/oschwald/geoip2-golang 提供了更多的api操作， 底层也是调用了maxminddb-golang包来做数据的解析，仅仅做了一层接口上的封装，和对应地理数据格式（企业、城市、国家、AnonymousIP、Domain、ISP）的定义。 但是只适用于[GeoLite2](http://dev.maxmind.com/geoip/geoip2/geolite2/) and [GeoIP2](http://www.maxmind.com/en/geolocation_landing) databases，有database type限制。

我们使用的是第三方的ip库，只能通过maxminddb-golang解析数据，根据业务场景封装自己的api，具体实现如下：

```go
type Enterprise struct {
   City struct {
      Names map[string]string `maxminddb:"names"`
   } `maxminddb:"city"`
   Continent struct {
      Code  string            `maxminddb:"code"`
      Names map[string]string `maxminddb:"names"`
   } `maxminddb:"continent"`
   Country struct {
      IsoCode string            `maxminddb:"iso_code"`
      Names   map[string]string `maxminddb:"names"`
   } `maxminddb:"country"`
   ISP struct {
      Names map[string]string `maxminddb:"names"`
   } `maxminddb:"isp"`
   Location struct {
      Latitude  float64 `maxminddb:"latitude"`
      Longitude float64 `maxminddb:"longitude"`
   } `maxminddb:"location"`
   Province struct {
      IsoCode string            `maxminddb:"iso_code"`
      Names   map[string]string `maxminddb:"names"`
   } `maxminddb:"province"`
   Subdivisions []struct {
      IsoCode string            `maxminddb:"iso_code"`
      Names   map[string]string `maxminddb:"names"`
   } `maxminddb:"subdivisions"`
}

//file = ipip.mmdb
func NewGeoIp2(file string) (d *maxminddb.Reader, f func(), err error) {
	 //打开.mmdb文件
   d, err = maxminddb.Open(file)
   if err != nil {
      return
   }
   //程序结束关闭函数 闭包
   f = func() {
      if d != nil {
         _ = d.Close()
      }
   }
   return d, f, nil
}

type GeoIpRepo struct {
   ipDb *maxminddb.Reader
}

func NewGeoIpRepo(data *maxminddb.Reader) *GeoIpRepo {
   return &GeoIpRepo{
      ipDb: data,
   }
}
type GeoIP struct {
	Country  string
	Province string
	City     string
}

func (g *GeoIpRepo) GetIpLocation(ctx context.Context, ip string) (r GeoIP, err error) {
  //解析ip
  ipV := net.ParseIP(ip)
  //如果不能确定mmdb结构 可以使用map去接收数据 然后再定义结构体
  //var record =make(map[string]interface{})
   var record = Enterprise{}
  //二分查找数据
   err = g.ipDb.Lookup(ipV, &record)
   if err != nil {
      log.Warnf("ip %s get location failed,%v", ip, err)
      return
   }
  //转换
   if c, ok := record.Country.Names["zh-CN"]; ok {
      r.Country = c
   }
   if c, ok := record.City.Names["zh-CN"]; ok {
      r.City = c
   }
   if c, ok := record.Province.Names["zh-CN"]; ok {
      r.Province = c
   }
   return
}
```
