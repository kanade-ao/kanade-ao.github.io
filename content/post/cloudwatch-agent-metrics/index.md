---
title: CloudWatch AgentによるEC2メトリック収集
description: メモリ使用率、ディスク使用率などのメトリック収集方法を紹介する
date: 2026-01-09 10:00:00+0900
tags:
  - aws
  - cloudwatch
  - ec2
categories:
  - web
  - aws
---

## はじめに

EC2の運用において、CPU使用率やネットワークI/O、ディスクI/Oといった **「標準メトリクス」** は、特別な設定なしで自動的にCloudWatchへ送信されます。

しかし、**「メモリ使用率」** や **「ディスク空き容量」** といったOS内部のメトリクスは、標準メトリクスには含まれていません。
これらを監視するためには、CloudWatch Agentをインストールして収集する必要があります。

CloudWatch Agentは、サーバーからメトリクス、ログ、トレースを収集するソフトウェアコンポーネントです。

本記事では、Auto Scaling Group配下で複数のEC2インスタンスが稼働している環境を想定し、
SSM Parameter Storeを活用してCloudWatch Agentを効率的に設定・管理する方法を紹介します。

## 設定ファイル (config.json)

```json
{
    "metrics": {
        "append_dimensions": {
            "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
            "InstanceId": "${aws:InstanceId}"
        },
        "metrics_collected": {
            "mem": {
                "measurement": [
                    "mem_used_percent"
                ],
                "metrics_collect_interval": 60
            },
            "disk": {
                "measurement": [
                    "disk_used_percent"
                ],
                "resources": [
                    "/"
                ],
                "metrics_collect_interval": 60
            }
        }
    }
}
```

CloudWatch Agentの設定ファイルを見てみましょう。

- append_dimensions
  - ディメンション設定をASGグループ名＋EC2インスタンスIDの組み合わせにより、ASG配下のEC2ごとに、独自のメトリックが収集できます。
- metrics_collected
  - メモリ使用率とディスク使用率を60秒間隔でデータを収集します。

## 実装手順
### IAMロールの作成

EC2インスタンスにアタッチされているIAMロールに、「CloudWatchAgentServerPolicy」と「AmazonSSMManagedInstanceCore」ポリシーが含まれる必要があります。

### Agentのインストール

```shell
sudo dnf install amazon-cloudwatch-agent -y
```

### SSMパラメータの作成

設定ファイルをSSMパラメータストアに登録すると、保守性が向上します。AWS CLIを使用する場合、以下のコマンドでアップロードできます

```shell
aws ssm put-parameter \
    --name "AmazonCloudWatch-Config-MyASG" \
    --type "String" \
    --value file://config.json \
    --overwrite
```

### 設定の適用

```shell
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c ssm:AmazonCloudWatch-Config-MyASG \
    -s
```

パラメータの詳細を説明します。
- -a fetch-config: 設定を読み込む
- -m ec2: EC2モードで動作
- -c file:config.json: 設定ファイルのパスを指定
- -s: サービスを起動

### 動作確認

ステータスが `running` になっていることを確認します。

```shell
systemctl status amazon-cloudwatch-agent
```

## まとめ

CloudWatch Agentを利用することで、OS内部の重要なリソース状況をモニタリングできます。

必要に応じて、CloudWatch Alarmsと連携して「ディスク使用率80%超えた場合SNSでメール通知する」などの運用自動化へ繋げていくと良いでしょう。
