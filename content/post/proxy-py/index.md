---
title: デザインパターン-Proxy
description: PythonにおけるProxyの実装パターンまとめ
date: 2025-06-23 19:00:00+0900
tags:
  - python
categories:
  - design-pattern
---

## Proxy - 定義

> A class that functions as an interface to a particular resource. That resource may be remote, expensive to construct, or may require logging or some other added functionality.

- A proxy has the same interface as the underlying object
- To create a proxy, simply replicate the existing interface of an object
- Add relevant functionality to the redefined member functions
- Different proxies (communication, logging, caching, etc.) have completely different behaviors

## 前言

ソフトウェア設計において、あるオブジェクトに「直接アクセスさせたくない」「まだアクセスしたくない」という場面は意外と多くあります。
- セキュリティ：特定の権限がないと実行させたくない。
- パフォーマンス：初期化に時間がかかるので、本当に必要になるまで作りたくない。
- ロギング：実行の前後にログを残したい。

これらを解決するのがProxyパターンです。 Proxyは、対象となるオブジェクトと同じインターフェースを持ち、クライアントと対象オブジェクトの間に入って制御を行います。

Pythonによる具体的なコード例を使って、代表的な2つのProxyパターンの実装をまとめます。

## 実装方法
### Protection Proxy - アクセス制御
```py
class SensitiveDocument:
    def __init__(self, content):
        self.content = content

    def read(self):
        print(f"Reading content: {self.content}")

class DocumentProxy:
    def __init__(self, user, document):
        self.user = user
        self.document = document

    def read(self):
        # アクセス制御ロジック: 役職がManagerの場合のみ許可
        if self.user.role == 'Manager':
            self.document.read()
        else:
            print(f"Access Denied: User '{self.user.name}' does not have permission.")

class User:
    def __init__(self, name, role):
        self.name = name
        self.role = role

        
>>> employee = User('Alice', 'Staff')
>>> secret_doc = SensitiveDocument('Company Secrets...')
>>> proxy = DocumentProxy(employee, secret_doc)
>>> proxy.read()
Access Denied: User 'Alice' does not have permission.

>>> manager = User('Bob', 'Manager')
>>> proxy_for_manager = DocumentProxy(manager, secret_doc)
>>> proxy_for_manager.read()
Reading content: Company Top Secrets...
```

ここでの`DocumentProxy`は、本物の`SensitiveDocument`と同じ`read()`メソッドを持っています。

クライアントから呼び出された際に、まずユーザーの役職をチェックします。そして、「Managerである」という条件を満たしたときだけ、
内部で保持している本物の`SensitiveDocument`に処理を委譲しています。

これにより、「`SensitiveDocument`自体に、権限確認のロジックを書かなくて済む」 という大きなメリットが生まれます。 

`SensitiveDocument`クラスは純粋に「中身を表示する」ことだけに集中し、 面倒な権限管理はすべてProxyが担当する。
これによって、コードの役割分担がきれいに実現できます。

### Virtual Proxy - 遅延初期化

```py
import time

class RealDatabase:
    def __init__(self):
        print("Connecting to the database... ")
        # 接続に時間がかかることをシミュレート
        time.sleep(3)
        print("Connected to Database!")

    def query(self, sql):
        print(f"Executing SQL: {sql}")

class LazyDatabaseProxy:
    def __init__(self):
        self._real_db = None

    def query(self, sql):
        # 実際にクエリを投げるときまで、接続(インスタンス化)を遅らせる
        if self._real_db is None:
            self._real_db = RealDatabase()
        self._real_db.query(sql)

# クライアントコード
def run_analytics(db):
    print("Preparing analytics report...")
    # ここで初めてデータベース接続が発生する
    db.query("SELECT * FROM sales")
    print("Report finished.")


# この時点ではまだ接続しない（3秒待たされない）
>>> db_proxy = LazyDatabaseProxy()

# ユーザーの操作などで、必要になったタイミングで処理を開始
>>> run_analytics(db_proxy)
Preparing analytics report...
Connecting to the database... 
Connected to Database!
Executing SQL: SELECT * FROM sales
Report finished.
```

`RealDatabase`クラスは初期化の時点でデータベースへの接続処理を行ってしまうため、インスタンスを作っただけで待ち時間が発生します。
仮にインスタンスを作った後で、一度もクエリを実行しなかった場合、その接続コストは完全に無駄になります。

LazyDatabaseProxyは、実際の`query()`メソッドが呼ばれるその瞬間まで、本物の`RealDatabase`に接続しません。

インターフェースが同じであるため、既存のビジネスロジックを一切変更することなく、
「必要なときだけ接続する」というパフォーマンス改善を導入することができます。