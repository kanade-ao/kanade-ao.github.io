---
title: デザインパターン-Singleton
description: PythonにおけるSingletonの実装パターンまとめ
date: 2025-06-15 14:00:00+0900
tags:
  - python
categories:
  - design-pattern
---

## Singleton - 定義

> A component which is instantiated only once.

- Different realizations of Singleton: custom allocator, decorator, metaclass
- Laziness is easy, just init on first request

## 前言

PythonでDesign Patternの一つであるSingletonを実装する方法はいくつかありますが、それぞれ挙動やメリット・デメリットが異なります。
今回は、代表的な3つの実装方法についてコードと共にまとめます。

## 実装方法
### `__new__` を使用した実装
```py
import random

class Database:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            # 親クラスの__new__を呼び出してインスタンスを作成
            cls._instance = super(Database, cls).__new__(cls, *args, **kwargs)
        return cls._instance

    def __init__(self):
        self.id = random.randint(1, 1000)
        print('Database id = ', self.id)
```

Pythonでは、`__new__` がインスタンスを返した後、必ず `__init__` が呼ばれます。
つまり、Singletonとして同じオブジェクトを返しているにもかかわらず、`Database()` を呼ぶたびに `__init__` が再実行されます。

上記のコードの場合、`Database()` を2回呼ぶと、インスタンスは同一ですが `self.id` は毎回書き換わってしまいます。

### Decoratorを使用した実装

```py
import random

def singleton(class_):
    instance = {}

    def get_instance(*args, **kwargs):
        if class_ not in instance:
            instance[class_] = class_(*args, **kwargs)
        return instance[class_]

    return get_instance

@singleton
class Database:
    def __init__(self):
        self.id = random.randint(1, 1000)
        print('Database id = ', self.id)
```

Decoratorを使うことで、Singletonのロジックをクラス定義から分離できます。
また、この方法であればインスタンス作成時のみ `__init__` が走るため、前述の「`__init__` 毎回呼ばれる問題」は発生しません。

注意点: `@singleton`を適用した時点で、`Database`はクラスではなく`get_instance`関数に置き換わります。
そのため、クラスメソッドや静的メソッドへのアクセス、isinstanceメソッドなどで直感的でない挙動をする場合があります。

```py
@singleton
class Database:
    def __init__(self):
        self.name = "MySQL"

    @staticmethod
    def version():
        return "1.0.0"

>>> print(f"Type of Database: {type(Database)}")
Type of Database: <class 'function'>

>>> print(f"Database represents: {Database}")
Database represents: <function singleton.<locals>.get_instance at 0x1046531c0>

>>> print(Database.version())
AttributeError: 'function' object has no attribute 'version'

>>> isinstance(db, Database)
TypeError: isinstance() arg 2 must be a type, a tuple of types, or a union
```

### Metaclassを使用した実装

```py
class Singleton(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            # インスタンスがなければ作成して保存
            cls._instances[cls] = super(Singleton, cls).__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=Singleton):
    def __init__(self):
        print('Init database')
```

Metaclassは「クラスを作るためのクラス」です。
クラスのインスタンス化は、実はメタクラスの`__call__`メソッドの呼び出しです。

Metaclassをオーバーライドすることで、`Database`はそのままクラスとしての性質を保ちます。
`__init__`も最初の1回しか呼ばれません。（`super().__call__` の中で `__new__` と `__init__` がセットで呼ばれるため）

実務でSingletonを実装する場合、このMetaclassアプローチが最も推奨されます。