+++
title = "CDK Watch"
weight = 300
+++

## 高速な開発

{{% notice info %}} このセクションは必須ではありませんが、`cdk deploy --hotswap` と `cdk watch` がデプロイをどのようにスピードアップできるか確認することをお勧めします。
{{% /notice %}}

Lambda 関数が機能しているのは素晴らしいことですが、`"Hello, CDK"` の代わりに `"Good Morning, CDK!"` で応答させるよう Lambda 関数のコードを微調整してみましょう。

スタックを更新するために使うコマンドは `cdk deploy` ですが、CloudFormation スタックをデプロイし、「lambda」ディレクトリをブートストラップバケットにアップロードする必要があり時間がかかりすぎます。Lambda 関数のコードを変更するだけなら、CloudFormation スタックを実際に更新する必要はないので、`cdk deploy` の一部の時間は本来不要です。

Lambda 関数のコードの更新だけを行うための仕組みがあったらいいですよね？

## `cdk deploy` の時間

まず、`cdk deploy` の実行にかかる時間を計りましょう。CloudFormation のデプロイを完了させるためにかかる時間が参考になります。そのために、`lambda/hello.py` 内のコードを変更します。

{{<highlight python "hl_lines=10">}}
import json

def handler(event, context):
    print('request: {}'.format(json.dumps(event)))
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'text/plain'
        },
        'body': 'Good Morning, CDK! You have hit {}\n'.format(event['path'])
    }
{{</highlight>}}

これで `cdk deploy` を実行することができます。

```
cdk deploy
```

出力は次のようになります。

```text
✨  Synthesis time: 6s

CdkWorkshopStack: deploying...
CdkWorkshopStack: creating CloudFormation changeset...



 ✅  CdkWorkshopStack

✨  Deployment time: 66.82s

Stack ARN:
arn:aws:cloudformation:REGION:ACCOUNT-ID:stack/CdkWorkshopStack/STACK-ID

✨  Total time: 72.82s
```

正確な時間は異なりますが、通常の CloudFormation の完全なデプロイにかかる時間がわかりました。

## ホットスワップデプロイ

{{% notice info %}} このコマンドは、デプロイを高速化するために CloudFormation スタックにドリフトを意図的に発生させます。このため、開発目的でのみ使用してください。実稼働環境ではホットスワップを使用しないでください！
{{% /notice %}}

デプロイ時間を短縮するために利用する `cdk deploy --hotswap` では、CloudFormation デプロイの代わりにホットスワップデプロイを実行できるかどうかを評価します。可能であれば、CDK CLI は AWS サービス API を使用して直接変更を行います。それ以外の場合は、CloudFormation の完全なデプロイの実行にフォールバックします。

ここでは、`cdk deploy --hotswap` を使用して、ホットスワップ可能な変更を AWS Lambda アセットコードにデプロイします。

## `cdk deploy --hotswap`の時間

`lambda/hello.py` の Lambda 関数のコードをもう一度変更してみましょう

{{<highlight python "hl_lines=10">}}
import json

def handler(event, context):
    print('request: {}'.format(json.dumps(event)))
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'text/plain'
        },
        'body': 'Good Afternoon, CDK! You have hit {}\n'.format(event['path'])
    }
{{</highlight>}}

`cdk deploy --hotswap`を実行します。

```
cdk deploy --hotswap
```

出力は次のようになります。

```text
✨  Synthesis time: 6.44s

⚠️ The --hotswap flag deliberately introduces CloudFormation drift to speed up deployments
⚠️ It should only be used for development - never use it for your production Stacks!

CdkWorkshopStack: deploying...
✨ hotswapping resources:
   ✨ Lambda Function 'CdkWorkshopStack-HelloHandler2E4FBA4D-tEZTcXqG8YYe'
✨ Lambda Function 'CdkWorkshopStack-HelloHandler2E4FBA4D-tEZTcXqG8YYe' hotswapped!

 ✅  CdkWorkshopStack

✨  Deployment time: 3.07s

Stack ARN:
arn:aws:cloudformation:REGION:ACCOUNT-ID:stack/CdkWorkshopStack/STACK-ID

✨  Total time: 9.51s
```

CloudFormation のデプロイが完了するには 67秒かかったのに対し、ホットスワップされた変更をデプロイするのにかかったのは 3秒です。
警告メッセージをよく見てください。非常に重要です！（開発目的でのみ使用してください。実稼働環境ではホットスワップを使用しないでください！）

```text
⚠️ The --hotswap flag deliberately introduces CloudFormation drift to speed up deployments
⚠️ It should only be used for development - never use it for your production Stacks!
```

