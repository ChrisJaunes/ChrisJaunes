---
title: DRF-ModelSerializer-源码浅析-序列化
date: 2022-09-05 19:54:21
categories: [django]
tags: 
- django 
- 序列化
excerpt: 本文浅析了 DRF 中的 ModelSerializer 序列化实现
---

Django 中常常使用Model去描述数据，一般来说，每一个Model都映射一张数据库表。

但是在前后端交互的过程中, 需要将对象转化成容易传输和保持的格式，这种转化过程就是序列化的过程。

Serializer允许把QuerySet、Model Instance等复杂数据转化成Python原生数据类型，容易处理成JSON、XML或者其他内容形式。

ModelSerializer 类提供了一种快捷方式，可让自动创建具有与 Model 字段对应的字段的 Serializer 类

```python
from rest_framework.serializers import ModelSerializer
class Example(models.Model):
    example_id = models.BigAutoField(primary_key=True, verbose_name="样例ID")
    detail = models.TextField(verbose_name="详细信息")

class ExampleSerializer(ModelSerializer):
    class Meta:
        model = Example
        fields = "__all__"

example = Example(example_id=1, detail="测试样例")
ExampleSerializer(example).data
```

为了自动生成Serializer，我们需要告诉ModelSerializer需要序列化哪一个Model的哪些Field

需要提供 fields 或者 exclude 之一，不能同时提供，也不能都不提供

fields 必须是 "\_\_all\_\_" 或者是 (list, tuple) 的实例

exclude 必须是 (list, tuple) 的实例

ModelSerializer 首先需要解析出提供的 field_name

{% spoiler "解析出提供的 field_name" %}
源代码：[github](https://github.com/encode/django-rest-framework/blob/master/rest_framework/serializers.py#L1019-L1091)

ModelSerializer 使用了 get_field_names 获取到了 配置 Field

首先对 fields 和 exclude 进行校验

如果 fields 非空，那么校验 ModelSerializer 定义的 field_name 是否在 fields 中，然后返回 fields

这个校验方法非常的有意思，SerializerMetaclass创建的每个类都有一个_declared_fields代表这个类中定义的属性，通过集合相减的方式可以获得当前类定义的属性

从 Model 中获取全部的名称，排除掉exclude中的field_name

```python
class ModelSerializer(Serializer):
    ...
    def get_field_names(self, declared_fields, info):
        fields = getattr(self.Meta, 'fields', None)
        exclude = getattr(self.Meta, 'exclude', None)

        if fields and fields != ALL_FIELDS and not isinstance(fields, (list, tuple)):
            raise TypeError

        if exclude and not isinstance(exclude, (list, tuple)):
            raise TypeError

        assert not (fields and exclude), 

        assert not (fields is None and exclude is None)

        if fields == ALL_FIELDS:
            fields = None

        if fields is not None:
            required_field_names = set(declared_fields)
            for cls in self.__class__.__bases__:
                required_field_names -= set(getattr(cls, '_declared_fields', []))

            for field_name in required_field_names:
                assert field_name in fields
            return fields

        fields = self.get_default_field_names(declared_fields, info)

        if exclude is not None:
            for field_name in exclude:
                assert field_name not in self._declared_fields
                assert field_name in fields
                fields.remove(field_name)
        return fields
    ...
```
{% endspoiler %}

ModelSerializer 接着 创建 Field 实例

{% spoiler "构造 Field 实例" %}
fields 是 Serializer 中的方法，被cached_property修饰成属性，其中用到了get_fields去获取field，在ModelSerializer中将重写get_fields

fields 可以通过 extra_kwargs 中配置 source 去指定Model来源字段名称

```python
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):
    @cached_property
    def fields(self):
        """
        A dictionary of {field_name: field_instance}.
        """
        fields = BindingDict(self)
        for key, value in self.get_fields().items():
            fields[key] = value
        return fields


class ModelSerializer(Serializer):
    def get_fields(self):
        """
        Return the dict of field names -> field instances that should be
        used for `self.fields` when instantiating the serializer.
        """
        if self.url_field_name is None:
            self.url_field_name = api_settings.URL_FIELD_NAME

        assert hasattr(self, 'Meta')
        assert hasattr(self.Meta, 'model')
        if model_meta.is_abstract_model(self.Meta.model):
            raise ValueError

        declared_fields = copy.deepcopy(self._declared_fields)
        model = getattr(self.Meta, 'model')
        depth = getattr(self.Meta, 'depth', 0)

        if depth is not None:
            assert depth >= 0, "'depth' may not be negative."
            assert depth <= 10, "'depth' may not be greater than 10."

        info = model_meta.get_field_info(model)
        field_names = self.get_field_names(declared_fields, info)

        extra_kwargs = self.get_extra_kwargs()
        extra_kwargs, hidden_fields = self.get_uniqueness_extra_kwargs(
            field_names, declared_fields, extra_kwargs
        )

        fields = OrderedDict()

        for field_name in field_names:
            if field_name in declared_fields:
                fields[field_name] = declared_fields[field_name]
                continue

            extra_field_kwargs = extra_kwargs.get(field_name, {})
            source = extra_field_kwargs.get('source', '*')
            if source == '*':
                source = field_name

            field_class, field_kwargs = self.build_field(
                source, info, model, depth
            )

            field_kwargs = self.include_extra_kwargs(
                field_kwargs, extra_field_kwargs
            )

            fields[field_name] = field_class(**field_kwargs)

        fields.update(hidden_fields)

        return fields
    ...
```
{% endspoiler %}

ModelSerializer 进行序列化并且获得一个 ReturnDict 对象

{% spoiler "序列化实例" %}

data是Serializer的一个属性，该方法中继续调用BaseSerializer中的data方法

```python
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):
    ...
    @property
    def data(self):
        ret = super().data
        return ReturnDict(ret, serializer=self)
    ...


class BaseSerializer(Field):
    ...
    @property
    def data(self):
        ...
        self._data = self.to_representation(self.instance)
        return self._data
    ...
```

to_representation 就是 序列化 的核心了，这是一个责任链模式 所有的Field都实现了to_representation方法

```python
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):
    ...
    def to_representation(self, instance):
        ret = OrderedDict()
        fields = self._readable_fields

        for field in fields:
            try:
                attribute = field.get_attribute(instance)
            except SkipField:
                continue

            check_for_none = attribute.pk if isinstance(attribute, PKOnlyObject) else attribute
            if check_for_none is None:
                ret[field.field_name] = None
            else:
                ret[field.field_name] = field.to_representation(attribute)

        return ret
    ...
```

{% endspoiler %}


参考文档：

1. Serializers.drf.https://www.django-rest-framework.org/api-guide/serializers/#serializers