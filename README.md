# humidifier_sample

CloudFormationテンプレートをRubyスクリプトで作成できるHumidifierというgemを使ってVPC環境を構築します。Humidifierは標準のテンプレート記法(JSON)では実装できない環境変数や設定ファイルの取り込み、条件判断、ループなどをCloudFormationテンプレートの作成時に組み込むことが出来ます。

# 参照元
https://github.com/localytics/humidifier

# 前提条件
* OS: macOS Sierra
* Rubyバージョン: 2.2.5

# 準備

## AWSアカウントの設定

環境変数にAWS認証情報とCloudFormationスタックをデプロイするリージョンを設定します。

```bash
export AWS_REGION=ap-northeast-1
export AWS_ACCESS_KEY_ID=AKIA****************
export AWS_SECRET_ACCESS_KEY=******************************
```

## Humidifierのインストール

```ruby:Gemfile
gem "humidifier"
gem "rake"
```

```bash
bundle install
```

# 基本的な使い方

最初にCloudFormationスタックを作成します。

```ruby
stack = Humidifier::Stack.new(name: "sample-stack", aws_template_format_version: "2010-09-09")
```

次に作成したスタックにリソースを追加します。この例では`AWS::EC2::VPC`をスタックに追加しています。

```ruby
vpc = Humidifier::EC2::VPC.new(
  cidr_block: Humidifier.ref("VpcCidr"),
  enable_dns_support: true,
  enable_dns_hostnames: true
)
stack.add('VPC', vpc)
```

パラメータの仕様はCloudFormationテンプレートのリファレンスのパラメータをスネークケース化したものです。
引用元: https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html

```json
{
   "Type" : "AWS::EC2::VPC",
   "Properties" : {
      "CidrBlock" : String,
      "EnableDnsSupport" : Boolean,
      "EnableDnsHostnames" : Boolean,
      "InstanceTenancy" : String,
      "Tags" : [ Resource Tag, ... ]
   }
}
```

パラメータも追加できます。

```ruby
stack.add_parameter("VpcCidr", type: "String", no_echo: false)
```
上記`Humidifier::EC2::VPC.new`メソッドで`Humidifier.ref`メソッドを使用して`VpcCidr`の値を参照しています。

最後にパラメータを指定してデプロイを実行します。

```ruby
stack.deploy_and_wait(parameters: [ { parameter_key: "VpcCidr", parameter_value: "10.0.0.0/16" } ])
```

チェンジセットを作成する場合は以下です。

```ruby
stack.create_change_set(parameters: [ { parameter_key: "VpcCidr", parameter_value: "10.0.0.0/16" } ])
```

Templateファイルは`to_cf`メソッドで取得できます。

```ruby
puts stack.to_cf
```

# スクリプトの例
CloudFormationスタックのクラスを作成し、Rakefileから実行するスクリプトの例です。

## CloudFormationスタックを構築するクラス
VPCを作成するクラスです

