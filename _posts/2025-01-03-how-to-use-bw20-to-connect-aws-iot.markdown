---
layout: post
title:  "如何使用安信可 BW20 模组对接 AWS IoT Core"
date:   2025-01-03 17:56:46 +0800
categories: iot realtek amebadplus
---

# 1. 准备工作
按照 [安信可 BW20-12F / BW20-07S 模组（RTL8711DAx）开发环境搭建指南](https://blog.csdn.net/u011300834/article/details/141448802) 配置好编译环境。
# 2. 安装并配置 AWS 命令行工具
## 2.1. 安装 `aws-cli`
```shell
$ sudo snap install aws-cli --classic
```
```
aws-cli (v2/stable) 2.22.21 from Amazon Web Services (aws✓) installed
```
## 2.2. 创建 AWS 访问凭证

登录 AWS 控制台，打开 [Create access key](https://us-east-1.console.aws.amazon.com/iam/home?region=ap-southeast-1#/security_credentials/access-key-wizard) 页面，点击按钮创建 Access Key，保存好刚才创建的 Access Key。

> **注意⚠：这个页面只会显示一次，所以务必保存好**

## 2.3. 配置 AWS 命令行工具

```shell
$ aws configure
```

根据向导提示输入刚才创建的 AWS 访问凭证

```
AWS Access Key ID [None]: 上一步生成的 Access key
AWS Secret Access Key [None]: 上一步生成的 Secret access key
Default region name [None]: 选一个延迟低的区域，比如 ap-southeast-1（新加坡）
Default output format [None]: text
```

## 2.4. 检查配置和网络连通性

```shell
$ aws account get-contact-information
```

此处显示你的账号信息。若提示错误，需要检查设置或者配置 HTTP 代理访问

# 3. 创建 AWS S3 Bucket 存放 OTA 固件

## 3.1. 创建 AWS S3 Bucket
运行下面的命令创建 AWS S3 Bucket，`ap-southeast-1` 替换为要创建的区域，`afr-ota-aithinker` 替换为实际要创建的名字

> **注意⚠：名称必须以 `afr-ota` 开头，否则需要手动添加 OTA 角色的 Bucket 访问权限**

```shell
$ aws s3api create-bucket \
    --bucket afr-ota-aithinker \
    --create-bucket-configuration LocationConstraint=ap-southeast-1
```

此处返回访问 URL。若提示名称冲突，可以换个名字。

## 3.2. 打开 S3 Bucket 的版本控制功能

打开 S3 Bucket 的版本控制功能，`afr-ota-aithinker` 替换为实际要创建的名字

```shell
$ aws s3api put-bucket-versioning \
    --bucket afr-ota-aithinker \
    --versioning-configuration Status=Enabled
```

## 3.3. 测试上传下载文件

1. 本地创建测试文件 `test.txt`

```shell
$ echo "hello world" > test.txt
```

2. 将测试文件上传到 `afr-ota-aithinker`，并改名为 `test2.txt`

```shell
$ aws s3 cp test.txt s3://afr-ota-aithinker/test2.txt
```
```
upload: ./test.txt to s3://afr-ota-aithinker/test2.txt
```

3. 将改名后的测试文件下载到本地

```shell
$ aws s3 cp s3://afr-ota-aithinker/test2.txt test2.txt
```
```
download: s3://afr-ota-aithinker/test2.txt to ./test2.txt
```

4. 输出测试文件内容

```shell
$ cat test2.txt
```
```
hello world
```

5. 从 `afr-ota-aithinker` 删除测试文件

```shell
$ aws s3 rm s3://afr-ota-aithinker/test2.txt
```
```
delete: s3://afr-ota-aithinker/test2.txt
```

6. 删除本地测试文件

```shell
$ rm test.txt test2.txt
```

# 4. 创建 OTA 服务角色

## 4.1. 创建角色的 Trust Entities 策略文件

运行下面命令创建 `MyOTAServiceRolePolicy.json`，允许角色访问 AWS IoT 服务

```shell
$ cat << EOF > MyOTAServiceRolePolicy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "iot.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF
```

## 4.2. 创建 OTA 服务角色

创建角色  `MyOTAServiceRole`，指定上一步创建的策略文件

```shell
$ aws iam create-role \
    --role-name MyOTAServiceRole \
    --assume-role-policy-document file://`realpath MyOTAServiceRolePolicy.json`
```
```
ROLE    arn:aws:iam::991983452270:role/MyOTAServiceRole 2024-12-26T01:25:20+00:00       /       AROA6N5WSIBXGVQKCUJM3   MyOTAServiceRole
ASSUMEROLEPOLICYDOCUMENT        2012-10-17
STATEMENT       sts:AssumeRole  Allow
PRINCIPAL       iot.amazonaws.com
```

## 4.3. 创建策略文件

1. 创建策略文件 `AccessMyOTAServiceRolePolicy.json`

```shell
$ cat << EOF > AccessMyOTAServiceRolePolicy.json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"iam:GetRole",
				"iam:PassRole"
			],
			"Resource": "arn:aws:iam::`aws sts get-caller-identity --query Account --output text`:role/MyOTAServiceRole"
		}
	]
}
EOF
```

2. 创建策略文件 `AccessSignerPolicy.json`

```shell
$ cat << EOF > AccessSignerPolicy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "signer:*"
            ],
            "Resource": "*"
        }
    ]
}
EOF
```

## 4.4. 添加策略到角色

```shell
$ aws iam put-role-policy \
    --role-name MyOTAServiceRole \
    --policy-name AccessMyOTAServiceRole \
    --policy-document file://`realpath AccessMyOTAServiceRolePolicy.json`
```
```shell
$ aws iam put-role-policy \
    --role-name MyOTAServiceRole \
    --policy-name AccessSigner \
    --policy-document file://`realpath AccessSignerPolicy.json`
```
```shell
$ aws iam attach-role-policy \
    --role-name MyOTAServiceRole \
    --policy-arn arn:aws:iam::aws:policy/AWSIoTFullAccess
```
```shell
$ aws iam attach-role-policy \
    --role-name MyOTAServiceRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

## 4.5. 查询策略是否成功添加

```shell
$ aws iam list-role-policies --role-name MyOTAServiceRole
```
```
POLICYNAMES     AccessMyOTAServiceRole
POLICYNAMES     AccessSigner
```
```shell
$ aws iam list-attached-role-policies --role-name MyOTAServiceRole
```
```
ATTACHEDPOLICIES        arn:aws:iam::aws:policy/AWSIoTFullAccess        AWSIoTFullAccess
ATTACHEDPOLICIES        arn:aws:iam::aws:policy/AmazonS3FullAccess      AmazonS3FullAccess
```

## 4.6. 删除本地策略文件

```shell
$ rm AccessMyOTAServiceRolePolicy.json MyOTAServiceRolePolicy.json AccessSignerPolicy.json
```

# 5. 配置代码签名服务 AWS Signer

## 5.1. 创建自签名证书配置文件

创建自签名证书配置文件 `req.cnf`，指定证书用途为代码签名，公司信息根据实际替换即可

```shell
$ cat << EOF > req.cnf
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = CN
ST = Guangdong Province
L = Shenzhen
O = Shenzhen Anxinke Technology Co., Ltd.
CN = www.ai-thinker.com

[v3_req]
keyUsage = digitalSignature
extendedKeyUsage = codeSigning
subjectAltName = @alt_names

[alt_names]
DNS.1 = www.ai-thinker.com
DNS.2 = ai-thinker.com
DNS.3 = www.aithinker.com
DNS.4 = aithinker.com
EOF
```

## 5.2. 生成自签名证书私钥

```shell
$ openssl genpkey -algorithm EC \
    -pkeyopt ec_paramgen_curve:P-256 \
    -pkeyopt ec_param_enc:named_curve \
    -outform PEM \
    -out ecdsa-sha256-signer.key.pem
```

## 5.3. 生成自签名证书

```shell
openssl req -new -x509 \
    -config req.cnf \
    -nodes \
    -days 3650 \
    -key ecdsa-sha256-signer.key.pem \
    -out ecdsa-sha256-signer.crt.pem
```

## 5.4. 上传生成的证书和私钥到 AWS Certificate Manager

```shell
$ aws acm import-certificate \
    --certificate fileb://`realpath ecdsa-sha256-signer.crt.pem` \
    --private-key fileb://`realpath ecdsa-sha256-signer.key.pem`
```

```
arn:aws:acm:ap-southeast-1:991983452270:certificate/7e352118-27d3-4376-bbb2-6a12179d4ec8
```

上传成功后，命令行返回证书对应的 ARN

## 5.5. 创建 AWS Signer 配置

创建 AWS Signer 配置 `MyOTASigningProfile3`，`certificateArn=` 后的值替换为上一步生成的 ARN

```shell
$ aws signer put-signing-profile \
     --profile-name MyOTASigningProfile3 \
     --platform-id AWSIoTDeviceManagement-SHA256-ECDSA \
     --signing-parameters certname="aithinker.com",certificatePathOnDevice="ecdsa-sha256-signer.crt.pem" \
     --signing-material certificateArn=arn:aws:acm:ap-southeast-1:991983452270:certificate/7e352118-27d3-4376-bbb2-6a12179d4ec8
```
```
arn:aws:signer:ap-southeast-1:991983452270:/signing-profiles/MyOTASigningProfile3       Hjr4mjO9JM      arn:aws:signer:ap-southeast-1:991983452270:/signing-profiles/MyOTASigningProfile3/Hjr4mjO9JM
```


# 6. 使用 AWS Signer 对固件进行签名
## 6.1. 上传待签名固件

1. 上传 `ota_all.bin` 文件，该文件为待签名的 OTA 固件

```shell
$ aws s3 cp ota_all.bin s3://afr-ota-aithinker/ota_all.bin
```
```
upload: ./ota_all.bin to s3://afr-ota-aithinker/ota_all.bin
```

2. 查询 `ota_all.bin` 对应 AWS S3 Bucket 中的版本 ID

```shell
$ aws s3api list-object-versions --bucket afr-ota-aithinker --prefix ota_all.bin --query Versions[0].VersionId
```
```
CF5s4DSv1jgGaU0S5F5pa4NV5jyiZP3x
```

## 6.2. 创建 OTA 签名任务

创建 OTA 签名任务，对固件进行签名，`version=` 后的值替换为上一步的版本 ID

```shell
$ aws signer start-signing-job \
    --source 's3={bucketName=afr-ota-aithinker,key=ota_all.bin,version=CF5s4DSv1jgGaU0S5F5pa4NV5jyiZP3x}' \
    --destination 's3={bucketName=afr-ota-aithinker,prefix=signed-}' \
    --profile-name MyOTASigningProfile3
```
```
8141658f-f0ed-4b0d-b144-e5d8b442e088    991983452270
```

## 6.3. 获取生成的 OTA 签名

1. 使用上一步返回的任务 ID 查询 OTA 签名任务状态

```shell
$ aws signer describe-signing-job \
    --job-id 8141658f-f0ed-4b0d-b144-e5d8b442e088 \
    --output json
```
```json
{
    "jobId": "8141658f-f0ed-4b0d-b144-e5d8b442e088",
    "source": {
        "s3": {
            "bucketName": "afr-ota-aithinker",
            "key": "ota_all.bin",
            "version": "CF5s4DSv1jgGaU0S5F5pa4NV5jyiZP3x"
        }
    },
    "signingMaterial": {
        "certificateArn": "arn:aws:acm:ap-southeast-1:991983452270:certificate/7e352118-27d3-4376-bbb2-6a12179d4ec8"
    },
    "platformId": "AWSIoTDeviceManagement-SHA256-ECDSA",
    "platformDisplayName": "AWS IoT Device Management SHA256-ECDSA ",
    "profileName": "MyOTASigningProfile3",
    "profileVersion": "Hjr4mjO9JM",
    "signingParameters": {
        "certificatePathOnDevice": "ecdsa-sha256-signer.crt.pem",
        "certname": "aithinker.com"
    },
    "createdAt": "2025-01-03T08:53:22+08:00",
    "completedAt": "2025-01-03T08:53:22+08:00",
    "requestedBy": "arn:aws:iam::991983452270:root",
    "status": "Succeeded",
    "statusReason": "Signing Succeeded",
    "signedObject": {
        "s3": {
            "bucketName": "afr-ota-aithinker",
            "key": "signed-8141658f-f0ed-4b0d-b144-e5d8b442e088"
        }
    },
    "jobOwner": "991983452270",
    "jobInvoker": "991983452270"
}
```

可以看见存储签名信息的文件为 `signed-8141658f-f0ed-4b0d-b144-e5d8b442e088`

2. 下载存储签名信息的文件，查看内容

```shell
$ aws s3 cp s3://afr-ota-aithinker/signed-7b04fcbc-fd83-4801-8328-c5861821e0d2 .
```
```
download: s3://afr-ota-aithinker/signed-8141658f-f0ed-4b0d-b144-e5d8b442e088 to ./signed-8141658f-f0ed-4b0d-b144-e5d8b442e088
```
```shell
$ cat signed-7b04fcbc-fd83-4801-8328-c5861821e0d2
```
```json
{
    "rawPayloadSize": 1109376,
    "signature": "MEUCIQD+e5a9Hd+eDSmMDOmMl92EzObRh0JonAYOMvlh56zySAIgZRTdZb5fOhzg+YmDxv5qfliBgMZuWr4CUvcP1gX5W9Y=",
    "signatureAlgorithm": "SHA256withECDSA",
    "payloadLocation": {
        "s3": {
            "bucketName": "afr-ota-aithinker",
            "key": "ota_all.bin",
            "version": "CF5s4DSv1jgGaU0S5F5pa4NV5jyiZP3x"
        }
    }
}
```

可以看见生成的 OTA 固件签名经 Base64 编码后为 `MEUCIQD+e5a9Hd+eDSmMDOmMl92EzObRh0JonAYOMvlh56zySAIgZRTdZb5fOhzg+YmDxv5qfliBgMZuWr4CUvcP1gX5W9Y=`

## 6.4. 验证 AWS Signer 生成的签名

1. 通过 `openssl` 解析出证书公钥，保存为 `ecdsa-sha256-signer.pubkey.pem`

```shell
$ openssl x509 \
    -in ecdsa-sha256-signer.crt.pem \
    -noout -pubkey > ecdsa-sha256-signer.pubkey.pem
```
```shell
$ cat ecdsa-sha256-signer.pubkey.pem
```
```
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWUEA30iw1a4fxyfRQvof9Jy0l4iX
daAx9+DGJu/nOxNqMqIEkUid9LXJzy5c+4SjQYPGeISRjV8PR0byBwHWFw==
-----END PUBLIC KEY-----
```

2. 通过 `openssl`，使用公钥验证签名

```shell
$ openssl dgst -sha256 \
    -verify ecdsa-sha256-signer.pubkey.pem \
    -signature ota_all.bin.dgst ota_all.bin
```
```
Verified OK
```

签名验证成功，代表 AWS Signer 工作正常
# 7. 创建 OTA 测试设备

## 7.1. 创建设备类型

运行下面的命令创建设备类型 `BW20Module`

```shell
$ aws iot create-thing-type --thing-type-name BW20Module
```
```
arn:aws:iot:ap-southeast-1:991983452270:thingtype/BW20Module    6f425319-b8e4-4091-91eb-7e1b88d26ab4    BW20Module
```

## 7.2. 创建设备组

运行下面的命令创建设备组 `BW20ModuleGroup`

```shell
$ aws iot create-thing-group --thing-group-name BW20ModuleGroup
```
```
arn:aws:iot:ap-southeast-1:991983452270:thinggroup/BW20ModuleGroup      23be07cd-2d85-4397-b722-cc2a50d6f297    BW20ModuleGroup
```

## 7.3. 配置 MQTT Topic 权限

1. 运行下面命令创建权限策略配置文件 `BW20ModuleGroupPolicy.json`

```shell
$ cat << EOF > BW20ModuleGroupPolicy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:ap-southeast-1:991983452270:client/\${iot:Connection.Thing.ThingName}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iot:Publish",
        "iot:Receive",
        "iot:PublishRetain"
      ],
      "Resource": "arn:aws:iot:ap-southeast-1:991983452270:topic/*"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": "arn:aws:iot:ap-southeast-1:991983452270:topicfilter/*"
    }
  ]
}
EOF
```

2. 运行下面的命令，创建策略 `BW20ModuleGroupPolicy`

```shell
$ aws iot create-policy \
    --policy-name BW20ModuleGroupPolicy \
    --policy-document file://`realpath BW20ModuleGroupPolicy.json`
```
```
arn:aws:iot:ap-southeast-1:991983452270:policy/BW20ModuleGroupPolicy    {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:ap-southeast-1:991983452270:client/${iot:Connection.Thing.ThingName}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iot:Publish",
        "iot:Receive",
        "iot:PublishRetain"
      ],
      "Resource": "arn:aws:iot:ap-southeast-1:991983452270:topic/*"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": "arn:aws:iot:ap-southeast-1:991983452270:topicfilter/*"
    }
  ]
}
        BW20ModuleGroupPolicy   1
```

3. 将权限附加到设备组 `BW20ModuleGroup`

```shell
$ aws iot attach-policy \
    --policy BW20ModuleGroupPolicy \
    --target arn:aws:iot:ap-southeast-1:991983452270:thinggroup/BW20ModuleGroup
```

## 7.4. 创建 OTA 测试设备

按照惯例，我们使用模组的 Wi-Fi MAC 地址 `94c96058af41` 作为设备名称，运行下面的命令创建 OTA 测试设备

```shell
$ aws iot create-thing --thing-name 94c96058af41 --thing-type-name BW20Module
```
```
arn:aws:iot:ap-southeast-1:991983452270:thing/94c96058af41      980940fd-8c18-4e36-b104-7f039bd20a79    94c96058af41
```

## 7.5. 将 OTA 测试设备加入设备组

运行下面命令将 `94c96058af41` 加入 `BW20ModuleGroup`

```shell
$ aws iot add-thing-to-thing-group \
    --thing-group-name BW20ModuleGroup \
    --thing-name 94c96058af41
```

## 7.6. 生成设备的证书

运行下列命令生成证书文件，证书文件只会显示一次，所以务必保存好

```shell
$ aws iot create-keys-and-certificate \
    --set-as-active \
    --certificate-pem-outfile 94c96058af41.crt.pem \
    --public-key-outfile 94c96058af41.pub.pem \
    --private-key-outfile 94c96058af41.key.pem
```
```
arn:aws:iot:ap-southeast-1:991983452270:cert/f67c5a8edfb3705c5feacb1f1b052dd5f6ee23881a60ac101af9fdec08b34fe8   f67c5a8edfb3705c5feacb1f1b052dd5f6ee23881a60ac101af9fdec08b34fe8
        -----BEGIN CERTIFICATE-----
xxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxx
-----END CERTIFICATE-----

KEYPAIR -----BEGIN RSA PRIVATE KEY-----
xxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxx
-----END RSA PRIVATE KEY-----
        -----BEGIN PUBLIC KEY-----
xxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxx
-----END PUBLIC KEY-----
```

该证书的 ARN 为 `arn:aws:iot:ap-southeast-1:991983452270:cert/f67c5a8edfb3705c5feacb1f1b052dd5f6ee23881a60ac101af9fdec08b34fe8`

## 7.7. 绑定证书到设备

运行下面命令绑定证书到设备

```shell
$ aws iot attach-thing-principal \
    --thing-name 94c96058af41 \
    --principal arn:aws:iot:ap-southeast-1:991983452270:cert/f67c5a8edfb3705c5feacb1f1b052dd5f6ee23881a60ac101af9fdec08b34fe8 \
    --thing-principal-type "EXCLUSIVE_THING"
```

查询是否绑定成功

```shell
$ aws iot list-thing-principals-v2 \
    --thing-name 94c96058af41 \
    --output json
```
```json
{
    "thingPrincipalObjects": [
        {
            "principal": "arn:aws:iot:ap-southeast-1:991983452270:cert/f67c5a8edfb3705c5feacb1f1b052dd5f6ee23881a60ac101af9fdec08b34fe8",
            "thingPrincipalType": "EXCLUSIVE_THING"
        }
    ]
}
```

可以看见证书绑定成功

# 8. 连接 AWS IoT Core
## 8.1 写入设备证书

根据设备证书生成 `aws_clientcredential_keys.h` 文件

```shell
$ cat << EOF > ~/ameba-rtos/component/application/amazon/amazon-freertos/demos/include/aws_clientcredential_keys.h
#ifndef AWS_CLIENT_CREDENTIAL_KEYS_H
#define AWS_CLIENT_CREDENTIAL_KEYS_H

#define keyCLIENT_CERTIFICATE_PEM \\
$(awk '{ print "    \"" $0 "\\n\" \\" }' 94c96058af41.crt.pem)
    ""

#define keyJITR_DEVICE_CERTIFICATE_AUTHORITY_PEM NULL

#define keyCLIENT_PRIVATE_KEY_PEM \\
$(awk '{ print "    \"" $0 "\\n\" \\" }' 94c96058af41.key.pem)
    ""

#endif /* AWS_CLIENT_CREDENTIAL_KEYS_H */
EOF
```

## 8.2. 写入 MQTT 配置

根据 Wi-Fi 配置和 AWS IoT Core 配置生成 `aws_clientcredential.h` 文件

```shell
cat << EOF > ~/ameba-rtos/component/application/amazon/amazon-freertos/demos/include/aws_clientcredential.h
#ifndef __AWS_CLIENTCREDENTIAL__H__
#define __AWS_CLIENTCREDENTIAL__H__

#define clientcredentialMQTT_BROKER_ENDPOINT         "$(aws iot describe-endpoint)"
#define clientcredentialIOT_THING_NAME               "94c96058af41"
#define clientcredentialMQTT_BROKER_PORT             8883
#define clientcredentialGREENGRASS_DISCOVERY_PORT    8443
#define clientcredentialWIFI_SSID                    "AIOT@FAE"
#define clientcredentialWIFI_PASSWORD                "fae12345678"
#define clientcredentialWIFI_SECURITY                eWiFiSecurityWPA3

#endif /* ifndef __AWS_CLIENTCREDENTIAL__H__ */
EOF
```

## 8.3. 编译固件

```shell
$ cd ~/ameba-rtos/amebadplus_gcc_project
$ ./build.py -a amazon_freertos
```
```
current host platform is linux
Default toolchain path: /opt/rtk-toolchain
ToolChain Had Existed
Toolchain Version Matched
Git found: /usr/bin/git
Python3 found: /usr/bin/python3
project : amebadplus
project : km4
ccache found
DAILY_BUILD = 0
EXAMPLE: amazon_freertos
THE PATH of example_amazon_freertos.c is /home/zxf/ameba-rtos//component/example/amazon_freertos/
project : km0
ccache found
DAILY_BUILD = 0
-- Configuring done (0.1s)
-- Generating done (0.0s)
-- Build files have been written to: /home/zxf/ameba-rtos/amebadplus_gcc_project/build
[9/14] Generating /home/zxf/ameba-rtos/amebadplus_gcc_proj...gcc_project/project_km0/asdk/image/target_img2_otrcore.asm
  BIN      km4_image2_all.bin
========== Image Info HEX ==========
/home/zxf/ameba-rtos/amebadplus_gcc_project/project_km4/asdk/image/target_img2.axf  :
section                               size         addr
.ram_image2.entry                     0x20   0x20004da0
.xip_image2.text                   0x996a0    0xe000020
.sram_timer_idle_task_stack.bss     0x1000   0x20006000
.sram_image2.text.data               0xcc0   0x2000b020
.ARM.exidx                             0x8    0xe0996c0
.ram_image2.bss                     0x8d84   0x2000bd00
.ram_image2.nocache.data              0x1c   0x20014a84
.debug_info                       0x17b62c          0x0
.debug_abbrev                      0x311d2          0x0
.debug_loc                        0x118ee1          0x0
.debug_aranges                      0x6e30          0x0
.debug_ranges                      0x1b340          0x0
.debug_line                        0xd18a3          0x0
.debug_str                         0x2d6a9          0x0
.comment                              0x37          0x0
.ARM.attributes                       0x3c          0x0
.debug_frame                       0x17174          0x0
Total                             0x4a1aaa


   text    data     bss     dec     hex filename
0x996a8   0xce0  0x9da0  672040   a4128 /home/zxf/ameba-rtos/amebadplus_gcc_project/project_km4/asdk/image/target_img2.axf
0x996a8   0xce0  0x9da0  672040   a4128 (TOTALS)
========== Image Info HEX ==========
========== Image Info DEC ==========
/home/zxf/ameba-rtos/amebadplus_gcc_project/project_km4/asdk/image/target_img2.axf  :
section                              size        addr
.ram_image2.entry                      32   536890784
.xip_image2.text                   628384   234881056
.sram_timer_idle_task_stack.bss      4096   536895488
.sram_image2.text.data               3264   536916000
.ARM.exidx                              8   235509440
.ram_image2.bss                     36228   536919296
.ram_image2.nocache.data               28   536955524
.debug_info                       1553964           0
.debug_abbrev                      201170           0
.debug_loc                        1150689           0
.debug_aranges                      28208           0
.debug_ranges                      111424           0
.debug_line                        858275           0
.debug_str                         186025           0
.comment                               55           0
.ARM.attributes                        60           0
.debug_frame                        94580           0
Total                             4856490


   text    data     bss     dec     hex filename
 628392    3296   40352  672040   a4128 /home/zxf/ameba-rtos/amebadplus_gcc_project/project_km4/asdk/image/target_img2.axf
 628392    3296   40352  672040   a4128 (TOTALS)
========== Image Info DEC ==========
========== Image manipulating start ==========
image_size= 963616
checksum=05ff08a4
image_size= 963616
checksum=05ff08a4
========== Image manipulating end ==========
[10/14] Generating /home/zxf/ameba-rtos/component/wifi/wifi_make/wifi_feature_disable/wifi_intf_drv_to_app_ext_noused.c
Gen file: /home/zxf/ameba-rtos/component/wifi/wifi_make/wifi_feature_disable/wifi_intf_drv_to_app_ext_noused.c!
[14/14] Generating /home/zxf/ameba-rtos/amebadplus_gcc_project/project_km0/asdk/image/target_img2.axf
/opt/rtk-toolchain/asdk-10.3.1/linux/newlib/bin/arm-none-eabi-nm: /home/zxf/ameba-rtos/amebadplus_gcc_project/project_km0/asdk/image/target_pure_img2.axf: no symbols
  BIN      km0_image2_all.bin
========== Image Info HEX ==========
/home/zxf/ameba-rtos/amebadplus_gcc_project/project_km0/asdk/image/target_img2.axf  :
section                              size         addr
.ram_image2.entry                    0x40   0x20004d20
.xip_image2.text                  0x4d880    0xc000020
.sram_rtos_static_0.bss             0xa10   0x20008000
.sram_rtos_static_1.bss             0x7e0   0x20005600
.sram_timer_idle_task_stack.bss    0x1000   0x20002000
.sram_image2.text.data              0xae0   0x20068020
.ARM.exidx                            0x8    0xc04d8a0
.ram_image2.bss                    0x3758   0x20068b00
.ram_image2.bd.data                   0x8   0x2006c258
.ram_image2.nocache.data            0x3a0   0x2006c260
.coex_trace.text                   0x1536   0xca000000
.debug_info                       0x2863c          0x0
.debug_abbrev                      0x8a33          0x0
.debug_loc                        0x1acb1          0x0
.debug_aranges                     0x1650          0x0
.debug_ranges                      0x3398          0x0
.debug_line                       0x1b2b6          0x0
.debug_str                         0xae66          0x0
.comment                             0x37          0x0
.ARM.attributes                      0x30          0x0
.debug_frame                       0x34c8          0x0
.stabstr                            0x14f          0x0
Total                             0xcf470


   text    data     bss     dec     hex filename
0x4f89e    0x40  0x5cf0  349646   555ce /home/zxf/ameba-rtos/amebadplus_gcc_project/project_km0/asdk/image/target_img2.axf
0x4f89e    0x40  0x5cf0  349646   555ce (TOTALS)
========== Image Info HEX ==========
========== Image Info DEC ==========
/home/zxf/ameba-rtos/amebadplus_gcc_project/project_km0/asdk/image/target_img2.axf  :
section                             size         addr
.ram_image2.entry                     64    536890656
.xip_image2.text                  317568    201326624
.sram_rtos_static_0.bss             2576    536903680
.sram_rtos_static_1.bss             2016    536892928
.sram_timer_idle_task_stack.bss     4096    536879104
.sram_image2.text.data              2784    537296928
.ARM.exidx                             8    201644192
.ram_image2.bss                    14168    537299712
.ram_image2.bd.data                    8    537313880
.ram_image2.nocache.data             928    537313888
.coex_trace.text                    5430   3388997632
.debug_info                       165436            0
.debug_abbrev                      35379            0
.debug_loc                        109745            0
.debug_aranges                      5712            0
.debug_ranges                      13208            0
.debug_line                       111286            0
.debug_str                         44646            0
.comment                              55            0
.ARM.attributes                       48            0
.debug_frame                       13512            0
.stabstr                             335            0
Total                             849008


   text    data     bss     dec     hex filename
 325790      64   23792  349646   555ce /home/zxf/ameba-rtos/amebadplus_gcc_project/project_km0/asdk/image/target_img2.axf
 325790      64   23792  349646   555ce (TOTALS)
========== Image Info DEC ==========
========== Image manipulating start ==========
image_size= 963616
checksum=05ff08a4
image_size= 963616
checksum=05ff08a4
========== Image manipulating end ==========
Build done
```

## 8.4. 运行 AWS IoT Core 测试固件

> **需要使用带 4MB PSRAM 的 BW20 模组进行测试**

烧录运行固件，波特率设成 1.5Mbps，发送下面的命令连接 Wi-Fi。其中 `TEST` 和 `12345678` 替换为实际的热点名称和密码

```
AT+WLCONN=ssid,TEST,pw,12345678
```
```
AT+WLCONN=ssid,TEST,pw,12345678
[WLAN-A] IPS out
[WLAN-A] set ssid TEST
259 523260 [example_amazon_freertos] [INFO] Waiting for the network link up event...
260 525261 [example_amazon_freertos] [INFO] Waiting for the network link up event...
261 527262 [example_amazon_freertos] [INFO] Waiting for the network link up event...
[WLAN-A] start auth to d6:d8:53:54:f0:a6
[WLAN-A] auth success, start assoc
[WLAN-A] assoc success(1)
[WLAN-A] set group key 4 2
[WLAN-I] set cam: gtk alg 4 0

 write_fast_connect_data_to_flash():not the same ssid/passphrase/channel, write new profile to flash 
[+WLCONN] Connected after 5228 ms.
[WLAN-A] set pairwise key 4(WEP40-1 WEP104-5 TKIP-2 AES-4)

Interface 0 IP address : 192.168.137.15
[+WLCONN] Got IP after 6348 ms.

+WLCONN:OK

[MEM] After do cmd, available heap 290560


#
262 529263 [example_amazon_freertos] [INFO] Creating a TLS connection to a1wpsaryt4080n-ats.iot.ap-southeast-1.amazonaws.com:8883.
263 533069 [example_amazon_freertos] [DEBUG] PKCS #11 module was successfully initialized.
264 533070 [example_amazon_freertos] [INFO] PKCS #11 successfully initialized.
265 533070 [example_amazon_freertos] [DEBUG] Successfully Returned a PKCS #11 slot with ID 1 with a count of 1.
266 533071 [example_amazon_freertos] [DEBUG] Assigned a 0x2 Type Session.
267 533071 [example_amazon_freertos] [DEBUG] Assigned Mechanisms to no operation in progress.
268 533072 [example_amazon_freertos] [DEBUG] Current session count at 0
269 533073 [example_amazon_freertos] [DEBUG] C_Login is not implemented.
270 533073 [example_amazon_freertos] [DEBUG] Successfully generated 32 random bytes.
271 533074 [example_amazon_freertos] [DEBUG] Successfully generated 16 random bytes.
272 533708 [example_amazon_freertos] [INFO] Creating an MQTT connection to a1wpsaryt4080n-ats.iot.ap-southeast-1.amazonaws.com.
273 533709 [example_amazon_freertos] [DEBUG] Encoded size for length 80 is 1 bytes.
274 533709 [example_amazon_freertos] [DEBUG] CONNECT packet remaining length=80 and packet size=82.
275 533710 [example_amazon_freertos] [DEBUG] CONNECT packet size is 82 and remaining length is 80.
276 533711 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=12, Bytes Remaining=70
277 533712 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=2, Bytes Remaining=68
278 533713 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=12, Bytes Remaining=56
279 533713 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=2, Bytes Remaining=54
280 533714 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=54, Bytes Remaining=0
281 533891 [example_amazon_freertos] [DEBUG] Encoded size for length 2 is 1 bytes.
282 533892 [example_amazon_freertos] [DEBUG] BytesReceived=2, BytesRemaining=0, TotalBytesReceived=2.
283 533892 [example_amazon_freertos] [DEBUG] Packet received. ReceivedBytes=2.
284 533893 [example_amazon_freertos] [DEBUG] CONNACK session present bit not set.
285 533893 [example_amazon_freertos] [DEBUG] Connection accepted.
286 533894 [example_amazon_freertos] [DEBUG] Received MQTT CONNACK successfully from broker.
287 533894 [example_amazon_freertos] [INFO] MQTT connection established with the broker.
288 533895 [example_amazon_freertos] [INFO] An MQTT connection is established with a1wpsaryt4080n-ats.iot.ap-southeast-1.amazonaws.com.
289 533896 [example_amazon_freertos] [INFO] Attempt to subscribe to the MQTT topic 94c96058af41/example/topic.
290 533897 [example_amazon_freertos] [DEBUG] Encoded size for length 31 is 1 bytes.
291 533897 [example_amazon_freertos] [DEBUG] Subscription packet remaining length=31 and packet size=33.
292 533898 [example_amazon_freertos] [DEBUG] SUBSCRIBE packet size is 33 and remaining length is 31.
293 533899 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=4, Bytes Remaining=29
294 533900 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=2, Bytes Remaining=27
295 533901 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=26, Bytes Remaining=1
296 533902 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=1, Bytes Remaining=0
297 533902 [example_amazon_freertos] [INFO] SUBSCRIBE sent for topic 94c96058af41/example/topic to broker.
298 534075 [example_amazon_freertos] [DEBUG] Encoded size for length 3 is 1 bytes.
299 534076 [example_amazon_freertos] [DEBUG] Received packet of type 90.
300 534076 [example_amazon_freertos] [DEBUG] Packet identifier 1.
301 534077 [example_amazon_freertos] [DEBUG] Topic filter 0 accepted, max QoS 1.
302 534077 [example_amazon_freertos] [INFO] Subscribed to the topic 94c96058af41/example/topic with maximum QoS 1.
303 534078 [example_amazon_freertos] [INFO] Publish to the MQTT topic 94c96058af41/example/topic.
304 534079 [example_amazon_freertos] [DEBUG] Encoded size for length 42 is 1 bytes.
305 534079 [example_amazon_freertos] [DEBUG] Encoded size for length 42 is 1 bytes.
306 534080 [example_amazon_freertos] [DEBUG] PUBLISH packet remaining length=42 and packet size=44.
307 534081 [example_amazon_freertos] [DEBUG] Encoded size for length 42 is 1 bytes.
308 534081 [example_amazon_freertos] [DEBUG] Adding QoS as QoS1 in PUBLISH flags.
309 534082 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=4, Bytes Remaining=40
310 534083 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=26, Bytes Remaining=14
311 534084 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=2, Bytes Remaining=12
312 534084 [example_amazon_freertos] [DEBUG] sendMessageVector: Bytes Sent=12, Bytes Remaining=0
313 534085 [example_amazon_freertos] [INFO] Attempt to receive publish message from broker.
314 534242 [example_amazon_freertos] [DEBUG] Encoded size for length 2 is 1 bytes.
315 534243 [example_amazon_freertos] [DEBUG] Received packet of type 40.
316 534243 [example_amazon_freertos] [DEBUG] Packet identifier 2.
317 534244 [example_amazon_freertos] [INFO] Ack packet deserialized with result: MQTTSuccess.
318 534245 [example_amazon_freertos] [INFO] State record updated. New state=MQTTPublishDone.
319 534245 [example_amazon_freertos] [INFO] PUBACK received for packet Id 2.
320 534270 [example_amazon_freertos] [DEBUG] Encoded size for length 42 is 1 bytes.
321 534271 [example_amazon_freertos] [DEBUG] QoS is 1.
322 534271 [example_amazon_freertos] [DEBUG] Retain bit is 0.
323 534271 [example_amazon_freertos] [DEBUG] DUP bit is 0.
324 534272 [example_amazon_freertos] [DEBUG] Topic name length: 26.
325 534272 [example_amazon_freertos] [DEBUG] Packet identifier 1.
326 534273 [example_amazon_freertos] [DEBUG] Payload length 12.
327 534273 [example_amazon_freertos] [INFO] De-serialized incoming PUBLISH packet: DeserializerResult=MQTTSuccess.
328 534274 [example_amazon_freertos] [INFO] State record updated. New state=MQTTPubAckSend.
329 534274 [example_amazon_freertos] [INFO] Incoming QoS : 1

330 534275 [example_amazon_freertos] [INFO] Incoming Publish Topic Name: 94c96058af41/example/topic matches subscribed topic.Incoming Publish Message : Hello World!
331 534277 [example_amazon_freertos] [DEBUG] sendBuffer: Bytes Sent=4, Bytes Remaining=0
332 534277 [example_amazon_freertos] [INFO] Keeping Connection Idle...
```

出现上面信息代表模组已经成功连接上 AWS IoT Core

# 9. OTA 测试

## 9.1. 创建 OTA 描述文件

```shell
$ cat << EOF > ota-firmware-files.json
[
    {
        "codeSigning": {
            "awsSignerJobId": "8141658f-f0ed-4b0d-b144-e5d8b442e088"
        },
        "fileName": "ota_all.bin"
    }
]
EOF
```

## 9.2. 创建 OTA 任务

运行下面的命令创建 OTA 任务，其中 `bw20-ota-task-$(uuidgen)` 为随机生成的任务 ID，`arn:aws:iot:ap-southeast-1:991983452270:thinggroup/BW20ModuleGroup` 为步骤 7.2 中创建的设备组的 ARN，`arn:aws:iam::991983452270:role/MyOTAServiceRole` 为步骤 4.2 中创建的 OTA 服务角色。

```shell
$ aws iot create-ota-update \
    --ota-update-id bw20-ota-task-$(uuidgen) \
    --targets arn:aws:iot:ap-southeast-1:991983452270:thinggroup/BW20ModuleGroup \
    --files file://`realpath ota-firmware-files.json` \
    --protocols HTTP MQTT \
    --role-arn arn:aws:iam::991983452270:role/MyOTAServiceRole
```
```
arn:aws:iot:ap-southeast-1:991983452270:otaupdate/bw20-ota-task-91a9f124-b43c-4c8f-aed3-e836d60d87e2    bw20-ota-task-91a9f124-b43c-4c8f-aed3-e836d60d87e2      CREATE_PENDING
```
## 9.3. 编译 OTA 测试固件

去除 `~/ameba-rtos/component/application/amazon/amazon-freertos/ports/amebaDplus/aws_main.c` 中的`RunOtaCoreMqttStreamsDemo()` 前的注释，启用 OTA 例程，并重新编译

```c
    //mqtt mutual auto demo
    //RunCoreMqttMutualAuthDemo(0, NULL, NULL, NULL, NULL);

    //http mutual auto demo
    //RunCoreHttpMutualAuthDemo(0, NULL, NULL, NULL, NULL);

    //device shadow demo
    //RunDeviceShadowDemo(0, NULL, NULL, NULL, NULL);

    //device defender demo
    //RunDeviceDefenderDemo(0, NULL, NULL, NULL, NULL);

    // ota over mqtt demo
    //RunOtaCoreMqttDemo(0, NULL, NULL, NULL, NULL);

    //ota over mqtt streams demo (NEW!)
    /// 去除下面这行注释
    RunOtaCoreMqttStreamsDemo(0, NULL, NULL, NULL, NULL);
```
```shell
$ ./build.py -a amazon_freertos
```

编译完成后，烧录固件进行测试

（待续）

