---
title: "CodePipelineの実行結果をSlackに通知する2つの方法を試してみた（AWS Chatbot / Lambda）"
emoji: "🔔"
type: "tech"
topics: ["aws", "codepipeline", "slack", "chatbot", "lambda"]
published: false
---

## はじめに

業務でCodePipelineのSlack通知設定を触る機会があったので、せっかくなので2つの方式を比較検証してみました。

- **方式A**: AWS Chatbot を使用した標準的な通知
- **方式B**: EventBridge + Lambda を使用したカスタム通知

「どっちを使えばいいの？」という疑問に答えられるよう、実際に両方構築して違いを確認してみます。最後にCloudFormationでのIaC化もやってみました。

本記事は「CodePipelineシリーズ」の第2弾です。

- 第1弾: パスフィルタによる条件付きトリガー
- **第2弾: Slack通知設定（本記事）**
- 第3弾: SlackからCodePipelineを実行するChatOps（次回予定）

## 2つの通知方式の概要

まず、それぞれの方式がどんな構成になるのか整理しておきます。

### 方式A: AWS Chatbot による通知

```
CodePipeline → 通知ルール → AWS Chatbot → Slack
```

AWS Chatbotは、AWSサービスからの通知をSlackやAmazon Chimeに送信するためのマネージドサービスです。CodePipelineには「通知ルール」機能が組み込まれており、Chatbotと連携するだけで簡単にSlack通知ができます。

**特徴**
- セットアップが簡単
- AWS標準のリッチなメッセージフォーマット
- 「Get info」「Stop pipeline」などのインタラクティブボタン付き
- 追加コストなし

### 方式B: EventBridge + Lambda による通知

```
CodePipeline → EventBridge → Lambda → Slack Webhook
```

EventBridgeでCodePipelineの状態変更イベントをキャプチャし、Lambda関数でSlack Incoming Webhookに送信する方式です。

**特徴**
- 通知内容を完全にカスタマイズ可能
- 条件分岐やフィルタリングが柔軟
- Lambda実行料金が発生（ただし微小）
- インタラクティブボタンは自前実装が必要

## 前提条件・環境準備

### 前提条件

- AWSアカウント
- Slackワークスペース（通知先）
- AWS CLIがインストール済み

### 検証用パイプラインの作成

通知の検証用に、シンプルなパイプラインを作成しました。本題は通知設定なので、パイプライン自体はできるだけシンプルにしています。

#### 1. S3バケットの作成（ソース用）

S3コンソールで以下の設定でバケットを作成しました。

| 項目 | 値 |
|------|-----|
| バケット名 | `codepipeline-slack-demo-source-{アカウントID}` |
| リージョン | `ap-northeast-1`（東京） |
| バージョニング | **有効** |

:::message
CodePipelineのS3ソースを使用する場合、バケットのバージョニングは必須です。
:::

#### 2. ソースファイルの準備

以下の内容で `buildspec.yml` を作成し、zipファイルにしました。

```yaml:buildspec.yml
version: 0.2

phases:
  build:
    commands:
      - echo "Hello from CodeBuild!"
      - echo "Build started at $(date)"
      - sleep 10
      - echo "Build completed!"
```

```bash
# zipファイルを作成
zip source.zip buildspec.yml

# zipの構造を確認（buildspec.ymlがルート直下にあること）
zipinfo source.zip
```

:::message alert
最初、ディレクトリごとzipしてしまいCodeBuildがbuildspec.ymlを見つけられずエラーになりました。`buildspec.yml` はzipのルート直下に配置する必要があります。
:::

作成した `source.zip` をS3バケットにアップロードします。

```bash
aws s3 cp source.zip s3://codepipeline-slack-demo-source-{アカウントID}/
```

#### 3. CodeBuildプロジェクトの作成

CodeBuildコンソールで以下の設定でプロジェクトを作成しました。

