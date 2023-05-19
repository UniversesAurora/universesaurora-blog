---
title: Apple Clangd 掉坑记
date: 2023-05-19 15:51:21
updated: 2023-05-19 15:51:21
cover: https://cdn.pixabay.com/photo/2023/05/15/15/33/landscape-7995369_1280.jpg
thumbnail: https://cdn.pixabay.com/photo/2023/05/15/15/33/landscape-7995369_1280.jpg
categories:
- 随笔
tags:
- 编辑器
- vscode
- clangd
- llvm
toc: true
---

clangd 是 llvm 项目推出的 C/C++ 语言服务器，通过 LSP（Language Server Protocal）协议向编辑器如 vscode/vim/emacs 提供语法补全、错误检测、跳转、格式化等等功能。据传是比 vscode 自己的 IntelliSense 更好一些，不过我开始用 clangd 的时候 IntelliSense 貌似还不支持使用 `compile_commands.js` 文件辅助代码分析，所以也没有具体对比过两者的差异。

macOS 上附带的 llvm 是 Apple 自己管理的，属于 Xcode 的一部分，这个版本落后于目前主线的版本很远。按理来说这种等待上游稳定后再选择使用的版本稳定性更好、问题比较少，然而没想到这次居然在更新了 Xcode 版本后翻车了。

<!-- more -->

今天打开 vscode 准备看下内核代码时，看到有 vscode 的更新就顺便更新了下，vscode 重启过后我发现 clangd 提供的语法分析功能完全用不了了，

一开始 clangd 一直在报 `Error while reading shard xxx.c: wrong version: want 17, got 16` 这个错。我猜测可能更新了 Xcode 之后缓存的某些东西版本和当前不符导致的，于是删除了项目目录下的 `.cache` `.tmp_*` 之类的文件，这个问题倒是没有了，但是 clangd 插件还是报告崩溃（好崩溃...

打开输出窗口看了下 clangd 插件的日志，报了一个`Signalled during AST worker action: InlayHints`的错误，看起来是被某个信号中断了，这里能推测下也许是某个 bug 导致访问了非法内存地址。

![错误输出](https://s2.loli.net/2023/05/19/FQ1HDPUSep2EaIw.png)

开启时只打开没有 EXPORT_SYMBOL 的文件，没有发生崩溃。

Google 了下只找到了一个似乎很符合目前情况的 [issue](https://github.com/clangd/clangd/issues/1120)，正巧他也是在用 clangd 分析内核代码时出现的，而且是访问了空指针导致的。不过我的输出中并没有栈回溯，不知道是不是 Apple 版的 clangd 去掉了栈回溯打印，因此还不能完全确定就是同一个问题。

![没有 EXPORT_SYMBOL，一切正常](https://s2.loli.net/2023/05/19/KBxtWy3ORpMfjGq.png)

不过没关系，他自己不打印，我可以挂 lldb 帮他打啊～

![挂上 lldb](https://s2.loli.net/2023/05/19/biv4y13TLYMEoZu.png)

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

![clangd 版本](https://s2.loli.net/2023/05/19/RTW8s2QO1eYxAhg.png)

![Xcode 版本](https://s2.loli.net/2023/05/19/gAvtwyS8WZRO5af.png)

去[这个网站](https://xcodereleases.com/alpha.html)看了下目前 Xcode 版本对应的 Clang 版本。

但是目前主线已经到 16.0.4 了，而主线的 14.0.3 是在2022年4月29日发布的。

这个修复的 commit 刚好在 14.0.3 发布的一周后。

最后 brew 上下了一个最新的 llvm 版本解决了这个问题。不过默认 brew 安装的版本并不会出现在 PATH 中（这点很好，毕竟 macOS 上 Xcode 之类主要用的还是这个版本，并且安装时也提示了混用版本可能出现问题），需要在 VScode 的 clangd 插件里指定了下使用 brew 安装的 `/usr/local/opt/llvm/bin/clangd` 路径上的 clangd。

终于，clangd 愉快的跑了起来，vscode 的嵌入提示也能正常显示了！

![恢复正常了！可喜可贺！](https://s2.loli.net/2023/05/19/AghnrWmdsq6butQ.png)
