---
title: ALBによるERR_CONNECTION_RESETエラー調査
description: 一日数回だけERR_CONNECTION_RESETエラーが発生する不具合の調査をまとめた
date: 2025-12-26 11:00:00+0900
tags:
  - aws
  - alb
categories:
  - web
  - aws
---

## 障害内容

最初は、お客様から特定のユーザーがChromeからドメインにアクセスし、ERR_CONNECTION_RESETエラーが続いているという報告がありました。

その際に、Safariに切り替えて、正常アクセスができたとの報告もあったため、偶発的な不具合だと思い込み、深く調査しませんでした。

数日後、またERR_CONNECTION_RESETエラーがありました。CloudWatchでの5XXと4XX系エラーが一切なかったため、不審と思い、ALBアクセスログから調べることにしました。

## アーキテクチャ概要

ウェブサーバーを通常のALB-EC2-RDSの三段階に設計し、CloudFormationスタックによるデプロイ計画です。お客様の要望により、Single-AZの配置が必須です。（コスト削減のため）

ビジネス関係で、テストも本番アカウントでの実施となりました。本番運用は東京リージョンで、該当リージョン下は別のシステムが稼働中の状態です。

テストのリスクを最小限するために、大阪リージョンでの先行テストにしました。しかし、大阪リージョンでデプロイ成功したIaCテンプレートが東京リージョンでは失敗しました。
原因はALBは東京リージョンでMulti-AZにする必要がありました。故に、使うことのないサブネットワークを作り、L11に追加しました。（ここは悪夢の始まりでした）

以下はIaCから一部抜粋したものです。

```yaml
Resources:
  # Application Load Balancer (ALB)
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${ProjectName}-alb'
      Scheme: internet-facing
      Type: application
      Subnets:
        - Fn::ImportValue: !Sub '${ProjectName}-PublicSubnetAIds'
        - Fn::ImportValue: !Sub '${ProjectName}-PublicSubnetBIds' # 東京リージョンデプロイのため追加したもの
      SecurityGroups:
        - Fn::ImportValue: !Sub '${ProjectName}-ALBSGId'
      LoadBalancerAttributes:
        - Key: waf.fail_open.enabled
          Value: 'true'
      Tags:
        - Key: Name
          Value: !Sub '${ProjectName}-alb'
```

## ALBアクセスログ分析

時間帯とクライアントIPから絞ってみると、1件のログのみでした。HTTPへのリクエストは正常にALBによるHTTPSへ301リダイレクトされていることがわかりました。

しかしリダイレクト発生後の3分間、ALBに後続リクエストのログが一切記録されていないです。これはALBに到達する前（TCP/TLSハンドシェイク段階）で通信が途絶えていたことが示しています。

(ドメインとIPはすべてダミーデータです)

```json
{
    "type": "http",
    "time": "2025-12-25T23:45:50.036400Z",
    "elb": "-----",
    "client_ip_port": "119.21.1.1:63228",
    "target_ip_port": "-",
    "request_processing_time": "-1.0",
    "target_processing_time": "-1.0",
    "response_processing_time": "-1.0",
    "elb_status_code": "301",
    "target_status_code": "-",
    "received_bytes": "473",
    "sent_bytes": "342",
    "request": "GET http://test:80/ HTTP/1.1",
    "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36",
    "ssl_cipher": "-",
    "ssl_protocol": "-",
    "target_group_arn": "-",
    "trace_id": "Root=1-693a062e-29b4d9b61542ea3034586a68",
    "domain_name": "-",
    "chosen_cert_arn": "-",
    "matched_rule_priority": "0",
    "request_creation_time": "2025-12-25T23:45:50.029000Z",
    "actions_executed": "waf,redirect",
    "redirect_url": "https://test.jp:443/",
    "error_reason": "-",
    "target_port_list": "-",
    "target_status_code_list": "-",
    "classification": "-",
    "classification_reason": "-",
    "29": "TID_c8908f22305e62458552dd06d871d365"
}
```

## ACM関連調査

ACM Certificateが正常に配置されることを確認してから、TCPの接続状況を確認します。

```bash
$ nc -zv test.jp 443
nc: connectx to test.jp port 443 (tcp) failed: Operation timed out
Connection to test.jp port 443 [tcp/https] succeeded!
```

