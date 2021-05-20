---
title: python-docx-源码浅析-创建Document
date: 2021-05-20 10:48:50
categories:
- [Python, Docx]
tags:
- Python
excerpt: 浅析了python-docx库
---



## Docx介绍

在介绍Docx之前，先了解一下Office Open XML， 参考：[wikipadia](https://zh.wikipedia.org/wiki/Office_Open_XML)

Office Open XML是Microsoft Office的默认格式，其文档就是docx文件。

我们如果用zip打开docx，可以看到一系列文件夹，如下图

```shell
$ tree word
word
├── [Content_Types].xml
├── _rels
├── customXml
│   ├── _rels
│   │   └── item1.xml.rels
│   ├── item1.xml
│   └── itemProps1.xml
├── docProps
│   ├── app.xml
│   └── core.xml
└── word
    ├── _rels
    │   └── document.xml.rels
    ├── document.xml
    ├── endnotes.xml
    ├── fontTable.xml
    ├── footnotes.xml
    ├── media
    │   ├── image1.png
    │   ├── image2.png
    ├── numbering.xml
    ├── settings.xml
    ├── styles.xml
    ├── theme
    │   └── theme1.xml
    └── webSettings.xml
```

我们有非常多的手段去解析xml文件，但分析OOXML仍然不是一件简单事情，而python-docx是一个可以用于解析docx的库。



## Document构建

Working with Documents ： [参考](https://python-docx.readthedocs.io/en/latest/user/documents.html)

我们可以构建一个新的document

```python
from docx import Document
document = Document()
```

可以通过文件名打开一个docx文件

```python
from docx import Document
document = Document("filename.docx")
```

也可以通过文件流打开一个docx文件

```python
from docx import Document
with open("filename.docx", "rb") as f:
    stream = StringIO(f.read())
doucment = Document(stream)
stream.close()
```



### Document构建流程

构建一个新的document

{% plantuml %}
actor User
participant "module docx\n file api.py\n func Documnet" as Document

participant "module docx.opc\n file package.py\n class OpcPackage\n func open" as Package

participant "module docx.opc\n file pkgreader.py\n class PackageReader\n func from_file" as PackageReader

participant "module docx.opc\n file phys_pkg.py\n class PhysPkgReader\n func ~__new__.py" as PhysPkgReader

User-> Document: 用户要求创建一个新的Document, 参数为docx

activate Document

Document -> Document: 假如没有提供docx参数\n 就设置docx为docx/templates/default.docx

Document -> Package: 参数为docx

activate Package

Package -> PackageReader: 调用PackageReader.from_file\n这是一个静态函数\n 参数为pkg_file, 其实就是docx

activate PackageReader

PackageReader -> PhysPkgReader: 获取DirPkgReader或者ZipPkgReader

activate PhysPkgReader

PhysPkgReader -> PhysPkgReader : 通过判断来确定是文件路径、字符流、zip文件等的哪一种

PhysPkgReader -> PackageReader: 返回对应的解析器，由于重写了new，实际上是以DirPkgReader或者ZipPkgReader作为其实例

deactivate PhysPkgReader

PackageReader -> PackageReader: 获取了content_types、pkg_srels、sparts

PackageReader -> Package: 返回构造的PackageReader 实例

deactivate PackageReader

Package -> Package: 调用Unmarshaller对其反序列化

Package -> Document: 返回package

deactivate Package

Document -> Document: 检查其类型是否为CT.WML_DOCUMENT_MAIN

Document -> User: 返回一个document

deactivate Document 

{% endplantuml %}



### Document

源代码：[GitHub](https://github.com/python-openxml/python-docx/blob/master/docx/api.py#L17)

```python
def Document(docx=None):
    docx = _default_docx_path() if docx is None else docx
    document_part = Package.open(docx).main_document_part
    if document_part.content_type != CT.WML_DOCUMENT_MAIN:
        tmpl = "file '%s' is not a Word file, content type is '%s'"
        raise ValueError(tmpl % (docx, document_part.content_type))
    return document_part.document


def _default_docx_path():
    _thisdir = os.path.split(__file__)[0]
    return os.path.join(_thisdir, 'templates', 'default.docx')
```

可以看出如果没有提供docx的话，会加载模板，也就是docx/templates/default.docx文件

然后会调用Package.open将docx加载到内存

接着会判断文件类型是不是CT.WML_DOCUMENT_MAIN

然后就把document返回给调用者。



### Package.open

源代码：[GitHub](https://github.com/python-openxml/python-docx/blob/master/docx/opc/package.py#L122)

```python
@classmethod
def open(cls, pkg_file):
    pkg_reader = PackageReader.from_file(pkg_file)
    package = cls()
    Unmarshaller.unmarshal(pkg_reader, package, PartFactory)
    return package

```

这是一个类方法，通过PackageReader.from_file可以获得一个解析器，然后利用这个解析器去对pkg_file进行解析



### pkgreader

源代码：[GitHub](https://github.com/python-openxml/python-docx/blob/master/docx/opc/pkgreader.py#L27)

```python
    @staticmethod
    def from_file(pkg_file):
        """
        Return a |PackageReader| instance loaded with contents of *pkg_file*.
        """
        phys_reader = PhysPkgReader(pkg_file)
        content_types = _ContentTypeMap.from_xml(phys_reader.content_types_xml)
        pkg_srels = PackageReader._srels_for(phys_reader, PACKAGE_URI)
        sparts = PackageReader._load_serialized_parts(
            phys_reader, pkg_srels, content_types
        )
        phys_reader.close()
        return PackageReader(content_types, pkg_srels, sparts)
```



### PhysPkgReader

源代码：[GitHub](https://github.com/python-openxml/python-docx/blob/master/docx/opc/phys_pkg.py#L18)

```python
class PhysPkgReader(object):
    def __new__(cls, pkg_file):
        if is_string(pkg_file):
            if os.path.isdir(pkg_file):
                reader_cls = _DirPkgReader
            elif is_zipfile(pkg_file):
                reader_cls = _ZipPkgReader
            else:
                raise PackageNotFoundError(
                    "Package not found at '%s'" % pkg_file
                )
        else:
            reader_cls = _ZipPkgReader

        return super(PhysPkgReader, cls).__new__(reader_cls)
```

PhysPkgReader将返回一个解析器，根据情况而言：

如果pkg_file是字符串，那么 判断pkg_file，是文件路径的话使用\_DirPkgReader，是zip文件使用_ZipPkgReader, 其他抛出异常

如果不是字符串，一律使用\_ZipPkgReader

注意到

```python
super(PhysPkgReader, cls).__new__(reader_cls)
```

这句话其实等价于

```python
object.__new__(reader_cls)
```

等价于利用reader_cls对象作为本类的实例，这是一个非常有趣的点。

不过这里无论是DirPkgReader、还是ZipPkgReader都是PhysPkgReader的子类。