| セクション | 項目 | 値 |
|-----------|------|-----|
| プロジェクトの設定 | プロジェクト名 | `slack-demo-build` |
| ソース | ソースプロバイダ | Amazon S3 |
| | バケット | 作成したバケット |
| | S3オブジェクトキー | `source.zip` |
| 環境 | 環境イメージ | マネージド型イメージ |
| | OS | Amazon Linux |
| | ランタイム | Standard |
| | イメージ | `aws/codebuild/amazonlinux2-x86_64-standard:5.0` |
| Buildspec | 使用 | ソースコードのbuildspec.ymlを使用 |
| アーティファクト | タイプ | アーティファクトなし |

#### 4. CodePipelineの作成

CodePipelineコンソールで以下の設定でパイプラインを作成しました。

| セクション | 項目 | 値 |
|-----------|------|-----|
| パイプラインの設定 | パイプライン名 | `slack-notification-demo` |
| | パイプラインタイプ | V2 |
| ソースステージ | ソースプロバイダ | Amazon S3 |
| | バケット | 作成したバケット |
| | S3オブジェクトキー | `source.zip` |
| | 変更検出オプション | Amazon CloudWatch Events |
| ビルドステージ | プロバイダ | AWS CodeBuild |
| | プロジェクト名 | `slack-demo-build` |
| デプロイステージ | | スキップ |

パイプライン作成後、自動で最初の実行が開始されました。無事に成功（緑）で完了したので、これで準備OKです。

## 方式A: AWS Chatbot による通知

まずは方式Aから試してみます。

### Step 1: Slackワークスペースとの連携

#### 1. AWS Chatbotコンソールを開く

https://console.aws.amazon.com/chatbot/ にアクセスしました。

#### 2. Slackワークスペースを連携

1. 「Configure new client」→「Slack」を選択
2. 「Configure」をクリック
3. Slackの認証画面でワークスペースへのアクセスを許可

連携すると、ワークスペース名が表示されました。

:::message
Slackワークスペースとの連携はOAuth認証が必要なため、コンソールでの手動設定が必須です。この部分はCloudFormationでは自動化できません。
:::

### Step 2: チャンネル設定の作成

AWS Chatbotコンソールで「Configure new channel」をクリックし、以下を設定しました。

| セクション | 項目 | 値 |
|-----------|------|-----|
| Configuration details | Configuration name | `codepipeline-notifications` |
| Slack channel | Channel type | パブリック |
| | Channel name | 通知先のチャンネルを選択 |
| Permissions | Role settings | Create a new role |
| | Policy templates | Notification permissions |
| Channel guardrail policies | Policy name | ReadOnlyAccess |

### Step 3: CodePipelineの通知ルールを作成

CodePipelineコンソールでパイプラインを開き、「Settings」タブから「Create notification rule」をクリックしました。

| セクション | 項目 | 値 |
|-----------|------|-----|
| Notification rule settings | Notification name | `slack-notification-demo-rule` |
| | Detail type | Full |
| Events that trigger notifications | Pipeline execution | Started, Succeeded, Failed にチェック |
| Targets | Target type | AWS Chatbot (Slack) |
| | Choose target | 作成したChatbot設定を選択 |

### Step 4: 動作確認

パイプラインを実行して通知を確認してみました。

1. CodePipelineコンソールで「Release change」をクリック
2. Slackチャンネルで通知を確認

以下のような通知が届きました！

**開始時（STARTED）**
```
📣 AWS CodePipeline Notification | ap-northeast-1 | Account: xxxxxxxxxxxx

CodePipeline pipeline execution STARTED.

Pipeline
slack-notification-demo

[Get info] [Stop pipeline]
```

**成功時（SUCCEEDED）**
```
✅ AWS CodePipeline Notification | ap-northeast-1 | Account: xxxxxxxxxxxx

CodePipeline pipeline execution SUCCEEDED.

Pipeline
slack-notification-demo

[Get info]
```

### 通知の特徴

実際に使ってみて気づいたAWS Chatbot通知の特徴です。

1. **スレッド形式**: 同一実行のSTARTED/SUCCEEDEDがスレッドでまとまる。チャンネルが荒れなくて良い
2. **インタラクティブボタン**: 「Get info」「Stop pipeline」がSlackから直接操作可能。これは便利
3. **リージョン・アカウント表示**: どの環境からの通知か一目で分かる

### 失敗通知の確認

