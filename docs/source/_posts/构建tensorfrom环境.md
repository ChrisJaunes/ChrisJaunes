---
title: 构建TensorFlow环境
date: 2021-02-22 02:43:37
tags: 
- TensorFlow
- Blockly
excerpt: 这是关于TensorFlow环境搭建的文章, 介绍了在线体验TensorFlow、利用Docker搭建TensorFlow、利用pip安装TensorFlow。
---
## 方案一： 在线体验TensorFlow

借助 Colaboratory（简称 Colab），可在浏览器中编写和执行 Python 代码，Python程序可以直接在浏览器中运行    

利用google colab可以在线体验TensorFlow
网站(需要翻墙)
https://colab.research.google.com/github/tensorflow/docs-l10n/blob/master/site/zh-cn/tutorials/quickstart/beginner.ipynb

## 方案二(推荐) 利用docker构建环境

### 第一步 安装Docker，参考网站

1. 利用以下方式安装Docker

    1. 在Ubuntu下安装Docker

        参考链接: [https://www.runoob.com/docker/ubuntu-docker-install.html](https://www.runoob.com/docker/ubuntu-docker-install.html) 

        利用命令安装Docker
        ```bash
        curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
        ```
        如果不是root用户建议执行
        ```bash
        sudo usermod -aG docker chrisjaunes
        ```

    2. 在Windows下安装Docker

        参考链接: [https://www.runoob.com/docker/windows-docker-install.html](https://www.runoob.com/docker/windows-docker-install.html)


2. 检查Docker是否安装完成
    
    1. 利用 docker -v 检查docker是否安装完成
    
        参考情况($代表命令提示符， 下同)

        ```bash
        $ docker -v
        Docker version 20.10.3, build 48d30b5
        ```
    2. 检查Docker是否运行(Linux)

        ```bash
        systemctl status docker.service
         ```

        Active为active (running)时表示Docker正在运行

    3. 运行Docker, 可能需要root权限, 可以使用sudo
        ```bash
        systemctl start docker.service
        ```


### 第二步 安装TensorFlow Docker镜像
1. Tensorflow Docker相关

    参考链接[https://tensorflow.google.cn/install/docker](https://tensorflow.google.cn/install/docker)

    Docker代码库[https://hub.docker.com/r/tensorflow/tensorflow/](https://hub.docker.com/r/tensorflow/tensorflow/)
    
    映像版本:
    |标记|说明 |
    | ------ | ------ |
    |latest| TensorFlow CPU 二进制映像的最新版本。（默认版本）|
    |nightly| TensorFlow 映像的每夜版。（不稳定）|
    |version| 指定 TensorFlow 二进制映像的版本，例如：2.1.0 |
    |devel| TensorFlow master 开发环境的每夜版。包含 TensorFlow 源代码。|
    |custom-op| 用于开发 TF 自定义操作的特殊实验性映像。|

    每个基本标记都有会添加或更改功能的变体：

    |标记变体|说明|
    | --- | --- |
    |tag-gpu|支持 GPU 的指定标记版本。|
    |tag-jupyter|针对 Jupyter 的指定标记版本|


2. 利用以下方式安装TensorFlow Docker
    
    在Docker运行情况下

    1. 安装二进制映像的最新版本

    ```bash
    docker pull tensorflow/tensorflow:latest
    ```

    2. 安装二进制映像的最新版本，有GPU支持，jupyter支持(推荐)

    ```bash
    docker pull tensorflow/tensorflow:latest-gpu-jupyter
    ```


### 第三步 启动 TensorFlow Docker 容器

1. 要启动配置 TensorFlow 的容器，请使用以下命令格式：

    ```shell
    docker run [-it] [--rm] [-p hostPort:containerPort] tensorflow/tensorflow[:tag] [command]
    ```

    例如:
    使用 TensorFlow Docker
    ```bash
    docker run -it tensorflow/tensorflow bash
    ```
    使用 TensorFlow:nightly-jupyter 启动 Jupyter 服务器(推荐)
    ```bash
    docker run -it -p 8888:8888 tensorflow/tensorflow:nightly-jupyter
    ```
2. 使用CPU时去除告警信息

    告警信息为: 

        Could not load dynamic library 'libcudart.so.11.0'; dlerror: libcudart.so.11.0: cannot open shared object file: No such file or directory
    
    解决方案：

    ```python
    import os
    os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'
    import tensorflow as tf
    ```
 
## 方案三 利用pip构建环境
1.  系统要求

    1. Python 3.5-3.8 若要支持 Python 3.8，需要使用 TensorFlow 2.2 或更高版本。
    
    2. pip 19.0 或更高版本（需要 manylinux2010 支持）
    
    3. Ubuntu 16.04 或更高版本（64 位）
    
        macOS 10.12.6 (Sierra) 或更高版本（64 位）（不支持 GPU）
    
        Windows 7 或更高版本（64 位）
        * 适用于 Visual Studio 2015、2017 和 2019 的 Microsoft Visual C++ 可再发行软件包
    
    4. Raspbian 9.0 或更高版本
    
    5. GPU 支持需要使用支持 CUDA® 的卡（适用于 Ubuntu 和 Windows）

2. 检查系统要求
    
    ```bash
    $ python3 --version
    $ pip3 --version
    ```

    如果不满足， 则安装

    ```bash
    $ sudo apt update
    $ sudo apt install python3-dev python3-pip python3-venv
    ```

3. 创建虚拟环境(推荐)

    参考资料: https://tensorflow.google.cn/install/pip

4. 安装tensorflow
    
    ```bash
    pip install --upgrade tensorflow
    ```
    
    如果速度太慢的换可以考虑换个源
    
    ```bash
    pip install --upgrade tensorflow  --default-timeout=100 -i http://pypi.douban.com/simple/ --trusted-host pypi.douban.com
    ```

    如果报错Memory Error 可以考虑参数: --no-cache-dir
