---
title: https加密的具体过程
date: 2024-09-22 10:22:11
tags: 计算机网络
cover: cover.jpeg
---

## 1. 客户端发起请求
- 用户在浏览器中输入网址（https://），浏览器向服务器发送一个请求，要求建立安全连接。这一步称为 **ClientHello**。
- 请求中包含客户端支持的 TLS/SSL 版本、加密套件（可用于通信的加密算法）、以及一个随机数（`Client Random`）。

## 2. 服务器响应
- 服务器收到请求后，选择双方都支持的最高版本的 TLS/SSL 以及一个加密套件，并将其发送给客户端。这个过程称为 **ServerHello**。
- 服务器还发送自己的数字证书（由受信任的证书颁发机构 CA 签发），包含服务器的公钥以及服务器的身份信息，并附带一个随机数（`Server Random`）。

## 3. 证书验证
- 客户端接收到服务器的数字证书后，首先会验证该证书的合法性，确保证书是由受信任的 CA 签发的，且证书没有过期或被吊销。
- 如果证书验证成功，客户端将继续下一步，否则会终止连接。

## 4. 生成对称密钥
- 客户端生成一个新的随机数（称为 `Pre-Master Secret`），并使用服务器的公钥对该随机数进行加密，然后将其发送给服务器。
- 服务器使用自己的私钥解密，获取 `Pre-Master Secret`。

## 5. 生成会话密钥
- 客户端和服务器现在都有 `Client Random`、`Server Random`、以及 `Pre-Master Secret`，通过这三个随机数，两端将通过相同的算法生成一个对称加密密钥（会话密钥）。
- 这个会话密钥将用于接下来的数据传输，加密和解密所有通信内容。

## 6. 完成握手
- 客户端和服务器都发送一个“Finished”消息，表示握手过程结束。这一消息经过会话密钥的加密处理，确保握手过程中没有被篡改。

## 7. 加密通信
- 握手完成后，客户端与服务器之间开始使用对称密钥加密传输数据。所有后续的请求和响应都将被加密，确保数据的保密性和完整性。
