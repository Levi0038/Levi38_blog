---
title: THM Tempest
description: 通过日志和流量包分析攻击者攻击行为的 THM 靶场练习
published: 2026-11-25
tags: [应急响应, 流量分析, TryHackMe]
category: 题目
draft: false
---

# Tempest

## 前言

这个靶场可以帮助我们学习如何通过日志和流量包分析攻击者的攻击行为

因为我是先打了靶机再书写笔记 所以整个笔记以更容易阅读的顺序进行书写

## 前期准备

因为整个靶场只是对静态文件进行分析 不会有动态交互 所以我选择先把文件从靶机上拿到自己电脑上来

用wsl起一个Impacket-SMBServer 

![smbserver1](/Levi38_blog/images/thm-tempest/smbserver1.png)

再在靶机上传输回来

`net use \\10.13.4.71\share /u:rse rse`

`copy .\* \\10.13.4.71\share\ `

最终得到标记后的三个文件

![image-20261125140410104](/Levi38_blog/images/thm-tempest/image-20261125140410104.png)

## 工具准备

**日志分析**

EvtxCmd

Timeline Explorer

sysmonView

Event Viewer

**流量分析**

wireshark

Brim（非必需 我就没有用到）

### EvtxECmd & Timeline Explorer

EvtxECmd是一个日志解析工具 它可以帮我们把EVTX 日志转换为 CSV 格式，再到Timeline Explorer中进行分析 这样日志文件就变得十分有序易于分析

```
.\EvtxECmd.exe -f 'C:\Users\user\Desktop\Incident Files\sysmon.evtx' --csv 'C:\Users\user\Desktop\Incident Files' --csvf sysmon.csv
```

用Timeline Explorer打开csv文件即可进行分析

![image-20261125133246752](/Levi38_blog/images/thm-tempest/image-20261125133246752.png)

### Event Viewer&SysmonView

Sysmon日志是Windows系统上由Sysmon工具生成的**详细系统活动遥测记录**。它写在Windows事件日志中，核心价值在于提供比系统默认日志更丰富的上下文（如完整命令行、哈希值、父进程关系），用于安全监控、威胁狩猎和取证分析。

在较新的Windows系统上，日志位于：
`应用程序和服务日志` > `Microsoft` > `Windows` > `Sysmon` > `Operational`

对于sysmon日志 我们可以先用Event Viewer导出为xml格式 再用Sysmon import导入分析

![image-20261125134110721](/Levi38_blog/images/thm-tempest/image-20261125134110721.png)

## 恶意文档

根据题目所说 此次入侵是从一个恶意文档开始的 分析师还汇总了由该警报产生的关键信息，具体如下：

- 该恶意文档的扩展名为.doc。
- 用户通过 chrome.exe 下载了该恶意文档。
- 该恶意文档随后执行了一连串的命令，以实现代码的执行。

因为我们知道是使用chrome.exe下载的 所以首先我们可以使用sysmonview来快速定位这个恶意文档

![image-20261125134656522](/Levi38_blog/images/thm-tempest/image-20261125134656522.png)

通过在Timeline Explorer中检索关键字我们可以发现这个文档是用WINWORD打开的

![image-20261125134931048](/Levi38_blog/images/thm-tempest/image-20261125134931048.png)

并且关注到这个进程的PID是496  在sysmonview中发现WINWORD中有一个很奇怪的域名phishteam.xyz

![image-20261125135335509](/Levi38_blog/images/thm-tempest/image-20261125135335509.png)

此时我们肯定很关心这个程序到底干了什么 那么根据这个PID496 如果我们找到了它调用的子进程 就可以知道干了什么

在payloaddata5里 我们可以筛选这个PPID

![image-20261125135628588](/Levi38_blog/images/thm-tempest/image-20261125135628588.png)

![image-20261125135729226](/Levi38_blog/images/thm-tempest/image-20261125135729226.png)

得到了执行的命令

让ds帮我们分析一下这个命令

![image-20261125135817111](/Levi38_blog/images/thm-tempest/image-20261125135817111.png)

## Follina执行

根据对恶意代码的分析 我们可以知道每次用户登录时 恶意代码就会执行  

根据之前恶意代码执行的时间 我们可以知道攻击一定是发生在这个时间点后

并且根据Sysmon 的 **Event ID 1** 代表 **“进程创建”** 事件 我们重点关注Event ID 1的事件

