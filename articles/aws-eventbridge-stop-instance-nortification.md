---
title: "AWS EventBridgeを使用し、EC2インスタンの停止を通知"
emoji: "🐔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "aws", "EventBirdge", "AWS CLI"]
published: false
---


# AWS EventBridge と Amazon SNS を使用して、EC2インスタンの停止をEメール通知する。

構築済みのEC2インスタンが停止したことを受けて、EventBridgeのイベン駆動でSNSからEメールを送信して通知してみたいと思う。

前半では、AWSコンソールから作成し、後半で同じ構成をAWS CLIを使用して作成してみる。

EventBridgeのターゲットとしてSNSトピックを指定する必要があるので、先にSNSトピックの作成と、サブスクリプションでEメールアドレスを登録していく。

### SNS トピックを作成

AWS コンソールにログインしたら、SNSサービスに移動し`SNSを作成`を選択

![](https://storage.googleapis.com/zenn-user-upload/4c4251d9f9e0-20220911.png)

次に`名前`と`表示名`を入力していく。
表示名を入力することで、Eメール通知の件名となる。

![](https://storage.googleapis.com/zenn-user-upload/ff19cdc10166-20220910.png)

今回はその他の項目は設定を変更せずに`トピックの作成`を選択。

![](https://storage.googleapis.com/zenn-user-upload/840ed0b787f2-20220911.png)

### サブスクリプションの作成

SNSトピックが作成されたら、次にサブスクリプションを作成していく。
作成したSNSトピックを選択したら`サブスクリプションの作成`を選択。

![](https://storage.googleapis.com/zenn-user-upload/9e9f1ba8dee9-20220911.png)

トピックARNは自動入力されていると思うので、プロトコルに`Eメール`を選択し、エンドポイントに使用するEメールアドレスを入力したら`サブスクリプションの作成`を選択。

![](https://storage.googleapis.com/zenn-user-upload/7a7af769c3f3-20220911.png)

サブスクリプションが登録されたが、スタータスが`保留中`の確認となっているので、次に登録したEメールアドレス宛に確認メールがAWSから送信されているので確認する。

![](https://storage.googleapis.com/zenn-user-upload/2d1ba236e2ed-20220911.png)

メール本文の`Confirm subscription`を選択し、Webページに飛んで、`Subscription confirmed!`と表示されれば登録は完了。
SNSサブスクリプションを確認すると、ステータスが`確認済み`となっていることが確認できる。

![](https://storage.googleapis.com/zenn-user-upload/0156a4d7da6c-20220911.png)

## EventBridgeルールの作成

次にEventBridgeサービスに移動し`ルールを作成`を選択。

![](https://storage.googleapis.com/zenn-user-upload/a9115a0c567b-20220911.png)

`名前`を入力したら次へ。

![](https://storage.googleapis.com/zenn-user-upload/5ec2a38c0382-20220911.png)

指定のEC2インスタンスのステータスが停止となったことを受けて通知を行うように`イベントパターン`を作成していく。

![](https://storage.googleapis.com/zenn-user-upload/9483bc4b0ecf-20220917.png)

続いて、ターゲットにSNSトピックを選択し、トピックに先程作成したものをプルダウンから選択していく。
AWSから標準出力されるものは、可読性が悪いので、イベントの出力内容に手を加えて行きたいと思う。
追加の設定のターゲット入力を設定で`入力トランスフォーマー`を選択し、入力トランスフォーマーを設定していく。
(標準の通知内容を一度、通知させてみてどのような内容が出力されるのか確認してみてもいいと思う)

![](https://storage.googleapis.com/zenn-user-upload/fa0483e0d6ee-20220917.png)

まず、`入力パス`にAWS標準出力されるJSONの内容から、JSONのキーバリューの形で書き出していく。

![](https://storage.googleapis.com/zenn-user-upload/862d6d044ff4-20220917.png)

続いて、テンプレートに実際出力したい文字列を入力していく。
このときに入力パスで指定したキーを`<key-name>`形式で使用可能。

![](https://storage.googleapis.com/zenn-user-upload/2026d9ef7bab-20220917.png)

他のステータスについては特に変更を加えずに`ルールの作成`を行う。
ルールが作成されステータスが`Enabled`となっていれば、EC2インスタンスを停止させ、Eメール通知が行われるか確認していく。

![](https://storage.googleapis.com/zenn-user-upload/9934a417f112-20220917.png)

問題なくメール通知が行われ、メール本文に入力トランスフォーマーの内容が反映されていることを確認できました。

![](https://storage.googleapis.com/zenn-user-upload/515762996768-20220917.png)

-----
# AWS CLI を使用して、同じ構成を作成していく

基本的に`input-json`ファイルを作成し、`aws cli` からコマンドを実行する際に読み込ませる形でリソースを構築していきたいと思う。
すべての作業は、aws cli のホームディレクトリ(/home/cloudshell-user)配下で行っていく。

## SNSトピックを作成

まずは、`input-json`ファイルを作成していく。

```sh:sns-create-topic.json
{
    "Name": "test-cli-topic",
    "Attributes": {
        "DisplayName": "EC2インスタンスが停止しました。"
    }
}
```

### input-json の書き方がわからないときは

```sh
aws sns create-tipic --generate-cli-skeleton > create-topic-template.json
```
オプション`--generate-cli-skeleton` でコマンドを実行することで`input-json`用のテンプレートを見ることができる。
ファイルに書き出して編集用に使用してもいい。

続いて`aws cli`で、SNSトピックの作成をしていく。

```sh
aws sns create-topic --cli-input-json file://sns-create-tipic.json
```

カレントディレクトリに`input-json`ファイルがある場合は相対バスで記述可能。
また、コマンドの必須オプションとして`--name`を指定する必要があるが`input-json`ファイル内に記述があれば省略可能。
詳しくは、コマンドリファレンスを参照してください。
[aws cli コマンドリファレンス](https://docs.aws.amazon.com/ja_jp/cli/latest/reference/sns/create-topic.html)

## SNSサブスクリプションを作成

ホームディレクトリ配下に`input-json`ファイルを作成していく。

```sh:sns-subscribe.json
{
    "TopicArn": "<your topic arn>",
    "Protocol": "email",
    "Endpoint": "<your email address>",
    "ReturnSubscriptionArn": true
}
```

コマンドを実行し、SNSサブスクリプションの作成をしていく

```sh
aws sns subscribe --cli-input-json file://sns-subscribe.json
```

確認通知がエンドポイントで指定したメールアドレス宛に送信されているので`Confirm subscription`で有効化する。

## EventBridge ルールの作成

EventBridge は、ルールとターゲットを別に設定する必要がある。
まずは、ルール用の`input-json`ファイルの作成をしていく。

```sh:events-put-rule.json
{
    "Name": "test-event-rule", 
    "EventPattern": "{ \"source\": [\"aws.ec2\"], \"detail-type\": [\"EC2 Instance State-change Notification\"], \"detail\": { \"state\": [\"stopping\"], \"instance-id\": [\"i-0a97dd1749bbd2cfd\"]}}",
    "State": "ENABLED", 
    "EventBusName": "default" 
} 
```

続いて、aws cliコマンドを実行して、EventBridgeルールを作成していく。

```sh
aws events put-rule --cli-input-json file://events-put-rule.json
```

## EventBridge ターゲットの作成

ターゲット用の`input-json`ファイルを作成していく。

```sh:events-put-target.json
{
    "Rule": "test-event-rule",
    "EventBusName": "default",
    "Targets": [
        {
            "Id": "1",
            "Arn": "<sns topic arn>",
            "InputTransformer": {
                "InputPathsMap": {
                    "detail-type": "$.detail-type",
                    "instance-id": "$.detail.instance-id",
                    "state": "$.detail.state",
                    "time": "$.time"
                },
                "InputTemplate": "\"インスタンスID「<instance-id」のステータスが「<state>」に変更されました。\"\n\n\"発生時刻は「<time>」です。\"\n" 
            }
        }
    ]
}
```

続いて、aws cliでコマンドを実行し、EventBridgeターゲットを反映していく。

```sh
aws events put-targets --cli-input-json file://events-put-target.json
```

EC2インスタンスを起動状態から停止にしたら、問題なくメール通知されました。

![](https://storage.googleapis.com/zenn-user-upload/0b8867bf69d5-20220919.png)

シェルスクリプトで、awsコマンドを実行することで、複数リソースを一気に作成することも可能なので、aws cli を使用した構築はおすすめです。
はじめは、AWSコンソール上でリソースを構築し、構築パラメータを取得することで`input-json`ファイルの書き方の参考にできたりもします。
例えば、EventBridge ルールやターゲットなら以下のコマンドで設定パラメータを取得することができます。

```sh
aws events list-rules --name-prefix "test" > test-list-rules.json
```
```sh
aws events list-targets-by-rule --rule "test-event-rule" > test-event-rule-targets.json
```

どなたかのお役に立てましたら幸いです。

#### --以上です。--
