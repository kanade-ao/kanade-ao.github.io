---
title: CloudWatch AgentによるEC2ログ収集
description: CloudWatch AgentでEC2のログをCloudWatch Logsに転送・集約する
date: 2026-01-14 14:00:00+0900
tags:
  - aws
  - cloudwatch
  - ec2
categories:
  - web
  - aws
---

## はじめに

[前回の記事](https://kanade-ao.github.io/p/cloudwatch-agent%E3%81%AB%E3%82%88%E3%82%8Bec2%E3%83%A1%E3%83%88%E3%83%AA%E3%83%83%E3%82%AF%E5%8F%8E%E9%9B%86/)
では、CloudWatch Agentを使ってEC2のメモリやディスク使用率を可視化する方法を紹介しました。

今回は「ログの収集」について解説します。特にASG環境で運用している場合、ログ管理には以下のような特有の課題があります。

- EC2スケールインによるログの紛失: EC2インスタンスが自動的に終了すると、そのインスタンス内に保存されていたログファイルも一緒に消滅してしまいます。
- 調査の煩雑さ: 複数台のサーバーが稼働している環境でエラーが発生した場合、「どのサーバーで起きたのか」を特定するのは困難です。1台ずつSSH接続してgrepするのは現実的ではありません。

これらの課題は、CloudWatch Agentの設定に`logs`セクションを追加し、ログファイルをCloudWatch Logsへリアルタイムに転送・集約することで解決できます。

今回はPHPアプリケーションのログ収集を例に、具体的な設定手順を紹介します。

## 設定ファイル (config.json)

前回の`metrics`設定に加え、`logs`セクションを追加します。

```json
{
    "metrics": {...},
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": [
                    {
                        "file_path": "/var/log/httpd/access_log",
                        "log_group_name": "httpd-access-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%d/%b/%Y:%H:%M:%S %z"
                    },
                    {
                        "file_path": "/var/log/httpd/error_log",
                        "log_group_name": "httpd-error-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%a %b %d %H:%M:%S.%f %Y"
                    },
                    {
                        "file_path": "/var/log/php-fpm/error.log",
                        "log_group_name": "php-fpm-error-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%d-%b-%Y %H:%M:%S"
                    },
                    {
                        "file_path": "/var/log/php-fpm/www-error.log",
                        "log_group_name": "php-fpm-www-error-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%d-%b-%Y %H:%M:%S %Z"
                    },
                    {
                        "file_path": "/var/log/php-fpm/www-slow.log",
                        "log_group_name": "php-fpm-www-slow-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%d-%b-%Y %H:%M:%S"
                    },
                    {
                        "file_path": "/var/log/php-fpm-check.log",
                        "log_group_name": "php-fpm-check-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "UTC",
                        "timestamp_format": "%Y-%m-%d %H:%M:%S"
                    },
                    {
                        "file_path": "/var/das/html/storage/logs/laravel.log",
                        "log_group_name": "laravel-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%Y-%m-%d %H:%M:%S"
                    },
                    {
                        "file_path": "/var/app/html/storage/logs/apache-action-[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9].log",
                        "log_group_name": "apache-action-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%Y-%m-%d %H:%M:%S"
                    },
                    {
                        "file_path": "/var/app/html/storage/logs/console-command-[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9].log",
                        "log_group_name": "console-command-log",
                        "log_stream_name": "{instance_id}",
                        "timezone": "Local",
                        "timestamp_format": "%Y-%m-%d %H:%M:%S"
                    }
                ]
            }
        }
    }
}
```

collect_listの構成を見てみましょう。

- file_path

  収集したいログファイルの絶対パスを指定します。「*」を使用して、ローテーションされるログなどをまとめて指定することも可能です。

  「apache-action-[0-9]...log」のような正規表現も使用できますので、日付ローテーションされるログファイルを正確にターゲットし、無関係なファイルの誤送信が防げます。
- log_group_name

  送信先ロググループ
- log_stream_name

  ログイベントを識別するためのストリーム名
　{instance_id} というプレースホルダーを使うことで、自動的に「i-1234567890abcdef0」のようなインスタンスIDに置き換わります。これにより、ASG環境でもログの混在を防げます。
- timezone
  
  ログが出力されているタイムゾーン（Local または UTC）が指定できます。システムログとアプリログのタイムゾーンを合わせるための設定です。
- timestamp_format
  
  ログ内の時刻フォーマットを指定することで、CloudWatch Agentがログ内の時刻を解析し、CloudWatch Log上の時系列表示になります。
  指定しない場合は、EC2から転送された時刻が使用され、CloudWatch上の時刻は実際のログ時刻とのズレが生じてしまいます。

## 実装手順

手順はメトリクス収集時とほぼ同じです。設定ファイルの内容が変わるだけです。（すでに配置した場合は、ステップ3からで大丈夫です。）

### IAMロールの作成

EC2インスタンスにアタッチされているIAMロールに、「CloudWatchAgentServerPolicy」と「AmazonSSMManagedInstanceCore」ポリシーが含まれる必要があります。

### Agentのインストール

```shell
sudo dnf install amazon-cloudwatch-agent -y
```

### SSMパラメータの更新

設定ファイルをSSMパラメータストアに登録すると、保守性が向上します。AWS CLIを使用する場合、以下のコマンドでアップロードできます

```shell
aws ssm put-parameter \
    --name "AmazonCloudWatch-Config-MyASG" \
    --type "String" \
    --value file://config.json \
    --overwrite
```

### 設定の再読み込み

```shell
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c ssm:AmazonCloudWatch-Config-MyASG \
    -s
```

パラメータの詳細説明は[前回記事](https://kanade-ao.github.io/p/cloudwatch-agent%E3%81%AB%E3%82%88%E3%82%8Bec2%E3%83%A1%E3%83%88%E3%83%AA%E3%83%83%E3%82%AF%E5%8F%8E%E9%9B%86/#%E8%A8%AD%E5%AE%9A%E3%81%AE%E9%81%A9%E7%94%A8)
を参照してください。

### 動作確認

ステータスが `running` になっていることを確認します。

```shell
systemctl status amazon-cloudwatch-agent
```

設定適用後、AWSマネジメントコンソールの **CloudWatch > ロググループ** で確認します。

1. 設定した`{log_group_name}`が表示されていることを確認します。

2. ロググループに入ると、各EC2インスタンスIDごとのログストリームが作成されています。

3. ストリームをクリックすると、サーバー内のログが転送されていることが確認できます。

## まとめ

CloudWatch Agentでログ転送を行うことで、サーバーが終了してもログがCloudWatch上に永続化されるようになり、障害調査や監査対応が非常に楽になります。

また、CloudWatch Logsに集約しておけば、CloudWatch Logs Insights という機能を使い、
「特定のキーワードを含むエラーログを、全サーバー横断で検索・集計する」といった高度な分析も可能になります。
