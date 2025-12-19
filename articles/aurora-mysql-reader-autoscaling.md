---
title: "Aurora MySQLでオートスケーリング設定時に手動でReaderを追加したらどうなるか検証してみた"
emoji: "🐬"
type: "tech"
topics: ["aws", "aurora", "mysql", "rds"]
published: false
---

## はじめに

Aurora MySQLでReaderインスタンスを増やす必要があり、手動で追加しようとしていました。ただ、クラスターにはオートスケーリングが設定されています。

ドキュメントを読むと「手動追加してもスケールインで削除されるから意味がないのでは？」と思ったのですが、本当にそうなのか検証してみたところ、実は自分の読み間違いだったことがわかりました。

## 状況

負荷増加に備えて、オートスケーリングが設定されているAurora MySQLクラスターのReaderインスタンスを増やすことになりました。

## ドキュメントを読んで思ったこと

[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.Integrating.AutoScaling.html)に以下の記載があります：

> Auto Scaling で管理するか、手動で追加するかにかかわらず、Aurora クラスター内のすべてのリーダーインスタンスを含めます。

この記述を読んで、「手動追加したReaderもオートスケーリングの管理対象になる」→「最低台数を超えていればスケールインで削除されるのでは？」と思いました。

本当にそうなのか、検証してみることにしました。

## 検証してみた

検証用のAuroraクラスターを作成して確認してみました。

### 構成

- Writerインスタンス: 1台
- Readerインスタンス: 1台
- オートスケーリング設定: 最低1台、最大3台

### 手動でReaderを追加

```bash
aws rds create-db-instance \
  --db-instance-identifier ${CLUSTER_NAME}-reader-manual \
  --db-cluster-identifier $CLUSTER_NAME \
  --db-instance-class db.t4g.medium \
  --engine aurora-mysql
```

これでReaderが2台になりました。最低台数は1台なので、私の理解が正しければスケールインで削除されるはず...

### スケールインを待つ

CloudWatchのアラームを確認すると、スケールイン用のアラーム（AlarmLow）が「ALARM」状態になっていました。

```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix TargetTracking-cluster:${CLUSTER_NAME} \
  --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue}'
```

```json
[
    {
        "Name": "TargetTracking-cluster:aurora-test-cluster-AlarmLow-...",
        "State": "ALARM"
    }
]
```

スケールインの条件を満たしているのに、30分以上待っても削除されません。

```bash
aws application-autoscaling describe-scaling-activities \
  --service-namespace rds \
  --resource-id cluster:$CLUSTER_NAME
```

```json
{
    "ScalingActivities": []
}
```

スケーリングアクティビティは空のまま。削除されない...？

## ドキュメントとコンソールを再確認

削除されなかったので、改めてドキュメントとコンソールを確認してみました。

AWSコンソールでオートスケーリングの設定を見ると、以下の記載がありました：

![Scale in設定画面](/images/aurora-autoscaling/scale-in-setting.png)

> **Scale in**
> Enable to allow this Auto Scaling policy to remove Aurora Replicas. **Aurora Replicas created by you are not removed by Auto Scaling.**

**手動で作成したReaderはオートスケーリングで削除されない**と明記されていました。

## 正しい理解

ドキュメントの「すべてのリーダーインスタンスを含めます」は：

- ✅ **カウント対象として含める**（現在のReader数を計算する際に含める）
- ❌ ~~削除対象として含める~~

私の読み間違いでした。

つまり：

1. 手動でReaderを追加する → Reader 2台になる
2. オートスケーリングは「現在2台ある」と認識する
3. 最低台数1台なので、スケールインの条件は満たす
4. **でも手動作成分は削除できない** → 何も起きない

手動追加したReaderは削除されないので、**手動追加しても問題ない**ということです。

## 運用上の注意点

手動追加したReaderは削除されませんが、オートスケーリングのカウント対象にはなります。

例えば、最低台数1台の設定で手動で2台目を追加した場合：
- オートスケーリングは「2台ある」と認識
- 負荷が上がっても「まだ余裕がある」と判断してスケールアウトしない可能性がある