失敗時の通知も確認しておきました。`buildspec.yml` を以下のように変更してわざとエラーを発生させます。

```yaml:buildspec.yml
version: 0.2

phases:
  build:
    commands:
      - echo "This will fail"
      - exit 1
```

S3に再アップロードするとパイプラインが自動実行され、失敗通知が届きました。

```
❌ AWS CodePipeline Notification | ap-northeast-1 | Account: xxxxxxxxxxxx

1 action failed in stage: Build.

  Build
    Additional Information: Build terminated with state: FAILED. 
    Phase: BUILD, Code: COMMAND_EXECUTION_ERROR, 
    Message: Error while executing command: exit 1. Reason: exit status 1

Pipeline
slack-notification-demo

[Get info] [Start pipeline]
```

失敗時は詳細なエラー情報が含まれていて、「Start pipeline」ボタンで再実行もできます。これは実用的ですね。

:::message alert
検証中に気づいたのですが、パイプライン作成時に「Enable automatic retry on stage failure」を有効にしていると、失敗時に自動リトライが行われて通知が複数回届きました。「1 action failed」「2 actions failed」と連続で来て最初は焦りました。通知の重複が気になる場合は、リトライを無効にするか、通知イベントを調整してください。
:::

確認後、`buildspec.yml` を正常版に戻しておきました。

## 方式B: EventBridge + Lambda による通知

次に方式Bを試してみます。こちらは通知内容を自由にカスタマイズできるのが魅力です。

### Step 1: Slack Incoming Webhook の作成

1. Slackの Apps から「Incoming Webhooks」を検索して追加
2. 通知先チャンネルを選択
3. Webhook URLをコピー

```
https://hooks.slack.com/services/T.../B.../xxx
```

:::message
Webhook URLは秘密情報として扱い、コードにハードコードしないようにしましょう。
:::

### Step 2: Lambda関数の作成

Lambdaコンソールで以下の設定で関数を作成しました。

| 項目 | 値 |
|------|-----|
| 関数名 | `codepipeline-slack-notifier` |
| ランタイム | Python 3.12 |
| アーキテクチャ | x86_64 |
| 実行ロール | 基本的なLambdaアクセス権限で新しいロールを作成 |

以下のコードを設定しました。状態に応じて絵文字と色を変える簡単なカスタマイズを入れています。

```python:lambda_function.py
import json
import urllib.request
import os

SLACK_WEBHOOK_URL = os.environ['SLACK_WEBHOOK_URL']

def lambda_handler(event, context):
    print(json.dumps(event))
    
    detail = event.get('detail', {})
    pipeline = detail.get('pipeline', 'Unknown')
    state = detail.get('state', 'Unknown')
    execution_id = detail.get('execution-id', 'Unknown')
    region = event.get('region', 'ap-northeast-1')
    
    # 状態に応じた絵文字と色
    status_config = {
        'STARTED': {'emoji': '🚀', 'color': '#3498db'},
        'SUCCEEDED': {'emoji': '✅', 'color': '#2ecc71'},
        'FAILED': {'emoji': '❌', 'color': '#e74c3c'},
        'CANCELED': {'emoji': '⚪', 'color': '#95a5a6'},
    }
    
    config = status_config.get(state, {'emoji': '❓', 'color': '#95a5a6'})
    
    # パイプラインへのリンク
    pipeline_url = f"https://{region}.console.aws.amazon.com/codesuite/codepipeline/pipelines/{pipeline}/view"
    
    # Slack メッセージ（カスタマイズ例）
    message = {
        "attachments": [
            {
                "color": config['color'],
                "blocks": [
                    {
                        "type": "section",
                        "text": {
                            "type": "mrkdwn",
                            "text": f"{config['emoji']} *CodePipeline {state}*\n\n*Pipeline:* <{pipeline_url}|{pipeline}>\n*Execution ID:* `{execution_id[:8]}`"
                        }
                    }
                ]
            }
        ]
    }
    
    req = urllib.request.Request(
        SLACK_WEBHOOK_URL,
        data=json.dumps(message).encode('utf-8'),
        headers={'Content-Type': 'application/json'}
    )
    
    urllib.request.urlopen(req)
    
    return {'statusCode': 200}
```

