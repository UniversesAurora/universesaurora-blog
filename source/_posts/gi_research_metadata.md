---
title: 小探 GI global-metadata
date: 2022-06-10 20:07:34
updated: 2022-06-10 20:07:46
cover: https://s2.loli.net/2022/06/10/ztySZrhFvx9wQfN.jpg
thumbnail: https://s2.loli.net/2022/06/10/ztySZrhFvx9wQfN.jpg
categories:
- 逆向
tags:
- 逆向
- 游戏
toc: true
---

算是博客的第一篇正式文章。

最近尝试了一下原神私服，在弄模型替换的时候出了点问题，正好借此机会对游戏做一点探索。

<!-- more -->

## 起因

想要在游戏中进行模型替换需要一个叫做 Melonloader 的框架，但这个框架在 2.7 版本中无法使用了^[[该问题的 Github Issue](https://github.com/lassedds/Melonloader-AnimeGaming/issues/3)]。查看产生的 log 可以发现该问题出在 Il2CppDumper 上：

```
...
[20:57:40.389] Executing Il2CppDumper...
[20:57:40.392] "C:\Users\Miguel\Downloads\GrassCutPer\Genshin Impact\Genshin Impact Game\MelonLoader\Dependencies\Il2CppAssemblyGenerator\Il2CppDumper\Il2CppDumper.exe" "C:\Users\Miguel\Downloads\GrassCutPer\Genshin Impact\Genshin Impact Game\YuanShen_Data\Native\UserAssembly.dll" "C:\Users\Miguel\Downloads\GrassCutPer\Genshin Impact\Genshin Impact Game\YuanShen_Data\Native\Data\Metadata\global-metadata.dat"
[20:57:40.766] Initializing metadata...
[20:57:46.401] System.InvalidOperationException: 序列不包含任何元素
[20:57:46.401]    在 System.Linq.Enumerable.Max[TSource](IEnumerable`1 source)
[20:57:46.402]    在 Il2CppDumper.Metadata.<>c.<ProcessingMetadataUsage>b__38_0(KeyValuePair`2 x)
[20:57:46.403]    在 System.Linq.Enumerable.WhereSelectEnumerableIterator`2.MoveNext()
[20:57:46.404]    在 System.Linq.Enumerable.Max[TSource](IEnumerable`1 source)
[20:57:46.405]    在 Il2CppDumper.Metadata.ProcessingMetadataUsage()
[20:57:46.406]    在 Il2CppDumper.Metadata..ctor(Stream stream, StringDecryptionData decData, String nameTranslationPath)
[20:57:46.407]    在 Il2CppDumper.Program.Init(String il2cppPath, String metadataPath, String nameTranslationPath, Metadata& metadata, Il2Cpp& il2Cpp)
[20:57:46.408]    在 Il2CppDumper.Program.Main(String[] args)
[20:57:46.419] Executing Il2CppAssemblyUnhollower...
...
[20:57:46.516] [ERROR] 未经处理的异常:  System.IO.DirectoryNotFoundException: 未能找到路径“C:\Users\Miguel\Downloads\GrassCutPer\Genshin Impact\Genshin Impact Game\MelonLoader\Dependencies\Il2CppAssemblyGenerator\Il2CppDumper\DummyDll”的一部分。
...
```

经过一番搜索后，我发现这个 Il2CppDumper 实际上是被修改过的，原本的 Il2CppDumper 只能解析出 unity 正常生成的 global-metadata.dat，但游戏将 global-metadata.dat 文件进行了一些混淆，所以不将其解密是无法成功将元数据导出来的。Il2CppDumper 大概的作用就是把经过 Il2Cpp 转换后需要用到的符号还原出来并生成一系列 dll 文件，而 Melonloader 的运行应该需要用到这些 dll 文件。

这样看来应该是 2.7 版本的混淆方法有了变化，导致这个针对之前设计的 Il2CppDumper 没法正确解密了。虽然静等作者更新这个程序也不是不行，但刚好我之前有过探索下这个游戏背后的程序结构之类的想法，于是就打算自己分析分析看看能不能把这个解密方法修复下。

## 行为

用二进制编辑器看了下 metadata 文件，头部的标志已经没有了，和之前版本的对比了下也没看出来什么门道。我首先想到的是抓抓系统调用看看打开 metadata 文件的前后发生了些啥，然而抓了后又把 map/read 之类操作和 metadata 文件对了下，依然看不出什么东西，显然这种简单的分析是完全没用的。更诡异的是运行几次后连 read 都抓不到了，难不成这东西还有缓存的？

![Process Monitor 记录](https://s2.loli.net/2022/06/10/VgBNwJjQX9mWU82.png)

## UserAssembly.dll

于是把 UserAssembly.dll 丢进 ida 分析了下。

一测的metadata有mark已经不一样了
找到异常抛出点回溯一下，header.metadataUsageListsCount为0，发现还是DecryptMetadataBlocks有问题，头部读出的信息错误
DecryptMetadataBlocks开头的几个判断条件都是对的啊，看起来文件可能只被微调了
以异或为主要逻辑加一点偏移
上调试看看原神怎么算的 忘记了原神有壳 没法动态调试了 不熟

```bash
private_server_launch.cmd 127.0.0.1 443 true "<YuanShen.exe 路径>" "<GrassClipper 路径>" false true
```
