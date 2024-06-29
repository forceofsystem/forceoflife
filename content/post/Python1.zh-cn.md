---
title: "从零开始学 CPython - 0"
date: 2023-12-19T14:28:43+08:00
draft: false
comments: true
toc: true
tags:
  - Python
  - 编译原理
  - 计算机
---

<!--more-->

## 前言

Python 是时下最流行的编程语言，在 [TIOBE](https://www.tiobe.com/tiobe-index/) 排行榜上连续多年位居榜首，作为一名计算机相关专业的学生，掌握 Python 是非常有必要的。可我还不会 CPython，真是闻着伤心见者落泪。于是我痛定思痛，打算新开一个系列，从零开始学习 CPython，努力成为一个会调包的合格大学生（误）。你可能注意到了我说我要从零开始学习 CPython，如果你对 CPython 不够了解的话，~~CPython 是 Python 的别称，意思是像 C 一样快的 Python。~~

好吧，玩笑到此就结束了，正式介绍一下，CPython 是 Python 解释器的官方实现，对于大多数人来说，平常写的 Python 代码就是由 CPython 来解释执行的。这个系列是我从零开始阅读 Python 源码，将编译原理、虚拟机等理论知识与工程上具体的实现相结合的尝试。

## 环境搭建

### 获取 CPython 源码

既然是学习 Python 的源码，那么就有必要搭建一个环境来支持我们阅读、运行、修改源代码。在笔者写这篇文章时，Python 已经完全迁移到 Github 上开发了，可以用 git 将 CPython 的源码直接克隆到本地:

```bash
$ git clone https://github.com/python/cpython.git
```

笔者的电脑是一台 M1 芯片的 Macbook Air：

```bash
$ uname -m -s
Darwin arm64
```

之后可以运行下面的指令来编译 CPython:
```bash
$ ./configure --with-pydebug && make -j
```

> 简单解释一下，`./configure --with-pydebug` 是执行 [GNU Autoconf](https://www.gnu.org/software/autoconf/) 来生成 Makefile。

编译完成后可以运行一下测试：

```bash
./python.exe -m test -j3 
```

你可能会奇怪为什么是 python.exe，这明明不是 Windows 系统。这是因为 Mac 系统是大小写不敏感的，如果不加后缀名的话会与目录中的 Python 目录冲突，加上 .exe 后缀可以避免这种情况。

```bash
$ file python.exe
python.exe: Mach-O 64-bit executable arm64
```

到了这里，你就成功拥有了一份完整的 CPython 源码，笔者使用的源码是 3.13.0a2+。如果你在编译中遇到了问题，你可以访问 Python 的[开发者指导](https://devguide.python.org)网站来获取不同操作系统的教程。

### 阅读代码环境搭建

有了源码，接下来是如何阅读源码。对于 CPython 这个体量的项目，一般的工具肯定是不行的。所以，~~我推荐大家使用 Notepad.exe。~~

我没有尝试过将 CPython 的代码导入 IDE，但我猜这会吃掉我笔记本的所有内存并占用所有的 CPU 来建立索引。我无意挑起所谓的 [Editor War](https://en.wikipedia.org/wiki/Editor_war)，我只是来分享一下我的方案。我使用 Vim + Clangd + Ctags + Ripgrep (rg) 来阅读代码，在 CPython 巨大的代码量下，LSP（Clangd）几乎失灵，我用 Ctags 建立了基本的索引，在无法跳转时，用 rg 搜索对应函数的位置（没有安装 rg 也可以用 grep）。不过有一点我觉得很重要，就是关闭 LSP 的 diagnostics 功能，不然你可能会看到“山河一片红”。

> 如果你对我的 Vim 配置感兴趣，欢迎[查看](https://github.com/forceofsystem/dotvim)并送我一颗星星，顺带一提，我的配置很简洁而轻量。

无论如何，不管你选择 Vim、Emacs、Source Insight、VSCode 或是使用其他 IDE，你的目标都是能够方便地查看代码，而不是像以前的我一样为了配置编辑器而配置编辑器～。

### 调试环境搭建

没错，你还需要一个调试器。当你捋不清函数的调用关系时，你需要打开调试器，查看调用栈来获取更多信息。

> 我在撰写本文时大量采用这种方法。

在 Linux 上，你可以选择 GDB，因为 GDB 没有适配 M1，所以我选择了 LLDB，在 Windows 上，你可以选择 [MinGW64](https://www.mingw-w6b4.org) 自带的 GDB。

> 我的 LLDB 是通过 [Homebrew](https://brew.sh) 安装的，不是 `xcode-select --install` 安装的版本。

## Python 学习基础

### 目录

在开始阅读源代码之前，我们先来了解一下 CPython 的目录结构。绝大多数 CPython 代码都在下面几个文件夹中：

- Include：头文件
- Objects：各种对象的实现
- Python：解释器、字节码编译器和其他重要的基础组件
- Parser：词法分析器、语法分析器以及语法分析生成器
- Modules：标准库模块以及 main.c
- Programs：包含了程序的入口函数 main()

如果你是在 Linux 或 BSD （不包括 Mac OS X），这就是与你有关的所有目录；而如果你在用 Mac 或者 Windows，还有以下目录：

- Mac：专用于 Mac OS X 的代码
- PC：专用于 Windows 的代码（旧）
- PCBuild：专用于 Windows 使用的 MSVC 的代码（新）

在这个系列的博客中，我们只关心公共部分的代码，而不会关注这些特定平台的代码。

### 命名约定

在阅读之前，我们还需要学习一下 Python 的命名方式，根据 [PEP 7](https://peps.python.org/pep-0007/#naming-conventions) 中对于命名约定的说明：

- 除 static 函数外，对于所有的 public 函数，使用 `Py` 作前缀；对于 global service routines，使用 `Py_` 前缀，如 `Py_FatalError`；对于特定类型的例程，使用与之相关的较长的前缀，如 `PyString_` 之于字符串相关函数。
- public 函数和变量的命名混合使用大小写和下划线，如：`PyObject_GetAttr`，`Py_BuildValue`，`PyExc_TypeError`。
- 需要将 internal 的函数可见时，使用 `_Py` 前缀，如：`_PyObject_Dump`。
- 对于宏，其前缀是大小写混合，之后的部分全部使用大写，如：`PyString_AS_STRING`，`Py_PRINT_RAW`。
- 宏参数应该使用 ALL_CAPS 风格，以便与变量和结构成员进行区分。
  - ALL_CAPS 风格：用下划线作分割，全部字母大写。

## 学习开始

### 学习目标

下面我们就可以开始正式学习 CPython 的代码了，虽然我迫不及待地想看看 Python 是如何进行语法分析（Parse），Python 的虚拟机是如何实现的、采用了什么垃圾回收算法，但是俗话说：“心急吃不了热豆腐”，我们还是从一个简单的目标开始。

现在在你的终端运行我们刚编译好的 Python，什么参数也不要加，就像这样：

``` bash
$ ./python.exe
Python 3.13.0a2+ (heads/main:21d52995ea) [Clang 15.0.0 (clang-1500.0.40.1)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

现在我们就进入了 Python 的 REPL(Read-Eval-Print-Loop) 模式，作为一个“第一次”使用 Python 的新人，我非常好奇这个 Header 和交互提示符（Prompt）是怎么被打出来的。所以，我决定先探寻一下 Python 是怎么跑起来的，又是怎么打出那个经典的提示符 `>>>` 的。

### 找到入口

就想所有的 C 程序那样，CPython 也有一个入口函数，CPython 的入口函数位于 Programs 的 `python.c` 中：

```C
int
main(int argc, char **argv)
{
    return Py_BytesMain(argc, argv);
}
```
可以看到这个函数调用了定义在 Modules/main.c 中的 `Py_BytesMain` 函数。

> 据 Guido 所说，main.c 不在 `Python` 目录下而在 `Modules` 目录下是由于一些不太重要的历史原因。

可以看到，在保存了命令行参数后，`Py_BytesMain` 函数就调用了 `pymain_main` 函数。在这个函数里，首先执行了对解释器的初始化。

> `Py_BytesMain` 是对 `Py_Main` 的包装，用来防治因为 locale 和编码模式不同而导致的错误。

### 初始化解释器

在 `pymain_main` 中，调用 `pymain_init` 来初始化解释器：

```C
    PyStatus status = pymain_init(args);
```

首先来看一下 `PyStatus` 这个类型，根据 [PEP587](https://peps.python.org/pep-0587/)，其是用来存储初始化函数的状态，成功、错误或是退出，并且还会存储造成错误的函数名。简单看一下 `PyStatus` 的字段组成：
- `exitcode` (int): Argument passed to exit().
- `err_msg` (const char*): Error message.
- `func` (const char *): Name of the function which created an error, can be NULL.
- private `_type` field: for internal usage only.

进入 `pymain_init` 函数，可以看到初始化包括三个部分：运行时、preconfig、config。

```C
static PyStatus
pymain_init(const _PyArgv *args)
{
    PyStatus status;

    status = _PyRuntime_Initialize();

    PyPreConfig preconfig;
    PyPreConfig_InitPythonConfig(&preconfig);

    status = _Py_PreInitializeFromPyArgv(&preconfig, args);

    PyConfig config;
    PyConfig_InitPythonConfig(&config);

    if (args->use_bytes_argv) {
        status = PyConfig_SetBytesArgv(&config, args->argc, args->bytes_argv);
    }
    else {
        status = PyConfig_SetArgv(&config, args->argc, args->wchar_argv);
    }

    status = Py_InitializeFromConfig(&config);

    status = _PyStatus_OK();
}
```
> 代码省略了部分错误处理。

这对应着 [PEP432](https://peps.python.org/pep-0432/) 中提到的解释器初始化的三个步骤：
- Python 核心运行时预初始化（Python core runtime preinitiallization）:
  - 启动内存管理；
  - 决定系统接口使用的编码；
- Python 核心运行时初始化（Python core runtime initialization）:
  - 确保 C API 已经可以使用；
  - 确保内置模块与冻结模块（`frozen`）是可访问的；
- 主解释器配置（Main interpreter configuration）:
  - 确保外部模块是可访问的；

> 在 3.8 之前，初始化过程都是分为 2 步。CPython 的开发者们从 2012 年末到 2020 中期，用了 8 年的时间来重构，以让 Python 的启动过程更容易维护，同时也更容易嵌入到大型应用中。大家可以去阅读 PEP 432 和 PEP 587 来获取更完整的信息。
> 在现在的设计中，Python 的初始化过程分为以下四个阶段：
  - 未初始化：还未开始初始化过程；
  - 预初始化：解释器还不能使用；
  - 运行时已初始化：主解释器部分可用，还不能创建子解释器；
  - 初始化完成：主解释器完全可用，可以创建子解释器。

在 Python 3.8 中，为上述步骤都添加了一些数据结构，我们可以在上面的代码中看到。

`PyPreConfig` 结构体用来预初始化 Python 的下述功能：
- 设置内存分配器；
- 配置 LC_CTYPE locale；
- 设置 UTF-8 模式；

在上面的代码中可以看到与预初始化相关的函数：`PyPreConfig_InitPythonConfig` 和 `Py_PreInitializeFromPyArgv`，前者用来初始化默认配置（preconfiguration），后者则用来预初始化 Python。

预初始化结束后，开始初始化。首先初始化默认配置，将命令行参数存储到 `config->argv`，之后调用 `Py_InitializeFromConfig` 完成之后的初始化过程。`PyConfig` 是一个相当庞大的结构体，其定义有足足 100 行之多。

### 运行开始

在执行完初始化过程后，`pymain_main` 函数调用了 `Py_RunMain` 函数，终于要开始正式运行了，初始化过程可真是漫长，呜呼～。进入这个函数，它的代码意外地简单，我还以为会很复杂呢。

```C
int
Py_RunMain(void)
{
    int exitcode = 0;

    pymain_run_python(&exitcode);

    if (Py_FinalizeEx() < 0) {
        /* Value unlikely to be confused with a non-error exit status or
           other special meaning */
        exitcode = 120;
    }

    pymain_free();

    if (_PyRuntime.signals.unhandled_keyboard_interrupt) {
        exitcode = exit_sigint();
    }

    return exitcode;
}
```

很容易发现，这个函数是对 `pymain_run_python` 的一个包装，要想探究真正的运行过程，我们还需要继续抽丝剥茧地向里探查，`pymain_run_python` 启动！

一进入这个函数，这个近 100 行的函数体就让我感到头晕。跳过一些获取解释器状态的代码，我们看到这个函数首先加载了 `readline` 模块,这个模块可以为我们提供获取输入的能力。`pymain_import_readling` 函数是对 `PyImport_ImportModule` 的封装，在这个函数里一共引入了两个模块，分别是 `readline` 和 `rlcompleter`，后者用于提供在交互式环境下的自动补全功能。

```C
    pymain_import_readline(config);

    PyObject *mod = PyImport_ImportModule("readline");
    ...
    mod = PyImport_ImportModule("rlcompleter");
```

让我们越过那些冗长的错误处理（虽然它们是必要的，但是对于梳理代码运行逻辑可真没什么用），我们看到了今天的第一个目标，`pymain_header`，没错，这个函数会输出我们在进入 python repl 时显示的那几行关于版本的信息。

```C
    fprintf(stderr, "Python %s on %s\n", Py_GetVersion(), Py_GetPlatform());
    if (config->site_import) {
        fprintf(stderr, "%s\n", COPYRIGHT);
    }
```

> COPYRIGHT 是一个定义在 main.c 中的宏。

之后我们就会看到通往下一个阶段的大门，一个 if-else if-else 语句，用来调用不同运行模式下的函数。

```C
    if (config->run_command) {
        *exitcode = pymain_run_command(config->run_command);
    }
    else if (config->run_module) {
        *exitcode = pymain_run_module(config->run_module, 1);
    }
    else if (main_importer_path != NULL) {
        *exitcode = pymain_run_module(L"__main__", 0);
    }
    else if (config->run_filename != NULL) {
        *exitcode = pymain_run_file(config);
    }
    else {
        *exitcode = pymain_run_stdin(config);
    }
```

因为我们是在终端运行的，所以会进入到 `pymain_run_stdin` 函数中去，跳过对能否交互和错误处理的部分，我们最终会进入到这一行：

```C
    int run = PyRun_AnyFileExFlags(stdin, "<stdin>", 0, &cf);
```

我们不妨猜测一下，应该马上就要开始运行了。可以看出，这个函数是对标准输入和文件输入做了统一，当处于交互式模式（REPL）、用标准输入传递脚本文件（./python.exe < hello.py）或是正常的运行脚本文件时都可以使用这个函数。在对文件系统的编码进行转换后，这个函数继续调用 `_PyRun_AnyFileObject`。

```C
    PyObject *filename_obj;
    if (filename != NULL) {
        filename_obj = PyUnicode_DecodeFSDefault(filename);
        if (filename_obj == NULL) {
            PyErr_Print();
            return -1;
        }
    }
    else {
        filename_obj = NULL;
    }
    int res = _PyRun_AnyFileObject(fp, filename_obj, closeit, flags);
```

在这个函数中，与运行直接相关的代码是下面这个条件语句：

```C
    int res;
    if (_Py_FdIsInteractive(fp, filename)) {
        res = _PyRun_InteractiveLoopObject(fp, filename, flags);
        if (closeit) {
            fclose(fp);
        }
    }
    else {
        res = _PyRun_SimpleFileObject(fp, filename, closeit, flags);
    }
```

对于可交互环境，它会调用 `_PyRun_InteracticeLoopObject` 函数，从命名就可以看出，这里面会包括一个循环用于不断进行交互；而如果是执行一个脚本文件，则会调用下面这个函数。因为我们的目标是找到 `>>>` 是在哪里输出的，所以我们继续进入上面这个函数。一进入这个函数，我们就看到了我们想要的东西：

```C
    PyObject *v = _PySys_GetAttr(tstate, &_Py_ID(ps1));
    if (v == NULL) {
        _PySys_SetAttr(&_Py_ID(ps1), v = PyUnicode_FromString(">>> "));
        Py_XDECREF(v);
    }
    v = _PySys_GetAttr(tstate, &_Py_ID(ps2));
    if (v == NULL) {
        _PySys_SetAttr(&_Py_ID(ps2), v = PyUnicode_FromString("... "));
        Py_XDECREF(v);
    }
```

是的，我们看到了 `>>>`，不过别急，离它被输出到终端模拟器还有好一段距离。这段代码将系统的提示符，`sys.ps1` 和 `sys.ps2` 分别设置为 `>>>` 和 `...`。由于对 sys 模块的属性操作非常频繁，所以有专门的辅助函数来完成设置，这两个函数定义在 Python 目录下的 sysmoudle.c 中。再往下看，我们果然看到了一个 do-while 循环，在 Guido 的教程里，我们可以看到最初使用的是 for 循环，不知道为什么改为了 do-while 循环，也许是使从无限循环用 break 跳出，do-while 循环结束的条件更清晰。在这个循环里，很明显我们需要关注下面这个函数：

```C
     ret = PyRun_InteractiveOneObjectEx(fp, filename, flags);
```

进入这个函数后，我一眼就看到了 `pyrun_one_parse_ast` 和 `run_mod`，根据我开发解释器的经验，前者是用来进行语法分析，建立 AST （Abstract Syntax Tree，抽象语法树）的，后者则是用来执行字节码的。据此，我对 Python 的编译器架构有了一定的猜测，Python 是用 Parser 来驱动 Lexer，而不是同一级别顺次执行。进入 `pyrun_one_parse_ast` 函数，我们又看到了熟悉的 `sys.ps1`、`sys.ps2`，这个函数先设置了编码和两个提示符，之后作为参数传递给了 `_PyParser_InteractiveASTFromFile` 函数，后者作为包装函数又调用了 `_PyPegen_run_parser_from_file_pointer`。

> 好深的调用关系qwq。

在这个名字超长的函数里，我注意到了几行关键代码：

```C
    struct tok_state *tok = _PyTokenizer_FromFile(fp, enc, ps1, ps2);
    Parser *p = _PyPegen_Parser_New(tok, start_rule, parser_flags, PY_MINOR_VERSION,
                                    errcode, arena);
    result = _PyPegen_run_parser(p);
```

也就是说，CPython 是由 Parser 驱动 Tokenizer，Tokenizer 驱动 Lexer 的。第一行代码是获取一个 Tokenizer，第二行是创建一个新的 Parser，第三行则是执行这个 parser。

> Pegen 是 Parser generator 的缩写，与 Python Parser 的设计有关，之后的文章再写。

我们现在只想知道 `>>>` 是怎么来的，所以直接查看 `_PyPegen_run_parser` 的代码，而这个函数是对 `_PyPegen_parse` 的包装。在这个函数里，我们又看到了 if-else if-else 的代码结构来区分不同的执行方式：

```C
    if (p->start_rule == Py_file_input) {
        result = file_rule(p);
    } else if (p->start_rule == Py_single_input) {
        result = interactive_rule(p);
    } else if (p->start_rule == Py_eval_input) {
        result = eval_rule(p);
    } else if (p->start_rule == Py_func_type_input) {
        result = func_type_rule(p);
    }
```

我们是交互模式，所以进入 `interactive_rule` 函数，发现里面有一块没有被条件语句或循环语句包围的作用域：

```C
    { // statement_newline
        if (p->error_indicator) {
            p->level--;
            return NULL;
        }
        D(fprintf(stderr, "%*c> interactive[%d-%d]: %s\n", p->level, ' ', _mark, p->mark, "statement_newline"));
        asdl_stmt_seq* a;
        if (
            (a = statement_newline_rule(p))  // statement_newline
        )
        {
            D(fprintf(stderr, "%*c+ interactive[%d-%d]: %s succeeded!\n", p->level, ' ', _mark, p->mark, "statement_newline"));
            _res = _PyAST_Interactive ( a , p -> arena );
            if (_res == NULL && PyErr_Occurred()) {
                p->error_indicator = 1;
                p->level--;
                return NULL;
            }
            goto done;
        }
        p->mark = _mark;
        D(fprintf(stderr, "%*c%s interactive[%d-%d]: %s failed!\n", p->level, ' ',
                  p->error_indicator ? "ERROR!" : "-", _mark, p->mark, "statement_newline"));
    }
```

### 进入 Tokenizer

根据注释我们可以知道，这个作用域是用来处理新一行的。这里代码有些复杂，`D()` 是一个宏，用来在调试模式下输出一些内容，分析一下可以发现，prompt 是在每一行的开头输出的，所以我们的目标应该是 `statement_newline_rule` 函数。进入这个函数后，里面的内容印证了我前文中的猜测，Python 的 Tokenizer 是由 Parser 驱动的，理由就是这个函数：`_PyPegen_fill_token`。从名字可以看出，这个函数是用来填充 token 来供 Parser 解析的，函数内的前三行代码也反映了这一功能：

```C
    struct token new_token;
    _PyToken_Init(&new_token);
    int type = _PyTokenizer_Get(p->tok, &new_token);
```

新建，初始化，获取一气呵成。因为目前为止还没有看到获取输入的部分，所以我们还没有触及到输出提示符的位置。而根据编译器的性质，直接获取输入的都是 lexer，所以我们需要继续向下。

> 代码真的好复杂呜呜。

进入 `_PyTokenizer_Get` 函数，发现其是对 `tok_get` 的简单包装。后者对需要获取的 token 类型进行了判断，因为我们什么都没输入，所以不属于 f-string 类型，所以会调用 `tok_get_normal_mode` 函数。

```C
    if (current_tok->kind == TOK_REGULAR_MODE) {
        return tok_get_normal_mode(tok, current_tok, token);
    } else {
        return tok_get_fstring_mode(tok, current_tok, token);
    }
```

> f-string 是 Python 的一种特性，支持格式化字符串。
> ```Python
> name = "Alice"
> age = 30
> formatted_string = f"My name is {name} and I am {age} years old."
> ```

在函数中我们看到了一个无限的 for 循环：

```C
        for (;;) {
            c = tok_nextc(tok);
            if (c == ' ') {
                col++, altcol++;
            }
            else if (c == '\t') {
                col = (col / tok->tabsize + 1) * tok->tabsize;
                altcol = (altcol / ALTTABSIZE + 1) * ALTTABSIZE;
            }
            else if (c == '\014')  {/* Control-L (formfeed) */
                col = altcol = 0; /* For Emacs users */
            }
            else if (c == '\\') {
                   if ((c = tok_continuation_line(tok)) == -1) {
                    return MAKE_TOKEN(ERRORTOKEN);
                }
            }
            else {
                break;
            }
        }
```

很明显可以看出，这个循环是获取缩进的，因为 Python 是一个缩进敏感的语言，而 `tok_nextc` 则很明显是我们接下来的目标。

> 据 Guido 说，他把 Python 设计成这样是为了让程序员们能够好好格式化他们的代码，因为以前的编辑器自动格式化功能总是没办法使人满意。

里面又是一个无限循环，与上同理，我们的关注目标是：`rc = tok->underflow(tok);`，rc 是我们获取的 token，而赋值右边的这个代码，如果你熟悉 C 语言的话就会认出，`tok->underflow` 是一个函数指针，它以 tok 作为参数调用了某个函数，那它究竟调用了哪个函数？

因为 tok 的一个字段是函数指针，所以我们有必要回头关注一下 tok 的初始化过程。在 `_PyTokenizer_FromFile` 中，tok->underflow 被设置为了 `&tok_underflow_interactive`。

```C
    if (ps1 || ps2) {
        tok->underflow = &tok_underflow_interactive;
    } else {
        tok->underflow = &tok_underflow_file;
    }
```

> 这并不是一个 C 语言的教程，所以我假定你对函数指针有一定的了解，并简单使用过它。

也就是说 `tok->underflow(tok)` 会调用 tok_underflow_interactive(tok)，而在这个函数中会调用 PyOS_Readline 函数，经过初始化和获取可读锁后，调用了 PyOS_StdioReadline 函数。

```C
tok_underflow_interactive:
    char *newtok = PyOS_Readline(tok->fp ? tok->fp : stdin, stdout, tok->prompt);

PyOS_Readline:
    if (PyOS_ReadlineFunctionPointer == NULL) {
        PyOS_ReadlineFunctionPointer = PyOS_StdioReadline;
    }

    _PyOS_ReadlineTState = tstate;
    Py_BEGIN_ALLOW_THREADS
    PyThread_acquire_lock(_PyOS_ReadlineLock, 1);

    if (!isatty(fileno(sys_stdin)) || !isatty(fileno(sys_stdout)) || !_Py_IsMainInterpreter(tstate->interp))
    {
        rv = PyOS_StdioReadline(sys_stdin, sys_stdout, prompt);
    }
    else {
        rv = (*PyOS_ReadlineFunctionPointer)(sys_stdin, sys_stdout, prompt);
    }
```

这里你可能会疑惑为什么需要一个条件判断，而且条件判断执行的内容还是一样的。据 Guido 所说，因为 PyOS_ReadlineFunctionPointer 是一个公开的 C api，所以可以编写一个 C 扩展来自定义它，这会方便那些希望 GUI 能够处理 python 输入的开发者来更好地将 python 嵌入到他们自己的程序中。更多信息可以查看 Guido 自己的教程。

不论如何，我们都进入到了 PyOS_StdioReadline 函数里，跳过关于 Windows console 的宏，我们就看到了最终的目标：
```C
    fflush(sys_stdout);
    if (prompt) {
        fprintf(stderr, "%s", prompt);
    }
    fflush(stderr);
```

到了这里，你可以长舒一口气了，不过 prompt 是在哪里被设置的呢？我们先追溯这个参数第一次被传入的位置，我们刚刚在寻找 `tok->underflow` 是何时被绑定的时，最终确定是在 `_PyTokenizer_FromFile` 函数里，如果你观察仔细的话，你会发现它的上一行就是对提示符的赋值：

```C
  tok->prompt = ps1;
  tok->nextprompt = ps2;
```
呜呼，所以我们知道了 Python 是如何初始化，REPL 中的 `>>>` 是怎么打出来的，多么寻常而又不寻常的一条 `fprintf` 语句啊。那么这次的文章就到这里，之后我们可以再来看看 Python 是怎么运行起来的（lexer，tokenizer，parser）。

等等，我还有一个疑问，就是在 `pymain_run_python` 函数中的 if-else if-else 语句中：它的下面还有一行：

```C
if (config->run_command) {
        *exitcode = pymain_run_command(config->run_command);
    }
    else if (config->run_module) {
        *exitcode = pymain_run_module(config->run_module, 1);
    }
    else if (main_importer_path != NULL) {
        *exitcode = pymain_run_module(L"__main__", 0);
    }
    else if (config->run_filename != NULL) {
        *exitcode = pymain_run_file(config);
    }
    else {
        *exitcode = pymain_run_stdin(config);
    }

    pymain_repl(config, exitcode);
```

其内部调用的是 `PyRun_AnyFileFlags`，这个函数是 `pymain_run_stdin` 调用的函数的一个宏包装：

```C
#define PyRun_AnyFileFlags(fp, name, flags) \
    PyRun_AnyFileExFlags((fp), (name), 0, (flags))
```

但是在 `pymain_run_stdin` 中调用 `PyRun_AnyFileExFlags` 时其 flags 前的参数也是 0，我不知道为什么还保留 pymain_run_repl 的原因是什么，是为了向后兼容吗？欢迎和我交流你的想法～。

## 参考资料

1. Guido van Rossum [Yet another guided tour of CPython](https://paper.dropbox.com/doc/Yet-another-guided-tour-of-CPython-XY7KgFGn88zMNivGJ4Jzv)
2. Louie Lu [A guide from parser to objects, observed using GDB](https://hackmd.io/@klouielu/ByMHBMjFe?type=view)
3. Anthony Shaw [Your Guide to the CPython Source Code](https://realpython.com/cpython-source-code-guide/#establishing-runtime-configuration)
4. [Python devguide](https://devguide.python.org/internals/exploring/)
5. [Python's Compiler design](https://devguide.python.org/internals/compiler/)
6. [PEP 432](https://peps.python.org/pep-0432/)
7. [PEP 587](https://peps.python.org/pep-0587/)