手動追加とオートスケーリングを併用する場合は、この点を考慮して最小キャパシティを調整するのが良いでしょう。

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace rds \
  --resource-id cluster:your-cluster-name \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --min-capacity 2 \
  --max-capacity 5
```

## ついでに：Readerの上限は15台

Aurora MySQLのReaderインスタンスは、1クラスターあたり**最大15台**までという制限があります。オートスケーリングの最大キャパシティを設定する際も、この上限を超えることはできません。

## まとめ

- オートスケーリング設定済みのクラスターでも、**手動追加したReaderは削除されない**
- ドキュメントの「すべてのリーダーインスタンスを含めます」は削除対象ではなくカウント対象の話
- コンソールには「Aurora Replicas created by you are not removed by Auto Scaling.」と明記されている
- 手動追加とオートスケーリングを併用する場合は、カウント対象になることを考慮して運用する

ドキュメントをちゃんと読めという話ですが、検証して初めて気づくこともありますね。

---

## おまけ：検証環境の構築で学んだこと

今回の検証環境を構築する際に、いくつか学びがあったので共有します。

### エンジンバージョンの確認方法

Aurora MySQLのエンジンバージョンは以下のコマンドで確認できます：

```bash
aws rds describe-db-engine-versions \
  --engine aurora-mysql \
  --query 'DBEngineVersions[*].EngineVersion' \
  --region ap-northeast-1
```

```json
[
    "8.0.mysql_aurora.3.04.0",
    "8.0.mysql_aurora.3.04.1",
    "8.0.mysql_aurora.3.04.4",
    "8.0.mysql_aurora.3.08.0",
    ...
]
```

リージョンによって利用可能なバージョンが異なるので、事前に確認しておくと良いです。

### サポートされているインスタンスクラスの確認方法

エンジンバージョンによってサポートされているインスタンスクラスが異なります。以下のコマンドで確認できます：

```bash
aws rds describe-orderable-db-instance-options \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.4 \
  --query 'OrderableDBInstanceOptions[*].DBInstanceClass' \
  --region ap-northeast-1 \
  --output text | tr '\t' '\n' | sort -u
```

```
db.r5.large
db.r5.xlarge
...
db.r6g.large
db.r7g.large
...
db.t3.medium
db.t3.large
db.t4g.medium
db.t4g.large
```

例えば `db.t3.small` は Aurora MySQL 3.04.4 ではサポートされていません。検証用に小さいインスタンスを使おうとしてエラーになり、このコマンドで確認しました。

### 検証環境の構築手順

参考までに、今回の検証環境の構築手順を記載します。

#### 変数の設定

```bash
REGION=ap-northeast-1
PREFIX=aurora-test
PROFILE=your-profile

CLUSTER_NAME=${PREFIX}-cluster
MASTER_USERNAME=admin
MASTER_PASSWORD='YourPassword123!'
INSTANCE_CLASS=db.t4g.medium
ENGINE_VERSION=8.0.mysql_aurora.3.04.4
```

#### VPC・サブネット・セキュリティグループの作成

```bash
# VPC作成
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${PREFIX}-vpc}]" \
  --region $REGION \
  --profile $PROFILE \
  --query 'Vpc.VpcId' \
  --output text)

# サブネット作成（2つのAZ）
SUBNET_ID_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ${REGION}a \
  --region $REGION \
  --profile $PROFILE \
  --query 'Subnet.SubnetId' \
  --output text)

SUBNET_ID_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ${REGION}c \
  --region $REGION \
  --profile $PROFILE \
  --query 'Subnet.SubnetId' \
  --output text)

# セキュリティグループ作成
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name ${PREFIX}-sg \
  --description "Security group for Aurora test" \
  --vpc-id $VPC_ID \
  --region $REGION \
  --profile $PROFILE \
  --query 'GroupId' \
  --output text)

# DBサブネットグループ作成
aws rds create-db-subnet-group \
  --db-subnet-group-name ${PREFIX}-subnet-group \
  --db-subnet-group-description "Subnet group for Aurora test" \
  --subnet-ids $SUBNET_ID_1 $SUBNET_ID_2 \
  --region $REGION \
  --profile $PROFILE
