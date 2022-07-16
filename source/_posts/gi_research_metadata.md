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

## 分析 UserAssembly.dll

于是把 UserAssembly.dll 丢进 ida 分析了下。直接搜索字符串 global-metadata，找到了看起来可能是打开文件的地方：

![疑似打开文件](https://s2.loli.net/2022/06/10/ZbP5GaLAdsYv2D9.png)

于是我参考了网上相关文章研究了下，怀疑到下面有一处调用的解密函数，这个函数初始化在一个导出函数中，其中还初始化了另一个函数。

![初始化解密函数](https://s2.loli.net/2022/06/10/E4GAKo6kPxJReO8.png)

不过解密函数目前来看都不在这个 dll 中，四处翻了翻后我就暂时结束了对 UserAssembly.dll 的探索。

## 分析 Il2CppDumper

接下来我把目光转向了 Il2CppDumper，想通过它分析下之前版本的 metadata 是怎么解密出来的（没找到这个被修改过的 Il2CppDumper 源码）（更新：其实是有源码的，在另一个[仓库](https://github.com/Three-taile-dragon/MelonLoader-GenshinImpact)）。于是把他丢进 ida，搜索打印的 log 字符串 "Initializing metadata..."，很快我就发现了一个叫做 DecryptMetadata 的函数，显然是解密函数：

![DecryptMetadata](https://s2.loli.net/2022/06/10/SQeip7WVCR6atUl.png)

ida 好像反汇编不了 .net 的程序，换个反编译工具 dnSpy，这下源码基本完全解析出来了。首先动态调试了下，找到异常抛出的位置向前回溯，发现问题出自 header.metadataUsageListsCount 为 0，这个 header 又是通过之前解密的 metadata 数据初始化出来的，这样看 metadata 果然还是没有被正确解密。

```csharp
public static MetadataDecryption.StringDecryptionData DecryptMetadata(byte[] metadata)
{
	MetadataDecryption.DecryptMetadataBlocks(metadata);
	return MetadataDecryption.DecryptMetadataStringInfo(metadata);
}
```

细看第一个解密函数 DecryptMetadataBlocks，先是从文件后部复制了点数据，之后做了个比较。奇怪的是这个比较通过了，我又在二进制编辑器里对了下这个值，确实是对的，看来加密逻辑没有变动太多？

```csharp 第一个解密函数
private static void DecryptMetadataBlocks(byte[] metadata)
  {
   byte[] array = new byte[16384];
   Buffer.BlockCopy(metadata, metadata.Length - array.Length, array, 0, array.Length);
   if (array[200] != 46 || array[201] != 252 || array[202] != 254 || array[203] != 44)
   {
    throw new ArgumentException("*((uint32_t*)&footer[0xC8]) != 0x2CFEFC2E");
   }
```

后面主要就是数据复制，用异或方法压缩出了一个 key，之后又异或了一堆奇怪的预定义的值。看来之前加密的逻辑主要还是异或，还用到了一些预先定义好的数据。到这里感觉从 Il2CppDumper 也不再能看出太多了。

## 搭建主程序调试环境

推测下解密函数很有可能应该在主程序 YuanShen.exe 里面了，研究了下找到个方法能通过调试器直接启动游戏本体进私服，方便之后调试。（更新：其实只要用的是 Fiddler 这种抓包软件就行了，效果一样且更方便）

修改 GrassClipper 的 scripts/private_server_launch.cmd，找到这里：

```
:: Launch game
"%GAME_PATH%"
```

把 `"%GAME_PATH%"` 这部分注释掉，加个 pause。之后启动先把 grasscutter 服务端跑起来，然后在命令行运行这个脚本，带上下面这些参数：

```bash
private_server_launch.cmd 127.0.0.1 443 true "<YuanShen.exe 路径>" "<GrassClipper 路径>" false true
```

这里服务地址端口都是默认的配置，之后这个脚本会把代理开起来，接下来直接运行游戏本体就能进私服了，关闭记得在终端输入字符把剩下的关闭流程跑完，否则代理设置不会被清除掉没法正常联网。

搭好调试环境后，我发现这游戏主程序加了壳的没法直接调试（废话），想在游戏过程中挂 x64dbg 也挂不上，不知道用了什么魔法。另外游戏会不断检查并尝试杀掉 x64dbg 等调试器，启动时也会检查，以及如果杀不掉甚至游戏直接退出。所以这个部分遇到了点困难。

![本体有 vmp 壳](https://s2.loli.net/2022/06/10/JWm9QEqiTSVgLX3.png)

## 尝试新的 Il2CppDumper

回来更新文章了，发现已经有人更新了针对 2.7+ 版本的 Il2CppDumper^[[2.7+ 版本的 Il2CppDumper](https://github.com/1582421598/Il2CppDumper-Genshin)]，所以尝试直接用它替换掉原本的 Il2CppDumper。

运行后 Il2CppDumper 成功生成了 DummyDll，但是还是有其他错误，下面是比较关键的部分：

```
[01:25:26.036] ------------------------------
[01:25:26.037] Game Name: 原神
[01:25:26.037] Game Developer: miHoYo
[01:25:26.038] Unity Version: 2017.4.30f1
[01:25:26.038] Game Version:
[01:25:26.039] ------------------------------
[01:25:26.379] Preferences Loaded!
[01:25:26.398] [Il2CppUnityTls] Patching mono_unity_get_unitytls_interface...
[01:25:26.401] Loading Plugins...

[01:25:26.409] ------------------------------
[01:25:26.409] No Plugins Loaded!
[01:25:26.410] ------------------------------
[libil2cpp] Failed to resolve 3441514353 at startup
[libil2cpp] Failed to resolve 2641773275 at startup
[libil2cpp] Failed to resolve 3753878500 at startup
[libil2cpp] Failed to resolve 1688788800 at startup
[libil2cpp] Failed to resolve 226400704 at startup
[libil2cpp] Failed to resolve 58785463 at startup
[libil2cpp] Failed to resolve 1939797683 at startup
[libil2cpp] Failed to resolve 2903149294 at startup
...

[00:50:23.258] "C:\Users\Miguel\Downloads\gi-priv\games\gi_2.7.0_self\Genshin Impact Game\MelonLoader\Dependencies\Il2CppAssemblyGenerator\Il2CppAssemblyUnhollower\AssemblyUnhollower.exe" "--input=C:\Users\Miguel\Downloads\gi-priv\games\gi_2.7.0_self\Genshin Impact Game\MelonLoader\Dependencies\Il2CppAssemblyGenerator\Il2CppDumper\DummyDll" "--output=C:\Users\Miguel\Downloads\gi-priv\games\gi_2.7.0_self\Genshin Impact Game\MelonLoader\Dependencies\Il2CppAssemblyGenerator\Il2CppAssemblyUnhollower\Managed" "--mscorlib=C:\Users\Miguel\Downloads\gi-priv\games\gi_2.7.0_self\Genshin Impact Game\MelonLoader\Managed\mscorlib.dll" "--unity=C:\Users\Miguel\Downloads\gi-priv\games\gi_2.7.0_self\Genshin Impact Game\MelonLoader\Dependencies\Il2CppAssemblyGenerator\UnityDependencies" "--gameassembly=C:\Users\Miguel\Downloads\gi-priv\games\gi_2.7.0_self\Genshin Impact Game\YuanShen_Data\Native\UserAssembly.dll" "--add-prefix-to=ICSharpCode" "--add-prefix-to=Newtonsoft" "--add-prefix-to=TinyJson" "--add-prefix-to=Valve.Newtonsoft" "--no-xref-cache"
[00:50:23.348] Reading assemblies... 
[00:50:23.538] Done in 00:00:00.1889345
[00:50:23.539] Reading system assemblies... 
[00:50:23.552] Done in 00:00:00.0138266
[00:50:23.553] Reading unity assemblies... 
[00:50:23.563] Done in 00:00:00.0100095
[00:50:23.564] Creating rewrite assemblies... 
[00:50:23.584] Done in 00:00:00.0218627
[00:50:23.585] Computing renames... 
[00:50:23.616] [ERROR] 
[00:50:23.657] [ERROR] 未经处理的异常:  Mono.Cecil.AssemblyResolutionException: Failed to resolve assembly: 'System.Private.CoreLib, Version=5.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e'
[00:50:23.658] [ERROR]    在 Mono.Cecil.BaseAssemblyResolver.Resolve(AssemblyNameReference name, ReaderParameters parameters)
[00:50:23.659] [ERROR]    在 Mono.Cecil.DefaultAssemblyResolver.Resolve(AssemblyNameReference name)
[00:50:23.659] [ERROR]    在 Mono.Cecil.MetadataResolver.Resolve(TypeReference type)
[00:50:23.660] [ERROR]    在 Mono.Cecil.TypeReference.Resolve()
[00:50:23.661] [ERROR]    在 AssemblyUnhollower.Passes.Pass05CreateRenameGroups.NameOrRename(TypeReference typeRef, RewriteGlobalContext context)
[00:50:23.662] [ERROR]    在 AssemblyUnhollower.Passes.Pass05CreateRenameGroups.GenericNameToStrings(TypeReference typeRef, RewriteGlobalContext context)
[00:50:23.663] [ERROR]    在 AssemblyUnhollower.Passes.Pass05CreateRenameGroups.GetUnobfuscatedNameBase(RewriteGlobalContext context, TypeDefinition typeDefinition, Boolean allowExtraHeuristics)
[00:50:23.664] [ERROR]    在 AssemblyUnhollower.Passes.Pass05CreateRenameGroups.ProcessType(RewriteGlobalContext context, TypeDefinition originalType, Boolean allowExtraHeuristics)
[00:50:23.665] [ERROR]    在 AssemblyUnhollower.Passes.Pass05CreateRenameGroups.ProcessType(RewriteGlobalContext context, TypeDefinition originalType, Boolean allowExtraHeuristics)
[00:50:23.665] [ERROR]    在 AssemblyUnhollower.Passes.Pass05CreateRenameGroups.DoPass(RewriteGlobalContext context)
[00:50:23.666] [ERROR]    在 AssemblyUnhollower.Program.Main(UnhollowerOptions options)
[00:50:23.667] [ERROR]    在 AssemblyUnhollower.Program.Main(String[] args)
[00:50:25.289] Done in 00:00:01.7042110


[00:50:25.370] Loading Mods...
[00:50:25.389] [ERROR] No MelonInfoAttribute Found in C:\Users\Miguel\Downloads\gi-priv\games\gi_2.7.0_self\Genshin Impact Game\Mods\ClassLibrary3.dll

...

[00:50:25.499] [ERROR] Field internalEncoding was not found on class StreamWriter
[00:50:25.500] [ERROR] Field internalStream was not found on class StreamWriter
[00:50:25.502] [ERROR] Field iflush was not found on class StreamWriter
[00:50:25.503] [ERROR] Field byte_buf was not found on class StreamWriter
[00:50:25.504] [ERROR] Field byte_pos was not found on class StreamWriter
[00:50:25.504] [ERROR] Field decode_buf was not found on class StreamWriter
[00:50:25.505] [ERROR] Field decode_pos was not found on class StreamWriter
[00:50:25.506] [ERROR] Field DisposedAlready was not found on class StreamWriter
[00:50:25.506] [ERROR] Field preamble_done was not found on class StreamWriter
[00:50:25.507] [ERROR] Field internalFormatProvider was not found on class TextWriter
```

这里放的 log 比较多，大部分内容在之前的 log 中也是有的，不同的部分在于 Il2CppDumper 生成 DummyDll 后接着又运行的 Il2CppAssemblyUnhollower 给出的错误。

从 trace 来看 AssemblyUnhollower 又用到了 Mono.Cecil 这个库，在该库中抛出了一个异常。除此之外开头部分的 libil2cpp 也还不清楚是谁打印的，这部分错误从一开始就存在，且没有被写入 log 文件中，可能比较特殊。

施工中...