## コードは本当に変更されているのか？

デプロイは速かったですが、コードは実際に変更されているでしょうか？ AWS Lambda コンソールに移動して、確認してみましょう。

1. [AWS Lambda Console](https://console.aws.amazon.com/lambda/home#/functions) を開きます。(正しいリージョンにいることを確認してください)

    Lambda 関数が表示されます。

    ![](./lambda-1.png)

2. 関数名をクリックして、コンソールを移動します。

3. コードが画面に読み込まれます。コードは変更されていますか？

    ![](./lambda-5.png)

## CDK Watch

`cdk watch` は 毎回 `cdk deploy` または `cdk deploy --hotswap` を呼び出すよりも便利です。。`cdk deploy` に似ていますが、ワンショット操作ではなく、コードとアセットに変更がないか監視し、変更が検出されると自動的にデプロイを試みます。デフォルトでは、`cdk watch` は `--hotswap` フラグを使います。変更内容を調べて、ホットスワップできるかどうかを判断します。`cdk watch --no-hotswap` を呼び出すと、ホットスワップ動作が無効になります。

一度設定したら、`cdk watch`を使用して、ホットスワップ可能な変更と、完全な CloudFormation デプロイを必要とする変更の両方を検出できます。

## `cdk.json` ファイルを見てみましょう

`cdk watch` コマンドが実行されると、監視対象のファイルが `cdk.json` ファイルの `"watch"` 設定によって決定されます。`"include"` と `"exclude"` の 2つのサブキーがあり、それぞれが単一の文字列または文字列の配列のいずれかになります。
各エントリは、`cdk.json` ファイルの場所からの相対パスとして解釈されます。なお、`*` と `**` の両方が使えます。

`cdk.json` ファイルは次のようになります。

```json
{
  "app": "python3 app.py",
  "watch": {
    "include": [
      "**"
    ],
    "exclude": [
      "README.md",
      "cdk*.json",
      "requirements*.txt",
      "source.bat",
      "**/__init__.py",
      "python/__pycache__",
      "tests"
    ]
  },
  "context": {
    // ...
  }
}

```

ご覧のとおり、サンプルアプリには推奨される `"watch"` 設定が付属しています。このユースケースでは何も変更する必要はありませんが、`cdk watch` に他のファイルを監視させたい場合は、ここで設定を変更できます。
それでは試してみましょう。

## `cdk watch` の時間

はじめに `cdk watch` を呼び出します。

```
cdk watch
```

これにより、初期デプロイが実行され、すぐに `cdk.json` で指定したファイルの監視が開始されます。
`lambda/hello.py` の Lambda アセットコードをもう一度変更してみましょう。

{{<highlight python "hl_lines=10">}}
import json

def handler(event, context):
    print('request: {}'.format(json.dumps(event)))
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'text/plain'
        },
        'body': 'Good Night, CDK! You have hit {}\n'.format(event['path'])
    }
{{</highlight>}}

Lambda 関数のコードファイルを変更すると、`cdk watch` は変更を認識し、新しいデプロイを実行します。この場合、Lambda アセットコードをホットスワップできることが認識されるため、CloudFormation デプロイはせず、代わりに Lambda に直接デプロイします。

どれくらいの時間がかかりましたか？

```text
Detected change to 'lambda/hello.py' (type: change). Triggering 'cdk deploy'

✨  Synthesis time: 5.57s

⚠️ The --hotswap flag deliberately introduces CloudFormation drift to speed up deployments
⚠️ It should only be used for development - never use it for your production Stacks!

CdkWorkshopStack: deploying...
✨ hotswapping resources:
   ✨ Lambda Function 'CdkWorkshopStack-HelloHandler2E4FBA4D-tEZTcXqG8YYe'
✨ Lambda Function 'CdkWorkshopStack-HelloHandler2E4FBA4D-tEZTcXqG8YYe' hotswapped!

 ✅  CdkWorkshopStack

✨  Deployment time: 2.54s

Stack ARN:
arn:aws:cloudformation:REGION:ACCOUNT-ID:stack/CdkWorkshopStack/STACK-ID

✨  Total time: 8.11s
```

## まとめ

このチュートリアルの残りの部分では、引き続き `cdk watch` の代わりに `cdk deploy` を使用します。
しかし、もし望むなら、単に `cdk watch` をオンにしておくことができます。完全なデプロイを行う必要がある場合、`cdk watch` は `cdk deploy` を呼び出します。

`cdk watch` を使う方法については、[Increasing Development Speed with CDK Watch](https://aws.amazon.com/blogs/developer/increasing-development-speed-with-cdk-watch/) をお読みください
