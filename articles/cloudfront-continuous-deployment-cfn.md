---
title: "CloudFrontの継続的デプロイメントをCloudFormationで構築してプレビュー環境を作る"
emoji: "🚀"
type: "tech"
topics: ["AWS", "CloudFront", "CloudFormation", "CICD"]
published: true
---

## はじめに

本番環境のリリース前に「本番と同じドメイン上で、一般公開せずに次バージョンの動作確認をしたい」というニーズは多くの現場にあると思います。

CloudFrontには**継続的デプロイメント（Continuous Deployment）**という機能があり、プライマリディストリビューションに対して**ステージングディストリビューション**を紐づけることで、同一ドメイン上でトラフィックの振り分けが可能です。振り分け方法にはヘッダーベースと重み付けの2種類があり、ヘッダーベースを使えば「特定のヘッダーを付与したリクエストだけステージング側に流す」ことができます。一般のリクエストには一切影響を与えずに、本番ドメイン上でリリース候補の動作確認ができるわけです。

本記事では、以下の構成をCloudFormationで段階的に構築し、運用フローまで検証した内容をまとめます。

**想定するユースケース**

```
GitHub
├── main ブランチ
│   └── 更新 → プライマリディストリビューションのオリジンを更新（一般公開用）
└── main-preview ブランチ
    └── 更新 → ステージングディストリビューションのオリジンを更新（事前確認用）
```

- **プライマリディストリビューション**: 一般ユーザーのトラフィックを受ける（現行バージョン）
- **ステージングディストリビューション**: ヘッダーベースで特定のリクエストのみルーティングし、次のリリース候補を事前確認
- 確認完了後、Promotionでステージングをそのままプライマリに昇格させるのではなく、`main-preview`の内容を`main`にマージしてプライマリ側を更新
- ステージングは常に「次のリリース候補の確認場所」として維持する

## 構成図

```
                    ┌─────────────────────────────────┐
                    │    CloudFront Primary Domain     │
                    │   (dxxxxxxxxxxxxx.cloudfront.net) │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────┴──────────────────┐
                    │  Continuous Deployment Policy    │
                    │  (Header: aws-cf-cd-staging)     │
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │ ヘッダーなし         │         ヘッダーあり │
              ▼                                         ▼
┌──────────────────────┐              ┌──────────────────────┐
│  Primary Distribution │              │  Staging Distribution │
│  (Staging: false)     │              │  (Staging: true)      │
└──────────┬───────────┘              └──────────┬───────────┘
           │ OAC                                  │ OAC
           ▼                                      ▼
┌──────────────────────┐              ┌──────────────────────┐
│  S3: primary-origin   │              │  S3: staging-origin   │
│  (本番コンテンツ)       │              │  (確認用コンテンツ)     │
└──────────────────────┘              └──────────────────────┘
```

## 検証ステップ

| ステップ | 内容 |
|---------|------|
| Step 1 | S3バケット + プライマリディストリビューションを作成 |
| Step 2 | ステージング用S3バケット + ステージングディストリビューションを追加 |
| Step 3 | ContinuousDeploymentPolicyを作成し、ヘッダーベースの振り分けを設定 |
| Step 4 | 運用フローの検証（コンテンツ更新 → 確認 → 本番反映） |

## Step 1: S3バケット + プライマリディストリビューション

まずはS3をオリジンとしたシンプルなCloudFrontディストリビューションを作成します。

