---
title: 构建 Tensorflow+BlocklyJupyter 环境
date: 2021-06-11 23:52:04
tags: 
- Tensorflow
- Blockly
- Jupyter

excerpt: 这是关于 Tensorflow+BlocklyJupyter 服务器环境搭建的文章。
---

## 构建服务器环境

1. 由于需要支持TensorFlow，利用google提供的Docker

    参考: [构建tensorfrom环境](https://chrisjaunes.github.io/ChrisJaunes/2021/02/22/%E6%9E%84%E5%BB%BAtensorfrom%E7%8E%AF%E5%A2%83/)

    按照方案二，安装Docker镜像，然后启动Docker，在Docker中继续操作

    本处采用的docker镜像是 tensorflow/tensorflow:latest

2. 安装nodejs

    因为使用了typescript，所以要安装nodejs， 参考:[ubuntu快速安装最新版nodejs，只需2步](https://blog.csdn.net/Ezreal_King/article/details/78659810)

    ```
    curl -sL https://deb.nodesource.com/setup_14.x | bash -
    apt-get install -y nodejs
    ```
    nodejs 版本 v14.17.0
    npm 版本 6.14.13

    可以考虑使用国内源, 参考:[npm安装慢的俩种方法](https://blog.csdn.net/qq_43500877/article/details/89449837)
    
    ```
    npm config set registry http://registry.npm.taobao.org
    ```

3. 安装typescript 
    ```
    npm install typescript -g
    ```

4. 安装git
    tensorflow/tensorflow:latest默认没有git 需要安装
    ```
    apt-get install git
    ```

5. 下载BlocklyJupyter

    ```
    git clone git@github.com:ChrisJaunes/blockly_teaching.git
    ```
    当然有可能出现以下报错:
    ```
    Warning: Permanently added 'github.com,52.74.223.119' (RSA) to the list of known hosts.
    git@github.com: Permission denied (publickey).
    fatal: Could not read from remote repository.

    Please make sure you have the correct access rights
    and the repository exists.
    ```
    这是公钥出现问题了，你可以添加公钥到你的账户下面，也可以用https链接。
    
    采用https链接：
    ```
    git clone https://github.com/ChrisJaunes/blockly_teaching.git
    ```

6. 进入BlocklyJupyter
    ```
    cd blockly_teaching/BlocklyJupyter
    ```

7. 安装jupyter、jupyterlab等库
   目录下有一个requirements.txt的文件，这个文件定义了所需要的jupyter等lib及其版本
   ```
    pip install -r requirements.txt --default-timeout=100 -i http://pypi.douban.com/simple/ --trusted-host pypi.douban.com
   ```
   由于使用root账户，有些包无法自动安装，可以手动安装。
   
   不过通常按照jupyter和jupyterlab就可以了，其余根据需要安装。

8.  npm安装
    ```
    npm init
    npm install --save-dev
    npm run build
    ```
    如果出现 codemirror 的报错
    {% spoiler "codemirror报错的具体信息" %}
    ```shell
    node_modules/@jupyterlab/codemirror/lib/editor.d.ts:1:23 - error TS2688: Cannot find type definition file for '@types/codemirror/addon/search/searchcursor'.

    1 /// <reference types="@types/codemirror/addon/search/searchcursor" />
                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    node_modules/@jupyterlab/codemirror/lib/editor.d.ts:2:24 - error TS7016: Could not find a declaration file for module 'codemirror'. '/data/blockly_teaching/jupyterlab-3.0.16/examples/cell/node_modules/codemirror/lib/codemirror.js' implicitly has an 'any' type.
    Try `npm i --save-dev @types/codemirror` if it exists or add a new declaration (.d.ts) file containing `declare module 'codemirror';`

    2 import CodeMirror from 'codemirror';
                            ~~~~~~~~~~~~

    node_modules/@jupyterlab/codemirror/lib/tokens.d.ts:2:24 - error TS7016: Could not find a declaration file for module 'codemirror'. '/data/blockly_teaching/jupyterlab-3.0.16/examples/cell/node_modules/codemirror/lib/codemirror.js' implicitly has an 'any' type.
    Try `npm i --save-dev @types/codemirror` if it exists or add a new declaration (.d.ts) file containing `declare module 'codemirror';`

    2 import CodeMirror from 'codemirror';
                            ~~~~~~~~~~~~


    Found 3 errors.

    npm ERR! code ELIFECYCLE
    npm ERR! errno 1
    npm ERR! @jupyterlab/example-cell@3.0.11 build: `tsc -p src && webpack`
    npm ERR! Exit status 1
    npm ERR!
    npm ERR! Failed at the @jupyterlab/example-cell@3.0.11 build script.
    npm ERR! This is probably not a problem with npm. There is likely additional logging output above.

    npm ERR! A complete log of this run can be found in:
    npm ERR!     /root/.npm/_logs/2021-06-11T17_06_02_154Z-debug.log
    ```
    {% endspoiler %}
    ```
    $ npm init
    $ npm install --save-dev
    $ npm run build (出错, 按提示安装新依赖包)
    $ npm i --save-dev @types/codemirror
    $ npm run build(成功)
    ```

9.  运行
    ```
    python main.py --allow-root --ip=0.0.0.0
    ```