---
title: Apple Clangd 掉坑记
date: 2023-05-19 15:51:21
updated: 2023-05-19 20:33:45
cover: https://cdn.pixabay.com/photo/2023/05/15/15/33/landscape-7995369_1280.jpg
thumbnail: https://cdn.pixabay.com/photo/2023/05/15/15/33/landscape-7995369_1280.jpg
categories:
- 杂谈
tags:
- vscode
- clangd
- llvm
toc: true
---

clangd 是 llvm 项目推出的 C/C++ 语言服务器，通过 LSP（Language Server Protocal）协议向编辑器如 vscode/vim/emacs 提供语法补全、错误检测、跳转、格式化等等功能。据说是比 vscode 自己的 IntelliSense 更好一些，我开始用 clangd 的时候 IntelliSense 貌似还不支持使用 `compile_commands.js` 文件辅助代码分析，因此并没有具体对比过两者的差异。

macOS 上附带的 llvm 是 Apple 自己管理的，属于 Xcode 的一部分，这个版本其实是落后于主线版本很多的。理论上来说这种等待上游稳定后再选择使用的版本稳定性更好，问题会比较少，然而意外的这次在更新了 Xcode 后 clangd 就翻车了。

<!-- more -->

我个人比较喜欢在 vscode 上配合 clangd 插件阅读内核代码（虽然在工作时只能用 vim + ctags 这种古朴的方式，不过 tag 文件确实更灵活些，尤其是需要看不同架构的代码实现时）。为了获得更精确的分析，一般我会在 make 时配合 bear 生成 `compile_commands.js` 文件，用于辅助 clangd 语法分析。这样几乎所有符号都能正确解析，对分析代码有很大的帮助。并且分析过程不需要编译，可以把 `compile_commands.js` 文件稍微修改一下，拿到其他安装了 llvm 的平台上也可以工作的很好。

前几天 Xcode 在后台自动更新了，今天打开 vscode 时顺手更新了下 vscode，重新启动后 clangd 提供的代码分析功能就完全坏掉了。

一开始 clangd 一直在报 `Error while reading shard xxx.c: wrong version: want 17, got 16` 这个错。我猜测可能是更新了 Xcode 后缓存的某些文件的版本和当前不符导致的，于是删除了项目目录下的 `.cache` `.tmp_*` 之类的文件，然后这个报错就消失了，但是 clangd 插件还是报告崩溃（我也很崩溃...

打开输出窗口看了下 clangd 插件的日志，报了一个 `Signalled during AST worker action: InlayHints` 错误，看起来是被某个信号中断了（其实当时通过这个大概能推测到是某个 bug 导致访问了非法内存地址）。这里 InlayHints 其实指的就是 vscode 中指针处在符号上方后会跳出来的悬浮嵌入提示，也就是 clangd 插件向 clangd server 请求提示信息时发生的错误。

