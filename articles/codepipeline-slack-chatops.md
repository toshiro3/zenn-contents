---
title: "SlackからCodePipelineを実行するChatOpsを検証してみた"
emoji: "🚀"
type: "tech"
topics: ["AWS", "CodePipeline", "Slack", "ChatOps", "AmazonQ"]
published: true
---

## はじめに

本記事は、AWS CodePipelineシリーズの最終回です。

これまでの記事では、CodePipelineのパスフィルタ機能とSlack通知について検証してきました。今回は、SlackからCodePipelineを直接実行する「ChatOps」の設定を検証しました。

:::message
本記事は、前回の記事「[CodePipelineのSlack通知設定](https://zenn.dev/toshiro3/articles/codepipeline-slack-notification)」で構築した環境を流用しています。新規で環境を構築する場合は、先に前回の記事をご確認ください。
:::

ChatOpsを導入することで、以下のようなメリットがあります。

- **コンテキストスイッチの削減** - AWSコンソールを開かずにSlackからデプロイ実行
- **操作の可視化** - チームメンバー全員がデプロイ状況をリアルタイムで把握
- **迅速な対応** - 通知を確認してすぐに再デプロイなどの対応が可能

前回設定したSlack通知と組み合わせることで、「通知を受け取る → 状況確認 → 再実行」というフローがSlack上で完結できるか検証しました。

## 検証した構成

今回検証した構成は以下の通りです。

```
┌─────────────────────────────────────────────────────────────────┐
│  Slack                                                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ #pipeline-channel                                         │  │
│  │                                                           │  │
│  │  User: @Amazon Q codepipeline start-pipeline-execution    │  │
│  │        --name my-pipeline                                 │  │
│  │                                                           │  │
│  │  Amazon Q: I can run the command...                       │  │
│  │           [Run command] [Cancel]                          │  │
│  │                                                           │  │
│  │  Webhook: 🚀 CodePipeline STARTED                         │  │
│  │           ✅ CodePipeline SUCCEEDED                       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  AWS                                                            │
│                                                                 │
│  ┌──────────────────┐      ┌──────────────────┐                │
│  │  Amazon Q        │      │   IAM Role       │                │
│  │  Developer       │─────▶│   (Chatbot用)    │                │
│  └──────────────────┘      └──────────────────┘                │
│           │                         │                           │
│           │                         │ AssumeRole                │
│           ▼                         ▼                           │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  CodePipeline                                        │      │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐       │      │
│  │  │  Source  │───▶│  Build   │───▶│  Deploy  │       │      │
│  │  └──────────┘    └──────────┘    └──────────┘       │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 検証環境

今回の検証は、以下の環境で行いました。

- AWSアカウント
- Slackワークスペース（管理者権限あり）
- 前回の記事で作成したCodePipelineとSlack通知設定

前回の記事で設定したSlack通知の環境を流用しています。

## 検証手順

### Step 1: Slackワークスペースとの連携

まず、Amazon Q DeveloperとSlackワークスペースの連携を確認しました。

前回の記事でSlack通知を設定済みだったため、この手順はスキップしました。

新規で設定する場合は、以下の手順になります。

1. AWSコンソールで「chatbot」と検索し、「Amazon Q Developer in chat applications」を開く
2. 「Configure a chat client」から「Slack」を選択
3. 「Configure client」をクリック
4. Slackの認証画面が開くので、ワークスペースを選択して許可

### Step 2: チャンネル設定の確認

前回の記事で作成したチャンネル設定を確認しました。

| 項目 | 設定値 |
|------|--------|
| Configuration name | `codepipeline-slack-demo-slack-channel` |
| Slack channel | `#codepipeline-notifications` |
| Role settings | Channel role |
| Channel role | `codepipeline-slack-demo-chatbot-role` |
| Channel guardrail policies | `AdministratorAccess` |

### Step 3: IAMロールへの権限追加

前回の記事で作成したチャンネルロールには `CloudWatchReadOnlyAccess` のみが付与されていたため、CodePipelineの実行権限を追加しました。

IAMコンソールで `codepipeline-slack-demo-chatbot-role` を開き、以下のインラインポリシーを追加しました。

**ポリシー名**: `CodePipelineChatOpsPolicy`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "codepipeline:StartPipelineExecution",
                "codepipeline:GetPipelineState",
                "codepipeline:ListPipelines"
            ],
            "Resource": "*"
        }
    ]
}
```

:::message alert
**セキュリティに関する注意**: 上記の例では `Resource: "*"` としていますが、これは検証用の設定です。本番環境では、特定のパイプラインARNを指定して権限を絞ることを推奨します。詳細は「権限管理について」のセクションで解説しています。
:::

追加後のロールは以下のようになりました。

| ポリシー名 | タイプ |
|-----------|--------|
| CloudWatchReadOnlyAccess | AWS managed |
| CodePipelineChatOpsPolicy | Customer inline |

各アクションの用途は以下の通りです。

| アクション | 用途 |
|-----------|------|
| `StartPipelineExecution` | パイプラインの実行（必須） |
| `GetPipelineState` | 実行状態の確認 |
| `ListPipelines` | パイプライン一覧の表示 |

CloudFormationで管理する場合は、以下のように記述できます。

```yaml
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
      - arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess  # 通知用
    Policies:
      - PolicyName: CodePipelineChatOpsPolicy
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - codepipeline:StartPipelineExecution
                - codepipeline:GetPipelineState
                - codepipeline:ListPipelines
              Resource: '*'
