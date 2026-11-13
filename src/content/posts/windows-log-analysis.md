---
title: Windows 日志分析
description: 应急响应中 Windows 事件日志的分析方法与排查思路
published: 2026-11-13
tags: [应急响应, Windows, 日志分析]
category: 笔记
draft: false
---

# windows日志分析

## 前言

在应急响应和渗透测试的过程中，日志分析是很重要的一个环节。 事件日志记录了系统在运行过程中发生的各种事件，从而提供了可用于分析系统行为并诊断问题的审计线索

Windows 事件日志不是普通文本文件，无法使用文本编辑器进行查看。但是可以通过 Windows API 将原始数据转换为 XML 格式。这些日志文件中的事件以专有二进制格式存储，扩展名为.evt 或.evtx。带有.evtx 扩展名的日志文件通常位于 `C:\Windows\System32\winevt\Logs` 目录下

## 日志类别

windows日志主要有这三类

**System Logs**

记录与操作系统相关的事件

**Security Logs**

记录与设备登录和登出相关的事件 这些日志可以作为分析的重要依据

**Application Logs**

记录与系统中安装的应用程序相关的事件

在windows中，想要查看日志可以有以下三种方式

1. **事件查看器**
2. **Wevtutil.exe**
3. **Get-WinEvent** 

## 事件查看器

这是Windows内置的图形化日志查看软件 可以win+r 输入eventvwr.msc来启动

![image-20261112113012508](/Levi38_blog/images/win-log-analysis/image-20261112113012508.png)

在应用程序和服务日志的`Microsoft > Windows > PowerShell > Operational` 部分，PowerShell 会将有关引擎，提供程序和命令的操作记录到 Windows 事件日志中

右键Operational进入到属性中，可以看到日志的位置、日志的大小，以及日志创建、修改和最后访问的时间，同时右下角还有清除日志的功能

![image-20261112113913053](/Levi38_blog/images/win-log-analysis/image-20261112113913053.png)

同时每个事件有对应的级别 时间 来源 id 任务类别记录

![image-20261112114337994](/Levi38_blog/images/win-log-analysis/image-20261112114337994.png)

比如id4104在PowerShell日志下就记录了脚本块日志 4103是执行管道操作

然后 在右侧的操作区域 可以使用筛选当前日志或创建自定义视图的功能来进行各种条件的过滤

![image-20261112114731181](/Levi38_blog/images/win-log-analysis/image-20261112114731181.png)

## Wevtutil

命令行工具 相比于事件查看器更加灵活

可以用wevtutil.exe /?查看帮助

![image-20261113102855282](/Levi38_blog/images/win-log-analysis/image-20261113102855282.png)

如果我们要查询某个具体命令的用法 

比如 wevtutil qe /? 用于从日志或日志文件中查询事件

![image-20261113103333418](/Levi38_blog/images/win-log-analysis/image-20261113103333418.png)

以下是一些常见命令示例

```
wevtutil.exe el | Measure-Object     #查询这台机器上共有多少个日志名称
wevtutil qe Application /c:3 /rd:true /f:text #返回Application中的最近三条日志的文本格式
```

## Get-WinEvent

Get-WinEvent是另一个powershell工具

![image-20261113104829610](/Levi38_blog/images/win-log-analysis/image-20261113104829610.png)

常见命令

```
Get-WinEvent -ListLog *      #提取所有日志信息
Get-WinEvent -ListLog *openssh* #筛选和openssh有关的日志
Get-WinEvent -ListProvider * #获取事件日志的提供程序以及日志名称
(Get-WinEvent -ListProvider Microsoft-Windows-PowerShell).Events | Format-Table Id, Description #查询某个Provider能提供的事件类型
```

哈希表筛选

```
Get-WinEvent -FilterHashtable @{
  LogName='Application' 
  ProviderName='WLMS' 
}
或者用分号分隔
Get-WinEvent -FilterHashtable @{LogName='Application' ;ProviderName='WLMS' }
```

![Get-WinEvent acceptable key/value pairs for the FilterHashtable parameter](/Levi38_blog/images/win-log-analysis/filter-hashtable.png)

在事件查看器中也可以查看哈希表的构建信息

![Windows Event Viewer displaying information about MsiInstaller application](/Levi38_blog/images/win-log-analysis/build-hash-table.png)

## XPath 查询

XPath（XML Path Language）是一种**用于在 XML 文档中定位和筛选节点的查询语言**

可以根据事件查看器的xml视图编写XPath语句

![image-20261113112148991](/Levi38_blog/images/win-log-analysis/image-20261113112148991.png)

第一个标签可以是 `*` ，也可以是 `Event` 

下一层是System

然后EventID 是16384

如果用Get-WinEvent查询 查询语句就是

```
Get-WinEvent -LogName Application -FilterXPath '*/System/EventID=16384'
```

![image-20261113112721617](/Levi38_blog/images/win-log-analysis/image-20261113112721617.png)

如果用wevtutil 查询语句就是

