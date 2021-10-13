---
title: Python-文档字符串
date: 2021-10-13 15:38:04
categories: 
- [Python, 内置函数]
- [programing_language, 比较学习]
tags: python
excerpt: 本文介绍了python中的文档字符串规范
---
# 文档字符串
## 简介

Donald Knuth 在 "Literate Programming" 中提到   
>Instead of imagining that our main task is to instruct a computer what to do, let us concentrate rather on explaining to human beings what we want a computer to do.

如何更好去和别人沟通我们希望计算机去做什么？我们自然会想到使用注释，然而混杂在代码里面的注释并不好管理。python提供了文档字符串来帮助我们管理注释，但是光提供语法是不够的，我们需要一个规范来指导我们不同层级的文档字符串应该包含什么。

PEP 257 记录了与Python文档字符串相关的语义和约定

## PEP 257

[PEP257链接](https://www.python.org/dev/peps/pep-0257)

这个PEP的目的是标准化关于文档字符串的高级结构：它们应该包含什么以及如何表达(不涉及文档字符串中的任何标记语法)。PEP包含约定，而不是规定或者语法，因此你可以不遵守这些约定，但这样你的内容看起来可能不够整齐简洁，而一些类型docutils的文档字符串处理系统需要你遵守约定才能更好的工作。

### 定义

文档字符串在脚本、模块、函数、类或者方法定义中作为第一条语句出现的字符串，这样的字符串成为该对象的 \_\_doc\_\_ 特殊属性。

Python 代码中其他地方出现的字符串可以当作文档，但它们不能被Python解析器识别，也不能作为运行时的对象属性访问(没有分配给 \_\_doc\_\_)

为了保持一致性，PEP257 建议 文档字符串使用三重引号。

即使是一行也应该使用三重引号，虽然单引号和双引号也可以被正确识别，但不推荐这么做

### 单行引号

[链接](https://www.python.org/dev/peps/pep-0257/#one-line-docstrings)

### 多行引号

多行文档字符串由一个摘要行组成，后面跟着一个空行，后面跟着一个更详细的描述。
自动索引工具可能会使用摘要行。摘要行可以与开始引号在同一行，也可以在下一行。摘要行适合一行，并通过一个空行与文档字符串的其余部分分开。整个文档字符串的缩进与其第一行的引号相同。

### 脚本的文档字符串

{% spoiler "脚本的文档字符串" %}
The docstring of a script (a stand-alone program) should be usable as its "usage" message, printed when the script is invoked with incorrect or missing arguments (or perhaps with a "-h" option, for "help"). Such a docstring should document the script's function and command line syntax, environment variables, and files. Usage messages can be fairly elaborate (several screens full) and should be sufficient for a new user to use the command properly, as well as a complete quick reference to all options and arguments for the sophisticated user.
{% endspoiler %}

脚本(独立程序)的文档字符串应该可用作其“使用方法”消息，当使用不正确或缺失的参数调用脚本时显示该文本字符串(或者可能使用“-h”选项，表示“help”)。这样的文档字符串应该记录脚本的功能、命令行语法、环境变量和文件。使用消息可以是相当详细的(占满几个屏幕)，并且应该足以指导新用户正确使用命令，以及高级用户也可以完整快速的索引到全部选项和参数。

### 模块的文档字符串

{% spoiler "模块的文档字符串" %}
The docstring for a module should generally list the classes, exceptions and functions (and any other objects) that are exported by the module, with a one-line summary of each. (These summaries generally give less detail than the summary line in the object's docstring.) The docstring for a package (i.e., the docstring of the package's \_\_init\_\_.py module) should also list the modules and subpackages exported by the package.
{% endspoiler %}

模块的文档字符串通常应列出由该模块导出的类、异常和函数（以及任何其他对象），并在一行中列出每一项。（这些摘要通常比对象文档字符串中的摘要行提供的详细信息更少）。软件包的文档字符串（即包的__init__.py模块的文档字符串）还应列出软件包导出的模块和子包。

### 函数和方法的文档字符串
{% spoiler "函数和方法的文档字符串" %}
The docstring for a function or method should summarize its behavior and document its arguments, return value(s), side effects, exceptions raised, and restrictions on when it can be called (all if applicable). Optional arguments should be indicated. It should be documented whether keyword arguments are part of the interface.
{% endspoiler %}

函数或方法的文档字符串应总结其行为并记录其参数、返回值、副作用、引发的异常以及何时可以调用它的限制（所有如果适用）。应指明可选参数。应该记录关键字参数是否是接口的一部分。

### 类的文档字符串

{% spoiler "类的文档字符串" %}
The docstring for a class should summarize its behavior and list the public methods and instance variables. If the class is intended to be subclassed, and has an additional interface for subclasses, this interface should be listed separately (in the docstring). The class constructor should be documented in the docstring for its \_\_init\_\_ method. Individual methods should be documented by their own docstring. 
If a class subclasses another class and its behavior is mostly inherited from that class, its docstring should mention this and summarize the differences. Use the verb "override" to indicate that a subclass method replaces a superclass method and does not call the superclass method; use the verb "extend" to indicate that a subclass method calls the superclass method (in addition to its own behavior).
{% endspoiler %}

类的文档字符串应该总结其行为并列出公共方法和实例变量。如果该类旨在被子类化，并且有一个子类的附加接口，则该接口应该单独列出(在文档字符串中)。类构造函数应该在文档字符串中记录其\_\_init\_\_方法。私有方法应该由他们自己的文档字符串记录。
如果一个类对另一个类进行子类化，并且其行为主要从该类继承，则其文档字符串应该提及这一点并总结出差异。 使用动词“override”表示子类方法替换超类方法，不调用超类方法; 使用动词“extend”来表示一个子类方法调用超类方法(除了自己的行为)