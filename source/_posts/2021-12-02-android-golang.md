---
title: android运行golang程序
date: 2021-12-02 05:15:31
tags:
  - android
  - golang
---

写了个golang程序来根据当前ip自动修改阿里云ecs的安全组.现在需要在手机运行,找了下资料实现了.
android未root的情况下.

## 编译出android环境的程序
把main.go文件编译出anroid arm64的程序.大部分android都是arm64, 可以在anroid上运行`getprop ro.product.cpu.abi`查看具体架构.
```
set CGO_ENABLED=0
set GOOS=linux
set GOARCH=arm64
go build -o xxx main.go
```

## 运行环境termux
[termux](https://termux.com/) 是免费的android终端模拟器.  
打开termux, 安装下面的包.

### 安装termux-setup-storage
默认工作目录是termux的安装目录,想要获取手机存储则需要安装它.    
在终端里面输入 `termux-setup-storage`, 安装好了后termux主目录会生成 `storage` 子目录,这时我们就可以访问手机存储了

### 安装termux-chroot
termux没有传统的linux文件系统, 例如`/etc` ,这个可以设置一个最小的正常的文件系统.
这样就可以通过修改文件`/etc/resolv.conf` 来设置 dns
```
pkg install termux-chroot
```

### 安装vim
golang程序里面有网络访问, 在termux里面报`read: connection refused` 错误,
所以要用它来改dns设置
```
pkg install vim
```

设置dns
```
vim /etc/resolv.conf

# /etc/resolv.conf
nameserver 8.8.8.8
```

## 运行golang
在手机存储目录是没有权限的,需要把程序复制到termux目录
```
cp xxx ~/xxx
./xxx
```
这样就好了

## 源代码
```golang
# main.go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"path"
	"strings"
	"time"

	"github.com/aliyun/alibaba-cloud-sdk-go/services/ecs"
)

type ConfigurationModel struct {
	ACCESS_KEY_ID     string
	ACCESS_KEY_SECRET string
	REGION_ID         string
	SECURITY_GROUP_ID string
	IP_PROTOCOL       string
	PORT_RANGE        string
	DESCRIPTION       string
	DURATION          int
}

type IpResponse struct {
	Origin string `json:"origin"`
}

var configModel ConfigurationModel

// 公网 IP 查询站点的 URL 地址。
const GetPublicIpUrl = "http://www.httpbin.org/ip"

func main() {
	loadConfig()
	fmt.Println("配置：", configModel.SECURITY_GROUP_ID)

	filename := configModel.SECURITY_GROUP_ID + "_ip.txt"
	oldIp := readIp(filename)
	fmt.Printf("最后IP记录为: %s\n", oldIp)

	ecsClient, err := ecs.NewClientWithAccessKey(configModel.REGION_ID, configModel.ACCESS_KEY_ID, configModel.ACCESS_KEY_SECRET)
	if err != nil {
		// Handle exceptions
		panic(err)
	}

	for {
		newIp := getPublicIp()
		if oldIp != newIp {
			fmt.Printf("IP变更: %s => %s\n", oldIp, newIp)

			delSecurityGroup(ecsClient, configModel, oldIp)
			addSecurityGroup(ecsClient, configModel, newIp)

			oldIp = newIp
			writeIp(filename, newIp)
		}

		time.Sleep(time.Duration(configModel.DURATION) * time.Second)
	}

}

func loadConfig() {
	var configFile string

	dir, _ := os.Getwd()
	configFile = path.Join(dir, "settings.json")

	// 打开配置文件，并进行反序列化。
	f, err := os.Open(configFile)
	if err != nil {
		log.Fatalf("无法打开文件：%s", err)
		os.Exit(-1)
	}
	defer f.Close()
	data, _ := ioutil.ReadAll(f)

	if err := json.Unmarshal(data, &configModel); err != nil {
		log.Fatalf("数据反序列化失败：%s", err)
		os.Exit(-1)
	}
}

func getPublicIp() string {
	resp, err := http.Get(GetPublicIpUrl)
	if err != nil {
		log.Printf("获取公网 IP 出现错误，错误信息：%s", err)
		os.Exit(-1)
	}
	defer resp.Body.Close()

	bytes, _ := ioutil.ReadAll(resp.Body)

	var res IpResponse
	if err := json.Unmarshal(bytes, &res); err != nil {
		log.Fatalf("数据反序列化失败：%s", err)
		os.Exit(-1)
	}

	return res.Origin
}

func delSecurityGroup(client *ecs.Client, configModel ConfigurationModel, ip string) {
	request := ecs.CreateRevokeSecurityGroupRequest()
	request.SecurityGroupId = configModel.SECURITY_GROUP_ID
	request.PortRange = configModel.PORT_RANGE
	request.IpProtocol = configModel.IP_PROTOCOL
	request.SourceCidrIp = ip

	response, err := client.RevokeSecurityGroup(request)
	if err != nil {
		// 异常处理
		panic(err)
	}
	fmt.Printf("删除安全组成功: (%d)! %s\n", response.GetHttpStatus(), ip)
}

func addSecurityGroup(client *ecs.Client, configModel ConfigurationModel, ip string) {
	request := ecs.CreateAuthorizeSecurityGroupRequest()
	request.SecurityGroupId = configModel.SECURITY_GROUP_ID
	request.PortRange = configModel.PORT_RANGE
	request.IpProtocol = configModel.IP_PROTOCOL
	request.Description = configModel.DESCRIPTION
	request.SourceCidrIp = ip

	response, err := client.AuthorizeSecurityGroup(request)
	if err != nil {
		// 异常处理
		panic(err)
	}
	fmt.Printf("添加安全组成功: (%d)! %s\n", response.GetHttpStatus(), ip)
}

func readIp(filename string) string {
	ip := "127.0.0.1"

	contents, err := ioutil.ReadFile(filename)
	if err == nil {
		//因为contents是[]byte类型，直接转换成string类型后会多一行空格,需要使用strings.Replace替换换行符
		ip = strings.Replace(string(contents), "\n", "", 1)
	}

	return ip
}
func writeIp(filename string, content string) {
	data := []byte(content)
	ioutil.WriteFile(filename, data, 0644)
}

```
```
# settings.json
{
    "ACCESS_KEY_ID": "xx",
    "ACCESS_KEY_SECRET": "xx",
    "REGION_ID": "cn-shenzhen",
    "SECURITY_GROUP_ID": "xx",
    "IP_PROTOCOL": "tcp",
    "PORT_RANGE": "1/65535",
    "DESCRIPTION": "白名单ip",
    "DURATION": 60
}
```

## 参考链接
- [使用 Android 运行 Golang 程序](https://www.cnblogs.com/togettoyou/p/goandroidshell.html)