```yaml:step1-primary.yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Step1: Primary CloudFront Distribution with S3 Origin"

Parameters:
  ProjectName:
    Type: String
    Default: cf-continuous-deploy
    Description: Project name used as prefix for resource names

Resources:
  # ========================================
  # S3 Bucket (Primary Origin)
  # ========================================
  PrimaryOriginBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${ProjectName}-primary-origin"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  PrimaryOriginBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref PrimaryOriginBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: AllowCloudFrontServicePrincipal
            Effect: Allow
            Principal:
              Service: cloudfront.amazonaws.com
            Action: s3:GetObject
            Resource: !Sub "${PrimaryOriginBucket.Arn}/*"
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${PrimaryDistribution}"

  # ========================================
  # CloudFront OAC
  # ========================================
  CloudFrontOAC:
    Type: AWS::CloudFront::OriginAccessControl
    Properties:
      OriginAccessControlConfig:
        Name: !Sub "${ProjectName}-oac"
        OriginAccessControlOriginType: s3
        SigningBehavior: always
        SigningProtocol: sigv4

  # ========================================
  # CloudFront Distribution (Primary)
  # ========================================
  PrimaryDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Comment: !Sub "${ProjectName} - Primary Distribution"
        Enabled: true
        Staging: false
        DefaultRootObject: index.html
        Origins:
          - Id: S3Origin
            DomainName: !GetAtt PrimaryOriginBucket.RegionalDomainName
            S3OriginConfig:
              OriginAccessIdentity: ""
            OriginAccessControlId: !GetAtt CloudFrontOAC.Id
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6  # CachingOptimized
          Compress: true
        PriceClass: PriceClass_200
        HttpVersion: http2

Outputs:
  PrimaryDistributionId:
    Value: !Ref PrimaryDistribution
  PrimaryDistributionDomainName:
    Value: !GetAtt PrimaryDistribution.DomainName
  PrimaryOriginBucketName:
    Value: !Ref PrimaryOriginBucket
  CloudFrontOACId:
    Value: !GetAtt CloudFrontOAC.Id
```

ポイントとして、S3バケットはパブリックアクセスをすべてブロックし、**OAC（Origin Access Control）**を通じてCloudFrontからのみアクセス可能にしています。また、`Staging: false`を明示していますが、これはStep 2以降でステージングディストリビューションと区別するために重要です。

なお、バケットポリシー（`PrimaryOriginBucketPolicy`）がディストリビューション（`PrimaryDistribution`）を参照し、ディストリビューションがバケットを参照しているため、一見すると循環参照に見えます。しかし、バケットポリシーの`Condition`句で`!Sub`を使って文字列としてARNを組み立てているため、CloudFormationはこれを直接的なリソース依存とは解釈せず、デプロイは正常に完了します。ただし初回デプロイ時は、ディストリビューションの作成が完了するまでバケットポリシーの`AWS:SourceArn`に指定したARNが存在しない状態になります。S3のバケットポリシーは存在しないARNを条件値として受け入れるため、エラーにはなりません。

デプロイしてテスト用コンテンツをアップロードします。

```bash
aws cloudformation deploy \
  --template-file step1-primary.yaml \
  --stack-name cf-continuous-deploy \
  --capabilities CAPABILITY_IAM

echo '<h1>Primary Distribution - Production</h1><p>Version: 1.0</p>' > index.html
aws s3 cp index.html s3://cf-continuous-deploy-primary-origin/index.html \
  --content-type "text/html"
```

CloudFront経由でのアクセスを確認します。

```bash
DOMAIN=$(aws cloudformation describe-stacks \
  --stack-name cf-continuous-deploy \
  --query "Stacks[0].Outputs[?OutputKey=='PrimaryDistributionDomainName'].OutputValue" \
  --output text)

curl https://$DOMAIN
# => <h1>Primary Distribution - Production</h1><p>Version: 1.0</p>
```

プライマリディストリビューションの構築が完了しました。

## Step 2: ステージング用S3バケット + ステージングディストリビューション

ステージング用のS3バケットと、`Staging: true`を設定したディストリビューションを追加します。

Step 1からの差分（追加リソース）のみ抜粋します。

```yaml
  # ========================================
  # S3 Bucket (Staging Origin)
  # ========================================
  StagingOriginBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${ProjectName}-staging-origin"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  StagingOriginBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref StagingOriginBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: AllowCloudFrontServicePrincipal
            Effect: Allow
            Principal:
              Service: cloudfront.amazonaws.com
            Action: s3:GetObject
            Resource: !Sub "${StagingOriginBucket.Arn}/*"
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${StagingDistribution}"

  # ========================================
  # CloudFront Distribution (Staging)
  # ========================================
  StagingDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Comment: !Sub "${ProjectName} - Staging Distribution"
        Enabled: true
        Staging: true
        DefaultRootObject: index.html
        Origins:
          - Id: S3Origin
            DomainName: !GetAtt StagingOriginBucket.RegionalDomainName
            S3OriginConfig:
              OriginAccessIdentity: ""
            OriginAccessControlId: !GetAtt CloudFrontOAC.Id
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          Compress: true
        PriceClass: PriceClass_200
        HttpVersion: http2
```