![image-20261125140450112](/Levi38_blog/images/thm-tempest/image-20261125140450112.png)

在17:15:10的事件里 我们可以看到执行了一串powershell代码

![image-20261125141255289](/Levi38_blog/images/thm-tempest/image-20261125141255289.png)

它下载了phishteam.xyz里的first.exe然后运行

![image-20261125141412521](/Levi38_blog/images/thm-tempest/image-20261125141412521.png)

在sysmonview中我们还会发现另一个可疑的域名 resolvecyber.xyz first.exe一直在向这个域名发送数据

## C2流量分析

我们手里的流量包可以对其进行分析

在wireshark里筛选http.host matches "resolvecyber.xyz"

可以看到在往resolvecyber.xyz的/9ab62b5路由用q参数传递一些base64编码后的数据

![image-20261125142031040](/Levi38_blog/images/thm-tempest/image-20261125142031040.png)

![image-20261125142217829](/Levi38_blog/images/thm-tempest/image-20261125142217829.png)

我们可以用tshark将他们全部提取出来

```
tshark -r capture.pcapng -Y 'http.request.uri contains "/9ab62b5?q="' -T fields -e http.request.uri | grep -oP '(?<=/9ab62b5\?q=)\S+'
```

![image-20261125142654894](/Levi38_blog/images/thm-tempest/image-20261125142654894.png)

解码后发现这都是一些powershell命令

![image-20261125142728644](/Levi38_blog/images/thm-tempest/image-20261125142728644.png)

它通过某个脚本找到了用户的密码 并且对端口进行了侦察 可以看到5985端口开放

并且下载了一个ch.exe

![image-20261125142856795](/Levi38_blog/images/thm-tempest/image-20261125142856795.png)

![image-20261125143033430](/Levi38_blog/images/thm-tempest/image-20261125143033430.png)

![image-20261125143315742](/Levi38_blog/images/thm-tempest/image-20261125143315742.png)

可以看到利用这个ch.exe开了一个隧道

![image-20261125143408033](/Levi38_blog/images/thm-tempest/image-20261125143408033.png)

并且根据沙箱分析这就是chisel 是我们常用的隧道转发工具![屏幕截图 2026-11-24 113702](../../../../图片/Screenshots/屏幕截图 2026-11-24 113702.png)

并且在之后还有wsmprovhost的启动信息 这个进程是用于WINRM的启动 之后权限突然变成了system

也就是进行了提权

![image-20261125144817779](/Levi38_blog/images/thm-tempest/image-20261125144817779.png)

![image-20261125145133982](/Levi38_blog/images/thm-tempest/image-20261125145133982.png)

可以看到在17:20:06的时候 下载了spf.exe 同样的我们可以用sha256在沙箱中进行分析

发现执行了PrintSpoofer 因为具有SeImpersonatePrivilege所以进行了提权

![image-20261125145234182](/Levi38_blog/images/thm-tempest/image-20261125145234182.png)

![屏幕截图 2026-11-24 133614](../../../../图片/Screenshots/屏幕截图 2026-11-24 133614.png)

我们可以看到，这个spf.exe又执行了另一个二进制文件。

![final](/Levi38_blog/images/thm-tempest/final.png)

使用 SysmonView，我们可以看到与之前相同的域名连接，只不过连接的是8080端口

![final2](/Levi38_blog/images/thm-tempest/final2.png)

## 持久化

在成为 SYSTEM 之后 添加了两个账户用于持久化 其中一个加入了system组

![image-20261125145855132](/Levi38_blog/images/thm-tempest/image-20261125145855132.png)

![image-20261125145904589](/Levi38_blog/images/thm-tempest/image-20261125145904589.png)

我们可以通过在 Windows 日志中筛选Event ID 4720来确认这一点

![image-20261125150026875](/Levi38_blog/images/thm-tempest/image-20261125150026875.png)

攻击者还使用了一种名为 sc.exe 的持久性攻击手段

![image-20261125150151888](/Levi38_blog/images/thm-tempest/image-20261125150151888.png)

sc.exe 用于控制服务 auto=start 确保了 final.exe 能够在启动时运行

![image-20261125150236268](/Levi38_blog/images/thm-tempest/image-20261125150236268.png)

至此我们就摸清了攻击者的整个攻击流程