---
title: rsa加密
date: 2022-01-14 10:25:22
tags:
---

RSA加密算法是一种非对称加密算法，使用公钥加密，私钥解密。

RSA私钥存在PKCS1和PKCS8两种格式。

golang提供了encoding/pem包和crypto/x509包进行rsa的加解密。实现上需要注意公钥和私钥的格式，按照对应的包进行parse。

```go
var (
	privateKey = `-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDZsfv1qscqYdy4vY+P4e3cAtmvppXQcRvrF1cB4drkv0haU24Y
7m5qYtT52Kr539RdbKKdLAM6s20lWy7+5C0DgacdwYWd/7PeCELyEipZJL07Vro7
Ate8Bfjya+wltGK9+XNUIHiumUKULW4KDx21+1NLAUeJ6PeW+DAkmJWF6QIDAQAB
AoGBAJlNxenTQj6OfCl9FMR2jlMJjtMrtQT9InQEE7m3m7bLHeC+MCJOhmNVBjaM
ZpthDORdxIZ6oCuOf6Z2+Dl35lntGFh5J7S34UP2BWzF1IyyQfySCNexGNHKT1G1
XKQtHmtc2gWWthEg+S6ciIyw2IGrrP2Rke81vYHExPrexf0hAkEA9Izb0MiYsMCB
/jemLJB0Lb3Y/B8xjGjQFFBQT7bmwBVjvZWZVpnMnXi9sWGdgUpxsCuAIROXjZ40
IRZ2C9EouwJBAOPjPvV8Sgw4vaseOqlJvSq/C/pIFx6RVznDGlc8bRg7SgTPpjHG
4G+M3mVgpCX1a/EU1mB+fhiJ2LAZ/pTtY6sCQGaW9NwIWu3DRIVGCSMm0mYh/3X9
DAcwLSJoctiODQ1Fq9rreDE5QfpJnaJdJfsIJNtX1F+L3YceeBXtW0Ynz2MCQBI8
9KP274Is5FkWkUFNKnuKUK4WKOuEXEO+LpR+vIhs7k6WQ8nGDd4/mujoJBr5mkrw
DPwqA3N5TMNDQVGv8gMCQQCaKGJgWYgvo3/milFfImbp+m7/Y3vCptarldXrYQWO
AQjxwc71ZGBFDITYvdgJM1MTqc8xQek1FXn1vfpy2c6O
-----END RSA PRIVATE KEY-----`
	publicKey = `-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDZsfv1qscqYdy4vY+P4e3cAtmv
ppXQcRvrF1cB4drkv0haU24Y7m5qYtT52Kr539RdbKKdLAM6s20lWy7+5C0Dgacd
wYWd/7PeCELyEipZJL07Vro7Ate8Bfjya+wltGK9+XNUIHiumUKULW4KDx21+1NL
AUeJ6PeW+DAkmJWF6QIDAQAB
-----END PUBLIC KEY-----`
)
type RSACipher struct {
   privateKey []byte
   publicKey  []byte
}

func RsaDecryptBase64(content string) (string, error) {
   _data, err := base64.StdEncoding.DecodeString(content)
   if err != nil {
      return "", err
   }
   decrypt, err := newRSACipher(privateKey, publicKey).decode(_data)
   if err != nil {
      return "", err
   }
   return string(decrypt), nil
}

func RsaDecodeBase64(content string) (string, error) {
   decrypt, err := newRSACipher(privateKey, publicKey).encrypt([]byte(content))
   if err != nil {
      return "", err
   }
   return base64.StdEncoding.EncodeToString(decrypt), nil
}

func newRSACipher(privateKey, publicKey string) *RSACipher {
   return &RSACipher{
      []byte(privateKey), []byte(publicKey),
   }
}

func (r *RSACipher) encrypt(origData []byte) ([]byte, error) {
   block, _ := pem.Decode(r.publicKey)
   if block == nil {
      return nil, errors.New("public key error")
   }
   var pub *rsa.PublicKey
   var ok bool
   var err error
   if block.Type == "RSA PUBLIC KEY" {
      pub, err = x509.ParsePKCS1PublicKey(block.Bytes)
      if err != nil {
         return nil, err
      }
   } else {
      pubInterface, err2 := x509.ParsePKIXPublicKey(block.Bytes)
      if err2 != nil {
         return nil, err2
      }
      pub, ok = pubInterface.(*rsa.PublicKey)
      if !ok {
         return nil, errors.New("key not public key")
      }
   }
   return rsa.EncryptPKCS1v15(rand.Reader, pub, origData)
}

func (r *RSACipher) decode(ciphertext []byte) ([]byte, error) {
   block, _ := pem.Decode(r.privateKey)
   if block == nil {
      return nil, errors.New("private key error")
   }
   priv, err := x509.ParsePKCS1PrivateKey(block.Bytes)
   if err != nil {
      return nil, err
   }
   /* rsaKey, ok := priv.(*rsa.PrivateKey)
      if !ok {
         return nil, errors.New("key not PrivateKey")
      }*/
   return rsa.DecryptPKCS1v15(rand.Reader, priv, ciphertext)
}
```

完整代码示例参数：[rsa加密](https://github.com/102er/apiserver/blob/main/pkg/cipher/rsa.go)