```
wevtutil.exe qe Application /q:*/System[EventID=16384] /f:text /c:1
```

![image-20261113112835271](/Levi38_blog/images/win-log-analysis/image-20261113112835271.png)

如果要根据提供者进行过滤，就需要使用 `Provider` 中的 `@Name` 属性 语句有所不同

```
 Get-WinEvent -LogName Application -FilterXPath '*/System/Provider[@Name="Microsoft-Windows-Security-SPP"]'
```

也可以合并两个查询语句

```
Get-WinEvent -LogName Application -FilterXPath '*/System/EventID=16384 and */System/Provider[@Name="Microsoft-Windows-Security-SPP"]'
```

如果要根据EventData中的属性查询

![image-20261113115335476](/Levi38_blog/images/win-log-analysis/image-20261113115335476.png)

```
Get-WinEvent -LogName Security -FilterXPath '*/EventData/Data[@Name="TargetUserName"]="System"'
```

以下是一些示例：

**找出系统时间为 2020-12-15T01:09:08.940277500Z 的 WLMS 事件**

```
Get-WinEvent -LogName Application -FilterXPath '*/System/Provider[@Name="WLMS"] and */System/TimeCreated[@SystemTime="2020-12-15T01:09:08.940277500Z"]'
```

**查询用户名为Sam有 Logon 事件 ID 为 4720 的事件**

注意：4720 事件（创建用户）属于安全日志，所以 `-LogName` 必须是 `Security`

```
Get-WinEvent -LogName Security -FilterXPath '*/EventData/Data[@Name="TargetUserName"]="Sam" and */System/EventID=4720'
```

## 事件 ID

通过具体的id我们可以查询到对应的事件类型

![image-20261113115832702](/Levi38_blog/images/win-log-analysis/image-20261113115832702.png)

![image-20261113115840782](/Levi38_blog/images/win-log-analysis/image-20261113115840782.png)

![image-20261113115850387](/Levi38_blog/images/win-log-analysis/image-20261113115850387.png)

![image-20261113115858854](/Levi38_blog/images/win-log-analysis/image-20261113115858854.png)

## 练习题

接下来的情景和问题都是基于位于桌面上的名为 `merged.evtx` 的外部事件日志文件

**What event ID is to detect a PowerShell downgrade attack?**

id400记录 PowerShell 引擎版本，用于检测降级攻击

**What is the Date and Time this attack took place? (MM/DD/YYYY H:MM:SS [AM/PM])**

可以使用Get-WinEvent语句查询 也可以直接用事件查看器进行筛选

```
Get-WinEvent -FilterHashtable @{path='merged.evtx';id=400} | Foreach-Object {$version = [Version] ($_.Message -replace '(?s).*EngineVersion=([\d\.]+)*.*','$1'); if($version -lt ([Version] "5.0")) { $_ } }
```

![image-20261113151258016](/Levi38_blog/images/win-log-analysis/image-20261113151258016.png)

答案 12/18/2020 7:50:33 AM

**A Log clear event was recorded. What is the 'Event Record ID'?**

已清除的日志事件可以通过两种不同的事件 ID 来查询 安全日志的 EID 1102 所有其他日志的 EID 均为 104

```
Get-WinEvent -FilterHashtable @{path='merged.evtx';id=104} | Select-Object -Property RecordId, TimeCreated, Message
```

或者用事件查看器

![image-20261113151553861](/Levi38_blog/images/win-log-analysis/image-20261113151553861.png)

答案27736

**What is the name of the computer?**

```
Get-WinEvent -FilterHashtable @{path='merged.evtx'} -MaxEvents 20 -Oldest | Select-Object -Property MachineName
```

![image-20261113151843106](/Levi38_blog/images/win-log-analysis/image-20261113151843106.png)

在每个事件的MachineName字段可以找到答案 PC01.example.corp

**What is the name of the first variable within the PowerShell command?**

根据PowerShell命令可以锁定id在4104

```
Get-WinEvent -FilterHashtable @{path='merged.evtx';id=4104} -Oldest -MaxEvents 1 | Format-Table -AutoSize -Wrap
```

![image-20261113152042649](/Levi38_blog/images/win-log-analysis/image-20261113152042649.png)

答案 $Va5w3n8

**What is the Group Security ID of the group she enumerated?**

根据微软官方文档，事件 **4799** 的触发条件就是：**当一个进程枚举计算机或设备上启用了安全性的本地组成员时**生成

![image-20261113152627371](/Levi38_blog/images/win-log-analysis/image-20261113152627371.png)

 题目还说了调用该事件的进程是C:\Windows\System32\net1.exe

所以我们可以用CallerprocessName和EventID来确定

```
Get-WinEvent -Path "merged.evtx" -FilterXPath '*/EventData/Data[@Name="CallerProcessName"]="C:\Windows\System32\net1.exe" and */System[EventID=4799]' | Format-List
```

![image-20261113152828800](/Levi38_blog/images/win-log-analysis/image-20261113152828800.png)

答案 S-1-5-32-544
