---
title: PlanPlayerAnalytics使用Cloudflare生成的JKS证书
date: 2023-02-04 15:51:00
tags: 技术
---
Em，我昨天在做服的时候一直在配置Plan Player Analytics插件的jks证书问题，搜集了很多资料，但是发现都太麻烦了，而且有的根本不能用，官方Wiki那个方法我也没搞懂，所以我自己整了个歪门邪道的方法能快速获取到jks证书供Plan插件使用

## 准备
Cloudflare不多说，不知道的自己百度都有，把你的域名dns解析到上面然后就可以用Cloudflare给域名签发证书了，我是签了15年的源服务器证书。然后我们就可以拿到PEM和Key了。我们将它复制下来，然后在本地创建pem和key文件用来存储这两个的内容。之后就可以进行操作了。我们接下来要用到openssl和JDK（笔者用的版本为17，亲测1.8会报错，所以最好用11往上的）中自带的keytool。

## 开始
找到openssl的bin目录，找到openssl.exe文件，单击右键以管理员身份运行，打开命令行，输入命令：
```
pkcs12 -export -out xxx.pfx -in xxx.pem -inkey xxx.key
```
该操作会要求你输入两次密码，切记这两次密码，后续还需要使用
之后就可以找到保存在你指定的位置的pfx文件了。
使用Keytool工具，将PFX格式证书文件转换成JKS格式，得到“xxx.jks”文件
```
keytool -importkeystore -srckeystore xxx.pfx -destkeystore xxx.jks -srcstoretype PKCS12 -deststoretype JKS
```
输入两次jks设定的密码，可以与上文的一样，然后输入一次上文设置的密码。
之后要切记你输入的两份密码，最好创建一个新的文件保存起来。

## 使用
接下来你指定的目录下应该就生成了jks文件了，将它塞到你的Plan插件文件夹下之后改名为Cert.jks
打开你的Plan/config.yml找到如下键值
```yml
    Security:
        SSL_certificate:
            KeyStore_path: Cert.jks
            Key_pass: pixelab   #pfx的密码
            Store_pass: pixelab #jks的密码
            Alias: 1            #别名，cloudflare一般生成的都是 1 ，如果不是请看下面的解决方法
```
之后重启服务器就可以看到SSL被启用了，同时也可以正常注册、登录了

## 部分问题解决方法
### 别名不正确
输入 keytool -list -v -keystore xxx.jks 然后找到 alias一项，后面的就是别名了 