OACはプライマリとステージングで共有しています。

### ハマりポイント① HttpVersionの制約

ステージングディストリビューションでは`http2and3`（HTTP/3）がサポートされていません。設定するとデプロイ時に以下のエラーが発生します。

```
The parameter HTTP3 is not supported for Continuous Deployment Policy.
```

`HttpVersion: http2` に設定する必要があります。また、Step 3でContinuousDeploymentPolicyを紐づける際にプライマリとステージングの設定一致が求められるため、**プライマリ側もhttp2に揃えておく**必要があります。

### ハマりポイント② ステージングディストリビューションのドメインには直接アクセスできない

デプロイ後、マネジメントコンソールではステージングディストリビューションにも`dxxxxxxxxxxxxx.cloudfront.net`のようなドメイン名が表示されます。しかし、このドメインのDNS解決はできません。

```bash
curl https://dxxxxxxxxxxxxx.cloudfront.net
# => curl: (6) Could not resolve host: dxxxxxxxxxxxxx.cloudfront.net
```

これは仕様通りの挙動です。ステージングディストリビューションはプライマリのドメインを共有し、ContinuousDeploymentPolicyのルーティング条件（ヘッダーや重み付け）を通じてのみアクセスされます。つまり、ステージングの動作確認はStep 3でContinuousDeploymentPolicyを設定してから行います。

また、ステージングディストリビューションに代替ドメイン名（CNAME）を設定してアクセスしようとしても、プライマリと重複するため設定できません。ステージングへのアクセス手段はあくまでContinuousDeploymentPolicyによるルーティングに限定されます。

## Step 3: ContinuousDeploymentPolicyの作成とヘッダーベースルーティング

ContinuousDeploymentPolicyを作成し、プライマリディストリビューションに紐づけます。

```yaml
  # ========================================
  # Continuous Deployment Policy
  # ========================================
  ContinuousDeploymentPolicy:
    Type: AWS::CloudFront::ContinuousDeploymentPolicy
    Properties:
      ContinuousDeploymentPolicyConfig:
        Enabled: true
        StagingDistributionDnsNames:
          - !GetAtt StagingDistribution.DomainName
        TrafficConfig:
          Type: SingleHeader
          SingleHeaderConfig:
            Header: aws-cf-cd-staging
            Value: "true"
```

プライマリディストリビューションの`DistributionConfig`にContinuousDeploymentPolicyIdを追加します。

```yaml
  PrimaryDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        # ... 既存の設定 ...
        ContinuousDeploymentPolicyId: !GetAtt ContinuousDeploymentPolicy.Id
```

### ハマりポイント③ CloudFormationのスキーマ

CloudFormationで`ContinuousDeploymentPolicy`を定義する際、ヘッダー設定は`TrafficConfig`の中にネストする必要があります。以下のように`SingleHeaderPolicyConfig`をトップレベルに書くとエラーになります。

```yaml
# ❌ これはエラーになる
ContinuousDeploymentPolicyConfig:
  Enabled: true
  StagingDistributionDnsNames:
    - !GetAtt StagingDistribution.DomainName
  SingleHeaderPolicyConfig:
    Header: aws-cf-cd-staging
    Value: "true"
```

```
The parameter Type is not valid.
```

正しくは`TrafficConfig.Type`と`TrafficConfig.SingleHeaderConfig`の組み合わせです。

### 動作確認

同じドメインに対してヘッダーの有無でレスポンスが切り替わることを確認します。