IPv4の接続確認は成功しましたので、特に問題なさそうです。しかし、TCPが成功したのに、どうしてALBログに残されていないでしょうか。

ALBはSSLが成功した上に、アクセスログとして記録することは、
[re:Post](https://repost.aws/articles/AR6uOvOqgRSgWLuDRr3FmAnw/why-do-ssl-tls-negotiation-errors-occur-when-connecting-to-an-application-load-balancer-over-https-and-how-can-i-identify-the-responsible-client-ip)
記事から判明しました。

> If the ALB Connection logs exist but no corresponding entry appears in the ALB Access logs, 
> this indicates that the handshake failed before a valid request was processed.

SSL段階でなにかドラブルがあったのは間違いないでしょう。IPv4のみで接続詳細を確認しましょう。

ALB IPのうちの一つ（2.2.2.2）がタイムアウトになっていることが判明しました。
東京リージョンのIaCに、二番目のサブネットを追加したのですが、ルーティングテーブルを関連するのを忘れていました。

リクエストがプライベートネットワークに入り、そのまま消えてしまいました。

```bash
$ curl -v -4 https://test.jp
* Host test.jp:443 was resolved.

* IPv6: (none)
* IPv4: 2.2.2.2, 1.1.1.1
*   Trying 2.2.2.2:443...
* connect to 2.2.2.2 port 443 from 192.168.0.41 port 49230 failed: Operation timed out
*   Trying 1.1.1.1:443...
* Connected to test.jp (1.1.1.1) port 443
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-AES128-GCM-SHA256 / [blank] / UNDEF
...
```

## 復旧確認

サブネットをパブリックルーティングテーブルに関連した後に、SSLできることを確認します。

```bash
$ openssl s_client -connect 2.2.2.2:443 -servername test.jp
Connecting to 2.2.2.2
CONNECTED(00000003)
depth=2 C=US, O=Amazon, CN=Amazon Root CA 1
verify return:1
depth=1 C=US, O=Amazon, CN=Amazon RSA 2048 M04
verify return:1
depth=0 CN=test.jp
verify return:1
---
Certificate chain
 0 s:CN=test.jp
   i:C=US, O=Amazon, CN=Amazon RSA 2048 M04
   a:PKEY: RSA, 2048 (bit); sigalg: sha256WithRSAEncryption
   v:NotBefore: Oct 23 00:00:00 2025 GMT; NotAfter: Nov 21 23:59:59 2026 GMT
 1 s:C=US, O=Amazon, CN=Amazon RSA 2048 M04
   i:C=US, O=Amazon, CN=Amazon Root CA 1
   a:PKEY: RSA, 2048 (bit); sigalg: sha256WithRSAEncryption
   v:NotBefore: Aug 23 22:26:35 2022 GMT; NotAfter: Aug 23 22:26:35 2030 GMT
 2 s:C=US, O=Amazon, CN=Amazon Root CA 1
   i:C=US, ST=Arizona, L=Scottsdale, O=Starfield Technologies, Inc., CN=Starfield Services Root Certificate Authority - G2
   a:PKEY: RSA, 2048 (bit); sigalg: sha256WithRSAEncryption
   v:NotBefore: May 25 12:00:00 2015 GMT; NotAfter: Dec 31 01:00:00 2037 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
subject=CN=test.jp
issuer=C=US, O=Amazon, CN=Amazon RSA 2048 M04
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: rsa_pss_rsae_sha256
Peer Temp Key: X25519, 253 bits
---
SSL handshake has read 4354 bytes and written 1619 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Protocol: TLSv1.3
Server public key is 2048 bit
This TLS version forbids renegotiation.
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_128_GCM_SHA256
    Session-ID: ...
    Session-ID-ctx:
    Resumption PSK: ...
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 59043 (seconds)
    TLS session ticket:
    ...

    Start Time: 1765415144
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
```

SSLが`Verify return code: 0 (ok)`になりましたので、問題解決です。

## 反省点

Multi-AZの配置からSingle-AZの調整する際に、AWSの仕様を確認すること。業務上使わないサブネット配置もしっかり確認していきましょう。
