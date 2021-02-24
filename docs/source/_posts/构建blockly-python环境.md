---
title: 构建 blockly+python 远程环境
date: 2021-02-24 19:18:04
tags: 
- Jupyter
- Python
- Blockly
- metakernel

excerpt: 这是关于 blockly+python 服务器环境搭建的文章。
---
## 简介

### 引子

Blockly是google发布的可视化编程工具,基于web技术构建

Python是一种可以在多数平台上写脚本和快速开发应用的编程语言

利用Blockly去引导Python的学习可以显著降低同学们的入门难度

但是Blockly基于web技术构建，而如何去在浏览器中直接运行Python代码是一个有趣的问题

在浏览器中运行Python代码有很多解决方案，比如skulpt

然而这些直接在浏览器中运行Python的方案目前难以解决一个问题：如何导入Python库？

比如skuplt并没有tensorflow支持，但我们想要在浏览器中运行这类代码

我们可以搭建一个服务器，然后提供API供前端使用

显然，这么容易想到的东西已经有人去做了

Jupyter Notebook是基于网页的用于交互计算的应用程序。其可被应用于全过程计算:开发、文档编写、运行代码和展示结果。

本篇主要是利用Jupyter Notebook服务去搭建一个可以使用Blockly的服务器环境

### 构建服务器环境

1. 由于需要支持TensorFlow，利用google提供的Docker

    参考: [构建tensorfrom环境](https://chrisjaunes.github.io/ChrisJaunes/2021/02/22/%E6%9E%84%E5%BB%BAtensorfrom%E7%8E%AF%E5%A2%83/)

    按照方案二，安装Docker镜像，然后启动Docker，在Docker中继续操作

2. metakernel是一个很棒的Ipythor内核，可以安装该内核

    ```bash
    pip3 install metakernel
    ```

3. 添加metakernel内核

    ```bash
    python3 -m ipykernel install --user --name metakernel
    ```

    检查jupyter已经安装的内核
    
    ```bash
    jupyter kernelspec list
    ```

4. 由于GFW、CORF、CORS、chrome的sandbox等缘故，metakernel提供的jigsaw无法使用, 我重写了一个metakernel的magic
    源代码位于：[blockly_magic](https://github.com/ChrisJaunes/blockly-kernel/blob/master/metakernel/magics/blockly_magic.py)
    
    ```bash
    wget https://raw.githubusercontent.com/ChrisJaunes/blockly-kernel/master/metakernel/magics/blockly_magic.py
    ```
    
    下载该文件，并且将其放到metakernel库的meatkernel/magics目录下

    这个目录在哪？

    方法一：利用find 或者 tree命令找一下metakernel

    方法二：打开python, 执行
    
    ```python
    import metakernel
    metakernel.__file__
    ```

    方法三：在命令行执行
    ```bash
    pip show metakernel
    ```

5. 启动jupyter

    ```bash
    jupyter notebook
    ```

    如果没有配置过允许所有ip地址访问:

    ```bash
    jupyter notebook --no-browser --ip='0.0.0.0'
    ```

    建议使用nohup命令后台运行

    找不到Token可以使用
    ```bash
    jupyter notebook list
    ```

6. 出错了。。。

    1. 在服务器系统上利用netstat命令检查端口是否被监听

        ```bash
        netstat -ano
        ```

    2. 如果(1)没有监听，检查Docker服务是否正常运行, Docker容器是否正常运行

    3. 如果(2)正常, 在Docker容器中检查Jupyter是否正常运行

    4. 如果(3)正常，检查Docker的端口映射是否正确

    5. 如果(1)在监听状态，检查接受的端口

    6. 如果(5)没有问题，检查防火墙是否开放端口