```bash
# ヘッダーなし → プライマリ（Version: 1.0）
curl https://$DOMAIN
# => <h1>Primary Distribution - Production</h1><p>Version: 1.0</p>

# ヘッダーあり → ステージング（Version: 2.0-rc1）
curl -H "aws-cf-cd-staging: true" https://$DOMAIN
# => <h1>Staging Distribution - Preview</h1><p>Version: 2.0-rc1</p>
```

同一ドメイン上で、ヘッダーを付与したリクエストだけがステージングのコンテンツを返すことが確認できました。

確認担当者には、ブラウザ拡張（ModHeaderなど）やプロキシで`aws-cf-cd-staging: true`ヘッダーを自動付与する設定を共有すれば、普段のブラウジングと同じ操作でプレビュー環境にアクセスできます。

:::message
プライマリとステージングは同一ドメインでコンテンツを配信するため、ブラウザのディスクキャッシュに注意が必要です。ヘッダーの有無でレスポンスが切り替わっても、ブラウザがキャッシュ済みのレスポンスを返してしまう場合があります。ステージングでの確認時はブラウザのキャッシュを無効化するか、シークレットウィンドウを使うと確実です。CloudFront側のキャッシュについては、プライマリとステージングでキャッシュは独立しているため混在の心配はありません。
:::

## Step 4: 運用フローの検証

### リリースサイクルのシミュレーション

実際の運用を想定して、ステージングでの確認 → プライマリへの反映の流れを検証します。

**1. ステージングに新しいリリース候補をデプロイ**

```bash
echo '<h1>Staging Distribution - Preview</h1><p>Version: 2.0-rc2</p>' > index.html
aws s3 cp index.html s3://cf-continuous-deploy-staging-origin/index.html \
  --content-type "text/html"

# 必要に応じてキャッシュをInvalidation
STAGING_ID=$(aws cloudformation describe-stacks \
  --stack-name cf-continuous-deploy \
  --query "Stacks[0].Outputs[?OutputKey=='StagingDistributionId'].OutputValue" \
  --output text)

aws cloudfront create-invalidation \
  --distribution-id $STAGING_ID \
  --paths "/*"
```

**2. ステージングで動作確認**

```bash
curl -H "aws-cf-cd-staging: true" https://$DOMAIN
# => <h1>Staging Distribution - Preview</h1><p>Version: 2.0-rc2</p>
```

**3. 確認OK → プライマリに反映**

```bash
echo '<h1>Primary Distribution - Production</h1><p>Version: 2.0</p>' > index.html
aws s3 cp index.html s3://cf-continuous-deploy-primary-origin/index.html \
  --content-type "text/html"

PRIMARY_ID=$(aws cloudformation describe-stacks \
  --stack-name cf-continuous-deploy \
  --query "Stacks[0].Outputs[?OutputKey=='PrimaryDistributionId'].OutputValue" \
  --output text)

aws cloudfront create-invalidation \
  --distribution-id $PRIMARY_ID \
  --paths "/*"
```

**4. 両方を確認**

```bash
curl https://$DOMAIN
# => <h1>Primary Distribution - Production</h1><p>Version: 2.0</p>

curl -H "aws-cf-cd-staging: true" https://$DOMAIN
# => <h1>Staging Distribution - Preview</h1><p>Version: 2.0-rc2</p>
```

プライマリが`Version: 2.0`に更新され、ステージングには`Version: 2.0-rc2`が残っています。それぞれが独立して更新できることが確認できました。

### ステージングの長期維持確認

ステージングを常設の確認環境として維持し続けられるかを検証します。Promotionを実行せずに次のリリースサイクルに入ります。

```bash
# 次のリリース候補をステージングに投入
echo '<h1>Staging Distribution - Preview</h1><p>Version: 3.0-rc1</p>' > index.html
aws s3 cp index.html s3://cf-continuous-deploy-staging-origin/index.html \
  --content-type "text/html"

aws cloudfront create-invalidation --distribution-id $STAGING_ID --paths "/*"
```

```bash
# プライマリ → Version: 2.0（変更なし）
curl https://$DOMAIN
# => <h1>Primary Distribution - Production</h1><p>Version: 2.0</p>

# ステージング → Version: 3.0-rc1（新しいリリース候補）
curl -H "aws-cf-cd-staging: true" https://$DOMAIN
# => <h1>Staging Distribution - Preview</h1><p>Version: 3.0-rc1</p>
```