```

### Step 4: Slackチャンネルへの招待

権限設定が完了したので、Slackチャンネルからコマンドを実行してみました。

まず、パイプライン実行コマンドを試してみました。

```
@Amazon Q codepipeline start-pipeline-execution --pipeline-name codepipeline-slack-demo-pipeline
```

すると、以下のような応答が返ってきました。

```
You mentioned @Amazon Q, but they're not in this channel.
[Add Them] [Do Nothing]
```

Amazon Qアプリがチャンネルに招待されていなかったため、「Add Them」をクリックして招待しました。

### Step 5: コマンド実行テスト（最初のハマりポイント）

Amazon Qを招待した後、再度コマンドを実行してみました。

```
@Amazon Q codepipeline start-pipeline-execution --pipeline-name codepipeline-slack-demo-pipeline
```

しかし、以下のような応答が返ってきました。

```
I can't answer that question. For an enhanced natural language experience using Amazon Q, visit Enabling Amazon Q Developer in Slack in the Amazon Q Developer in Slack User Guide.
```

CLIコマンドとして認識されず、自然言語の質問として処理されているようでした。

### Step 6: 原因の切り分け

原因を特定するため、いくつかのコマンドを試してみました。

**ヘルプコマンド**

```
@Amazon Q help
```

これは正常に動作し、Amazon Qの機能説明が表示されました。

**読み取り専用コマンド**

```
@Amazon Q codepipeline list-pipelines
```

このコマンドは認識され、リージョンを聞いてきました。`ap-northeast-1` を選択すると、パイプライン一覧が正常に表示されました。

```
I ran the read-only command

@Amazon Q codepipeline list-pipelines --region ap-northeast-1

in account 485326832412 with role codepipeline-slack-demo-chatbot-role in region Asia Pacific (Tokyo). These are items 1 to 3.

Pipelines (3):
-
    Name: cfn-path-filter-demo-pipeline
    Updated: 2026-01-29 15:39 UTC
    ...
-
    Name: codepipeline-slack-demo-pipeline
    Updated: 2026-01-31 11:11 UTC
    ...
```

読み取り系は動くのに、実行系が認識されない状況でした。

### Step 7: オプション名の違いを発見

パラメータなしでコマンドを実行してみました。

```
@Amazon Q codepipeline start-pipeline-execution
```

すると、Amazon Qが以下のように補完してくれました。

```
I can run the command

@Amazon Q codepipeline start-pipeline-execution --name codepipeline-slack-demo-pipeline --region ap-northeast-1