```ruby:vpc.rb
require "humidifier"

class VPC

  ##
  #=== コンストラクタ
  def initialize(stack_name)
    @stack_name = stack_name
    stack.add_parameter("VpcCidr", type: "String", no_echo: false)
    stack.add("VPC", vpc)
    stack.add("Subnet1", subnet1)
    stack.add("Subnet2", subnet2)
    stack.add("InternetGateway", internet_gateway)
    stack.add("VPCGatewayAttachment", vpc_gateway_attachment)
    stack.add("PublicRouteTable", public_route_table)
    stack.add("PublicRoute", public_route)
    stack.add("Subnet1RouteTableAssociation", subnet1_route_table_association)
    stack.add("Subnet2RouteTableAssociation", subnet2_route_table_association)
    stack.add("PublicNetworkAcl", public_network_acl)
    stack.add("InboundHTTPPublicNetworkAclEntry", inbound_http_public_network_acl_entry)
    stack.add("InboundSSHPublicNetworkAclEntry", inbound_ssh_public_network_acl_entry)
    stack.add("InboundEphemeralPublicNetworkAclEntry", inbound_ephemeral_public_network_acl_entry)
    stack.add("OutboundPublicNetworkAclEntry", outbound_public_network_acl_entry)
    stack.add("Subnet1NetworkAclAssociation", subnet1_network_acl_association)
    stack.add("Subnet2NetworkAclAssociation", subnet2_network_acl_association)
    stack.add("DHCPOptions", dhcp_options)
    stack.add_output("VpcId", value: Humidifier.ref("VPC"))
    stack.add_output("Subnet1Id", value: Humidifier.ref("Subnet1"))
    stack.add_output("Subnet2Id", value: Humidifier.ref("Subnet2"))
  end

  ##
  #=== CloudFormationスタック
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-stack.html AWS::CloudFormation::Stack
  def stack
    @stack ||= Humidifier::Stack.new(
      name: @stack_name,
      aws_template_format_version: "2010-09-09",
      description: "Sample CloudFormation Stack"
    )
  end

  private

  ##
  #=== VPC
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html
  def vpc
    Humidifier::EC2::VPC.new(
      cidr_block: Humidifier.ref("VpcCidr"),
      enable_dns_support: true,
      enable_dns_hostnames: true
    )
  end

  ##
  #=== サブネット1
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html
  def subnet1
    Humidifier::EC2::Subnet.new(
      vpc_id: Humidifier.ref("VPC"),
      availability_zone: "ap-northeast-1b",
      cidr_block: "10.0.0.0/24"
    )
  end

  ##
  #=== サブネット2
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html
  def subnet2
    Humidifier::EC2::Subnet.new(
      vpc_id: Humidifier.ref("VPC"),
      availability_zone: "ap-northeast-1c",
      cidr_block: "10.0.1.0/24"
    )
  end

  ##
  #=== インターネットゲートウェイ
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-internet-gateway.html
  def internet_gateway
    Humidifier::EC2::InternetGateway.new
  end

  ##
  #=== VPCゲートウェイアタッチメント
  def vpc_gateway_attachment
    Humidifier::EC2::VPCGatewayAttachment.new(
      vpc_id: Humidifier.ref("VPC"),
      internet_gateway_id: Humidifier.ref("InternetGateway")
    )
  end

  ##
  #=== パブリックルートテーブル
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route-table.html
  def public_route_table
    Humidifier::EC2::RouteTable.new(
      vpc_id: Humidifier.ref("VPC")
    )
  end

  ##
  #=== パブリックルート
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html
  def public_route
    Humidifier::EC2::Route.new(
      route_table_id: Humidifier.ref("PublicRouteTable"),
      destination_cidr_block: "0.0.0.0/0",
      gateway_id: Humidifier.ref("InternetGateway")
    )
  end

  ##
  #=== サブネット1をルートテーブルに関連付け
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html
  def subnet1_route_table_association
    Humidifier::EC2::SubnetRouteTableAssociation.new(
      subnet_id: Humidifier.ref("Subnet1"),
      route_table_id: Humidifier.ref("PublicRouteTable")
    )
  end

  ##
  #=== サブネット2をルートテーブルに関連付け
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html
  def subnet2_route_table_association
    Humidifier::EC2::SubnetRouteTableAssociation.new(
      subnet_id: Humidifier.ref("Subnet2"),
      route_table_id: Humidifier.ref("PublicRouteTable")
    )
  end

  ##
  #=== サブネット2をルートテーブルに関連付け
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html
  def public_network_acl
    Humidifier::EC2::NetworkAcl.new(
      vpc_id: Humidifier.ref("VPC")
    )
  end

  ##
  #=== ネットワークACLにエントリ(http)を作成します
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-network-acl-entry.html
  def inbound_http_public_network_acl_entry
    Humidifier::EC2::NetworkAclEntry.new(
      network_acl_id: Humidifier.ref("PublicNetworkAcl"),
      rule_number: 100,
      protocol: 6,
      rule_action: "allow",
      egress: false,
      cidr_block: "0.0.0.0/0",
      port_range: { From: 80, To: 80 }
    )
  end

  ##
  #=== ネットワークACLにエントリ(ssh)を作成します
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-network-acl-entry.html
  def inbound_ssh_public_network_acl_entry
    Humidifier::EC2::NetworkAclEntry.new(
      network_acl_id: Humidifier.ref("PublicNetworkAcl"),
      rule_number: 102,
      protocol: 6,
      rule_action: "allow",
      egress: false,
      cidr_block: "0.0.0.0/0",
      port_range: { From: 22, To: 22 },
    )
  end

  ##
  #=== ネットワークACLにエントリ(エフェメラルポート)を作成します
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-network-acl-entry.html
  def inbound_ephemeral_public_network_acl_entry
    Humidifier::EC2::NetworkAclEntry.new(
      network_acl_id: Humidifier.ref("PublicNetworkAcl"),
      rule_number: 103,
      protocol: 6,
      rule_action: "allow",
      egress: false,
      cidr_block: "0.0.0.0/0",
      port_range: { From: 1024, To: 65535 },
    )
  end

  ##
  #=== ネットワークACLにエントリ(アウトバウンド)を作成します
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-network-acl-entry.html
  def outbound_public_network_acl_entry
    Humidifier::EC2::NetworkAclEntry.new(
      network_acl_id: Humidifier.ref("PublicNetworkAcl"),
      rule_number: 100,
      protocol: 6,
      rule_action: "allow",
      egress: true,
      cidr_block: "0.0.0.0/0",
      port_range: { From: 0, To: 65535 },
    )
  end

  ##
  #=== サブネット1をネットワーク ACL に関連付けます
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-network-acl-assoc.html
  def subnet1_network_acl_association
    Humidifier::EC2::SubnetNetworkAclAssociation.new(
      subnet_id: Humidifier.ref("Subnet1"),
      network_acl_id: Humidifier.ref("PublicNetworkAcl")
    )
  end

  ##
  #=== サブネット2をネットワーク ACL に関連付けます
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-network-acl-assoc.html
  def subnet2_network_acl_association
    Humidifier::EC2::SubnetNetworkAclAssociation.new(
      subnet_id: Humidifier.ref("Subnet2"),
      network_acl_id: Humidifier.ref("PublicNetworkAcl")
    )
  end

  ##
  #=== VPC 用の DHCP オプションセット
  # @see https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-dhcp-options.html
  def dhcp_options
    Humidifier::EC2::DHCPOptions.new(
      domain_name: "ap-northeast-1.compute.internal",
      domain_name_servers: [ "AmazonProvidedDNS" ]
    )
  end
end
```

