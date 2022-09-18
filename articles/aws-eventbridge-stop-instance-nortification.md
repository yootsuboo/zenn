---
title: "AWS EventBridgeを使用し、EC2インスタンの停止を通知"
emoji: "🐔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "aws", "EventBirdge", "AWS CLI"]
published: false
---


# AWS EventBridge と Amazon SNS を使用して、EC2インスタンの停止をEメール通知する。

構築済みのEC2インスタンが停止したことを受けて、EventBridgeのイベン駆動でSNSからEメールを送信して通知してみたと思う。

前半では、AWSコンソールから作成し、後半で同じ構成をAWS CLIを使用して作成してみる。

EventBridgeのターゲットとしてSNSトピックを指定する必要があるので、先にSNSトピックの作成と、サブスクリプションでEメールアドレスを登録していく。

### SNS トピックを作成

AWS コンソールにログインしたら、まずは「SNSを作成」を選択

![](https://storage.googleapis.com/zenn-user-upload/4c4251d9f9e0-20220911.png)

次に「名前」と「表示名」を入力していく。
表示名を入力することで、Eメール通知の件名となる。

![](https://storage.googleapis.com/zenn-user-upload/ff19cdc10166-20220910.png)

今回はその他の項目は設定せずにトピックの作成を選択。

![](https://storage.googleapis.com/zenn-user-upload/840ed0b787f2-20220911.png)

### サブスクリプションの作成

SNSトピックが作成されたら、次にサブスクリプションを作成していく。
作成したSNSトピックを選択したら、サブスクリプションの作成を選択。

![](https://storage.googleapis.com/zenn-user-upload/9e9f1ba8dee9-20220911.png)

トピックARNは自動入力されていると思うので、プロトコルに「Eメール」を選択肢、エンドポイントに使用するEメールアドレスを入力したら、サブスクリプションの作成を選択。

![](https://storage.googleapis.com/zenn-user-upload/7a7af769c3f3-20220911.png)

サブスクリプションが登録されたが、スタータスが「保留中」の確認となっているので、次に登録したEメールアドレス宛に確認メールがAWSから送信されているので確認する。

![](https://storage.googleapis.com/zenn-user-upload/2d1ba236e2ed-20220911.png)

メール本文の「Confirm subscription」を選択し、Webページに飛んで、「Subscription confirmed!」と表示されれば登録作業は完了。
SNSサブスクリプションを確認すると、ステータスが「確認済み」となっていることが確認できる。

![](https://storage.googleapis.com/zenn-user-upload/0156a4d7da6c-20220911.png)

## EventBridgeルールの作成

次にEventBridgeサービスに移動し「ルールを作成」を選択。

![](https://storage.googleapis.com/zenn-user-upload/a9115a0c567b-20220911.png)

名前を入力したら次へ。

![](https://storage.googleapis.com/zenn-user-upload/5ec2a38c0382-20220911.png)

指定のEC2インスタンスのステータスが停止となったことを受けて通知を行うように「イベントパターン」を作成していく。

![](https://storage.googleapis.com/zenn-user-upload/9483bc4b0ecf-20220917.png)

続いて、ターゲットにSNSトピックを選択し、トピックに先程作成したものをプルダウンから選択していく。
AWSから標準出力されるものは、可読性が悪いので、イベントの出力内容に手を加えて行きたいと思う。
追加の設定のターゲット入力を設定で「入力トランスフォーマー」を選択し、入力トランスフォーマーを設定していく。

![](https://storage.googleapis.com/zenn-user-upload/fa0483e0d6ee-20220917.png)

まず、入力パスにAWS標準出力されるJSONの内容から、JSONのキーバリューの形で書き出していく。

![](https://storage.googleapis.com/zenn-user-upload/862d6d044ff4-20220917.png)

続いて、テンプレートに実際出力したい文字列を入力していく。
このときに入力パスで指定したキーを<key-name>形式で使用可能。

![](https://storage.googleapis.com/zenn-user-upload/2026d9ef7bab-20220917.png)

他のステータスについては特に変更を加えずにルールの作成を行う。
ルールが作成されステータスが「Enabled」となっていれば、EC2インスタンスを停止させ、Eメール通知が行われるか確認していく。

![](https://storage.googleapis.com/zenn-user-upload/9934a417f112-20220917.png)

問題なくメール通知が行われ、メール本文に入力トランスフォーマーの内容が反映されていることを確認できました。

![](https://storage.googleapis.com/zenn-user-upload/515762996768-20220917.png)

# AWS CLI を使用して、同じ構成を作成していく

## SNSトピックを作成



