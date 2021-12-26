---
title: aes加密
date: 2021-05-25 23:02:20
tags:
---

## 介绍

go实现AES-CBC-128加解密，使用PKCS7算法进行填充

## AES

AES（Advanced Encryption Standard）高级加密标准，是流行的**对称加密**算法（<u>加密和解密用到的密钥是相同的</u>）。它有5种加密模式：

- 电子密码本模式（ECB，Electronic Code Book）
- 加密块链模式（CBC，Cipher Block Chaining），如果明文长度不是分组长度 16 字节的整数倍需要进行填充
- 计数模式（CTR，Counter）
- 密码反馈模式（CFB，Cipher FeedBack）
- 输出反馈模式（OFB，Output FeedBack）

AES 秘钥的长度只能是16、24 或 32 字节，分别对应三种加密模式 AES-128、AES-192 和 AES-256，三者的区别是加密轮数不同。

## PKCS7

`PKCS7`是当下各大加密算法都遵循的数据填充算法，且 `OpenSSL` 加密算法簇的默认填充算法就是 `PKCS7`。`AES-128`, `AES-192`, `AES-256` 的数据块长度分别为 `128/8=16bytes`, `192/8=24bytes`, `256/8=32bytes`。其实`PKCS7`理解起来非常简单，使用需填充长度的数值 `paddingSize` 所表示的`ASCII`码 `paddingChar = chr(paddingSize)`对数据进行冗余填充。

比如 `AES-128`的数据块长度是 16bytes，使用`PKCS7`进行填充时，填充的长度范围是 1 ~ 16。注意，当待加密数据长度为 16 的整数倍时，填充的长度反而是最大的，要填充 16 字节，为什么呢？因为 "PKCS7" 拆包时会按协议取最后一个字节所表征的数值长度作为数据填充长度，如果因真实数据长度恰好为 16 的整数倍而不进行填充，则拆包时会导致真实数据丢失。

为什么是冗余填充呢？因为即便你的数据长度符合`blockSize`的整数倍时，也需要填充，填充的长度反而是最大的，要填充`blockSize`个`char(blockSize)`字符在数据尾部，这样牺牲了数据长度的做法是为了更为灵活透明的去解包数据，发送端和接收端不需要约定好`blockSize`，接收端总能通过数据包的最后一个字符得到填充的数据长度。

当我们拿到一串`PKCS7`填充的数据时，取其最后一个字符`paddingChar`，此字符的`ASCII`码的十进制`ord(paddingChar)`即为填充的数据长度`paddingSize`，读取真实数据时去掉填充长度即可`substr(content, 0, -paddingSize)`。

## Go实现

CBC模式是先将明文切分成若干小段，然后每一小段与初始块或者上一段的密文段进行异或运算后，再与密钥进行加密。这时候就有个问题，那第一段的明文怎么加密呢？这时候就引入了**初始化向量**（英语：initialization vector，缩写为IV）。初始化向量是随机的，就是你可以自定义这个初始化向量，不同的初始化向量加密出来的结果也不一样。

PKCS7Padding填充模式：假设数据长度需要填充 n(n>0) 个字节才对齐，那么填充 n 个字节，每个字节都是 n 。如果数据本身就已经对齐了，则填充一块长度为块大小的数据，每个字节都是块大小。

```go
package cipher

import (
   "bytes"
   "crypto/aes"
   "crypto/cipher"
   "encoding/base64"
)

var (
   key = "1234567890abcdef"
   iv  = "iv0494f002332659"
)

func AesEncryptNoBase64(data []byte) (string, error) {
   return newAESCBCCipher(key, iv).AESCBCBase64Encode(data)
}

type AESCBCCipher struct {
   Key string
   IV  string
}

func newAESCBCCipher(key, iv string) *AESCBCCipher {
   return &AESCBCCipher{
      key, iv,
   }
}

// AESCBCBase64Encode 开始加密
func (a *AESCBCCipher) AESCBCBase64Encode(_data []byte) (string, error) {
   _key := []byte(a.Key)
   _iv := []byte(a.IV)

   _data = PKCS7Padding(_data)
   block, err := aes.NewCipher(_key)
   if err != nil {
      return "", err
   }
   mode := cipher.NewCBCEncrypter(block, _iv)
   mode.CryptBlocks(_data, _data)
   return base64.StdEncoding.EncodeToString(_data), nil
}

// AESCBCBase64Decode 开始解密
func (a *AESCBCCipher) AESCBCBase64Decode(data string) (string, error) {
   _data, err := base64.StdEncoding.DecodeString(data)
   if err != nil {
      return "", err
   }
   _key := []byte(a.Key)
   _iv := []byte(a.IV)

   block, err := aes.NewCipher(_key)
   if err != nil {
      return "", err
   }
   mode := cipher.NewCBCDecrypter(block, _iv)
   mode.CryptBlocks(_data, _data)
   _data = PKCS7UnPadding(_data)

   return string(_data), nil
}

func PKCS7Padding(data []byte) []byte {
   padding := aes.BlockSize - len(data)%aes.BlockSize
   padtext := bytes.Repeat([]byte{byte(padding)}, padding)
   return append(data, padtext...)
}

func PKCS7UnPadding(data []byte) []byte {
   length := len(data)
   unpadding := int(data[length-1])
   return data[:(length - unpadding)]
}
```