## Rakefileの作成
作成したVPCクラスを実行するRakefileを作成します。ここでスタック名やパラメータを指定しています。ENVなどで環境変数からも値を取得できます。

```ruby:Rakefile
require "rake"

stack_name = "sample-stack"
parameters = [
  { parameter_key: "VpcCidr", parameter_value: "10.0.0.0/16" }
]

namespace "VPC" do
  desc "VPCのデプロイ"
  task :deploy do
    require "./vpc"
    vpc = VPC.new(stack_name)
    vpc.stack.deploy_and_wait(parameters: parameters)
  end

  desc "VPCのチェンジセットの作成"
  task :create_change_set do
    require "./vpc"
    vpc = VPC.new(stack_name)
    vpc.stack.create_change_set(parameters: parameters)
  end

  desc "CloudFormationテンプレートの作成"
  task :template do
    require "./vpc"
    vpc = VPC.new(stack_name)
    puts vpc.stack.to_cf
  end
end
```

## スクリプトの実行

### VPCをデプロイする場合
```bash
rake VPC:deploy
```

スタックが作成されました
![CloudFormation_Management_Console.png](https://qiita-image-store.s3.amazonaws.com/0/81973/8317757b-5d7c-9ae8-9067-78a16f418299.png)

パラメータも指定されています
![CloudFormation_Management_Console.png](https://qiita-image-store.s3.amazonaws.com/0/81973/930fe96c-5515-5438-320a-1bc9dc719b90.png)

出力結果
![CloudFormation_Management_Console.png](https://qiita-image-store.s3.amazonaws.com/0/81973/7a9ff353-05fd-4cb8-30f6-c8a6a609b83a.png)

### テンプレートの出力
```bash
rake VPC:template
```

### チェンジセットの作成
```bash
rake VPC:create_change_set
```
チェンジセットが作成される
![CloudFormation_Management_Console.png](https://qiita-image-store.s3.amazonaws.com/0/81973/95ca085a-1187-f481-5ab5-fc2213b938d8.png)

変更される内容を確認して実行
![Change_Set_Detail.png](https://qiita-image-store.s3.amazonaws.com/0/81973/aa054722-57cc-431c-1209-b08f081af3ff.png)