環境変数も設定しました。

- キー: `SLACK_WEBHOOK_URL`
- 値: 取得したWebhook URL

### Step 3: EventBridgeルールの作成

EventBridgeコンソールで「ルールを作成」をクリックしました。

最近UIが新しくなっていて、ドラッグ＆ドロップで設定できるようになっていました。

#### イベントパターンの設定

左側の「AWS SERVICE EVENTS」から「CodePipeline」を検索し、「CodePipeline Pipeline Execution State Change」を選択してTriggering Eventsエリアにドロップしました。

#### ターゲットの設定

「Targets」タブで「Lambda function」を選択し、作成した `codepipeline-slack-notifier` を指定しました。

ルール名を `codepipeline-state-change-rule` として作成しました。

### Step 4: 動作確認

パイプラインを実行して通知を確認してみました。

```bash
aws codepipeline start-pipeline-execution \
  --name slack-notification-demo \
  --region ap-northeast-1
```

以下のような通知が届きました！

**開始時（STARTED）**
```
🚀 CodePipeline STARTED

Pipeline: slack-notification-demo
Execution ID: be19a3db
```

**成功時（SUCCEEDED）**
```
✅ CodePipeline SUCCEEDED

Pipeline: slack-notification-demo
Execution ID: be19a3db
```

方式Aと比べるとシンプルですが、パイプライン名がリンクになっていてクリックするとAWSコンソールに飛べます。

### カスタマイズの可能性

方式BではLambda関数を修正することで、様々なカスタマイズができます。思いつくものを挙げてみました。

- 特定のパイプラインのみ通知
- 失敗時のみ通知
- 実行者情報の追加
- CloudWatch Logsへのリンク追加
- メンション機能（`@channel`、`@here`、個人メンション）

例えば、失敗時のみ `@channel` でメンションする場合は以下のように修正できます。

```python
if state == 'FAILED':
    message['text'] = '<!channel> パイプラインが失敗しました！'
```

この柔軟性が方式Bの強みですね。

:::message
**Tips: 通知の重複排除と冪等性**

方式Bで自動リトライが有効なパイプラインを扱う場合、同一実行に対して複数のイベントが発行されることがあります。よりロバストな実装にするには以下のアプローチが考えられます。

- **Slackスレッドへの集約**: `execution-id` をキーにDynamoDBなどで Slack の `thread_ts` を管理し、同一実行の通知をスレッドにまとめる
- **冪等性の確保**: `execution-id` + `state` の組み合わせで重複チェックを行い、同じ通知を複数回送信しないようにする

ただし、これらの実装はDynamoDBなどの追加リソースが必要になり、方式Aのスレッド機能を自前で再現しようとすると工数が一気に跳ね上がります。シンプルな要件であれば方式Aの採用を検討してください。
:::

## 方式A vs 方式B の比較

実際に両方試してみて感じた違いをまとめます。

| 項目 | 方式A（AWS Chatbot） | 方式B（EventBridge + Lambda） |
|------|---------------------|------------------------------|
| **セットアップ** | 簡単（コンソールのみ） | やや複雑（Lambda作成が必要） |
| **通知フォーマット** | AWS標準（リッチ） | 完全カスタマイズ可能 |
| **インタラクティブボタン** | あり（Get info, Stop等） | 自前実装が必要 |
| **スレッド形式** | 対応（標準機能） | 非対応（実装には追加工数大） |
| **フィルタリング** | イベント種別のみ | 柔軟（パイプライン名、ステージ等） |
| **追加コスト** | 無料 | Lambda実行料金（微小） |
| **IaC化** | 一部制限あり | 完全にコード化可能 |

:::message
**スレッド機能のコスト対効果**

方式Aの「同一実行をスレッドにまとめる」機能は、チャンネルのノイズを抑える上で非常に重要です。方式Bでこれを自前実装しようとすると、DynamoDBなどで `ExecutionID` と Slack の `Thread ID` のマッピングを管理する必要があり、実装コストが大幅に増加します。

