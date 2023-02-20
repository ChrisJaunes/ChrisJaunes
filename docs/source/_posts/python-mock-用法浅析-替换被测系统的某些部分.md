---
title: python-mock-用法浅析
date: 2023-02-20 20:28:40
categories: [Python, Mock]
tags: [Python, Mock, test]
excerpt: 本文浅析了Python下的Mock库的常见用法
---

Mock允许我们用模拟对象替换被测系统的某些部分，并对它们的使用方式做出断言。

## 替换被测系统的某些部分

### 替换函数

```python
from unittest import mock
def main(*args, **kwargs):
    print("[main] 执行", args, kwargs)
    return False

# 直接调用
def test_1():
    print("[test1] start")
    print(main())
    print("[test1] finish")

# 替代调用
@mock.patch("__main__.main")
def test_2(mock_main: mock.MagicMock):
    print("[test2] start")
    mock_main.return_value = True
    print(main())
    print("[test2] finish")

if __name__ == "__main__":
    test_1()
    test_2()
```
output:
```text
[test1] start
[main] 执行 () {}
False
[test1] finish
[test2] start
True
[test2] finish
```

### 替换其余文件的函数
a.py
```python
def main_step(*args, **kwargs):
    print("[main_step", args, kwargs)
    return False

def main(*args, **kwargs):
    print("[main] 执行", args, kwargs)
    return main_step(*args, **kwargs)
```
test.py
```python
from unittest import mock
from a import main

def test_1():
    print("[test1] start=========================")
    print(main())
    print("[test1] finish========================")

@mock.patch("a.main_step")
def test_2(mock_a_main_step: mock.MagicMock):
    print("[test2] start=========================")
    mock_a_main_step.return_value = True
    print(main())
    print("[test2] finish========================")

@mock.patch("__main__.main")
def test_3(mock_a_main: mock.MagicMock):
    print("[test3] start=========================")
    mock_a_main.return_value = True
    print(main())
    print("[test3] finish========================")
    
if __name__ == "__main__":
    test_1()
    test_2()
    test_3()
```
output:
```text
[test1] start=========================
[main] 执行 () {}
[main_step () {}
False
[test1] finish========================
[test2] start=========================
[main] 执行 () {}
True
[test2] finish========================
[test3] start=========================
True
[test3] finish========================
```

### 替换类中的函数

a.py
```python
class MainClass(object):
    def __init__(self):
        self.a = 100
        
    def get_a(self):
        print("[MainClass get_a] start")
        print("[MainClass get_a] finish")
        return self.a
```
test.py
```python
from unittest import mock
from a import MainClass

def test_1():
    print("[test1] start=========================")
    print(MainClass().get_a())
    print("[test1] finish========================")

@mock.patch("a.MainClass.get_a")
def test_2(mock_a_main_class_get_a: mock.MagicMock):
    print("[test2] start=========================")
    mock_a_main_class_get_a.return_value = True
    print(MainClass().get_a())
    print("[test2] finish========================")
    
if __name__ == "__main__":
    test_1()
    test_2()
```
output:
```text
[test1] start=========================
[MainClass get_a] start
[MainClass get_a] finish
100
[test1] finish========================
[test2] start=========================
True
[test2] finish========================
```

### 替换模块
a.py
```python
def get_env():    
    from environs import Env
    env = Env()
    print(env.get("env_name"))
```
test.py
```python
import sys
from unittest import mock
from a import get_env

def test_1():
    print("[test1] start=========================")
    try:
        get_env()
        print("Successful")
    except:
        print("Error")
    print("[test1] finish========================")

def test_2():
    print("[test2] start=========================")
    environs_env = mock.Mock()
    environs_env.get.return_value = "TEST"
    environs = mock.Mock()
    environs.Env.return_value = environs_env
    sys.modules['environs'] = environs
    try:
        get_env()
        print("Successful")
    except:
        print("Error")
    print("[test2] finish========================")

if __name__ == "__main__":
    test_1()
    test_2()
```
output:
```text
[test1] start=========================
Error
[test1] finish========================
[test2] start=========================
TEST
Successful
[test2] finish========================
```