in account 485326832412 with role codepipeline-slack-demo-chatbot-role in region Asia Pacific (Tokyo).

What would you like me to do?

[Run command] [Add optional parameters] [Cancel command]
```

ここで原因が判明しました。**AWS CLIでは `--pipeline-name` ですが、Amazon Qでは `--name` に短縮されていました。**

### Step 8: パイプライン実行成功

「Run command」をクリックすると、パイプラインが正常に実行されました。

さらに、前回の記事で設定したSlack通知も連動して、同じチャンネルに通知が届きました。

```
🚀 CodePipeline STARTED
Pipeline: codepipeline-slack-demo-pipeline
Execution ID: 449be473

✅ CodePipeline SUCCEEDED
Pipeline: codepipeline-slack-demo-pipeline
Execution ID: 449be473
```

Slackからパイプラインを実行して、結果の通知もSlackで受け取るという、ChatOpsの流れが完成しました。

## コマンドエイリアスの検証

毎回長いコマンドを入力するのは面倒なので、コマンドエイリアス（ショートカット）機能を検証しました。

### エイリアスの作成

以下のコマンドでエイリアスを作成しました。

```
@Amazon Q alias create deploy-demo codepipeline start-pipeline-execution --name codepipeline-slack-demo-pipeline --region ap-northeast-1
```

正常に作成され、「I successfully created your alias.」と表示されました。

### エイリアスの実行

作成したエイリアスを実行してみました。

```
@Amazon Q run deploy-demo
```

登録したフルコマンドに展開され、確認画面が表示されました。「Run command」をクリックすると、パイプラインが実行されました。

### エイリアスの一覧表示

登録済みのエイリアスを確認しました。

```
@Amazon Q alias list
```

```
Here are all of the aliases created in this channel:
• deploy-demo ➡️ codepipeline start-pipeline-execution --name codepipeline-slack-demo-pipeline --region ap-northeast-1

To run a command alias, say:
@Amazon Q run <alias-name>
```

エイリアスはチャンネル単位で管理されており、チャンネルメンバー全員が共有して使用できることがわかりました。

### その他のエイリアス操作

以下の操作も確認しました。

**エイリアスの削除**

```
@Amazon Q alias delete deploy-demo
```

**変数を使ったエイリアス**

複数のパイプラインを運用している場合、変数を使ったエイリアスが便利です。

```
@Amazon Q alias create deploy codepipeline start-pipeline-execution --name $pipeline --region ap-northeast-1
```

実行時に変数を指定します。

```
@Amazon Q run deploy pipeline=my-other-pipeline
```

## 権限管理について

ChatOpsでは「誰がどのコマンドを実行できるか」の管理が重要です。今回は検証環境で1人しか参加していなかったため、実際の動作検証はできませんでしたが、権限管理の仕組みについて整理しました。

### 権限の構造

Amazon Q Developerの権限は、以下の2つの要素の**交差部分**（AND条件）で決まります。

```
実行可能な権限 = チャンネルロール（またはユーザーロール） ∩ ガードレールポリシー
```

| 要素 | 説明 |
|------|------|
| チャンネルロール / ユーザーロール | 実際に実行されるIAMロール |
| ガードレールポリシー | 許可するアクションの上限を定義 |

例えば、チャンネルロールにCodePipeline実行権限があっても、ガードレールポリシーが `ReadOnlyAccess` の場合は実行できません。

### Role Settings（ロール設定）

Amazon Q Developerでは、2種類のロール設定があります。

#### Channel role（チャンネルロール）

- 全メンバーが同じロールを使用
- 設定がシンプル
- 限定メンバーのプライベートチャンネル向け

今回の検証ではこちらを使用しました。

#### User-level roles（ユーザーレベルロール）

- メンバーごとに異なるロールを割り当て可能
- 各ユーザーが `@Amazon Q switch-roles` でロールを選択
- 公開チャンネルで権限を分けたい場合向け

:::message
**補足**: User-level rolesを使用する場合でも、ガードレールポリシーによる制限は有効です。たとえユーザーが強い権限を持つロールを選択しても、ガードレールで許可されていないアクションは実行できません。「ロール」と「ガードレール」の二重の壁で制御される仕組みになっています。
:::

### User-level rolesの制御の仕組み

User-level rolesを使用する場合、ユーザーがロールを選択するには**AWSコンソールへのログインが必要**です。

```
User-level rolesの流れ:
1. ユーザーが @Amazon Q switch-roles を実行
2. AWSコンソールへのリンクが表示される
3. AWSにサインインしてロールを選択
4. 以降、選択したロールでコマンド実行可能
```

つまり、AWSアカウントにアクセスできないユーザー（例：PM、デザイナー）は自動的にコマンド実行不可になります。

ただし、**AWSアカウントにログインできる全員がロールを選択可能**になるため、「AWSにログインできるが実行はさせたくない」というケースには対応できません。

### 推奨構成

セキュリティと運用のバランスを考慮すると、以下の構成が推奨です。

#### パターン1: チャンネルを分離（最も確実）

```
#pipeline-notifications（公開）
  └→ 通知のみ、コマンド実行設定なし
  └→ 誰でも参加可能