Promotionを実行しなくても、ステージングの長期維持と独立した更新サイクルに問題がないことが確認できました。

## 最終的なCloudFormationテンプレート（全体）

Step 1〜3のすべてのリソースを含む完成版のテンプレートです。

```yaml:cf-continuous-deploy.yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "CloudFront Continuous Deployment with header-based routing"

Parameters:
  ProjectName:
    Type: String
    Default: cf-continuous-deploy
    Description: Project name used as prefix for resource names
  StagingAccessHeader:
    Type: String
    Default: aws-cf-cd-staging
    Description: Header name for staging access
  StagingAccessValue:
    Type: String
    Default: "true"
    Description: Header value for staging access

Resources:
  # ========================================
  # S3 Bucket (Primary Origin)
  # ========================================
  PrimaryOriginBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${ProjectName}-primary-origin"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  PrimaryOriginBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref PrimaryOriginBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: AllowCloudFrontServicePrincipal
            Effect: Allow
            Principal:
              Service: cloudfront.amazonaws.com
            Action: s3:GetObject
            Resource: !Sub "${PrimaryOriginBucket.Arn}/*"
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${PrimaryDistribution}"

  # ========================================
  # S3 Bucket (Staging Origin)
  # ========================================
  StagingOriginBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${ProjectName}-staging-origin"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  StagingOriginBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref StagingOriginBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: AllowCloudFrontServicePrincipal
            Effect: Allow
            Principal:
              Service: cloudfront.amazonaws.com
            Action: s3:GetObject
            Resource: !Sub "${StagingOriginBucket.Arn}/*"
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${StagingDistribution}"

  # ========================================
  # CloudFront OAC (shared)
  # ========================================
  CloudFrontOAC:
    Type: AWS::CloudFront::OriginAccessControl
    Properties:
      OriginAccessControlConfig:
        Name: !Sub "${ProjectName}-oac"
        OriginAccessControlOriginType: s3
        SigningBehavior: always
        SigningProtocol: sigv4

  # ========================================
  # Continuous Deployment Policy
  # ========================================
  ContinuousDeploymentPolicy:
    Type: AWS::CloudFront::ContinuousDeploymentPolicy
    Properties:
      ContinuousDeploymentPolicyConfig:
        Enabled: true
        StagingDistributionDnsNames:
          - !GetAtt StagingDistribution.DomainName
        TrafficConfig:
          Type: SingleHeader
          SingleHeaderConfig:
            Header: !Ref StagingAccessHeader
            Value: !Ref StagingAccessValue

  # ========================================
  # CloudFront Distribution (Primary)
  # ========================================
  PrimaryDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Comment: !Sub "${ProjectName} - Primary Distribution"
        Enabled: true
        Staging: false
        ContinuousDeploymentPolicyId: !GetAtt ContinuousDeploymentPolicy.Id
        DefaultRootObject: index.html
        Origins:
          - Id: S3Origin
            DomainName: !GetAtt PrimaryOriginBucket.RegionalDomainName
            S3OriginConfig:
              OriginAccessIdentity: ""
            OriginAccessControlId: !GetAtt CloudFrontOAC.Id
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          Compress: true
        PriceClass: PriceClass_200
        HttpVersion: http2

  # ========================================
  # CloudFront Distribution (Staging)
  # ========================================
  StagingDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Comment: !Sub "${ProjectName} - Staging Distribution"
        Enabled: true
        Staging: true
        DefaultRootObject: index.html
        Origins:
          - Id: S3Origin
            DomainName: !GetAtt StagingOriginBucket.RegionalDomainName
            S3OriginConfig:
              OriginAccessIdentity: ""
            OriginAccessControlId: !GetAtt CloudFrontOAC.Id
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          Compress: true
        PriceClass: PriceClass_200
        HttpVersion: http2

Outputs:
  PrimaryDistributionId:
    Value: !Ref PrimaryDistribution
  PrimaryDistributionDomainName:
    Value: !GetAtt PrimaryDistribution.DomainName
  PrimaryOriginBucketName:
    Value: !Ref PrimaryOriginBucket
  StagingDistributionId:
    Value: !Ref StagingDistribution
  StagingDistributionDomainName:
    Value: !GetAtt StagingDistribution.DomainName
  StagingOriginBucketName:
    Value: !Ref StagingOriginBucket
  ContinuousDeploymentPolicyId:
    Value: !GetAtt ContinuousDeploymentPolicy.Id
```

