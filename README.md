
![](https://files.mdnice.com/user/166515/15860f87-2537-4491-b3a4-08739c77d910.jpg)
# Grok Build error sending request 怎么解决？问题出在这里

[点击查看文章 : 2026 国内最新 SuperGrok 升级订阅教程](https://clawdo.com/2026/02/supergrok-upgrade/)


前几天装 Grok Build，被一个特别隐蔽的问题卡了一晚上。

折腾完之后回头看，发现这个坑其实很多人都会踩，踩了之后第一反应基本都是错的。今天把过程整个写出来，看完这篇你应该可以省不少时间。


## 报错长这样

我本地装好 Grok Build 之后，初始化要跳转 xAI 登录认证。浏览器那边一切正常，但终端这边一直转圈，过一会儿就甩出来这段：`error sending request for url https://auth.x.ai/.well-known/openid-configuration`

![](https://files.mdnice.com/user/166515/5c3d51db-487d-4fea-85f5-0feaa5fc18f7.png)

翻译就是：Grok Build 想去请求 `auth.x.ai` 获取登录配置，但是请求根本发不出去。


## 我开始全猜错
我的第一反应是这么排查的：重装、换号、升级、清配置，全没用。

折腾快一小时才反应过来，报错里写得明明白白：`error sending request`



请求压根没发出去——这是网络层的问题，跟上面那些一个都不沾边。

## 最反直觉的一点

根本原因是：

>**浏览器能上不等于终端能上**

我们日常用的本地网络工具，默认只接管浏览器流量。PowerShell、CMD、Node.js 这些命令行工具，默认不读取系统网络配置，全走直连。Grok Build 的请求自然也没走配好的通道。




## 完整排查步骤

### 第 1 步：先确认浏览器能打开

浏览器手动访问：

`
https://auth.x.ai/.well-known/openid-configuration
`

能看到一大段 JSON 配置就说明你的本地网络环境没问题。打不开的话先解决网络环境本身。

### 第 2 步：找到本地网络工具的端口

不同网络工具端口不一样，**千万别照搬别人的端口号**。

打开你自己常用的那个本地网络工具，找到"本地端口"或"入站端口"，把 HTTP 和 SOCKS 两个端口号都记下来。一般在主界面就能看到，类似：

`
本地: socks:XXXXX | http:XXXXX
`


### 第 3 步：在终端里手动配置环境变量

**用 PowerShell 的话**，输入下面的命令，把端口号换成你自己的：

```powershell
$env:HTTP_PROXY="http://127.0.0.1:你的HTTP端口"
$env:HTTPS_PROXY="http://127.0.0.1:你的HTTP端口"
$env:ALL_PROXY="socks5://127.0.0.1:你的SOCKS端口"
```

**用 CMD 的话**，命令换成：

```cmd
set HTTP_PROXY=http://127.0.0.1:你的HTTP端口
set HTTPS_PROXY=http://127.0.0.1:你的HTTP端口
set ALL_PROXY=socks5://127.0.0.1:你的SOCKS端口
```

这是告诉这个终端窗口：你后续发出的请求都走本地这些端口。

**有个特别重要的细节**：

这个配置**只在当前窗口生效**，窗口一关就失效。所以下面所有的测试和最后跑 Grok Build，都必须在**同一个窗口**里完成。

### 第 4 步：用 curl 做对比测试

这一步是整个排查的关键，能精准定位问题。

先测带端口参数的版本：

`
curl.exe -x http://127.0.0.1:你的HTTP端口 https://auth.x.ai/.well-known/openid-configuration
`

返回一大段 JSON 就说明本地通道本身没问题。

再测不带端口参数的版本：

`
curl.exe https://auth.x.ai/.well-known/openid-configuration
`

如果**这个失败、上面那个成功**，就 100% 锁定了问题：命令行环境没读取系统的网络配置，必须靠手动设置环境变量解决。

### 第 5 步：在同一个窗口跑 Grok Build

`
grok-build
`

然后点击界面里的：

`
Login with Grok
`

正常情况下浏览器会自动弹出登录页，跳转完成回到终端就能继续了。

![授权登录](https://files.mdnice.com/user/166515/507a765e-ff1d-42fc-8153-e761ddba0596.png)



## 写在最后

整个问题的核心其实就一句话：

> **浏览器和终端是两套独立的网络通道，给浏览器配好了，不等于给终端配好了。**

排查方法简单四步：

1. 看到 `error sending request`，先怀疑网络可达性，不要先怀疑软件本身。
2. 分别验证浏览器侧和终端侧的可达性。
3. 用 curl 带参数和不带参数各测一次，对比结果就能定位问题。
4. 实在不行就开 TUN，一了百了。

这套思路不光适用于 Grok Build，所有需要联网的命令行工具都适用。各种 AI 命令行工具、`npm install` 超时之类的问题，本质上都是这一类。

希望这篇能帮你少走点弯路。