「通知内容のカスタマイズ」と「スレッド機能」のどちらを優先するかが、方式選択の大きな判断ポイントになります。
:::

### どちらを選ぶべきか

検証してみた結論として、以下のように使い分けるのが良さそうです。

**方式A（AWS Chatbot）がおすすめのケース**
- とにかく早く通知を設定したい
- AWS標準のリッチな通知で十分
- Slackからパイプラインを操作したい（Get info, Stop, Start）

**方式B（Lambda）がおすすめのケース**
- 通知内容をカスタマイズしたい
- 特定の条件でのみ通知したい
- メンション機能を使いたい
- 既存の通知システムと統合したい

**両方併用するケース**
- 通常の通知は方式A、失敗時のみ方式Bでメンション付き通知

個人的には、まず方式Aで始めて、物足りなくなったら方式Bを追加するのが良いと思いました。

## CloudFormationでのIaC化

せっかくなので、検証した構成をCloudFormationでコード化してみました。

### 前提

AWS ChatbotのSlackワークスペースとの「連携（ワークスペースの追加）」はOAuth認証が必要なため、コンソールでの手動設定が必須です。ただし、連携完了後の「構成（チャンネルの紐付け）」はCloudFormationで管理できます。

以下のCloudFormationテンプレートは、ワークスペース連携が完了していることを前提としています。

| 作業 | IaC化 |
|------|-------|
| ワークスペースの連携（OAuth認証） | ❌ 手動のみ |
| チャンネル設定（`AWS::Chatbot::SlackChannelConfiguration`） | ✅ 可能 |
| 通知ルール（`AWS::CodeStarNotifications::NotificationRule`） | ✅ 可能 |

### 必要なパラメータの確認

テンプレートを実行する前に、以下の情報を確認してください。

1. **SlackWorkspaceId**: AWS Chatbotコンソールで確認（例: `TXXXXXXXXX`）
2. **SlackChannelId**: Slackでチャンネルを右クリック →「チャンネル詳細を表示」→ 一番下の「チャンネルID」
3. **SlackWebhookUrl**: Incoming Webhookの URL
4. **SourceBucketName**: ソース用S3バケット名

### CloudFormationテンプレート

以下のテンプレートで、方式A・方式B両方の通知設定を含むパイプラインを作成できます。