#pipeline-operations（プライベート）
  └→ コマンド実行可能
  └→ 実行権限を持つべきエンジニアのみ招待
```

#### パターン2: リソースレベルで制限

特定のパイプラインのみ実行可能にする場合：

```json
{
    "Effect": "Allow",
    "Action": "codepipeline:StartPipelineExecution",
    "Resource": [
        "arn:aws:codepipeline:ap-northeast-1:123456789012:staging-pipeline",
        "arn:aws:codepipeline:ap-northeast-1:123456789012:dev-pipeline"
    ]
}
```

本番環境のパイプラインを除外することで、誤操作のリスクを軽減できます。

#### パターン3: ガードレールで制限

ガードレールポリシーでCodePipelineのみ許可：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "codepipeline:StartPipelineExecution",
                "codepipeline:GetPipelineState",
                "codepipeline:ListPipelines"
            ],
            "Resource": "*"
        }
    ]
}
```

今回の検証では `AdministratorAccess` を使用しましたが、本番環境では最小権限の原則に従って適切に制限することを推奨します。

## ハマりポイント・注意点

今回の検証で遭遇したハマりポイントをまとめます。

### 1. Amazon QがCLIコマンドを認識しない

**症状**

```
@Amazon Q codepipeline start-pipeline-execution --pipeline-name my-pipeline
```

に対して「I can't answer that question.」と返答される。

**原因と解決策**

複数の原因が考えられます。

| 原因 | 解決策 |
|------|--------|
| Amazon Qアプリがチャンネルに招待されていない | `/invite @Amazon Q` で招待 |
| ユーザーがロールを選択していない（User-level roles使用時） | `@Amazon Q switch-roles` でロールを選択 |
| オプション名が間違っている | パラメータなしで実行して補完させる |

### 2. CLIオプション名の違い

**症状**

AWS CLIのドキュメント通りに `--pipeline-name` を使用しても認識されない。

**原因**

Amazon Q Developerでは、一部のオプション名が短縮されています。

| AWS CLI | Amazon Q |
|---------|----------|
| `--pipeline-name` | `--name` |

**解決策**

パラメータなしでコマンドを実行すると、Amazon Qがインタラクティブに補完してくれるため、正しいオプション名を確認できます。

### 3. helpコマンドで詳細を確認

困ったときは、ヘルプを確認すると良いことがわかりました。

```
@Amazon Q help
```

表示されるメニューから「Commands」を選択すると、利用可能なコマンドの一覧と使い方が表示されます。

## CloudFormation テンプレート（参考）

今回の検証で使用した構成をCloudFormationで管理する場合のテンプレート例です。

:::message
SlackワークスペースIDは、Amazon Q Developerコンソールで初回連携を行った後に取得できます。
:::

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CodePipeline ChatOps Configuration

