---
layout: post
title: Jenkins + fastlane + fir 自动化打包工具配置 
tags: [Jenkins,fastlane,自动化打包]
---

## 概览

目的：手动点击构建，拉取 gitlab repo 进行构建打包，然后上传到fir.

本文里主要做如下事情：

- 配置 Jenkins 执行 fastlane 脚本，然后上传 fir
- 配置 match 匹配 certification 和 profile
- 配置 fastlane 打包，build number 修改


## Jenkins 配置

这里我们只是手动点击构建时，才会进行构建操作，因此没有【构建触发器】配置。又因为是用 fastlane 进行打包操作，所以没有【构建环境】操作。

- 创建实例
- General -> GitHub project 
	- 填入 project url
	- 填入 respository name
- 源码管理 -> Git
	- Respositories -> Respository URL 
		-  填入项目 gitlab 地址
	- branches to build 
		- 填入你想要啦取的分支，例如：*/master
- 构建
	- Execute shell
		- command 里填入内容：`fastlane adhoc`
- 构建后操作
	- Archive the artifacts 存档，这里是放倒工程的 jenkin_build 目录下
		- 填入：`jenkin_build/*.ipa`
	- Upload to fir.im 上传到 fir。jenkins-fir 插件可以去 fir 上去找。
		- 填入 fir.im Token ，然后验证一下。 


对于 project url 里，可以填入 repo 的 ssh 地址，也可以是 https 地址。如果是 ssh 地址的话，需要配置 gitlab 的 private key.

