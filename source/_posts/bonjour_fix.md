---
title: Bonjour 不 Bon：修复 Bonjour 中存在的 IPv6 栈溢出 bug
date: 2024-09-30 07:15:16
updated: 2024-09-30 07:15:16
cover: https://cdn.pixabay.com/photo/2023/08/08/21/25/france-8178164_1280.jpg
thumbnail: https://cdn.pixabay.com/photo/2023/08/08/21/25/france-8178164_1280.jpg
categories:
- 逆向
tags:
- 网络
- 逆向
- 代理
- Bonjour
toc: true
---

闲话少说，如果你在使用基于由 Apple 提供的 Bonjour 协议的服务（例如通过网络刷新 AltStore 应用，iTunes WiFi 同步等）时遇到无法正常连接的问题，并且查看 Windows 事件查看器的 Windows 日志中发现此应用程序错误（异常代码和错误偏移量一致）：

<!-- more -->

```sh
错误应用程序名称: mDNSResponder.exe，版本: 3.1.0.1，时间戳: 0x55cbcce6
错误模块名称: mDNSResponder.exe，版本: 3.1.0.1，时间戳: 0x55cbcce6
异常代码: 0xc0000409
错误偏移量: 0x00000000000437c3
错误进程 ID: 0x0x9BA8
错误应用程序启动时间: 0x0x1DB1204029AD2A1
错误应用程序路径: C:\Program Files\Bonjour\mDNSResponder.exe
错误模块路径: C:\Program Files\Bonjour\mDNSResponder.exe
报告 ID: 97706ac4-91ee-484e-a851-e12e37433bd0
错误程序包全名: 
错误程序包相对应用程序 ID: 
```