Parameters:
  SlackWorkspaceId:
    Type: String
    Description: Slack Workspace ID (starts with T)
  SlackChannelId:
    Type: String
    Description: Slack Channel ID (starts with C)

Resources:
  # Chatbot用IAMロール
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
      Policies:
        - PolicyName: CodePipelineChatOpsPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - codepipeline:StartPipelineExecution
                  - codepipeline:GetPipelineState
                  - codepipeline:ListPipelines
                Resource: '*'

  # Slackチャンネル設定
  SlackChannelConfiguration:
    Type: AWS::Chatbot::SlackChannelConfiguration
    Properties:
      ConfigurationName: !Sub ${AWS::StackName}-slack-channel
      IamRoleArn: !GetAtt ChatbotRole.Arn
      SlackWorkspaceId: !Ref SlackWorkspaceId
      SlackChannelId: !Ref SlackChannelId
      LoggingLevel: INFO
      # ガードレールポリシー（本番環境では適切に制限）
      GuardrailPolicies:
        - arn:aws:iam::aws:policy/AdministratorAccess

Outputs:
  ChatbotRoleArn:
    Description: Chatbot IAM Role ARN
    Value: !GetAtt ChatbotRole.Arn
  SlackConfigurationArn:
    Description: Slack Channel Configuration ARN
    Value: !Ref SlackChannelConfiguration
```

## よく使うコマンド一覧

今回の検証で使用したコマンドをまとめます。

### パイプライン操作

```bash
# パイプライン一覧
@Amazon Q codepipeline list-pipelines

# パイプライン実行
@Amazon Q codepipeline start-pipeline-execution

# パイプライン状態確認
@Amazon Q codepipeline get-pipeline-state --name <pipeline-name>
```

### エイリアス管理

```bash
# エイリアス作成
@Amazon Q alias create <alias-name> <command>

# エイリアス一覧
@Amazon Q alias list

# エイリアス実行
@Amazon Q run <alias-name>

# エイリアス削除
@Amazon Q alias delete <alias-name>
```

### ユーティリティ

```bash
# ヘルプ
@Amazon Q help

# ロール切り替え
@Amazon Q switch-roles

# 設定確認
@Amazon Q set preferences
```

## まとめ

今回の検証で、SlackからCodePipelineを実行するChatOpsの設定方法を確認できました。

### 検証結果のポイント

1. **IAMロールへの権限追加が必要** - `codepipeline:StartPipelineExecution` など
2. **CLIオプション名が異なる場合がある** - `--pipeline-name` → `--name`（パラメータなしで実行すると補完してくれる）
3. **エイリアスで効率化できる** - よく使うコマンドはエイリアスに登録
4. **権限管理は慎重に** - 本番環境ではチャンネル分離やリソースレベル制限を推奨

### ChatOpsの効果

前回設定したSlack通知と組み合わせることで、以下のワークフローがSlack上で完結しました。

```
通知を確認 → 状況把握 → 必要に応じて再実行 → 結果確認
```

AWSコンソールを開く必要がなく、チームメンバー全員が操作状況を把握できるため、運用効率の向上が期待できます。

---

## シリーズ記事

本記事は、AWS CodePipelineシリーズの第3回（最終回）です。

| 回 | タイトル | 内容 |
|----|----------|------|
| 第1回 | [CodePipelineのパスフィルタ設定](https://zenn.dev/toshiro3/articles/codepipeline-path-filter) | 特定のファイル変更時のみパイプラインを実行 |
| 第2回 | [CodePipelineのSlack通知設定](https://zenn.dev/toshiro3/articles/codepipeline-slack-notification) | パイプラインの実行結果をSlackに通知 |
| 第3回 | SlackからCodePipelineを実行（本記事） | ChatOpsでSlackからパイプラインを実行 |

シリーズを通して、CodePipelineの運用効率を高める設定を検証してきました。これらを組み合わせることで、より効率的なCI/CD環境を構築できます。

---

最後まで読んでいただきありがとうございました！
