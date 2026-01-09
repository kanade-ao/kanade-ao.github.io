---
title: Lambdaによるクラス初期化の注意事項
description: Lambdaハンドラーメソッドの外部宣言されたクラスなどの変数は再利用が可能で、予想外の不具合につながります。
date: 2025-03-17 12:00:00+0900
tags:
  - aws
  - lambda
categories:
  - aws
---

## はじめに

Lambdaでは、パフォーマンスを向上させるために、一度起動した実行環境を一定期間破棄せずに保持し、
後続のリクエストで再利用する仕組みがあります。

この仕組みにおいて、ハンドラー関数の外側で初期化された変数やクラスのインスタンスは、
リクエスト処理が完了してもメモリ上から消去されず、次のリクエスト処理時にそのまま共有されます。

通常、この仕様はデータベース接続の使い回しなどに利用されます。
しかし、「特定のリクエストに紐づくデータ（ユーザーの検索結果や許可リストなど）」をグローバル変数として保持しまうと、
前のユーザーのデータが次のユーザーのリクエストに混入するという重大なバグを引き起こします。

他人の個人情報が見えてしまう「情報漏洩」にもつながります。

下記のDemoコードを一緒に分析しましょう。

## Demoコード
```py
Database = {
    'User': {
        'user_1': {'equipment_1'},
        'user_2': {'equipment_2'},
    },
    'Equipment': {
        'equipment_1': {'name': 'equipment_info_1'},
        'equipment_2': {'name': 'equipment_info_2'},
    }
}

class DataGetter:
    def __init__(self):
        # 【Bugの原因】
        # __init__ はLambdaのコンテナ起動時（コールドスタート）に一度だけ実行される
        # そのため、self.allowed_equipment_id はリクエスト間で共有される
        self.allowed_equipment_id = set()

    def execute(self, body):
        user_id = body['userId']
        # リクエストした機器にユーザー権限があるかどうか
        for equipment_id in body['requestEquipmentId']:
            if equipment_id in Database['User'][user_id]:
                self.allowed_equipment_id.add(equipment_id)
                
        # 本来はそのリクエストで許可されたものだけを返すべきだが、
        # 前回のリクエストで追加されたIDも残っているため、
        # 本来権限がないはずの機器情報まで返却してしまう
        return [Database['Equipment'][equipment_id] for equipment_id in self.allowed_equipment_id]

# ハンドラーの外でクラスをインスタンス化している（グローバル変数）
# これにより、AWS Lambdaのコンテナが再利用される限り、
# 同じ data_getter オブジェクトが使い回される
data_getter = DataGetter()

def lambda_handler(event, context):
    response = data_getter.execute(event['body'])
    return response
```

## 発生現象

一回目のリクエストは、【機器1】のみの権限を持つ【ユーザー1】が実行する
```json
{
    "body": {
        "userId": "user_1",
        "requestEquipmentId": ["equipment_1", "equipment_2"]
    }
}
```
下記のレスポンスが返却される。これは正しい結果です。
```json
[
    {"name": "equipment_info_1"}
]
```

この実行によって、メモリ上のインスタンス内には `{'equipment_1'}` が残ったままとなります。

二回目のリクエストは、【機器2】のみの権限を持つ【ユーザー2】が実行する

```json
{
    "body": {
        "userId": "user_1",
        "requestEquipmentId": ["equipment_2"]
    }
}
```
下記のレスポンスが返却される。【機器1】がリクエストされていないにも関わらず、レスポンスの中に返却されてしまいました。

【ユーザー2】は本来閲覧権限のない【機器1】の情報を取得できてしまい、越権アクセス・情報漏洩が発生しました。

```json
[
    {"name": "equipment_info_1"},
    {"name": "equipment_info_2"}
]
```

## 修正方法

この問題を解決する最も確実な方法は、状態を持つオブジェクトの初期化をハンドラー関数の内部（ローカルスコープ）で行うことです。

```py
# 【修正ポイント】
# グローバル領域でのインスタンス化を削除する。
# data_getter = DataGetter()  <-- 削除

def lambda_handler(event, context):
    # 【修正ポイント】
    # ハンドラー内部でインスタンス化を行う
    # これにより、リクエストごとに新しい「空の」data_getterが生成される
    local_data_getter = DataGetter()

    response = local_data_getter.execute(event['body'])
    return response
```