那么[下载这个](https://1drv.ms/u/s!AiqwR6LDRXHbkytc6emn6FL0zUr3?e=rufbKu)修复后的 `mDNSResponder.exe`，替换掉 `C:\Program Files\Bonjour\mDNSResponder.exe`，然后去服务里重启 Bonjour 服务即可。如果是 AltStore 刷新不了的问题，还需要在这之后重启 Altserver。

# 问题背景

之前我在 Windows 上用代理软件一般不打开系统代理等设置，只通过 port 访问，然后通过 proxifier 或者手动设置 HTTP 代理（一般是命令行）方式访问代理。然而这种方式会带来一些问题：最主要就是有的软件用 HTTP 连不上用 SOCKS 却可以，或者反过来的情况，这就导致规则很难设置；还有就是有的规则要用全局有点规则要用分流导致要同时开两个代理软件等等。proxifier 有时候又会出现莫名奇妙地无法代理的问题，然后它这个软件原理又比较奇妙很难查明原因（不是用的系统设置中的代理设置，也没有创建网卡，感觉更像是注入），比如我最近代理 VRChat 的时候就有时候能用有时候用不了，查了半天也找不到原因...

之所以一直用着是一是因为我希望 Windows 上的软件尽量不走代理，我大部分的代理需求都来自浏览器，已经被 SmartProxy 这类扩展解决了，proxifier主要还是解决 Discord 这类不用代理就连不上的软件，或者 GitHub 这类域名等；二是因为我的系统上还有 Tailscale、Altserver 之类对本地网络由依赖的软件，我不太清楚用 TUN 这类方式会不会对它们造成干扰。

不过最近捣鼓 iOS 代理软件让我开始觉得 TUN 这种网络层代理才是更完美的代理方法，所以最终还是把 Windows 也换成了 TUN 代理模式。我目前用的代理软件是 Mihomo Party，内核是 Mihomo。

没过几天我就发现 AltStore 刷新软件时找不到 Altserver，Altserver 中也不显示 iOS 设备了。很快我就意识到这很可能是我前几天弄成 TUN 代理导致的。

我尝试关闭代理软件，这会删除创建的 TUN 网卡，然后重启 Altserver，但很奇怪的并没有作用。之前我遇到几次 Altserver 连接问题发现通过重启 Apple Mobile Device Service 服务可以修复，然而我尝试了很多次也没有作用。

然后我找到了[这个 issue](https://github.com/clash-verge-rev/clash-verge-rev/issues/1445)，于是尝试关闭代理自启然后重启，此时连接恢复正常。但打开代理软件后，刚开始 AltStore 还可以正常刷新，过一会之后就又连不上了。

# 抓包分析

因为 AltServer 通过网络刷新应用需要通过 iTunes 开启 iOS 设备的“通过Wi-Fi 与此设备同步”选项，我知道它的工作原理其实是通过 Bonjour 协议发现设备。猜想是不是 Bonjour 协议在开了 TUN 之后不走物理网卡设备，或者 TUN 向 物理网卡转发导致了问题（我对 TUN 代理的工作原理不太熟悉，所以有很多猜想并不合理）。总之我打算用 Wireshark 抓包看下具体情况。

首先要说的是 Apple 的  Bonjour 协议本质上就是 mDNS（Multicast DNS）协议，走 UDP 的5353端口。当 mDNS 客户端需要解析主机名时，它会发送一条 IP 多播查询消息，要求具有该名称的主机识别自己的身份。然后，该目标机器多播一条包含其 IP 地址的消息。[^1]

我先在 Bonjour 正常工作的情况下抓了下包，mDNS 在 IPv4/v6 下面都会工作，但目前我先只看 IPv4 的情况，于是我设置过滤规则 `mdns && ip.addr == 192.168.2.100 || mdns && ip.addr == 192.168.2.16`，`192.168.2.100` 是电脑的 IP，`192.168.2.16` 是 iOS 设备的 IP。

很快我就发现了目标，一些和 Altserver 有关的 mDNS 查询和回复：

```
Internet Protocol Version 4, Src: 192.168.2.16, Dst: 224.0.0.251
User Datagram Protocol, Src Port: 5353, Dst Port: 5353
Multicast Domain Name System (query)
    Transaction ID: 0x0000
        [Expert Info (Warning/Protocol): DNS response missing]
    Flags: 0x0000 Standard query
    Questions: 1
    Answer RRs: 0
    Authority RRs: 0
    Additional RRs: 0
    Queries
        MDDPC._altserver._tcp.local: type SRV, class IN, "QU" question
            Name: MDDPC._altserver._tcp.local
            [Name Length: 27]
            [Label Count: 4]
            Type: SRV (33) (Server Selection)
            .000 0000 0000 0001 = Class: IN (0x0001)
            1... .... .... .... = "QU" question: True

Internet Protocol Version 4, Src: 192.168.2.100, Dst: 224.0.0.251
User Datagram Protocol, Src Port: 5353, Dst Port: 5353
Multicast Domain Name System (response)
    Transaction ID: 0x0000
        [Expert Info (Warning/Protocol): DNS response retransmission. Original response in frame 21609]
    Flags: 0x8400 Standard query response, No error
    Questions: 0
    Answer RRs: 1
    Authority RRs: 0
    Additional RRs: 7
    Answers
        MDDPC._altserver._tcp.local: type SRV, class IN, cache flush, priority 0, weight 0, port 1382, target MDDPC.local
    Additional records
        MDDPC.local: type A, class IN, cache flush, addr 192.168.2.100
        MDDPC.local: type AAAA, class IN, cache flush, addr 2001:0db8:85a3:0000:0000:8a2e:0370:7334
        MDDPC.local: type AAAA, class IN, cache flush, addr 2001:0db8:85a3:0000:0000:abcd:ef12:3456
        MDDPC.local: type AAAA, class IN, cache flush, addr 2001:0db8:85a3:0000:1234:5678:9abc:def0
        MDDPC.local: type AAAA, class IN, cache flush, addr fe80::1a2b:3c4d:5e6f:7890
        MDDPC.local: type NSEC, class IN, cache flush, next domain name MDDPC.local
        MDDPC._altserver._tcp.local: type NSEC, class IN, cache flush, next domain name MDDPC._altserver._tcp.local
    [Retransmitted response. Original response in: 21609]
    [Retransmission: True]
```

首先可以看到这两个报文的目的地址都是 `224.0.0.251`，这是固定的 mDNS 多播 IP 地址。iOS 设备向我的计算机（MDDPC）请求了 `_altserver` 这一 TCP 服务。而应答中返回了该服务的端口为 1382，通过 `MDDPC.local` 这一域名访问，同时在 `Additional records` 中将该域名对应的 IP 均列了出来。这就是 mDNS 大致的工作原理了。

接下来我在开启 TUN 的情况下进行抓包，由于此时多了一个 TUN 网卡，所以我分别同时在物理网卡和 TUN 网卡上同时进行抓包。

我在 TUN 网卡上确实抓到了一些 mDNS 包，但是没有和 Altserver 有关的包，而且都是计算机自己发送的，没有其他人响应。因为其实 TUN 网卡和物理网卡不在一个网段上，如下，物理网卡网段为 192.168，TAP 网卡为 198.18：

```
未知适配器 Mihomo:

   连接特定的 DNS 后缀 . . . . . . . :
   描述. . . . . . . . . . . . . . . : Meta Tunnel
   物理地址. . . . . . . . . . . . . : AA-BB-CC-DD-EE-FF
   DHCP 已启用 . . . . . . . . . . . : 否
   自动配置已启用. . . . . . . . . . : 是
   IPv6 地址 . . . . . . . . . . . . : 2001:0db8:abcd::1(首选)
   IPv4 地址 . . . . . . . . . . . . : 198.18.0.1(首选)
   子网掩码  . . . . . . . . . . . . : 255.255.255.252
   默认网关. . . . . . . . . . . . . : ::
                                       0.0.0.0
   DNS 服务器  . . . . . . . . . . . : 2001:0db8:abcd::2
                                       198.18.0.2
   TCPIP 上的 NetBIOS  . . . . . . . : 已启用

以太网适配器 以太网:

   连接特定的 DNS 后缀 . . . . . . . :
   描述. . . . . . . . . . . . . . . : Intel(R) Ethernet Controller I226-V
   物理地址. . . . . . . . . . . . . : 11-22-33-44-55-66
   DHCP 已启用 . . . . . . . . . . . : 否
   自动配置已启用. . . . . . . . . . : 是
   IPv6 地址 . . . . . . . . . . . . : 2001:0db8:abcd::100(首选)
   获得租约的时间  . . . . . . . . . : 2024年9月28日 21:37:00
   租约过期的时间  . . . . . . . . . : 2024年9月30日 6:30:44
   IPv6 地址 . . . . . . . . . . . . : 2001:0db8:abcd::200(首选)
   临时 IPv6 地址. . . . . . . . . . : 2001:0db8:abcd::300(受到抨击)
   临时 IPv6 地址. . . . . . . . . . : 2001:0db8:abcd::400(首选)
   本地链接 IPv6 地址. . . . . . . . : fe80::1a2b:3c4d:5e6f:7890%21(首选)
   IPv4 地址 . . . . . . . . . . . . : 192.168.2.100(首选)
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : fe80::8ede:f9ff:fead:6d81%21
                                       192.168.2.1
   DHCPv6 IAID . . . . . . . . . . . : 554221496
   DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-2C-28-7A-02-11-22-33-44-55-66
   DNS 服务器  . . . . . . . . . . . : 2001:0db8::1
                                       2001:0db8::2
                                       119.29.29.29
                                       223.5.5.5
   TCPIP 上的 NetBIOS  . . . . . . . : 已启用
```

在物理网卡上我抓到了来自 iOS 对 Altserver 的 mDNS 查询请求，但计算机并没有响应。

# 谁是 Bonjour？

到这里我想看下到底是谁在监听5353端口，通过 `netstat -abn` 我发现监听 UDP 5353端口的程序不止一个，不太确定 Apple 的 Bonjour 由谁回复：

```sh
 [svchost.exe]
  UDP    0.0.0.0:5353           *:* 
 [chrome.exe]
  UDP    0.0.0.0:5353           *:*   
 [svchost.exe]
  UDP    192.168.2.100:5353     *:*                    
 [nvcontainer.exe]
  UDP    192.168.2.100:5353     *:*
...
```

于是我直接打开 Process Hacker 搜索 Bonjour，很快发现了一个叫做 `mDNSResponder.exe` 的程序，父进程是 `services.exe` ，他就是 Bonjour 服务。路径是 `C:\Program Files\Bonjour\mDNSResponder.exe`。

![Bonjour 服务](https://s2.loli.net/2024/09/29/Hw4MRCloYxhWNjD.png)

打开 TUN 代理后，我发现 `mDNSResponder.exe` 很快就退出了。`C:\Program Files\Bonjour\` 目录下也没有发现有 log。因此我就去系统日志中寻找，于是找到了文章开头的错误日志。

![错误日志](https://s2.loli.net/2024/09/29/kjrK9usXtPmbS1v.png)

看起来 `mDNSResponder.exe` 并非正常退出，否则一般也不会触发日志记录，并且是应用程序崩溃事件。这也解释了为什么关闭代理也无法恢复正常：Bonjour 服务并不会自动重启。在我关闭代理并手动启动 Bonjour 服务和 Altserver 后，果然一切恢复了正常。搜了下网上还有其他人也遇到过相同的错误：[Bonjour Service Crashes when Eddie Active - Off-Topic - AirVPN](https://airvpn.org/forums/topic/51827-bonjour-service-crashes-when-eddie-active/)

另外我发现 Clash for Windows 的 TUN 代理并不会导致 Bonjour 服务崩溃，这说明问题很可能并不出在 TUN 本身上。我又尝试更改 TUN 模式的设置，比如设置堆栈，自动全局路由，严格路由，DNS 劫持等等，但不管怎么改 `mDNSResponder.exe` 都会崩溃。

# 逆向时间

为了搞明白问题出在哪里，我准备上 x64dbg 上分析一下。不过 `mDNSResponder.exe` 默认情况下是以服务程序模式执行的，没办法直接执行：

```powershell
PS C:\Users\Fusion\Downloads> C:\'Program Files'\Bonjour\mDNSResponder.exe
start service dispatcher failed (1063)
```

不过好在它提供了选项 `-server` 可以用于直接执行：

```powershell
mDNSResponder 1.0d1

    <no args>    Runs the service normally
    -install     Creates the service and starts it
    -remove      Stops the service and deletes it
    -start       Starts the service dispatcher after processing all other arguments
    -server      Runs the service directly as a server (for debugging)
    -q           Toggles Quiet Mode (no events or output)
    -remote      Allow remote connections
    -cache n     Number of mDNS cache entries (defaults to 512)
    -h[elp]      Display Help/Usage
```

在 x64dbg 中添加参数 `-server` 后，就可以正常调试了。我发现它启动后很快触发了一个异常：

```
EXCEPTION_DEBUG_INFO:  
           dwFirstChance: 1  
           ExceptionCode: C0000005 (EXCEPTION_ACCESS_VIOLATION)  
          ExceptionFlags: 00000000  
        ExceptionAddress: mdnsresponder.[00007FF64BE59D37](x64dbg://localhost/address64#00007FF64BE59D37)  
        NumberParameters: 2  
ExceptionInformation[00]: [0000000000000001](x64dbg://localhost/address64#0000000000000001) Write  
ExceptionInformation[01]: [0000000000150000](x64dbg://localhost/address64#0000000000150000) Inaccessible Address  
第一次异常于 [00007FF64BE59D37](x64dbg://localhost/address64#00007FF64BE59D37) (C0000005, EXCEPTION_ACCESS_VIOLATION)

# 出错的指令
00007FF64BE59D37 | 41:8810                 | mov byte ptr ds:[r8],dl                      | r8:"Actx "
```

mov 指令访问了一个不可访问的地址，这个地址就是 r8 寄存器里的 0x150000，mov 试图将 dl 寄存器的内容写到这个地址。

这个地址是哪里呢，查看内存布局，可以看到该地址正好位于堆栈上方的只读内存段的开头。结合汇编和在 x64dbg 下的调试，这大概率是一个栈溢出错误。

![内存布局](https://s2.loli.net/2024/09/30/hnYrCtyGOU4N6mv.png)

于是打开 IDA 开始分析，找到出错位置的伪代码如下：

```c
if ( memcmp(src, sa_family, i->Address.iSockaddrLength) )
  {
    stack_mem_to_set_ff = 0i64;
    v56 = 0i64;
    stack_addr = &stack_mem_to_set_ff;
    do
    {
      if ( PrefixLength < 8 )
        val_ff = -1 << (8 - PrefixLength);
      else
        val_ff = -1;
      *(_BYTE *)stack_addr = val_ff;// crash here
      stack_addr = (__int64 *)((char *)stack_addr + 1);
      PrefixLength -= 8;
    }
    while ( PrefixLength );
```

上面代码已经被我解析过，比较容易能看到问题原因：`PrefixLength` 在非8的倍数时 while 循环永远不会结束，并且 `PrefixLength` 是一个 unsigned int，所以会把 `stack_mem_to_set_ff` 所在栈位置以上内存填充 `FF`，直到超过栈范围写入只读地址导致崩溃。

那么 `PrefixLength` 又是什么？这就要提到这部分代码开头调用的 `GetAdaptersAddresses` API了。

```c
AdaptersAddresses = GetAdaptersAddresses(
                          0,
                          0x3Eu,                // 返回单播地址,返回此适配器上的 IP 地址前缀列表,返回默认网关的地址等
                          0i64,                 // Reserved
                          pLinkList_of_IP_ADAPTER_ADDRESSES,
                          &Size);

pFirstPrefix = pLinkList_of_IP_ADAPTER_ADDRESSES_unwind2->FirstPrefix;

IPHLPAPI_DLL_LINKAGE ULONG GetAdaptersAddresses(
  [in]      ULONG                 Family,
  [in]      ULONG                 Flags,
  [in]      PVOID                 Reserved,
  [in, out] PIP_ADAPTER_ADDRESSES AdapterAddresses,
  [in, out] PULONG                SizePointer
);
```

这个函数的定义具体见：[GetAdaptersAddresses function (iphlpapi.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/iphlpapi/nf-iphlpapi-getadaptersaddresses)，简单来说它会返回系统中所有网卡的详细信息。其中 `0x3E` 为 Flags 参数，其中有一位是 `GAA_FLAG_INCLUDE_PREFIX`，表示要求系统返回 IPv6 和 IPv4 地址的 IP 地址前缀。所有程序调用这个 API 返回的内容应该是一样的，所以我在 Visual Studio 中写了个测试程序调用该 API 来 debug。`PIP_ADAPTER_ADDRESSES->FirstPrefix` 中保存的就是这些前缀了，对于 mihomo TUN 内容如下：

![mihomo TUN 的 FirstPrefix 结构](https://s2.loli.net/2024/09/30/aDZ98hQgJ6LxS4w.png)

可以看到所有 Prefix 通过 next 链接在一起，第二层 `PrefixLength` 为 `0x7e` 的 Prefix 就是问题所在，它并非8的倍数。

另外观察 `FirstPrefix->Address.lpSockaddr->sa_family` 成员，值为 `0x2` 代表 IPv4，`0x17` 代表 IPv6。`lpSockaddr` 这个成员是一个指向 [`sockaddr` 结构体](https://learn.microsoft.com/en-us/windows/win32/winsock/sockaddr-2)的指针，这个结构体长度根据所选协议不同而有所不同，具体长度储存在 `Address.iSockaddrLength`中，但第一个成员永远是地址族。

网卡 IP 地址储存在 `PIP_ADAPTER_ADDRESSES->FirstUnicastAddress` 中，其结构和 Prefix 类似，由于网卡可能有多个 IP 地址，其也有多个 `sockaddr` 结构，通过链表组织在一起。

整体代码的逻辑是遍历所有网卡->遍历网卡的所有 IP 地址->遍历网卡所有地址前缀。其作用为找到所有网卡的所有IP对应的子网前缀。出错代码位于遍历网卡所有地址前缀的逻辑内，并且仅在 IP 地址为 IPv6 时才会执行，功能为生成前缀对应的掩码来计算子网前缀。

这也就解释了为什么 Clash for Windows 的 TAP 模式不会崩溃，因为即便打开了 IPv6 功能，它的 TAP 网卡依然没有 IPv6 地址，因此就不会执行到错误处。下图为 Clash TAP 的 `FirstUnicastAddress`，可以看到只有一个 IPv4 类型的地址：

![Clash 只有 IPv4 地址](https://s2.loli.net/2024/09/30/eOukEG253aK6MNi.png)

那么如果关闭 mihomo 的 IPv6 功能呢？我测试了一下也不会报错，但查看网卡仍然有 IPv6 地址，此时查看所有 IPv6 Prefix 发现不再有非8倍数的 `PrefixLength` 了。另外物理网卡自动分配的 Prefix 长度也均为8的倍数。

问题的解决方案也很简单，只要将生成掩码的循环的判断条件从不为0修改为大于0即可。对应到汇编代码即将下图的 `jne` 指令修改为 `jg`，对应到程序文件就是将 `0x9140` 偏移处的 `75` 修改为 `7F`。

![崩溃处汇编指令](https://s2.loli.net/2024/09/30/Zuex8yAXFPf7js6.png)

![修补文件对比](https://s2.loli.net/2024/09/30/ILtHKNnBpeDOgT8.png)

# 后话

修好这个问题后我又去找了下，Bonjour 的源码其实已经被 Apple 公开了，感兴趣的可以去看下：

[mDNSResponder/mDNSWindows at main · apple-oss-distributions/mDNSResponder · GitHub](https://github.com/apple-oss-distributions/mDNSResponder/tree/main/mDNSWindows)

本篇文章分析的代码对应的是 mDNSWin32.c 中的 `getifaddrs_ipv6`。该库中最新代码已经修复了这个问题。然而我并不知道这个修复的版本被用在了哪里，Bonjour 本身似乎并没有独立的安装包，我系统中的 Bonjour 是安装非 Windows 商店版本的 iTunes 时自带的（因为 Altserver 不支持商店版 iTunes）。也许是用在了现在商店版本的“Apple 设备”这个软件中？

另外，我还尝试构建了下这个最新的开源版本，很不幸并没有构建成功。如果可以的话还想试下用最新版本的 Bonjour 替换下，现在这个版本修改日期还是15年的，算下来都快10年了（天呐）。

那么就到这里吧，这些工作都是我一晚上肝出来的，说实话开始时没想到会走到通过逆向把它修好这步，感觉学到很多。

# 参考

[^1]: [Multicast DNS - Wikipedia](https://en.wikipedia.org/wiki/Multicast_DNS)