![General](https://raw.githubusercontent.com/zlanchun/blogImages/master/jenkins_fastlane_fir_autobuild/jenkins_fastlane_fir_autobuild1.png)
![源码管理 ](https://raw.githubusercontent.com/zlanchun/blogImages/master/jenkins_fastlane_fir_autobuild/jenkins_fastlane_fir_autobuild2.png)
![构建 ](https://raw.githubusercontent.com/zlanchun/blogImages/master/jenkins_fastlane_fir_autobuild/jenkins_fastlane_fir_autobuild3.png)
![构建后操作](https://raw.githubusercontent.com/zlanchun/blogImages/master/jenkins_fastlane_fir_autobuild/jenkins_fastlane_fir_autobuild4.png)

## match 配置
参考：

- [Simplify your life with fastlane match -- Macoscope Blog](http://macoscope.com/blog/simplify-your-life-with-fastlane-match/#migration)
- [match - fastlane docs](https://docs.fastlane.tools/actions/match/)

match 是用 git 管理证书/profile. 然后，一套证书／profile 可以快速的用在多种设备上。

使用上大致上有2种方式：一种是直接生成新的证书/profile，进行管理。另一种是在之前的证书/profile上进行手工加密／解密，然后匹配存储到 gitlab 上。

对于第一种来说，直接创建一个 gitlab 私有库，空白的什么都没有，然后配置 match 指向这个 repo , fastlane 执行一下就好了。

对于第二种，就需要很复杂的操作了。

1. 导出当前正在用的 development/distribution 的 cer/p12。
2. 加密
3. 导出当前正在用的 profile 
4. 加密
5. 创建 repo ，并创建固定格式的文件夹，将之前创建好的文件放入到对应的目录下。

### Fastfile 配置

我们以 Adhoc 打包方式为例子。

在工程根目录下，执行：

```shell
fastlane init
// 需要填入repo地址
fastlane match init 
```

然后会生成 fastlane 目录。

```
fastlane/
├── Appfile
├── Fastfile
├── Matchfile
├── README.md
└── report.xml
```

在 Fastfile 里添加一个 lane

```ruby

	app_identifier = 'io.github.zlanchun.test'
	repo = 'http://gitlab.example.com/pacez/profiles.git'	

  desc "match adhoc profile"
  lane :matchAdhoc do
    match(git_url:"#{repo}",
      type:"adhoc",keychain_password:"123",app_identifier:"#{app_identifier}",
      readonly: true )
  end
```

match 命令的参数：

- git_url ：gitlab 私有库的地址
- type : 类型，可以是：adhoc/appstore/development/enterprise， 这里我们是打 adhoc 包，所以填入 adhoc
- keychain_password: 管理密码
- app_identifier: bundle id
- readonly : 是否可读，如果为 true 则只能从 repo 里拉取，而不能创建。如果为 false ，也是从 repo 里拉取，如果不匹配则自动创建。

我们手动加密证书/profile，所以 readonly 这里配置 true.

### 生成新证书/profile

把上面的 readonly 改为 false.

执行：

```shell
$ fastlane matchAdhoc
```

根据提示信息输入密码，然后就会创建新的证书/profile ，并推送到 repo 里。

repo 里的内容：

```
├── README.md
├── certs
│   ├── development
│   │   ├── 22XXXXX8G.cer
│   │   └── 22XXXXX8G.p12
│   └── distribution
│       ├── Y9XXXXX8R.cer
│       └── Y9XXXXX8R.p12
├── match_version.txt
└── profiles
    ├── adhoc
    │   └── AdHoc_io.github.zlanchun.test.mobileprovision
    ├── appstore
    │   └── AppStore_io.github.zlanchun.test.mobileprovision
    └── development
        └── Development_io.github.zlanchun.test.mobileprovision
```

### 匹配现有的 证书/profile

#### 获取正在用的证书的 certId

lookupCertId.rb 会找到你账号下的所有的证书信息（证书ID，时间什么的）

```ruby
# lookupCertId.rb
require 'spaceship'

Spaceship.login('your@appleid')
Spaceship.select_team

Spaceship.certificate.all.each do |cert| 
  cert_type = Spaceship::Portal::Certificate::CERTIFICATE_TYPE_IDS[cert.type_display_id].to_s.split("::")[-1]
  puts "Cert id: #{cert.id}, name: #{cert.name}, expires: #{cert.expires.strftime("%Y-%m-%d")}, type: #{cert_type}"
end
```

替换 lookupCertId.rb 中的 your@appleid 为你的 appid 账号，然后命令行里执行：

```shell
$ ruby lookupCertId.rb
```

类似于下面这个样子：

```
Cert id: 22XXXXX8G, name: iOS Development, expires: 2019-05-21, type: Development
```

找到你的正在用的证书，保存下来证书ID，之后会用到。

#### 创建 repo 目录结构

在 repo 里创建目录 certs （管理证书）、目录 profiles （管理 profiles）、文件 match_version.txt （记录 fastlane match —version 内容）

```
├── certs
│   ├── development
│   └── distribution
├── match_version.txt
└── profiles
    ├── adhoc
    ├── appstore
    └── development
```

#### 钥匙串导出证书

钥匙串 -> 登录 -> 我的证书 -> 右键导出 certificate.p12 / certificate.cer 

#### 将私钥提取到key.pem文件

```shell
$ openssl pkcs12 -nocerts -nodes -out key.pem -in certificate.p12
```

#### 加密 cer/p12

命令如下

`openssl aes-256-cbc -k password -in key.pem -out cert_id.p12 -a`

执行，123 是密码，需要替换成你自己的

```shell
$ openssl aes-256-cbc -k 123 -in key.pem -out cert_id.p12 -a
$ openssl aes-256-cbc -k 123 -in certificate.cer -out cert_id.cer -a
```

#### 加密 profile

把正在用的 profile 从 apple 上下载下来，然后加密，过程和上一步骤类似。

`openssl aes-256-cbc -k password -in development.mobileprovision -out Development_bundle.id.mobileprovision -a`

执行，注意导出的文件名是 Type_bundleId.mobileprovision，bundleId 里包含 `.` 不要写错了。

```shell
$ openssl aes-256-cbc -k 123 -in match_AdHoc_ioGithubZlanchunTest.mobileprovision -out AdHoc_io.github.zlanchun.test.mobileprovision -a
```

#### 推送到 repo

将这些生成的文件放到对应的目录文件里。目录结构参考上面生成新证书的目录结构。

```
├── certs
│   ├── development
│   │   ├── 22XXXXX8G.cer
│   │   └── 22XXXXX8G.p12
│   └── distribution
│       ├── Y9XXXXX8R.cer
│       └── Y9XXXXX8R.p12
├── match_version.txt
└── profiles
    ├── adhoc
    │   └── AdHoc_io.github.zlanchun.test.mobileprovision
    ├── appstore
    │   └── AppStore_io.github.zlanchun.test.mobileprovision
    └── development
        └── Development_io.github.zlanchun.test.mobileprovision
```

##### match 匹配

```shell
$ fastlane matchAdhoc
```

## fastlane 配置

以 Adhoc 为例，包含三个功能：

- 打包
- 匹配 adhoc profile
- 更新 build number (这里是改成当天时间: 年月日)

```ruby

platform :ios do

  app_identifier = 'io.github.zlanchun.test'
  currentTime = Time.new.strftime("%Y%m%d")
	
	 desc "adhoc"
  lane :adhoc do
		# 更新 build number，这里写 Info.plist 的相对地址
    set_info_plist_value(path: "./Test/Info.plist", key: "CFBundleVersion", value: "#{currentTime}")
    gym(
        workspace: "Test.xcworkspace",
        scheme: "Test",
        configuration: "Release",
        silent: true,
        clean: true,
        include_symbols: true,
        include_bitcode: true,
        export_method: "ad-hoc",
        output_directory: "jenkin_build/",
        export_options: {
			# 匹配 AdHoc profile
                provisioningProfiles: {
                    app_identifier => "match AdHoc #{app_identifier}"
                }
            }
        ) 
  end
end
```

测试：

```shell
$ fastlane adhoc
```

如果没有报错，则一切成功了。

## 问题集

### {"authType"=>"sa"} (Spaceship::Tunes::Error)

`ruby lookupCertInfo.rb` 时报错了，找了好久，原来是 iTunes 有个新协议需要确认，gg

参考：[Twitter](https://twitter.com/joshdholtz/status/998707288432136192)

登录并确认：[iTunes Connect](https://itunesconnect.apple.com/login) 
