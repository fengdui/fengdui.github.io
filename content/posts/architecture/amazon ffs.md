---
title: "amazon ffs"
date: "2021-11-02"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

# 借助亚马逊Alexa进行零接触配网

官方文档：[https://developer.amazon.com/zh/frustration-free-setup](https://developer.amazon.com/zh/frustration-free-setup)

## DAK 是什么？

DAK是设备证明密钥。它是一对非对称密码学密钥（一个私钥，一个公钥），在设备制造过程中被安全地注入到每一台设备里。

相关链接：[https://developer.amazon.com/frustration-free-setup/console/products/A25PYKXH7D1LXS/manage-daks](https://developer.amazon.com/frustration-free-setup/console/products/A25PYKXH7D1LXS/manage-daks)

## 下载dak.conf

```ini
oid_section = OIDs

[ req ]
default_bits = 256
prompt = no
encrypt_key = no
default_md = sha256
distinguished_name = dn
[ OIDs ]
DeviceTypeId=1.3.6.1.4.1.4843.1.3
DakType=1.3.6.1.4.1.4843.1.5

[ dn ]
DeviceTypeId=A25PYKXH7D1LXS
DakType=SoftwareProd
```

## Generate a DAK private key (dak_private_key.pem) and certificate signing request (dak.csr):

```bash
# 生成EC参数
openssl ecparam -name prime256v1 > dak-params.pem
# 生成DAK私钥和CSR
openssl req -new -nodes -config dak.conf -newkey ec:dak-params.pem -keyout dak_private_key.pem -out dak.csr
```

把csr文件上传让Amazon签名。

## generate DHA keys and certificates for each device

下载device.conf：

```ini
oid_section = OIDs

[ req ]
default_bits = 256
prompt = no
encrypt_key = no
default_md = sha256
distinguished_name = dn

[ OIDs ]
DeviceTypeId=1.3.6.1.4.1.4843.1.3

[ dn ]
DeviceTypeId=A25PYKXH7D1LXS

[ req_ext ]
authorityKeyIdentifier=keyid
keyUsage=digitalSignature,keyEncipherment
```

Generate a device private key (private-key.pem) and certificate signing request (device.csr):

```bash
# 生成EC参数
openssl ecparam -name prime256v1 > device-params.pem
# 生成设备私钥和CSR
openssl req -new -nodes -config device.conf -newkey ec:device-params.pem -keyout private_key.pem -out device.csr
```
Download your newly signed DAK certificate from the table above, save it as dak-certificate.pem and use the following commands to sign and obtain a DHA certificate chain for each device:

下面相当于用上面签发的证书再去对每一个设备签发一套证书

```bash
# 使用DAK证书和私钥为设备证书签名
openssl x509 -req -in device.csr -extfile device.conf -extensions req_ext -CA dak-certificate-XXXXXX.pem -CAkey dak_private_key.pem -days 1825 -out device-certificate.pem -outform PEM -CAcreateserial -sha256

# 合并证书
cat device-certificate.pem dak-certificate-XXXXXX.pem > certificate.pem
```

Use the following command to retrieve the DHA public key without headers for use with Control Logs. This can also be used to compress a public key for a 2D Barcode:

```bash
openssl x509 -in device-certificate.pem -pubkey -noout | openssl ec -pubin -conv_form compressed | openssl enc -base64 -d | openssl enc -base64 > dha-control-log-public-key.txt
```

## 控制日志

这是亚马逊为了反欺诈和确保供应链安全而要求的一份"生产日志"。

把上面一个个设备的证书公约等等信息上传到amazon，amazon会变更状态，然后这边定时获取设备控制日志的处理状态（成功/失败）、错误详情等关键信息。

大概生产过程要做这些。本质是将证书私钥这些烧录到设备里面去，这样设备带着证书去连接amazon能够通过验证，和我们平时ssh连接服务器一样，你有资格才能连到服务器。

## 配网过程

确保配网环境良好，执行配网测试，具体的配网步骤详见步骤三和四。

配网前准备条件：
1. echo配网，且联网成功；
2. 通过路由器新建一个ssid: simple_setup，无密码。

只要满足以上任何一条，都可以完成ffs配网，设备优先通过echo联网，10s后如果echo连接失败，会连接simple_setup。

### 1. echo配网

echo配网需要个音箱，目前我没有，配网流程待更新。

### 2. 路由热点配网

#### 2.1 默认ssid配网

- ssid名称：simple_setup
- 无密码

配网流程：
1. 通过路由器新建一个ssid: simple_setup，无密码
2. 设备设置配网模式为ap/ez共存模式
3. 设备上电

支持ffs配网的设备，在待配网模式下，上电后前10s会先去扫描配网热点，扫描到后自动配网连接。

### 配网详细流程

1. 激发echo生成隐藏ap；
2. 设备连接echo的隐藏ap；
3. 设备通过echo ap连接亚马逊dpss服务，获取用户ssid和passwd；
4. 设备连接用户ssid和passwd；
5. 设备连接mqtt服务；
6. 设备执行preactive；
7. 云端下发token；
8. 设备执行激活流程。

## FFS技术简介

FFS是亚马逊推出的一项新的技术，用于给WiFi设备快速配网。使用APP扫描产品的BarCode（贴在产品上的二维码），APP把BarCode的内容发送到服务器认证，通过认证后，亚马逊音响echo会建立一个隐藏的AP热点（SSID、Password由设备信息计算出来）；设备上电后，根据设备信息，去连接隐藏的AP热点，再到WSS Cloud上获取用户家里的AP信息（SSID、密码），之后再连接到用户的AP热点。

## 账号关联与授权

1. 亚马逊 App 账号需要与涂鸦账号进行关联；
2. 并且绑定Tuya Smart Skill技能；
3. 设备要先绑定到用户的 Alexa 账户下。

账户绑定是为了授权，绑定技能是为了开启某个品牌方的技能。

**示例流程说明**：
- 我用米家app添加一个第三方的设备 3
- 绑定Tuya Smart Skill技能 授权使用这个第三方品牌 2
- 亚马逊 App 账号需要与涂鸦账号进行关联 给米家授权第三方匹配方下面的数据 1