![错误输出](https://s2.loli.net/2023/05/19/FQ1HDPUSep2EaIw.png)

Google 了下只找到了一个似乎比较符合目前情况的 [issue](https://github.com/clangd/clangd/issues/1120)，很巧的是他也是在用 clangd 分析内核代码时出现的（而且是访问了空指针导致的）。

不过我的输出中并没有栈回溯，不知道是不是 Apple 版的 clangd 去掉了栈回溯打印，因此还不能完全确定就是同一个问题。不过他提到了错误是 `EXPORT_SYMBOL` 宏展开后的代码导致的问题，因此我只留下两个没有 `EXPORT_SYMBOL` 宏的源文件，重启 vscode...

![没有 EXPORT_SYMBOL，一切正常](https://s2.loli.net/2023/05/19/KBxtWy3ORpMfjGq.png)

这次并没有崩溃，嵌入提示也在这个源文件中正常工作了。目前差不多能确定是同一个问题了，为了再次确认下，我想到既然他自己不打印，我可以挂 lldb 自己看啊～

![挂上 lldb](https://s2.loli.net/2023/05/19/biv4y13TLYMEoZu.png)

只见我一个 attach，马上抓到了内存访问异常——

![异常栈回溯](https://s2.loli.net/2023/05/19/W4eavFUY6ThztpQ.png)

直接贴一下内容：

```cpp
(lldb) c
Process 27639 resuming
Process 27639 stopped
* thread #15, name = 'ASTWorker:dev.c', stop reason = EXC_BAD_ACCESS (code=1, address=0x8)
    frame #0: 0x0000000109a31953 clangd`clang::RecursiveASTVisitor<clang::clangd::(anonymous namespace)::InlayHintVisitor>::TraverseDecl(clang::Decl*) + 12051
clangd`clang::RecursiveASTVisitor<clang::clangd::(anonymous namespace)::InlayHintVisitor>::TraverseDecl:
->  0x109a31953 <+12051>: movl   0x8(%rdx), %edx
    0x109a31956 <+12054>: movq   %r13, %rdi
    0x109a31959 <+12057>: movq   %r15, %rsi
    0x109a3195c <+12060>: callq  0x109a4dcf0               ; clang::clangd::(anonymous namespace)::InlayHintVisitor::addReturnTypeHint(clang::FunctionDecl*, clang::SourceLocation)
Target 0: (clangd) stopped.
(lldb) bt
* thread #15, name = 'ASTWorker:dev.c', stop reason = EXC_BAD_ACCESS (code=1, address=0x8)
  * frame #0: 0x0000000109a31953 clangd`clang::RecursiveASTVisitor<clang::clangd::(anonymous namespace)::InlayHintVisitor>::TraverseDecl(clang::Decl*) + 12051
    frame #1: 0x0000000109a31cb8 clangd`clang::RecursiveASTVisitor<clang::clangd::(anonymous namespace)::InlayHintVisitor>::TraverseDecl(clang::Decl*) + 12920
    frame #2: 0x0000000109a2e680 clangd`clang::clangd::inlayHints(clang::clangd::ParsedAST&, llvm::Optional<clang::clangd::Range>) + 336
    frame #3: 0x0000000109949909 clangd`void llvm::detail::UniqueFunctionBase<void, llvm::Expected<clang::clangd::InputsAndAST> >::CallImpl<clang::clangd::ClangdServer::inlayHints(llvm::StringRef, llvm::Optional<clang::clangd::Range>, llvm::unique_function<void (llvm::Expected<std::__1::vector<clang::clangd::InlayHint, std::__1::allocator<clang::clangd::InlayHint> > >)>)::$_21>(void*, llvm::Expected<clang::clangd::InputsAndAST>&) + 89
    frame #4: 0x0000000109af3ae4 clangd`void llvm::detail::UniqueFunctionBase<void>::CallImpl<clang::clangd::(anonymous namespace)::ASTWorker::runWithAST(llvm::StringRef, llvm::unique_function<void (llvm::Expected<clang::clangd::InputsAndAST>)>, clang::clangd::TUScheduler::ASTActionInvalidation)::$_7>(void*) + 1524
    frame #5: 0x0000000109aeb56a clangd`clang::clangd::(anonymous namespace)::ASTWorker::runTask(llvm::StringRef, llvm::function_ref<void ()>) + 522
    frame #6: 0x0000000109ae953a clangd`void llvm::detail::UniqueFunctionBase<void>::CallImpl<clang::clangd::(anonymous namespace)::ASTWorker::create(llvm::StringRef, clang::clangd::GlobalCompilationDatabase const&, clang::clangd::TUScheduler::ASTCache&, clang::clangd::TUScheduler::HeaderIncluderCache&, clang::clangd::AsyncTaskRunner*, clang::clangd::Semaphore&, clang::clangd::TUScheduler::Options const&, clang::clangd::ParsingCallbacks&)::$_4>(void*) + 4250
    frame #7: 0x0000000109c6ec37 clangd`void* llvm::thread::ThreadProxy<std::__1::tuple<clang::clangd::AsyncTaskRunner::runAsync(llvm::Twine const&, llvm::unique_function<void ()>)::$_4> >(void*) + 71
    frame #8: 0x00007ff811b31259 libsystem_pthread.dylib`_pthread_start + 125
    frame #9: 0x00007ff811b2cc7b libsystem_pthread.dylib`thread_start + 15
```

和这个 issue 的栈回溯基本完全一致，到此我们确认了问题。

不过我还有最后一个疑问，这个 issue 被提出来的时间距今已过去一年了。我确认了下 clangd 的版本，clangd 的版本号和 clang 是一致的，我系统中的版本目前是 14.0.3。目前 llvm 主线已经到 16.0.4 了，而主线的 14.0.3 是在2022年4月29日发布的。也就是说 Apple 的 llvm 落后于主线大概一年，而这个问题的修复 commit 刚好在 14.0.3 发布的一周后，也就是说并不包括在 14.0.3 中。不过话说这么久的必现崩溃问题不应该早就向后移植了吗，难道 Apple 的 llvm 团队在更新的时候只把编译器相关的问题向后移植了下，这个小小（？）的 clangd 问题就忽略了？

![clangd 版本](https://s2.loli.net/2023/05/19/RTW8s2QO1eYxAhg.png)

![Xcode 版本](https://s2.loli.net/2023/05/19/gAvtwyS8WZRO5af.png)

又去[这个网站](https://xcodereleases.com/alpha.html)看了下 Xcode 版本对应的 Clang 版本，最新一次 Xcode 更新确实将 clang 从 14.0.0 更新到了 14.0.3，这一切也就说得通了。

最后我用 brew 安装了最新的 llvm 版本解决了这个问题。不过默认 brew 安装的版本并不会出现在 PATH 中（这点很好，毕竟 macOS 上 Xcode 之类主要用的还是这个版本，并且安装时也提示了混用版本可能出现问题），需要在 VScode 的 clangd 插件里指定了下使用 brew 安装的 `/usr/local/opt/llvm/bin/clangd` 路径上的 clangd。

终于，clangd 愉快的跑了起来，vscode 的嵌入提示也能正常显示了！

![恢复正常了！可喜可贺！](https://s2.loli.net/2023/05/19/AghnrWmdsq6butQ.png)

这个问题也让我想到了之前看到老雷发现的 Visual Studio 问题也是访问空指针（具体[在这](https://mp.weixin.qq.com/s/ezPkE6ZUNr5lQFRGSwRI_A)），即便是大公司发布的看起来可靠稳定的开发工具有时候还是会出 bug 的 ^ ^