## Promotionを使わない運用についての補足

CloudFrontの継続的デプロイメントには**Promotion**という機能があり、ステージングの設定をそのままプライマリに昇格させることができます。しかし、今回の構成ではPromotionを使わない運用を採用しています。

Promotionはステージングの**ディストリビューション設定**をプライマリにコピーする機能であり、**オリジンのコンテンツをコピーするものではありません**。今回の構成ではプライマリとステージングで異なるS3バケットをオリジンとしているため、Promotionを実行するとプライマリのオリジンがステージング用S3に切り替わってしまいます。

今回あえてバケットを分けているのは、環境の物理的な分離（独立性）を優先するためです。バケットを分けることで、プライマリとステージングのコンテンツライフサイクルが完全に独立し、誤って確認中のコンテンツが本番に露出するリスクを排除できます。この設計方針を採る以上、Promotionを使わない運用が合理的です。

代わりに、以下のフローで運用します。

1. `main-preview`ブランチの更新でステージング用S3を更新
2. ヘッダー付きリクエストでステージング上の動作を確認
3. 確認OK → `main-preview`を`main`にマージ
4. `main`ブランチの更新でプライマリ用S3を更新
5. ステージングは次のリリース候補の確認場所として継続利用

この運用により、ステージングを常設のプレビュー環境として維持できます。

## CloudFormation構築時のハマりポイントまとめ

| # | 内容 | エラーメッセージ |
|---|------|--------------|
| 1 | ステージングディストリビューションでは`HttpVersion: http2and3`が使えない | `The parameter HTTP3 is not supported for Continuous Deployment Policy.` |
| 2 | ステージングのドメイン名はDNS解決できない（プライマリのドメインを共有する仕様） | `Could not resolve host` |
| 3 | ContinuousDeploymentPolicyのヘッダー設定は`TrafficConfig`内にネストする必要がある | `The parameter Type is not valid.` |

特に1と3はCloudFormationで構築する場合に遭遇しやすいポイントです。マネジメントコンソールからの操作では自動的に制約が適用されるため気づきにくいですが、IaCで管理する場合は注意が必要です。

## リソースの削除

検証が完了したら、リソースを削除します。S3バケットにオブジェクトが残っているとスタック削除に失敗するため、先にバケットを空にします。

```bash
aws s3 rm s3://cf-continuous-deploy-primary-origin --recursive
aws s3 rm s3://cf-continuous-deploy-staging-origin --recursive

aws cloudformation delete-stack --stack-name cf-continuous-deploy
```

:::message
ContinuousDeploymentPolicyがプライマリに紐づいている状態での削除順序はCloudFormationが自動的に解決してくれます。手動で削除する場合は、プライマリからContinuousDeploymentPolicyの紐づけを解除 → Policy削除 → ステージング削除 → プライマリ削除の順序が必要です。
:::

## おわりに

CloudFrontの継続的デプロイメントを使って、ヘッダーベースで本番ドメイン上にプレビュー環境を構築する方法をCloudFormationで検証しました。

同一ドメイン上でヘッダーの有無だけでプライマリとステージングを切り替えられるため、ヘッダーを付与するだけでプレビュー環境にアクセスでき、一般ユーザーには一切影響を与えません。ステージングはPromotionで消費せず常設の確認環境として維持する運用とすることで、リリースサイクルごとにステージングを再作成する手間も不要です。

CloudFormationで構築する際にはいくつかの制約やスキーマの違いがあるため、本記事のハマりポイントが参考になれば幸いです。