```

#### Auroraクラスターの作成

```bash
# クラスター作成
aws rds create-db-cluster \
  --db-cluster-identifier $CLUSTER_NAME \
  --engine aurora-mysql \
  --engine-version $ENGINE_VERSION \
  --master-username $MASTER_USERNAME \
  --master-user-password $MASTER_PASSWORD \
  --db-subnet-group-name ${PREFIX}-subnet-group \
  --vpc-security-group-ids $SECURITY_GROUP_ID \
  --region $REGION \
  --profile $PROFILE

# Writerインスタンス作成
aws rds create-db-instance \
  --db-instance-identifier ${CLUSTER_NAME}-writer \
  --db-cluster-identifier $CLUSTER_NAME \
  --db-instance-class $INSTANCE_CLASS \
  --engine aurora-mysql \
  --region $REGION \
  --profile $PROFILE

# Readerインスタンス作成
aws rds create-db-instance \
  --db-instance-identifier ${CLUSTER_NAME}-reader-1 \
  --db-cluster-identifier $CLUSTER_NAME \
  --db-instance-class $INSTANCE_CLASS \
  --engine aurora-mysql \
  --region $REGION \
  --profile $PROFILE
```

#### オートスケーリングの設定

```bash
# スケーラブルターゲットの登録
aws application-autoscaling register-scalable-target \
  --service-namespace rds \
  --resource-id cluster:$CLUSTER_NAME \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --min-capacity 1 \
  --max-capacity 3 \
  --region $REGION \
  --profile $PROFILE

# スケーリングポリシーの作成
aws application-autoscaling put-scaling-policy \
  --service-namespace rds \
  --resource-id cluster:$CLUSTER_NAME \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --policy-name ${CLUSTER_NAME}-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "RDSReaderAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 300
  }' \
  --region $REGION \
  --profile $PROFILE
```

#### クリーンアップ

検証後は忘れずにリソースを削除しましょう。削除は作成と逆の順序で行います。

```bash
# オートスケーリング削除
aws application-autoscaling delete-scaling-policy \
  --service-namespace rds \
  --resource-id cluster:$CLUSTER_NAME \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --policy-name ${CLUSTER_NAME}-scaling-policy \
  --region $REGION \
  --profile $PROFILE

aws application-autoscaling deregister-scalable-target \
  --service-namespace rds \
  --resource-id cluster:$CLUSTER_NAME \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --region $REGION \
  --profile $PROFILE

# DBインスタンス削除
aws rds delete-db-instance \
  --db-instance-identifier ${CLUSTER_NAME}-reader-1 \
  --skip-final-snapshot \
  --region $REGION \
  --profile $PROFILE

aws rds delete-db-instance \
  --db-instance-identifier ${CLUSTER_NAME}-reader-manual \
  --skip-final-snapshot \
  --region $REGION \
  --profile $PROFILE

aws rds delete-db-instance \
  --db-instance-identifier ${CLUSTER_NAME}-writer \
  --skip-final-snapshot \
  --region $REGION \
  --profile $PROFILE

# クラスター削除（インスタンス削除完了後）
aws rds delete-db-cluster \
  --db-cluster-identifier $CLUSTER_NAME \
  --skip-final-snapshot \
  --region $REGION \
  --profile $PROFILE

# DBサブネットグループ削除（クラスター削除完了後）
aws rds delete-db-subnet-group \
  --db-subnet-group-name ${PREFIX}-subnet-group \
  --region $REGION \
  --profile $PROFILE

# セキュリティグループ・サブネット・VPC削除
aws ec2 delete-security-group --group-id $SECURITY_GROUP_ID --region $REGION --profile $PROFILE
aws ec2 delete-subnet --subnet-id $SUBNET_ID_1 --region $REGION --profile $PROFILE
aws ec2 delete-subnet --subnet-id $SUBNET_ID_2 --region $REGION --profile $PROFILE
aws ec2 delete-vpc --vpc-id $VPC_ID --region $REGION --profile $PROFILE
```
