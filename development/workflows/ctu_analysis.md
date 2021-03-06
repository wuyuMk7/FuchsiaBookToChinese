# Zircon 中的 Cross Translation Unit 静态分析

本文档介绍了：

* 如何在 Zircon 中使用 Clang 静态分析器 (**CSA**)设置 cross-translation-unit 分析(**CTU**);
* Kareem Khazem 在实习期间完成的关于 CTU 工作;
* 以及，在 Zircon 中完全支持 CTU 所需要做的剩余工作。


## 在 Zircon 中设置和运行 CTU

**摘要**：下载 Clang 的源代码，并在编译之前应用几个非主线补丁。围绕分析工具运行我的包装脚本。下载 CodeChecker 工具;使用它来理解分析的结果，并启动一个 Web 服务器以通过 Web 界面查看结果。

## CTU 授权补丁

已知的两个补丁集：

* ([三星](https://github.com/haoNoQ/clang/tree/summary-ipa-draft))补丁集，这是一个庞大的补丁，增加了对 Clang 的 AST 合并支持，主要包括对  `lib/AST/ASTImporter.cpp` 的补充。并且在 `tools/xtu-build/*` 下有整套 CTU 分析工具（原始版本，运行不稳定）。该补丁集基于 Clang 的旧版本和它补丁集庞大的事实，使得它很难在 tip-of-tree (**ToT**) Clang 之上大规模重写。
* （[爱立信](https://github.com/dkrupp/clang/blob/ctu-master/tools/xtu-build-new/xtu-analyze.py)）补丁集，包含了 Samsung 的 AST 合并工作并且加入了一些允许进行 CTU 分析的新工具（`tools/xtu-build-new/*`和 `tools/scan-build-py/*`）。Ericsson 的 xtu-build-new 工具是对 Samsung xtu-build 工具的改进但是又有些区别。该补丁集比 Samsung 的新许多，作者也一直致力于保持它基于 ToT 重写。

我们会采用 Ericsson 的补丁集来修补 Clang，因为 AST 的合并工作干净利落，我们也得到了更新的分析工具。但是，需要注意的是 CTU 对于 Zircon 的支持是不完整的，在某些情况下，Samsung 的补丁集包含了一些提供必需功能的代码（更多详情如下）。

### 构建 CTU-capable CSA 步骤

1. 以常规步骤下载并构建 Clang 和 LLVM。
2. 在一个单独的目录里，克隆 Ericsson 的 Clang 分支，然后切换到 ctu-master 分支。
3. 将[这个脚本](https://gist.github.com/karkhaz/d11efa611a1bde23490c2773dc0da60d)下载到 Ericsson 的分支中，然后运行。它会将一系列补丁转储到补丁目录中。我特意只转储了 Ericsson 从开始到 1bb3636 的提交 ，这是我在实习期 Ericsson 的最新版本。
    * 如果你想要使用 Ericsson 的最新改动版本，可以尝试在脚本中将 1bb3636 更改为 HEAD。通过在脚本中指定范围，来确保跳过将上游提交合并到 ctu-master 分支的提交。git log --graph 指令可以帮助确认上游提交与 Ericsson 提交的内容区别，如下是我使用的命令
    ```
    git log --graph  --decorate --date=relative --format=format:'%C(green)%h%C(yellow) %s%C(reset)%w(0,6,6)%C(bold green)\n%C(cyan)%G? %C(bold red)%aN%C(reset) %cr%C(reset)%w(0,0,0)\n%-D\n%C(reset)' --all
    ```
4.  将生成的补丁一次一个地应用到 *上游* Clang（而不是 Ericsson 分支）。
   ```
   for p in $(ls $PATCH_DIR/*.patch | sort -n); do git am < $p; done
   ```
11. 如果没有Kareem Khazem的补丁，则应用[下面](#zircon-patches)列出的补丁
12. 重新构建上游 Clang 和 LLVM。


## 运行 CTU 分析

**摘要:** 运行我的包装脚本。这会正常生成构建 Zircon，然后再次构建它——这其中，会转储序列化的 AST ，而不是转储目标文件，最后使用转储的 AST 来分析每个文件以实现 CTU。

### CTU 工作原理

首先，你需要了解这些前提：

非 CTU 静态分析分析每个 TU 的 AST;对外部函数的任何函数调用都被视为不透明。粗略地说，CTU 分析试图用该函数实现的 AST *代替* 不透明函数调用节点。

因此，CTU 分析将像往常一样开始分析一个 AST，但是当它遇到一个函数调用节点时，它将尝试 *合并* 该函数的 AST。这依赖于 AST 已预先序列化到磁盘的功能，以便分析器可以将 AST 重新加载到内存中。它也依赖于对 AST 合并的支持，这是 Samsung 补丁到`ASTImporter.cpp`（Ericsson 补丁也源自它）的补丁。

为了将 AST 序列化到磁盘上，我们需要模拟真正的构建过程。做到这一点的方法，是在记录编译器调用时，真正地构建一遍 Zircon ;这允许我们'回放'调用，但是此时编译器标志将被修改为转储 AST 文件而不是对象文件。


总而言之，现在我们发展出了：

* 使用 Clang 构建 zircon，并将构建过程包装在 [bear](https://github.com/rizsotto/Bear) 等程序中，以便记录编译器调用并生成 JSON 编译数据库。
* 重播相同的编译步骤，但转储 AST 文件而不是目标文件。这正是 [xtu-build.py] (https://github.com/dkrupp/clang/blob/ctu-master/tools/xtu-build-new/xtu-build.py)工具的工作。
* 像往常一样执行静态分析，但需要时将每个被调用函数的AST反序列化。
这是 [xtu-analyze.py](https://github.com/dkrupp/clang/blob/ctu-master/tools/xtu-build-new/xtu-analyze.py) 工具在顶层所做的，通过爱立信团队编写的瘦身版的 [scan-build replacement](https://github.com/dkrupp/clang/blob/ctu-master/tools/scan-build-py/bin/scan-build) 调用 [scan-build-py/libscanbuild](https://github.com/dkrupp/clang/tree/ctu-master/tools/scan-build-py/libscanbuild) 目录中的工具。


这些步骤被捕获在下面提到的 [Fuchsia 包装器](＃fuchsia-wrapper-script)中。所有的这些构成了一个以报告组成的目录，采用 [Apple plist](https://en.wikipedia.org/wiki/Property_list) 格式，主要内容为错误报告的详细信息。


### Ericsson 的包装脚本<a name="ericsson-wrapper-script"></a>

有两组工具用于运行 cross-translation-unit 分析:

* `tools/xtu-build-new` 下的工具是顶级脚本。由于底层分析器可能会错误（例如，由于 CSA 崩溃），所以我修补了 `xtu-analysis.py` （在 Ericsson 分支中），以便将分析器的输出（stdout/stderr，而不是报告）转储为一份文件。根据分析器的返回码，输出进入 ``OUT_DIR/{passes，failed}`` ，其中 `$ OUT_DIR` 是传递给`xtu-analyze.py` 的 `-o` 参数的目录。这些文件中特别有用的部分是*第二行*，它由 `libscanbuild` 工具（下一个项目符号点）发出的 `analyze：DEBUG：exec command in` 开头。该命令是修改其命令行的漫长而单调的过程后实际调用 CSA 的方式。因此，如果您想使用 gdb 在一个麻烦的文件上运行 CSA，则需要使用该命令。
* `tools/scan-build-py` 下的工具是一个用于包装实际调用 Clang 的工具包。它们负责修改命令行。我对它们不太熟悉，过去也不太需要干涉它们。


### Fuchsia 包装脚本<a name="fuchsia-wrapper-script"></a>

[这个非常小的 shell 脚本](https://gist.github.com/karkhaz/c8ded50e564d73853731266fec729454) 封装了 Ericsson `xtu-build-new` 封装。要对 zircon 进行完整分析，请务必先清理，然后指定正确的路径去构建 Clang。 然后再在 zircon 目录中执行：

```
make clean && ./run.sh
```

为了只构建内核，需指定 `TARGET` 作为环境变量:

```
make clean && TARGET=./build-zircon-pc-x86-64/zircon.elf ./run.sh
```

该脚本还需要提前将 [clangify.py](https://gist.github.com/karkhaz/2ab5e8c7a8783318d44ceca715f20438) 以可执行位集的形式放置在 zircon 目录中。分析完成后，会生成一个 `.result-xtu` 文件，它包含了:

* 一堆报告 BUG 的苹果 plist 文件;
* 一个包含失败内容的目录，包含了分析器的 std{out,err} 返回非零的调用;
* 一个包含成功内容的目录，包含了分析器的 std{out,err} 返回零的调用。

## 查看分析结果

此时，只有一种方法来解析 plist 报告文件，并且将他们以 web 接口的形式展示,这种方法就是使用 [CodeChecker](https://github.com/Ericsson/codechecker) 工具。该工具是 Ericsson 开发的，用于代码阅读和许多其它的任务。使用 CodeChecker 需要先安装大量的依赖项，相比使用 **apt-get**，建议优先使用 **pip** 或 **npm** 等其它管理工具来进行安装。总之，执行分析并将 plist 转储到 .result-xtu 之后，你可以调用 `CodeChecker plist` 指令来解析 plist 文件:

```
CodeChecker plist -d .result-xtu -n 2016-12-12T21:47_uniq_name -j 48
```

在 `CodeChecker plist` 的每次调用中，`-n` 的参数都必须是唯一的，因为它代表一个单独的分析运行。否则的话， CodeChecker 会抛出一个错误。之后，运行 `CodeChecker server` 指令，将在 `localhost:8001` 上启动一个 web 服务器，它会显示所有之前的分析运行报告。

## 获得帮助

Samsung 补丁集由 [Aleksei Sidorin](mailto：a.sidorin@samsung.com) 和他的团队撰写。 Aleksei 对 `ASTImporter.cpp` 和其他 AST 合并方面非常了解，并能针对这些内容提供有力的帮助。他和 [Sean Callanan](mailto：scallanan@apple.com) 很乐意帮助我检查我的 AST Importer 补丁。 Aleksei 还在 2016 年 LLVM 开发者大会上发表了关于基于概要的过程间分析的[相关讨论](https://www.youtube.com/watch?v=jbLkZ82mYE4)。

Ericsson 补丁集由 [Gábor Horváth](mailto:xazax.hun@gmail.com) 和他的团队撰写。对于如何使用 `xtu-build-new` 工具进行 CTU 分析，Gábor 的建议非常有帮助。 

我 ([Kareem Khazem](mailto:karkhaz@karkhaz.com)) 也很乐意尽我所能来提供帮助。

LLVM irc 频道也会有所帮助。


## Zircon特性分析

上游 Clang 非常乐意接受 Zircon 专用 Clang 语法检查器的补丁。[MutexInInterruptContext](https://reviews.llvm.org/D27854) 检查器就是一个例子（由 Farid Molazem Tabrizi 编写的 LLVM 通道移植而来），[SpinLockChecker](https://reviews.llvm.org/D26340) 和 [MutexChecker](https://reviews.llvm.org/D26342) 也是如此。 Clang 检查的潜在审查者有 Devin Coughlin（来自 Apple），Artem Dergachev（在三星的 Aleksei Sidorin 团队）和 Anna Zaks（也在 Apple）。

这些检查器通常是 *opt-in* 的，这意味着你需要将一个参数传递给分析器以启用它们：类似于 `-analyzer-checker=optin.zircon.MutexInInterruptContext` 。

如果 Clang 上没有这些补丁，你需要申请它们。要使用它们和 [Ericsson 包装脚本](#ericsson-wrapper-script) 来分析 Zircon，你需要修改[Fuchsia 封装脚本](#fuchsia-wrapper-script)，在文件末尾添加 `-e optin.zircon.MutexInInterruptContext` 选项到 `xtu-analyze.py` 的调用。 `MutexInInterruptContext` 的补丁有一个测试套件，可以用来作为分析的能力的一个例子。

# Zircon 支持的 CTU 的发展历程

## AST 导入器已修复的问题

上游 CSA 会在绝大多数的 Zircon 文件上崩溃。本节介绍 Kareem Khazem 遇到的一些问题及其修复。

### 不支持的 AST 节点<a name="zircon-patches"></a>

很多 Zircon 代码都无法在 Clang 静态分析器中直接导入，原因在于 Clang 静态分析器没有实现对某些类型的 AST 节点的导入支持。以下列出了支持这些节点的补丁：

<table>
  <tr>
    <td>AtomicType</td>
    <td>Patch merged into upstream</td>
  </tr>
  <tr>
    <td>CXXDependentScopeMemberExpr</td>
    <td>https://reviews.llvm.org/D26904</td>
  </tr>
  <tr>
    <td>UnresolvedLookupExpr</td>
    <td>https://reviews.llvm.org/D27033</td>
  </tr>
  <tr>
    <td>DependentSizedArray</td>
    <td></td>
  </tr>
  <tr>
    <td>CXXUnresolvedConstructExpr</td>
    <td></td>
  </tr>
  <tr>
    <td>UsingDecl</td>
    <td>https://reviews.llvm.org/D27181</td>
  </tr>
  <tr>
    <td>UsingShadowDecl</td>
    <td>https://reviews.llvm.org/D27181</td>
  </tr>
  <tr>
    <td>FunctionTemplateDecl</td>
    <td>https://reviews.llvm.org/D26904</td>
  </tr>
</table>


一般来说，在实现对新节点类型的支持时，必须在 `ASTImporter.cpp` 中实现 `VisitNode` 函数，还要进行单元测试和功能测试;以上 Kareem 的补丁中包含了相应的示例。此外，仍有不少不支持的 AST 节点等待被支持; 使用 grep 在分析器输出目录中查询 `error: cannot import unsupported AST node`。

Ericsson 补丁集仅包含 Samsung 补丁集中的一部分 `ASTImporter` 代码。在某些情况下，可以直接从 Samsung 补丁集中获取不受支持节点的 `Visit` 功能。但是，Samsung 补丁集不包含任何测试，所以在对该节点的支持进行升级之前，仍然有必要编写测试。

### 大量的错误

`ASTImporter.cpp` 中的代码包含大量 bug。有时候 Aleksei 会在一些 issues 中提供针对性补丁，如[this one](https://reviews.llvm.org/D26753)，所以值得给他 (**a-sid**）在 IRC 上快速 ping 一下。我的调试策略是在封装器输出中查找以 `analyze: DEBUG: exec command in` （后面跟着分析器的实际命令行）为开头的`第二个`字符串，并通过 gdb 运行该命令行。通常只需几个小时就可以找出段错误来自何处。

## CTU 之前和之后发现的错误

### VFS 中可能的错误？

这是 `oldparent` 的双重释放，它在 `system / ulib / fs / vfs.c：vfs_rename` 上声明未初始化。隔两行后，`vfs_walk`（同一个文件）被调用并以`oldparent`作为其第二个参数。通过输入 for 循环并在第一个循环中点击 `return r` 语句，可以从 `vfs_walk` 返回而不分配给 `oldparent`。如果 `r` 的值大于零，那么我们跳转到 `else if` 语句，该语句在 `oldparent`（它仍然未初始化）上调用了 `vn_release`。

### 线程中可能的错误？

这是一个“释放后使用”。路径是：

* `kernel/kernel/thread.c:thread_detach_and_resume`
    * 调用 `thread_detach(t)`
        * 返回 `thread_join(t, NULL, 0)`
            * 释放 `t` 并且返回 `NO_ERROR`
        * 返回 `NO_ERROR`
    * 检查错误是 1false1
    * 调用已被释放的 `thread_resume(t)`.
        * `thread_resume`然后访问 `t` 的字段.

## CTU 误报

* CSA 无法解析通过函数指针调用的函数的实现。这意味着它不能对函数的返回值做出任何假设，也不能对函数可能的输出参数产生任何影响。
* 有类函数无法实现对分析仪的访问。同样，分析器不知道这些函数会触及它们的输出参数，所以它们会错误地从抛弃值中抛出下面的报告：
  ```
  struct timeval tv;
  gettimeofday(&tv, NULL);
  printf("%d\n", tv.tv_usec);   // [SPURIOUS REPORT] Access to
                                // uninitialized variable ‘tv’
  ```
* 一些易受此类不精确影响的功能包括：
    * 系统调用 (例如 `gettimeofday`)
    * 编译器内置 (例如 `memcpy`)