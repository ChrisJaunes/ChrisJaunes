---
title: Python-Json-源码浅析-Load
date: 2021-05-04 10:59:43
categories: 
- [Python, Json]
tags:
- Python
excerpt: 本文浅析了Python下的Json库，介绍了Json库下Load过程、Json文法等。
---

# Load
## 概要

JSON 是一个轻量的数据交换格式

关于JSON的使用，参考 https://www.runoob.com/python/python-json.html

本文浅析了Python的Json库实现，其分析代码依照CPython的Lib库

分析源码位于:[github](https://github.com/python/cpython/tree/main/Lib/json)

## Load 中涉及的函数

### Load 时序图
<style> 
img{
    background-color: wheat;
    opacity : 1 !important;
}
</style>

解析JSON字符串默认流程
{% plantuml %}
@startuml
actor user
participant "module json\n file ~__init__.py\n func load" as load
participant "module json\n file ~__init__.py\n func loads" as loads
participant "module json\n file decoder.py\n class JSONDecoder\n func decode" as decode
participant "module json\n file decoder.py\n class JSONDecoder\n func raw_decode" as raw_decode
participant "module json\n file scanner.py\n class py_make_scanner\n func scan_once" as scan_once
user -> load : 读取JSON文件

activate load
load -> loads : 转成JSON字符串
deactivate load

user -> loads : 读取JSON字符串 (str、bytes or bytearray)

activate loads
loads -> decode : 调用解析器的decode函数\n默认解析器是JSONDecoder

activate decode
decode -> raw_decode : 在JSON Decoder中\n decode将解析任务交给raw_decode

activate raw_decode
raw_decode -> scan_once : 调用scan_once函数进行解析\n默认为py_make_scanner.scan_once\n这是个闭包

activate scan_once
scan_once -> scan_once : 使用递归下降算法解析JSON格式的字符串
scan_once -> raw_decode :  返回 (obj, end)
deactivate scan_once

raw_decode -> decode : 返回 (obj, end)
deactivate raw_decode

decode -> loads : 返回 obj
deactivate decode
loads -> user : 返回 obj
@enduml
{% endplantuml %}

### Load函数

json.load 用于解码 存有JSON数据的文件

源代码位于 [github](https://github.com/python/cpython/blob/main/Lib/json/__init__.py#L274)

```python
def load(fp, *, cls=None, object_hook=None, parse_float=None,
        parse_int=None, parse_constant=None, object_pairs_hook=None, **kw):
    return loads(fp.read(),
        cls=cls, object_hook=object_hook,
        parse_float=parse_float, parse_int=parse_int,
        parse_constant=parse_constant, object_pairs_hook=object_pairs_hook, **kw)
```

这个函数读取文件fp，然后将解析fp.read()的任务交给了loads


### Loads函数

json.loads 用于解码 JSON 数据。该函数返回 Python 字段的数据类型。

源代码位于 [github](https://github.com/python/cpython/blob/main/Lib/json/__init__.py#L299)

Loads 要接受一个需要解析的对象s，这个对象的类型可以是str，也可以是bytes或者bytearray

{% plantuml %}
(*) --> "处理要解析的对象"

if "s的类型是str" then
  -left->[true] "检查是否采用了UTF-8 BOM编码"
  --> if "s开头出现了\\ufeff" then
    -left->[true] "抛出JSONDecodeError异常"
  endif
else
  -->[false] "检查s的类型"
  --> if "s的类型既不 bytes 也不是 bytearray" then
    -right->[true] "抛出TypeError异常"
  else
    -->[false] "对s进行解码"
  endif
endif
{% endplantuml %}

如果就提供了s一个参数，使用默认的解析器_default_decoder，调用_default_decoder.decode(s)

否则构造解析器：

如果没有提供解析器类，则使用JSONDecoder作为解析器类，否则使用自定义的解析器， 自定义解析器要是JSONDecoder的子类。

消耗参数cls，其余参数当作解析器类的构造参数提供， 其余参数在py_make_scanner会进行介绍

特别注释: The ``encoding`` argument is ignored and deprecated.

```python
def loads(s, *, cls=None, object_hook=None, parse_float=None,
        parse_int=None, parse_constant=None, object_pairs_hook=None, **kw):
    if isinstance(s, str):
        if s.startswith('\ufeff'):
            raise JSONDecodeError("Unexpected UTF-8 BOM (decode using utf-8-sig)",
                                  s, 0)
    else:
        if not isinstance(s, (bytes, bytearray)):
            raise TypeError(f'the JSON object must be str, bytes or bytearray, '
                            f'not {s.__class__.__name__}')
        s = s.decode(detect_encoding(s), 'surrogatepass')

    if (cls is None and object_hook is None and
            parse_int is None and parse_float is None and
            parse_constant is None and object_pairs_hook is None and not kw):
        return _default_decoder.decode(s)
    if cls is None:
        cls = JSONDecoder
    if object_hook is not None:
        kw['object_hook'] = object_hook
    if object_pairs_hook is not None:
        kw['object_pairs_hook'] = object_pairs_hook
    if parse_float is not None:
        kw['parse_float'] = parse_float
    if parse_int is not None:
        kw['parse_int'] = parse_int
    if parse_constant is not None:
        kw['parse_constant'] = parse_constant
    return cls(**kw).decode(s)
```

### decode函数

decode是JSONDecoder类的成员方法

源代码位于 [github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L332)

```python
class JSONDecoder(object):
    ...
    def decode(self, s, _w=WHITESPACE.match):
        obj, end = self.raw_decode(s, idx=_w(s, 0).end())
        end = _w(s, end).end()
        if end != len(s):
            raise JSONDecodeError("Extra data", s, end)
        return obj
```

decode把解析任务交给了raw_decode方法

### raw_decode函数

raw_decode是JSONDecoder类的成员方法

源代码位于 [github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L343)

```python
class JSONDecoder(object):
    def raw_decode(self, s, idx=0):
        try:
            obj, end = self.scan_once(s, idx)
        except StopIteration as err:
            raise JSONDecodeError("Expecting value", s, err.value) from None
        return obj, end
```
raw_decode把解析任务交给了scan_once函数

scan_once是scanner.make_scanner(self)

### make_scanner方法

make_scanner是scanner文件中的函数

源代码位于 [github](https://github.com/python/cpython/blob/main/Lib/json/scanner.py#L73)

```python
make_scanner = c_make_scanner or py_make_scanner
```

make_scanner实际指向c_make_scanner 或者 py_make_scanner

c_make_scanner采用的c语言编写，源代码[github](https://github.com/python/cpython/blob/main/Modules/_json.c)

py_make_scanner采用python语言编写，源代码[github](https://github.com/python/cpython/blob/main/Lib/json/scanner.py#L15)

这两俩才是真正进行对JSON字符串解析的，为了分析这两个，需要进一步了解JSON格式文法

## JSON 格式文法

### JSON 文法

_JSON_ -> **true** | **false** | **null** | _number_ | _str_ | _JSONArray_ | _JSONObject_

_JSONObject_ -> **{** _ObjectMembers_ **}**

_ObjectMembers_ -> _Members_ | **ε**

_Members_ -> _Member_ _A_

_A_ -> **,** _Member_ |**ε**

_Member_ -> _Key_ **:** _Value_

_Key_ -> _str_

_Value_ -> _JSON_

_JSONArray_ -> **\[** _ArrayElements_ **\]**

_ArrayElements_ -> _Elements_ | **ε**

_Elements_ -> _Element_ _B_

_B_ -> **,** _Element_ | **ε**

_Element_ -> _JSON_

### 编写JSON解析器

### _scan_once 函数

这是一个闭包, 其自由变量位于py_make_scanner中

parse_object = context.parse_object = JSONDecoder中的parse_object = JSONObject，
JSONObject位于[github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L136)

parse_array = context.parse_array = JSONDecoder中的parse_array = JSONArray，
JSONArray位于[github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L217)

parse_string = context.parse_string = JSONDecoder中的parse_string = scanstring，
scanstring位于[github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L69)

match_number = NUMBER_RE.match = re.compile(r'(-?(?:0|[1-9]\d*))(\.\d+)?([eE][-+]?\d+)?', (re.VERBOSE | re.MULTILINE | re.DOTALL))

strict = context.strict = JSONDecoder中的strict = True or loads中kw存有strict参数

parse_float = context.parse_float = JSONDecoder中的parse_float = float or (parse_float = loads 中的 parse_float)

parse_int = context.parse_int = JSONDecoder中的parse_int = int or (parse_int = loads 中的 parse_int)

parse_constant = context.parse_constant = JSONDecoder中的parse_constant = \_CONSTANTS.\_\_getitem\_\_ or (parse_constant = loads 中的 parse_constant)

object_hook = context.object_hook= JSONDecoder中的object_hook = object_hook = loads 中的 object_hook

object_pairs_hook = context.object_pairs_hook= JSONDecoder中的object_pairs_hook = object_pairs_hook = loads 中的 object_pairs_hook

memo = context.memo= JSONDecoder中的memo = {}

采用了递归下降分析法

```python
def _scan_once(string, idx):
    try:
        nextchar = string[idx]
    except IndexError:
        raise StopIteration(idx) from None

    if nextchar == '"':
        return parse_string(string, idx + 1, strict)
    elif nextchar == '{':
        return parse_object((string, idx + 1), strict,
            _scan_once, object_hook, object_pairs_hook, memo)
    elif nextchar == '[':
        return parse_array((string, idx + 1), _scan_once)
    elif nextchar == 'n' and string[idx:idx + 4] == 'null':
        return None, idx + 4
    elif nextchar == 't' and string[idx:idx + 4] == 'true':
        return True, idx + 4
    elif nextchar == 'f' and string[idx:idx + 5] == 'false':
        return False, idx + 5

    m = match_number(string, idx)
    if m is not None:
        integer, frac, exp = m.groups()
        if frac or exp:
            res = parse_float(integer + (frac or '') + (exp or ''))
        else:
            res = parse_int(integer)
        return res, m.end()
    elif nextchar == 'N' and string[idx:idx + 3] == 'NaN':
        return parse_constant('NaN'), idx + 3
    elif nextchar == 'I' and string[idx:idx + 8] == 'Infinity':
        return parse_constant('Infinity'), idx + 8
    elif nextchar == '-' and string[idx:idx + 9] == '-Infinity':
        return parse_constant('-Infinity'), idx + 9
    else:
        raise StopIteration(idx)
```
### parse_string 函数

parse_string函数默认是c_scanstring 或者 py_scanstring 函数

这两个都是用来解析字符串的，c_scanstring采用c语言编写，py_scanstring采用python语言编写。

下面讨论py_scanstring函数， 源代码位于: [github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L69)

py_scanstring使用了正则表达式 ，其中：
```python
_m = re.compile(r'(.*?)(["\\\x00-\x1f])', re.VERBOSE | re.MULTILINE | re.DOTALL)
```
代表匹配以" \ \x00-\x1f结尾的最小字符串。groups会返回捕获的分组，这里设置了两个分组, 因此terminator会获得第二个分组的信息

然后

{% plantuml %}
(*) --> "通过正则式捕获合法的分组\n chunk = _m(s, end)"
if "没能捕获成功\n chunk is None" then
  -right->[true] "抛出JSONDecodeError异常"
else 
  -->[false] "利用groups获取捕获的分组, terminator会获得第二个分组\n content, terminator = chunk.groups()"
  --> "if content: _append(content)"
  if " 检查terminator是不是引号" then
    -right->[true] "1 break"
  else 
    if "检查是不是控制符号 \n terminator != \\\\ " then
      -right->[true] if "严格模式" then 
        -right->[true] "2 break" 
      else 
        -right->[false] "_append(terminator) continue"
      endif
    else
      -->[false 必定是反斜杠] 获取下一个字符
      if "检查是不是u" then
        -right->[不是] 不在反斜杠字典中抛出异常
      else
        -->[是] "对unicode解码"
        --> "_append(char)"
      endif
    endif
  endif
endif
{% endplantuml %}

### JSONObject 函数

parse_object函数默认是 JSONObject 函数

下面讨论JSONObject函数， 源代码位于: [github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L136)

在JSONObject中 跳过 空格、制表、回车、换行等空白字符是有必要的， 采用正则表达式来完成上述操作。

JSONObject的key 必须是string, 其原因可以参考 [stackoverflow](https://stackoverflow.com/questions/9304528/why-json-allows-only-string-to-be-a-key)

根据文法
_JSONObject_ -> **{** _ObjectMembers_ **}**

_ObjectMembers_ -> _Members_ | **ε**

处理JSONObject = {} 的情况
{% plantuml %}
(*) --> "1 获取下一个字符"
if "下一个字符不为引号" then
  --> if "下一个字符是空白字符" then
    -right->[true] "通过正则表达式跳过空白字符"
    --> "获取下一个字符"
    --> "----"
  else
    -->[false] "----"
  endif
  --> if "下一个字符是不是}括号" then
    -right->[是] "处理并返回结果"
  else
    --> if "下一个字符是不是引号" then
    -right->[不是] "引发异常"
{%endplantuml%}

根据文法

_Member_ -> _Key_ **:** _Value_

_Key_ -> _str_

_Value_ -> _JSON_ | **null**

{% plantuml %}
(*) --> "获取 key \n key, end = scanstring(s, end, strict)"
--> "1 跳过空白字符"
if "下一个字符是不是 : " then
  -right->[不是] "1 引发异常 JSONDecodeError"
else
  --> "2 跳过空白字符 "
  --> "获取value \n value, end = scan_once(s, end) "
  --> "3 跳过空白字符"
  if "下一个字符是不是 } " then
    -right->[是] "break"
  else
    if "下一个字符是不是 ," then
    -right->[不是] "2 引发异常 JSONDecodeError"
    else
      -->[是] "跳过空白字符"
      if "下一个字符是不是 引号" then
      -right->[不是] "3 引发异常 JSONDecodeError"
      endif
    endif
  endif
endif
{%endplantuml%}

如果 object_pairs_hook 不为空， 利用 object_pairs_hook 对于 pairs 进行处理

object_pairs_hook 接受一个 list 对象， 其中list的每一个元素是一个tuple

以下给一个demo:

```python
class Test(object):
    def __init__(self, li):
        self.li = li

    def __str__(self):
        return "[Test object_pairs_hook] : " + self.li.__str__()


def test():
    t = json.loads('{"123": 234}', object_pairs_hook=Test)
    print(t)
    print(type(t))
```
如果 object_hook 不为空， 利用 object_hook 对于dict(pairs) 进行处理
```python
class Test(object):
    def __init__(self, di):
        self.di = di

    def __str__(self):
        return "[Test object_pairs_hook] : " + self.di.__str__()


def test():
    t = json.loads('{"123": 234}', object_hook=Test)
    print(t)
    print(type(t))
```
### JSONArray 函数

parse_array函数默认是 JSONArray 函数

下面讨论 JSONArray 函数， 源代码位于: [github](https://github.com/python/cpython/blob/main/Lib/json/decoder.py#L217)

根据文法

_JSONArray_ -> **\[** _ArrayElements_ **\]**

_ArrayElements_ -> _Elements_ | **ε**

_Elements_ -> _Element_ _B_

_B_ -> **,** _Element_ | **ε**

_Element_ -> _JSON_

先判断是不是空的list, 利用正则表达式跳过空白字符

不断调用scan_once获取 _JSON_

### 判断null、true、false

```python
def _scan_once(string, idx):
  ...
  if nextchar == 'n' and string[idx:idx + 4] == 'null':
    return None, idx + 4
  elif nextchar == 't' and string[idx:idx + 4] == 'true':
    return True, idx + 4
  elif nextchar == 'f' and string[idx:idx + 5] == 'false':
    return False, idx + 5
```

### 处理数字

利用正则表达式进行匹配
```python
NUMBER_RE = re.compile(
    r'(-?(?:0|[1-9]\d*))(\.\d+)?([eE][-+]?\d+)?',
    (re.VERBOSE | re.MULTILINE | re.DOTALL))
```
然后利用groups()获取分组，根据分组情况转成int或者float, 如果无法转换，检查是否是NaN、Infinity、-Infinity

