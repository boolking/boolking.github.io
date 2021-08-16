---
title: "Verilog/VHDL加密浅析"
date: 2021-08-16T15:19:31+08:00
tags:
- Verilog
- SystemVerilog
- VHDL
- EDA
- FPGA
- reverse engineering
categories:
- hardware
summary: 简单分析一下IEEE-1735的使用
draft: false
---

# Verilog/VHDL加密浅析

## 背景

在EDA开发流程中，涉及了很多不同厂商的软件及IP（Intelligence Property），当用户购买了IP后，如何既能保护开发者的利益，又能够方便用户使用？每家厂商都有着自己的加密方案，有的采用了binary的格式，有的采用的是text的格式，当整个flow中采用了多个EDA厂家的产品时，互操作性很难保证，多家EDA厂商共同发布了IEEE1735这个规范，用于加密Verilog/VHDL语言的代码。

## IEEE-1735规范

IEEE-1735规范分为V1和V2两个版本，目前主要使用的是V1。

简单来说，采用的是对称加密算法使用随机生成的密钥(data_key)对代码进行加密并得到加密后的数据，然后将data_key使用每家EDA厂家自己的RSA私钥加密得到encrypted key，将加密后的数据与加密后的密钥编码为文本格式，加上EDA厂家及RSA key name构成一个encrypted envelope替换掉原始文件中的内容。

对数据进行加密的算法一般来说主要采用的是AES-CBC，也支持其他算法比如blowfish-cbc等，这个算法也会作为必要的信息包含在加密后描述信息中。对加密数据和RSA加密后的密钥一般采用的编码方式为BASE64，也有少数采用UUencode的方式。


例如下面的加密样本

