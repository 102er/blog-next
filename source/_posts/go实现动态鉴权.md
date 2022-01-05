---
title: go实现动态鉴权
date: 2021-10-15 20:16:32
categories:
    - GO
tags:
    - go开发
---

动态鉴权：对每个请求实现动态认证，保证访问安全可靠。客户端和服务端使用相同的秘钥以及规则进行加密请求，结果相等或者在有效期内，则认为请求合法。加密流程如图：

![osi](https://102er.github.io/uploads/AK_SK.png)

<!-- more -->

go实现：

```go
package signer

import (
   "bytes"
   "crypto/hmac"
   "crypto/sha256"
   "fmt"
   "io/ioutil"
   "net/http"
   "sort"
   "strconv"
   "strings"
   "time"
)

const (
   headerXAuthorization = "Authorization"
   headerXTimestamp     = "X-Api-Timestamp"
   algorithm            = "HMAC-SHA256"
)

type Signer struct {
   ID     string
   Secret string
}

func (s *Signer) Sing(r *http.Request) error {
   var (
      err error
      hxt = r.Header.Get(headerXTimestamp)
   )
   if len(hxt) == 0 {
      hxt = strconv.FormatInt(time.Now().Unix(), 10)
      r.Header.Set(headerXTimestamp, hxt)
   }
   XAuthHeader, err := BuildAuthorizationHeader(r, s.ID, s.Secret)
   if err != nil {
      return err
   }
   r.Header.Set(headerXAuthorization, XAuthHeader)
   return nil
}

func BuildAuthorizationHeader(r *http.Request, id, secret string) (string, error) {
   //构造请求
   hxt := r.Header.Get(headerXTimestamp)
   canonicalRequest, err := CanonicalRequest(r)
   if err != nil {
      return "", err
   }
   //组合签名字符串
   strSign, err := StringToSign(canonicalRequest, hxt)
   if err != nil {
      return "", err
   }
   //签名
   signature, err := SignStringToSign(strSign, []byte(secret))
   if err != nil {
      return "", err
   }
   return fmt.Sprintf("%s Access=%s,Signature=%s", algorithm, id, signature), nil
}

func hmacSha256(key []byte, data string) ([]byte, error) {
   h := hmac.New(sha256.New, key)
   if _, err := h.Write([]byte(data)); err != nil {
      return nil, err
   }
   return h.Sum(nil), nil
}

// SignStringToSign Create the  hmac sha 256 Signature.
func SignStringToSign(stringToSign string, signingKey []byte) (string, error) {
   hm, err := hmacSha256(signingKey, stringToSign)
   return fmt.Sprintf("%x", hm), err
}

// StringToSign Create a "String to Sign".
func StringToSign(canonicalRequest string, t string) (string, error) {
   hash := sha256.New()
   _, err := hash.Write([]byte(canonicalRequest))
   if err != nil {
      return "", err
   }
   //log.Print(fmt.Sprintf("%s\n%s\n%x",
   // Algorithm, t, hash.Sum(nil)))
   return fmt.Sprintf("%s\n%s\n%x",
      algorithm, t, hash.Sum(nil)), nil
}

// CanonicalRequest Build sign string
// CanonicalRequest =
// RequestMethod + '\n' +
// uri + '\n' +
// queryParams + '\n' +
// hexEncodeBody(Hash(RequestPayload))
func CanonicalRequest(r *http.Request) (string, error) {
   data, err := RequestPayload(r)
   if err != nil {
      return "", err
   }
   hexEncodeBody, err := HexEncodeSHA256Hash(data)
   if err != nil {
      return "", err
   }
   uri := CanonicalURI(r)
   queryParams := CanonicalQueryString(r)
   //log.Print(fmt.Sprintf("%s\n%s\n%s\n%s", r.Method, uri, queryParams, hexEncodeBody))
   return fmt.Sprintf("%s\n%s\n%s\n%s", r.Method, uri, queryParams, hexEncodeBody), nil
}

// CanonicalQueryString build query param
func CanonicalQueryString(r *http.Request) string {
   var keys []string
   query := r.URL.Query()
   for key := range query {
      keys = append(keys, key)
   }
   sort.Strings(keys)
   var a []string
   for _, key := range keys {
      k := escape(key)
      sort.Strings(query[key])
      for _, v := range query[key] {
         kv := fmt.Sprintf("%s=%s", k, escape(v))
         a = append(a, kv)
      }
   }
   queryStr := strings.Join(a, "&")
   r.URL.RawQuery = queryStr
   return queryStr
}

// CanonicalURI returns request uri
func CanonicalURI(r *http.Request) string {
   pattens := strings.Split(r.URL.Path, "/")
   var uri []string
   for _, v := range pattens {
      uri = append(uri, escape(v))
   }
   urlpath := strings.Join(uri, "/")
   if len(urlpath) == 0 || urlpath[len(urlpath)-1] != '/' {
      urlpath = urlpath + "/"
   }
   return urlpath
}
func RequestPayload(r *http.Request) ([]byte, error) {
   if r.Body == nil {
      return []byte(""), nil
   }
   b, err := ioutil.ReadAll(r.Body)
   if err != nil {
      return []byte(""), err
   }
   r.Body = ioutil.NopCloser(bytes.NewBuffer(b))
   return b, err
}
func HexEncodeSHA256Hash(body []byte) (string, error) {
   hash := sha256.New()
   if body == nil {
      body = []byte("")
   }
   _, err := hash.Write(body)
   return fmt.Sprintf("%x", hash.Sum(nil)), err
}
func shouldEscape(c byte) bool {
   if 'A' <= c && c <= 'Z' || 'a' <= c && c <= 'z' || '0' <= c && c <= '9' || c == '_' || c == '-' || c == '~' || c == '.' {
      return false
   }
   return true
}
func escape(s string) string {
   hexCount := 0
   for i := 0; i < len(s); i++ {
      c := s[i]
      if shouldEscape(c) {
         hexCount++
      }
   }

   if hexCount == 0 {
      return s
   }

   t := make([]byte, len(s)+2*hexCount)
   j := 0
   for i := 0; i < len(s); i++ {
      switch c := s[i]; {
      case shouldEscape(c):
         t[j] = '%'
         t[j+1] = "0123456789ABCDEF"[c>>4]
         t[j+2] = "0123456789ABCDEF"[c&15]
         j += 3
      default:
         t[j] = s[i]
         j++
      }
   }
   return string(t)
}
```
