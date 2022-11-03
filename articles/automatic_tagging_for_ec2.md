---
title: "EventBridgeとLambdaを使用して、新規構築したEC2に自動タグ付けする"
emoji: "🐔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "EventBridge", "Lambda", "CloudTrail"]
published: true
---

# AWSリソース管理のためにEC2インスタンスへの自動タグ付けをやってみました。

EC2を新規構築すると、CloudTrailにログが出力されます。それをEventBridgeで検出して、Lambdaで自動タグ付けをしてみたいと思います。
Lambdaの内容も難しくないものになっていますので、みなさんも楽しみながら自動化されてみてはいかがでしょうか。
リソース管理がやりやすくなるかと思います。

それでは、早速やっていきます。

## IAMポリシー/ロールの作成

まずは、Lambda関数に実行権限を付与するIAMポリシー/ロールを作成していきます。
IAMサービスからポリシーを選択し、ポリシーの作成を行います。

![](https://storage.googleapis.com/zenn-user-upload/47c27b027d19-20221030.png)

ポリシーの作成でJSONタブに切り替えて、以下のJSONを入力します。
EC2インスタンスへのタグ付け用と、LambdaではCloudWatchLogsにログを出力するので、ログ関連の許可を付与しています。

```json:Automatic-Tagging-Lambda-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "ec2:CreateTags"
            ],
            "Resource": "*"
        }
    ]
}
```

ポリシーの確認画面で、名前を入力して`ポリシーの作成`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/0a7ef01cfd85-20221103.png)

### 次に、IAMロールを作成していきます。

IAMサービスからロールを選択し`ロールを作成`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/eb88967efff3-20221030.png)

信頼されたエンティティタイプで`AWSのサービス`を選択し、ユースケースを`Lambda`にします。

![](https://storage.googleapis.com/zenn-user-upload/d526dbfcfbe6-20221030.png)

許可の追加で、先程作成したポリシーを選択します。

![](https://storage.googleapis.com/zenn-user-upload/f7f7500b6b40-20221103.png)

ロールの詳細で名前を入力し`ロールを作成`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/0ccbfd9db690-20221103.png)

## Lambdaの作成

次に、Lambda関数を作成していきます。
Lambdaサービスに移動し`関数の作成`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/3ea8b8edc84c-20221030.png)

以下の通り、ステータスを設定します。
デフォルトの実行ルールは、さきほど作成したIAMロールを選択してください。
詳細設定については、今回は変更していません。

![](https://storage.googleapis.com/zenn-user-upload/347fa77bbe4e-20221103.png)

Lambda関数が作成されました。
まだ、トリガーとなるものがありませんが、後ほど作成するEventBridgeルールがトリガーに追加されます。

次に、以下のようにコードソースを書き換えて`Deploy`を選択し、上書きします。
CloudTrailに作成されるログから、インスタンスID(instanceID)、作成ユーザー(userName)、構築日時(createDate)を取得して、EC2のタグ情報として渡しているだけです。

```py
import boto3, json

def lambda_handler(event, content):

    instanceId = event['detail']['responseElements']['instancesSet']['items'][0]['instanceId']

    userName = event['detail']['userIdentity']['userName']
    
    creationDate = event['detail']['userIdentity']['sessionContext']['attributes']['creationDate'][:10]
    
    
    ec2 = boto3.client('ec2')
    ec2.create_tags(
        Resources=[instanceId,],
        Tags=[
            {'Key': 'Owner', 'Value': userName},
            {'Key': 'CreationDate', 'Value': creationDate},
        ]
    )

    return {
        'statusCode': 200,
        'body': json.dumps('Excuted!')
    }
```

詳しいログの中身が知りたい方は、CloudTrailの証跡で`RunInstances`からすべてのログの中身が参照できます。
今回は、ログの内容については割愛します。

## EventBridgeルールの作成

最後に、EventBridgeのルールを作成していきます。
EC2インスタンスが新規構築されるとCloudTrailに`RunInstance`ログが発行されるので、
そのログをEventBridgeで検知して、後続のLambdaを起動するように設定していきます。

EventBridgeサービスのルールから`ルールを作成`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/29c77853d3ad-20221030.png)

ルールの詳細で、名前と説明を入力していきます。
イベントバスはAWSサービスの場合`default`で大丈夫です。

![](https://storage.googleapis.com/zenn-user-upload/482e78ff2156-20221103.png)

次に、イベントパターンを以下のように入力します。
CloudTrailに作成されるEC2に対するログの`RunInstances`と一致したときに、イベントルールが実行されます。

![](https://storage.googleapis.com/zenn-user-upload/69e2cd2f0d49-20221030.png)

ターゲットには、先程作成したLambda関数を入力します。
バージョン/エイリアス設定と、追加設定は変更していません。

![](https://storage.googleapis.com/zenn-user-upload/be1d6f338a0a-20221103.png)

確認と更新画面で、設定に問題がなければ`ルールを作成`を選択します。

EventBridgeのターゲットとしてLambdaを指定したことで、Lambda側のトリガーとしてEventBridgeが表示されるようになりました。

![](https://storage.googleapis.com/zenn-user-upload/3c82d769d055-20221103.png)

以上で準備は完了です。
早速EC2インスタンスを作成していきたいと思います。

### EC2インスタンスの作成

EC2インスタンスを作成していきます。
作成内容は割愛します。

![](https://storage.googleapis.com/zenn-user-upload/9b2e618227f3-20221103.png)

タグが自動で付与されているか確認していきます。

![](https://storage.googleapis.com/zenn-user-upload/4d4509c3111e-20221103.png)

んっ!?名前タグ以外に何も作成されていません。。。
CloudTrailで`RunInstances`ログが作成されているか確認していきます。

:::message
人によっては、問題なくログが追加されているかもしれません。
ログが自動付与されていた方は以上で終了です。
お疲れさまでした。
:::

イベント履歴を確認すると、問題なく`RunInstance`ログが作成されていました。
実は、CloudTrailではイベント履歴に表示されていても`証跡`を取得するように設定しないとイベントとして他のAWSサービスは認識してくれません。

![](https://storage.googleapis.com/zenn-user-upload/bfdd9d28e8af-20221030.png)

それでは、CloudTrailサービスから`証跡`を選択し、`証跡の作成`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/731fdd50ad88-20221030.png)

以下のようにパラメータを設定してください。
ログファイルの検証という項目を今回は無効にしています。
ログファイルの書き換えなどの不正をしていないか検証してくれますが、個人使用では必要ないかと思います。
また、CloudWatchLogsに証跡ログを出力してモニタリングすることもできますが、今回は無効にしています。

![](https://storage.googleapis.com/zenn-user-upload/3a02b3701132-20221103.png)

あとの設定はそのままで、確認と作成画面で`証跡の作成`を選択します。
証跡が取得され始めますので、再度EC2インスタンスを作成してみたいと思います。

![](https://storage.googleapis.com/zenn-user-upload/7c63d0d49331-20221103.png)

今度は問題なくタグが自動付与されました。
実は、証跡の作成が必要とわかるまで、数時間EventBridgeルールなどと格闘していました。。
意外なところに落とし穴があるものです。

# 参考資料

[新規作成したAWSリソースへの自動タグ付け処理(SKY BLOG)](https://www.sky365.co.jp/blog/aws/aws-2.html)
参考にさせていただきました、ありがとうございます。

**以上です、お疲れさまでした。**

先日、ブルージャイアントエクスプローラの7巻が発売されましたね!
私が、唯一読んでいる漫画なんですが、主人公の熱いところが最高に好きなんです。
興味がある方は、読まれてみてください。