```yaml:template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CodePipeline Slack Notification Demo - Method A (Chatbot) + Method B (Lambda)

Parameters:
  SlackWorkspaceId:
    Type: String
    Description: Slack Workspace ID (from AWS Chatbot console)
  SlackChannelId:
    Type: String
    Description: Slack Channel ID
  SlackWebhookUrl:
    Type: String
    Description: Slack Incoming Webhook URL for Method B
    NoEcho: true
  SourceBucketName:
    Type: String
    Description: S3 bucket name for pipeline source

Resources:
  # ============================================
  # CodeBuild
  # ============================================
  CodeBuildServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${AWS::StackName}-codebuild-role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
      Policies:
        - PolicyName: CodeBuildLogs
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: '*'

  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Sub ${AWS::StackName}-build
      ServiceRole: !GetAtt CodeBuildServiceRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/amazonlinux2-x86_64-standard:5.0
      Source:
        Type: CODEPIPELINE
        BuildSpec: buildspec.yml
      LogsConfig:
        CloudWatchLogs:
          Status: ENABLED

  # ============================================
  # CodePipeline
  # ============================================
  CodePipelineServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${AWS::StackName}-pipeline-role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codepipeline.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: CodePipelinePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:GetObjectVersion
                  - s3:GetBucketVersioning
                  - s3:PutObject
                Resource:
                  - !Sub arn:aws:s3:::${SourceBucketName}
                  - !Sub arn:aws:s3:::${SourceBucketName}/*
                  - !Sub arn:aws:s3:::${ArtifactBucket}
                  - !Sub arn:aws:s3:::${ArtifactBucket}/*
              - Effect: Allow
                Action:
                  - codebuild:BatchGetBuilds
                  - codebuild:StartBuild
                Resource: !GetAtt CodeBuildProject.Arn

  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub ${AWS::StackName}-artifacts-${AWS::AccountId}
      VersioningConfiguration:
        Status: Enabled

  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: !Sub ${AWS::StackName}-pipeline
      RoleArn: !GetAtt CodePipelineServiceRole.Arn
      PipelineType: V2
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
      Stages:
        - Name: Source
          Actions:
            - Name: Source
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: S3
                Version: '1'
              Configuration:
                S3Bucket: !Ref SourceBucketName
                S3ObjectKey: source.zip
                PollForSourceChanges: 'false'
              OutputArtifacts:
                - Name: SourceOutput
        - Name: Build
          Actions:
            - Name: Build
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref CodeBuildProject
              InputArtifacts:
                - Name: SourceOutput

  # ============================================
  # Method A: AWS Chatbot
  # ============================================
  ChatbotRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${AWS::StackName}-chatbot-role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: chatbot.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

  ChatbotSlackChannel:
    Type: AWS::Chatbot::SlackChannelConfiguration
    Properties:
      ConfigurationName: !Sub ${AWS::StackName}-slack-channel
      IamRoleArn: !GetAtt ChatbotRole.Arn
      SlackWorkspaceId: !Ref SlackWorkspaceId
      SlackChannelId: !Ref SlackChannelId
      LoggingLevel: INFO

  PipelineNotificationRule:
    Type: AWS::CodeStarNotifications::NotificationRule
    Properties:
      Name: !Sub ${AWS::StackName}-notification-rule
      DetailType: FULL
      Resource: !Sub arn:aws:codepipeline:${AWS::Region}:${AWS::AccountId}:${Pipeline}
      EventTypeIds:
        - codepipeline-pipeline-pipeline-execution-started
        - codepipeline-pipeline-pipeline-execution-succeeded
        - codepipeline-pipeline-pipeline-execution-failed
      Targets:
        - TargetType: AWSChatbotSlack
          TargetAddress: !GetAtt ChatbotSlackChannel.Arn

  # ============================================
  # Method B: Lambda + EventBridge
  # ============================================
  SlackNotifierLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${AWS::StackName}-lambda-role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  SlackNotifierLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub ${AWS::StackName}-slack-notifier
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt SlackNotifierLambdaRole.Arn
      Timeout: 10
      Environment:
        Variables:
          SLACK_WEBHOOK_URL: !Ref SlackWebhookUrl
      Code:
        ZipFile: |
          import json
          import urllib.request
          import os

          SLACK_WEBHOOK_URL = os.environ['SLACK_WEBHOOK_URL']

          def lambda_handler(event, context):
              print(json.dumps(event))
              
              detail = event.get('detail', {})
              pipeline = detail.get('pipeline', 'Unknown')
              state = detail.get('state', 'Unknown')
              execution_id = detail.get('execution-id', 'Unknown')
              region = event.get('region', 'ap-northeast-1')
              
              status_config = {
                  'STARTED': {'emoji': '🚀', 'color': '#3498db'},
                  'SUCCEEDED': {'emoji': '✅', 'color': '#2ecc71'},
                  'FAILED': {'emoji': '❌', 'color': '#e74c3c'},
                  'CANCELED': {'emoji': '⚪', 'color': '#95a5a6'},
              }
              
              config = status_config.get(state, {'emoji': '❓', 'color': '#95a5a6'})
              
              pipeline_url = f"https://{region}.console.aws.amazon.com/codesuite/codepipeline/pipelines/{pipeline}/view"
              
              message = {
                  "attachments": [
                      {
                          "color": config['color'],
                          "blocks": [
                              {
                                  "type": "section",
                                  "text": {
                                      "type": "mrkdwn",
                                      "text": f"{config['emoji']} *CodePipeline {state}*\n\n*Pipeline:* <{pipeline_url}|{pipeline}>\n*Execution ID:* `{execution_id[:8]}`"
                                  }
                              }
                          ]
                      }
                  ]
              }
              
              req = urllib.request.Request(
                  SLACK_WEBHOOK_URL,
                  data=json.dumps(message).encode('utf-8'),
                  headers={'Content-Type': 'application/json'}
              )
              
              urllib.request.urlopen(req)
              
              return {'statusCode': 200}

  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref SlackNotifierLambda
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt PipelineStateChangeRule.Arn

  PipelineStateChangeRule:
    Type: AWS::Events::Rule
    Properties:
      Name: !Sub ${AWS::StackName}-state-change-rule
      EventPattern:
        source:
          - aws.codepipeline
        detail-type:
          - CodePipeline Pipeline Execution State Change
      Targets:
        - Id: SlackNotifierLambda
          Arn: !GetAtt SlackNotifierLambda.Arn

Outputs:
  PipelineName:
    Value: !Ref Pipeline
  PipelineConsoleUrl:
    Value: !Sub https://${AWS::Region}.console.aws.amazon.com/codesuite/codepipeline/pipelines/${Pipeline}/view
  LambdaFunctionName:
    Value: !Ref SlackNotifierLambda
  EventBridgeRuleName:
    Value: !Ref PipelineStateChangeRule
```

