---
title: "AWS VPCのセカンダリCIDRでIPアドレス空間を拡張する"
emoji: "🔧"
type: "tech"
topics: ["aws", "vpc", "network"]
published: false
---

## はじめに

既存のVPCでサブネットを追加しようとしたとき、CIDRの空きが足りない——そんな状況に直面したことはないでしょうか。

私自身、業務で検証環境・本番環境を同一VPCで運用している構成に、新たに開発環境を追加する必要が生じました。VPCは`/16`で作成されていたものの、既存の予約ルールに従うと新規サブネットの配置が難しく、「VPCを分けてリソースを移行するしかないのか…」と考えていたところ、**セカンダリCIDR**という選択肢があることを知りました。

この記事では、セカンダリCIDRの概要と、VPCピアリングとの違いを整理した上で、実際にセカンダリCIDRを追加してホスト間で通信できることを検証します。

## セカンダリCIDRとは

VPCには作成時に指定するプライマリCIDRに加えて、後からCIDRブロックを追加できます。これがセカンダリCIDRです。

追加したCIDR範囲内に新しいサブネットを作成でき、プライマリCIDR側のリソースとも同一VPC内として通信できます。

### 制約

- **CIDRブロックのサイズ**: /28〜/16の範囲
- **VPCあたりの数**: 最大5つ（プライマリ1 + セカンダリ4）、上限緩和申請で増加可能
- **重複不可**: 既存CIDRと重複するブロックは追加できない
- **RFC 1918範囲の混在制限**: プライマリCIDRが`10.0.0.0/8`の範囲内にある場合、セカンダリCIDRも`10.0.0.0/8`の範囲内、またはRFC 1918外の範囲（100.64.0.0/10など）から選択する必要があります。`172.16.0.0/12`や`192.168.0.0/16`から追加することはできません。
- **ルートテーブルの既存ルートとの関係**: 既存ルートのdestinationと同じ、またはより大きいCIDRは追加不可。例えば、VPNやDirect Connect経由で`10.1.0.0/16`へのルートが既に存在する場合、そのVPCにセカンダリCIDRとして`10.1.0.0/16`を追加することはできません。

詳細は[公式ドキュメント](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)を参照してください。

## VPCピアリングとの違い

「IPアドレスが足りないならVPCを分ければいいのでは？」という発想もありますが、セカンダリCIDRとVPCピアリングは解決する課題が異なります。

| 観点 | セカンダリCIDR | VPCピアリング |
|------|---------------|---------------|
| **何を解決するか** | 既存VPCのIPアドレス空間拡張 | 異なるVPC間の通信 |
| **VPCの数** | 1つのまま | 2つ以上 |
| **通信経路** | VPC内部（local route） | ピアリング接続経由 |
| **ルートテーブル** | 自動でlocal routeに含まれる | 明示的に追加が必要 |
| **セキュリティグループ** | 同一VPC内として扱える | 同一リージョン内ならピア先VPCのSGを参照可能 |
| **名前解決 (Private DNS)** | 追加設定不要（同一VPC内で機能） | 「DNS解決の承諾」を有効にする必要あり |
| **データ転送料金** | なし（VPC内通信） | 同一AZ内は無料、AZをまたぐ場合は$0.01/GB |
| **クロスアカウント** | 不可 | 可能 |
| **クロスリージョン** | 不可 | 可能 |
| **設定の複雑さ** | 低 | 中（双方でルート設定・承認が必要） |