```verilog
`pragma protect begin_protected
`pragma protect version = 1
`pragma protect key_keyowner = "AAAA", key_keyname= "BBBB", key_method = "rsa"
`pragma protect key_block
ARIxiMyY/u314g+0H+bMrO2GAJk/FvLIb/AN0hFwJJ3PSmx56gBdOr1Erp45F1i68slteEk2xMxc
7qNtYVS/QcjeM9tl1pE9lqvCtcev/w+cLOdLwVNDWrx+UjXKJyJdZl4TQpNkGhg3jXezLUm7Y9t5
CIG3ztpiaUgwg4uqJDlcJ/DU2c4axryeYy4eJbDz8eBbB/bIkj8ghSMvBZurysZr+z5yMnLIzvNF
FTLCN7D3yz+ymWfmo6nub5jEQZq5fKim7L3VYfA38v4Hs1GcyWAQpn32zYJoaBThREZy5hDQ5wQx
JHxS2NPI+/U8EddEzettvGVl2WQamTv4aCfylA==

`pragma protect data_method = "AES128-CBC"
`pragma protect data_block
u+2vP4WS1vBobqzn052K08Twj4xC9k1w//2Xfp5dqULLFuUvuS5B4VtclRId07+/lwpkBmy5OKPa
H3N+t0hmaGzNtMkPtXP8EPDfJTmnQGNP6hfeL1ZFmGLdbAiT7GFJaeXA1EJcFtL9zocScGoYBg7p
nv1qpOPlZN4CFV/+xKG8MMvCgMRNhGkKqRKAf4QCTzIa2mUta1DjacltrYKoGhOK8BI8nTSipKdp
/NSfW/kkMrtnFmQAYVlPs+/H0Dyo1EC0n2g8c5Ho87we+oUX6s1eiMVSdkM5AcIpq8USxIiU/A48
0NmHieAis8b3ujiVkLO84ZeyS4LNioXMAy45w3jbSfTPDtg3s5SXbFQvUULYFylpHHlQJs0bXkrF

`pragma protect end_protected
```

加密参数：

|parameter|comment|
|---------|-------|
|version|1|
|密钥加密类型|rsa|
|密钥所有者|AAA|
|密钥名称|BBB|
|数据加密算法|AES128-CBC|


## 自定义加密方式

虽然主流的EDA厂家都支持IEEE-1735，各个EDA厂家也有大量遗留IP采用的是自定义的格式。比如Cadence有不少IP都是采用RC5算法对data key进行加密；Synopsys由于收购了不少的EDA厂商，因此加密算法也各不相同，比如Synplicity家有一种怀疑是简单替换密钥的加密方式，VCS也有自己的自定义加密方式VCS003等，下面列出我收集到的各种加密格式的样本列表。


## IP代码混淆

有一些厂商没有对代码直接进行加密，而是采用了混淆的方式来保护IP的代码。也有一些厂商是先混淆，然后再加密。
混淆虽然能够避免用户直接看到代码，但是还是无法抵抗手动还原。

Aldec就有提供一个[脚本](https://www.aldec.com/en/support/resources/documentation/articles/1586)来进行代码的混淆。

## RSA密钥废弃

为了保证互操作性，各大EDA厂商会将RSA的公钥公布到网上或者包含在自己的EDA工具中，方便第三方加密自己的IP。同时为了避免私钥泄露导致的代码被解密，各大EDA厂商也会经常更换新的RSA公钥，而已泄露的私钥将被废弃。比如，Mentor的MGC-VERIF-SIM-RSA-1和Aldec的ALDEC08_001就已经被废弃了。

## 已公开的RSA私钥

### MGC-DVT-MTI
* [URL](http://bbs.eetop.cn/thread-595345-1-1.html)

### cds_key
* [URL](http://bbs.eetop.cn/thread-597673-1-1.html)

### SNPS-VCS-RSA-1
* [URL](http://bbs.eetop.cn/forum.php?mod=redirect&goto=findpost&ptid=595345&pid=8963053)

### xilinx_2016_05
* [URL](http://bbs.eetop.cn/thread-622518-1-1.html)

### ALDEC08_001
* [URL](https://github.com/dmitrodem/p1735_decryptor)

另外，还有一篇[帖子](http://bbs.eetop.cn/thread-857123-1-1.html)对各种密钥的使用做了说明。

## 代码解密

为了解密IEEE-1735加密的代码，只需要得到对应的RSA私钥即可。获取RSA私钥有两种方式：
1. 通过公钥计算出私钥
2. 通过逆向工程的方式，从综合工具中找到私钥

### 通过公钥计算私钥

当RSA密钥长度比较短时，可以通过暴力穷举的方式进行因数分解计算其私钥，目前主流的方法是[NFS](https://en.wikipedia.org/wiki/General_number_field_sieve)。可以使用GNNFS和MSEIVE等工具来进行，我尝试过分解一个512bit的RSA私钥，在目前主流的配置上不到1周时间可以分解成功。具体流程可以参考看雪的[两篇](https://bbs.pediy.com/thread-156206.htm)[贴子](https://bbs.pediy.com/thread-268842.htm)。

当RSA密钥长度为1024甚至2048的时候，暴力因数分解的时间就是不可接受的了。

### 逆向工程找到私钥

由于EDA工具最终识别的是Verilog/VHDL代码，因此必须将加密后的代码进行解密才能继续后续的综合等工作，因此综合工具中必然包含了其RSA解密说必须的私钥。为了避免被逆向出私钥，EDA工具也是对私钥进行了很多的保护。从我遇到的情况来看，主要有如下几种：
* 不加密，直接字符串保存
  这种，直接搜索MII这个特征字符串，基本就很容易定位了
* 对私钥进行加密
  需要在解密后才能得到MII打头的私钥
* 不保存完整的私钥，而是保存RSA的p/q/e等参数，直接采用p/q/e进行解密
  根据获取的p/q/e计算出phi，n和d，然后编码为ASN.1格式并base64后加上头尾即可
* 对解密过程进行混淆
  去除混淆，找到密钥

### 样本列表

IEEE-1735的加密文件样本：

|vendor name                    |key name             |comment                      |
|-------------------------------|---------------------|-----------------------------|
|Synopsys                       |SNPS-VCS-RSA-1       |RSA                          |
|Synopsys                       |SNPS-VCS-RSA-2       |RSA                          |
|Cadence Design Systems.        |CDS_RSA_KEY          |RSA                          |
|Cadence Design Systems.        |CDS_RSA_KEY_VER_1    |RSA                          |
|Mentor Graphics Corporation    |MGC-VERIF-SIM-RSA-1  |RSA                          |
|Mentor Graphics Corporation    |MGC-VERIF-SIM-RSA-2  |RSA                          |
|Mentor Graphics Corporation    |MGC-VERIF-SIM-RSA-3  |RSA                          |
|MTI                            |MGC-DVT-MTI          |RSA                          |
|Xilinx                         |xilinx_2016_05       |RSA                          |
|Synplicity                     |SYNP05_001           |RSA                          |
|Synplicity                     |SYNP15_1             |RSA                          |
|ATRENTA                        |ATR-SG-2015-RSA-3    |RSA                          |
|Aldec                          |ALDEC08_001          |RSA                          |
|Aldec                          |ALDEC15_001          |RSA                          |

非标准加密方式：

|vendor name                    |key name             |comment                      |
|-------------------------------|---------------------|-----------------------------|
|Cadence Design Systems.        |cds_key              |RC5                          |
|Cadence Design Systems.        |cds_nc_key           |RC5                          |
|Cadence Design Systems.        |CDS_DATA_KEY         |直接加密data                  |
|Synopsys VCS                   |VCS001               |加密算法VCS003，使用uuencode编码|
|Intel Quartus                  |                     |使用binary文件，AES密码保存在FlexLM license的vendor_string中|


## 总结

为了保护IP，各大EDA厂商使用IEEE-1735进行保护，但是究其本质，IEEE-1735的V1版标准是无法避免私钥泄露或者被逆向工程利用的。

我通过对EDA工具的逆向工程，已经获取了上面这些密钥并编写了完整的解密工具，如果有需要，欢迎通过email与我联系:
bmV3bWFua2ltYmVyMjcxQGdtYWlsLmNvbQ==