### デプロイ方法

```bash
aws cloudformation create-stack \
  --stack-name codepipeline-slack-demo \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=SlackWorkspaceId,ParameterValue=TXXXXXXXXX \
    ParameterKey=SlackChannelId,ParameterValue=CXXXXXXXXXX \
    ParameterKey=SlackWebhookUrl,ParameterValue=https://hooks.slack.com/services/xxx/xxx/xxx \
    ParameterKey=SourceBucketName,ParameterValue=codepipeline-slack-demo-source-xxxxxxxxxxxx \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

実際にデプロイしてみたところ、スタック作成と同時にパイプラインが実行されて、Slack通知も正常に届きました。

### IaC化で気づいたポイント

1. **SlackWebhookUrlの保護**: `NoEcho: true` で CloudFormation の出力やログに表示されないようにしています。本番環境ではSecrets Managerの利用を検討してください。

2. **Lambda関数のインラインコード**: テンプレート内にコードを埋め込んでいますが、コードが大きくなる場合はS3やECRからデプロイする方式に変更してください。

3. **EventBridgeルールのスコープ**: 現在のテンプレートではすべてのCodePipelineのイベントをキャプチャします。検証用なのでこの構成にしていますが、本番環境ではそのリージョン内の全パイプラインの通知がLambdaに飛んでしまうため、以下のように特定のパイプラインに限定するのが一般的です。

```yaml
EventPattern:
  source:
    - aws.codepipeline
  detail-type:
    - CodePipeline Pipeline Execution State Change
  detail:
    pipeline:
      - !Sub ${AWS::StackName}-pipeline
```

:::message alert
フィルタリングなしで本番運用すると、意図しない大量通知が発生する可能性があります。必ずパイプライン名でフィルタリングするか、Lambda側で対象パイプラインを判定するロジックを入れてください。
:::

## まとめ

今回、CodePipelineの実行結果をSlackに通知する2つの方法を検証してみました。

**方式A（AWS Chatbot）**
- セットアップが簡単
- インタラクティブボタンで操作可能
- AWS標準のリッチな通知

**方式B（EventBridge + Lambda）**
- 通知内容を完全カスタマイズ可能
- 柔軟なフィルタリングが可能
- メンション機能など拡張が容易

どちらの方式も一長一短があります。シンプルな通知であれば方式A、カスタマイズが必要であれば方式Bを選択するのが良さそうです。両方を併用することも可能なので、用途に応じて使い分けてみてください。

## リソースのクリーンアップ

検証後は以下のリソースを削除してください。

```bash
# CloudFormationスタックの削除
aws cloudformation delete-stack \
  --stack-name codepipeline-slack-demo \
  --region ap-northeast-1

# S3バケットの削除（バケット内のオブジェクトを先に削除）
aws s3 rm s3://codepipeline-slack-demo-source-xxxxxxxxxxxx --recursive
aws s3 rb s3://codepipeline-slack-demo-source-xxxxxxxxxxxx
```

## 次回予告

次回は **SlackからCodePipelineを実行するChatOps** を検証予定です。

Slackのスラッシュコマンドやボタンからパイプラインを起動できるようにすることで、デプロイ作業をさらに効率化できます。お楽しみに！