詳細は[Amazon VPC Pricing](https://aws.amazon.com/vpc/pricing/)および[VPC peering basics](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)を参照してください。

### ユースケースの違い

**セカンダリCIDRが向いているケース**

- 既存VPCのCIDRが逼迫したが、環境は分けたくない
- 追加コストを抑えたい
- シンプルに拡張したい

**VPCピアリングが向いているケース**

- 環境（本番/検証など）を論理的・管理的に分離したい
- 別アカウント・別リージョンとの通信が必要
- 障害影響範囲を分けたい

## 検証

セカンダリCIDRを追加し、プライマリCIDR側とセカンダリCIDR側のEC2インスタンス間で通信できることを確認します。

### 構成図

```
┌─────────────────────────────────────────────────┐
│  VPC                                            │
│  ├─ Primary CIDR: 10.0.0.0/16                   │
│  └─ Secondary CIDR: 10.1.0.0/16                 │
│                                                 │
│  ┌──────────────────┐  ┌──────────────────┐     │
│  │ Subnet-Primary   │  │ Subnet-Secondary │     │
│  │ 10.0.1.0/24      │  │ 10.1.1.0/24      │     │
│  │                  │  │                  │     │
│  │  ┌────────────┐  │  │  ┌────────────┐  │     │
│  │  │   EC2-A    │  │  │  │   EC2-B    │  │     │
│  │  │ 10.0.1.x   │◄─┼──┼─►│ 10.1.1.x   │  │     │
│  │  └────────────┘  │  │  └────────────┘  │     │
│  └──────────────────┘  └──────────────────┘     │
└─────────────────────────────────────────────────┘
```

### 環境構築

AWS CLIで構築します。Session Managerで接続するため、IAMロールとIGWも作成します。

```bash
# 環境設定
export AWS_PROFILE=your-profile  # 使用するプロファイル
export AWS_DEFAULT_REGION=ap-northeast-1

# 変数設定
VPC_CIDR_PRIMARY="10.0.0.0/16"
VPC_CIDR_SECONDARY="10.1.0.0/16"
SUBNET_PRIMARY="10.0.1.0/24"
SUBNET_SECONDARY="10.1.1.0/24"
AZ="${AWS_DEFAULT_REGION}a"

# VPC作成
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block $VPC_CIDR_PRIMARY \
  --query 'Vpc.VpcId' \
  --output text)

aws ec2 create-tags \
  --resources $VPC_ID \
  --tags Key=Name,Value=secondary-cidr-test

# セカンダリCIDR追加
aws ec2 associate-vpc-cidr-block \
  --vpc-id $VPC_ID \
  --cidr-block $VPC_CIDR_SECONDARY

# IGW作成・アタッチ
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

# ルートテーブルにIGWルート追加
RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)

aws ec2 create-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# サブネット作成（プライマリ側）
SUBNET_PRIMARY_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block $SUBNET_PRIMARY \
  --availability-zone $AZ \
  --query 'Subnet.SubnetId' \
  --output text)

# サブネット作成（セカンダリ側）
SUBNET_SECONDARY_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block $SUBNET_SECONDARY \
  --availability-zone $AZ \
  --query 'Subnet.SubnetId' \
  --output text)

# セキュリティグループ作成（ICMP許可）
# 検証用として10.0.0.0/8と広めに許可しています
# 本番環境では必要最小限のCIDR範囲に絞ってください
SG_ID=$(aws ec2 create-security-group \
  --group-name secondary-cidr-test-sg \
  --description "Allow ICMP for testing" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol icmp \
  --port -1 \
  --cidr 10.0.0.0/8

# IAMロール作成（Session Manager用）
cat << 'EOF' > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name SSMRoleForTest \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name SSMRoleForTest \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile \
  --instance-profile-name SSMRoleForTest

aws iam add-role-to-instance-profile \
  --instance-profile-name SSMRoleForTest \
  --role-name SSMRoleForTest

# IAMの反映待ち
sleep 10

# AMI取得（Amazon Linux 2023）
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*-x86_64" \
            "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

# EC2作成（プライマリ側）
INSTANCE_A=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_PRIMARY_ID \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=SSMRoleForTest \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-primary}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

# EC2作成（セカンダリ側）
INSTANCE_B=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_SECONDARY_ID \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=SSMRoleForTest \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-secondary}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

# 起動待ち
aws ec2 wait instance-running --instance-ids $INSTANCE_A $INSTANCE_B

# プライベートIP確認
IP_A=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_A \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text)

IP_B=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_B \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text)

echo "EC2-A (Primary): $INSTANCE_A / $IP_A"
echo "EC2-B (Secondary): $INSTANCE_B / $IP_B"
```

### ルートテーブルの確認

セカンダリCIDRを追加すると、ルートテーブルのlocal routeに自動的に含まれます。

```bash
aws ec2 describe-route-tables \
  --route-table-ids $RT_ID \
  --query 'RouteTables[0].Routes'
```

```json
[
    {
        "DestinationCidrBlock": "10.1.0.0/16",
        "GatewayId": "local",
        "Origin": "CreateRoute",
        "State": "active"
    },
    {
        "DestinationCidrBlock": "10.0.0.0/16",
        "GatewayId": "local",
        "Origin": "CreateRouteTable",
        "State": "active"
    },
    {
        "DestinationCidrBlock": "0.0.0.0/0",
        "GatewayId": "igw-xxxxxxxxx",
        "Origin": "CreateRoute",
        "State": "active"
    }
]
```

プライマリCIDR（10.0.0.0/16）とセカンダリCIDR（10.1.0.0/16）の両方が`local`として登録されていることが確認できます。

### 疎通確認

Session Managerで各EC2に接続し、pingで疎通確認します。

**EC2-A（プライマリ側）からEC2-B（セカンダリ側）へ**

```bash
aws ssm start-session --target $INSTANCE_A
```

```
sh-5.2$ ping -c 3 10.1.1.104
PING 10.1.1.104 (10.1.1.104) 56(84) bytes of data.
64 bytes from 10.1.1.104: icmp_seq=1 ttl=127 time=0.459 ms
64 bytes from 10.1.1.104: icmp_seq=2 ttl=127 time=0.171 ms
64 bytes from 10.1.1.104: icmp_seq=3 ttl=127 time=0.174 ms

--- 10.1.1.104 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2080ms
rtt min/avg/max/mdev = 0.171/0.268/0.459/0.135 ms
```

**EC2-B（セカンダリ側）からEC2-A（プライマリ側）へ**

```bash
aws ssm start-session --target $INSTANCE_B
```

```
sh-5.2$ ping -c 3 10.0.1.62
PING 10.0.1.62 (10.0.1.62) 56(84) bytes of data.
64 bytes from 10.0.1.62: icmp_seq=1 ttl=127 time=0.168 ms
64 bytes from 10.0.1.62: icmp_seq=2 ttl=127 time=0.206 ms
64 bytes from 10.0.1.62: icmp_seq=3 ttl=127 time=0.179 ms

--- 10.0.1.62 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2055ms
rtt min/avg/max/mdev = 0.168/0.184/0.206/0.015 ms
```

双方向で問題なく通信できることが確認できました。

### クリーンアップ

検証が終わったらリソースを削除します。

```bash
# EC2インスタンス削除
aws ec2 terminate-instances \
  --instance-ids $INSTANCE_A $INSTANCE_B

aws ec2 wait instance-terminated \
  --instance-ids $INSTANCE_A $INSTANCE_B

# セキュリティグループ削除
aws ec2 delete-security-group --group-id $SG_ID

# サブネット削除
aws ec2 delete-subnet --subnet-id $SUBNET_PRIMARY_ID
aws ec2 delete-subnet --subnet-id $SUBNET_SECONDARY_ID

# IGWデタッチ・削除
aws ec2 detach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# セカンダリCIDR解除
aws ec2 disassociate-vpc-cidr-block \
  --association-id $(aws ec2 describe-vpcs \
    --vpc-ids $VPC_ID \
    --query 'Vpcs[0].CidrBlockAssociationSet[?CidrBlock==`10.1.0.0/16`].AssociationId' \
    --output text)

# VPC削除
aws ec2 delete-vpc --vpc-id $VPC_ID

# IAMリソース削除
aws iam remove-role-from-instance-profile \
  --instance-profile-name SSMRoleForTest \
  --role-name SSMRoleForTest

aws iam delete-instance-profile \
  --instance-profile-name SSMRoleForTest

aws iam detach-role-policy \
  --role-name SSMRoleForTest \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam delete-role --role-name SSMRoleForTest

# 一時ファイル削除
rm -f trust-policy.json
```

## どちらを選ぶべきか

最後に、セカンダリCIDRとVPCピアリングの選択基準をまとめます。

```
IPアドレス空間が足りない
    │
    ├─ 同一VPC内で拡張したい
    │   └─ セカンダリCIDR
    │
    └─ 環境を分離したい、または別アカウント/リージョン間の通信が必要
        └─ VPCピアリング（または Transit Gateway）
```

今回のように「既存VPCのCIDRが逼迫したが、リソースの移行はしたくない」というケースでは、セカンダリCIDRがシンプルで有効な選択肢です。

## まとめ

セカンダリCIDRを使えば、既存VPCのIPアドレス空間を簡単に拡張できます。ルートテーブルへの自動追加、同一VPC内通信として扱えるシンプルさ、追加コストなしという点で、VPC内の拡張には適した選択肢だと感じました。

一方で、環境の分離や、クロスアカウント・クロスリージョンの通信が必要な場合はVPCピアリングやTransit Gatewayを検討することになります。

この記事が、同じような課題に直面した方の参考になれば幸いです。
