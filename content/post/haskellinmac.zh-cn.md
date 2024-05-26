---
title: 在Mac上配置Haskell环境
date: 2023-06-28T18:10:03+08:00
draft: false
toc: true
comments: true
tags:
  - Haskell
categories:
- 计算机
---

> 这篇博客将介绍如何在**中国**安装、配置 Mac 的 Haskell 环境和编辑器的高亮、补全，其中编辑器包括 Vim/Neovim、Visual Studio Code。

<!--more-->

## 安装 Haskell

> 本文将带你安装 GHCup、GHC、Cabal。由于笔者写这篇文章时还是 Haskell 的初学者，所以没有办法细致的分析各种工具的优劣，故选择目前相对流行、配置简单的工具。

要安装 Haskell，首先要打开一个 terminal，之后的操作都会基于 terminal 来完成。

第一步我们要安装 GHCup，由于这个软件比较新，所以较少有镜像，这里我们选择[中科大的镜像网站](https://mirrors.ustc.edu.cn/help/ghcup.html)。但是由于 GHCup 在安装 Cabal 时会下载一个文件，所以我们必须提前替换 hackage 源，否则将无法正常安装。换源方法如下：

```bash
# 第一步在家目录下创建一个 .cabal 文件夹
cd ~ && mkdir .cabal
# 第二步创建并编辑一个 config 文件
vi .cabal/config
```

这里我选择用 vim，你可以用 nano，vscode 等等编辑器，不会有任何影响，~~当然，我不推荐你使用 emacs~~。然后将如下内容粘贴进去，保存退出：

```
repository mirrors.ustc.edu.cn
  url: https://mirrors.ustc.edu.cn/hackage/
  secure: True
```

之后就可以在终端执行命令了：

```sh
curl --proto '=https' --tlsv1.2 -sSf https://mirrors.ustc.edu.cn/ghcup/sh/bootstrap-haskell | BOOTSTRAP_HASKELL_YAML=https://mirrors.ustc.edu.cn/ghcup/ghcup-metadata/ghcup-0.0.7.yaml sh
```

接下来你会看到这样的界面：

![image-20230702042402797](https://cdn.jsdelivr.net/gh/zzxdyf1314/mycloudimg@master/image-20230702042402797.png)

**请仔细阅读所有提示，如果你不知道该做什么，就照我的选择去做：**

1. 是否自动将 ghcup 目录自动加入 PATH 直接回车
2. 是否安装 haskell 的 lsp
   - 如果你想使用 vscode 或 vim 等编辑器来补全、高亮、提示 Haskell 代码，就输入 Y；
3. 是否安装 Stack，输入 N，不安装。

之后的选项都直接回车，就完成了 GHCup 的安装，这里你需要设置当前 shell 的 PATH：

```bash
source ~/.ghcup/env
```

如果你不想这样做，也可以直接重启你的 shell。

这时你的 Haskell 环境已经安装完成了，可以简单测试一下：

```bash
ghc --version

cabal --version

ghci
# 输入 :q 以退出 ghci
```

接下来的步骤是配置 GHCup 的源

编辑 `~/.ghcup/config.yaml`文件，向其中添加：

```
url-source:
    OwnSource: https://mirrors.ustc.edu.cn/ghcup/ghcup-metadata/ghcup-0.0.7.yaml
```

## 编辑器配置

如果你想在 Vim/Neovim 上使用 HLS 来完成高亮、补全、提示、报错的话，你需要安装 [Coc.nvim](https://github.com/neoclide/coc.nvim)，并安装`coc-hls`插件，可以直接在 vim 的 command 模式下输入`:CocInstall coc-hls`，之后就可以正常使用了（一定要保证你安装了 HLS）。

在 vscode 上搜索 Haskell，安装同名插件即可。

不管是在 vim 还是 vscode 上都可能出现一种情况，插件提示说找不到对应版本的 HLS，一般来说问题出在你安装的 GHC 还不支持你安装的 HLS，可以在终端输入：

```
ghcup tui
```

进入 ghcup 的 ui 界面：

![image-20230702043610766](https://cdn.jsdelivr.net/gh/zzxdyf1314/mycloudimg@master/image-20230702043610766.png)

一定要保证你安装的 GHC 后面标有 hls-powered，你可以按上下键移动到对应的 GHC 版本上按`i`安装或按`u`卸载。同样的，你也可以安装或卸载其他的工具。

如果你是从这个界面安装的其他版本的 GHC，并卸掉了之前安装的，那么你需要替换掉原本的 ghc 文件和 ghci 文件，你可以在重启 shell 后发现 ghc 和 ghci 命令不存在或是还是原来的版本，这时你需要更新这两个文件。

ghc 和 ghci 的默认路径在家目录下的`.ghcup/bin/`下，是软连接到.ghcup/ghc/version code/bin/`下的文件，我们要做的就是重新软连接：

```
ln -s ~/.ghcup/ghc/verison code/bin ghci ~/.ghcup/bin/ghci
ln -s ~/.ghcup/ghc/verison code/bin ghc ~/.ghcup/bin/ghc
```

之后重启 shell/vscode，就会发现你的 HLS 已经可以运行了。

> 如果最开始有短暂的卡顿，是 HLS 正在启动，稍等即可。
