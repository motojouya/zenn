---
title: "EC2を開発環境にする"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["EC2", "lambda", "JavaScript"]
published: true
published_at: 2019-04-16
---

[EC2をスーパー快適な開発環境として使う](https://qiita.com/uraura/items/35d25f8ccf849f7fd1d7)を参考に、自分の技術スタックでできるように少しアレンジをくわえて自分の開発環境をEC2に構築しました。
とても参考になりました。ありがとうございます。
ちなみに僕はAWSに関してはまじの素人なので、間違っていることがあれば教えてもらえると助かります。

## モチベーション
まずは、なぜEC2で開発環境を作るのか、というモチベーションを確認しておこうと思います。

### ローカルPC
手元のPCが貧弱でも、EC2を使えば、ほしい時にほしい性能のマシンが手に入ります。
PCは3-5年くらい使うと思うのですが、家族がいる身としては、3-5年に一度10-20万のマシンなど買ってられないのです。
それだったら、5,6万のマシンでブラウザを操作しつつ、開発自体はEC2でできないだろうかと考えました。

### HTTPS
ローカルからでも、何かしら方法はありますが、Route53経由でのアクセスでLet's Encryptを使えば、もろもろ楽にHTTPS環境が手に入ります。
なかなかlocalhostや自己証明ではブラウザにイチイチ怒られたりと面倒だったので、ドメインとってRoute53を使えば楽になります。

### AWS Access Keyが不要
ローカルPCにsshの秘密鍵を入れておく必要はあるのですが、AWS Access Keyは不要になります。
EC2のロールで権限を付与しておけば問題ないからです。（という認識ですが合ってますよね？）
AWS Access Keyがなければ漏れる心配もありません。

### vimで開発
ねらいということではなく、むしろ前提条件なのですが、僕はvimとtmuxさえあれば開発できるので、こういった環境を作りました。
VSCodeじゃないとダメなんだよという人はEC2に開発環境を作っても幸せになれないかもしれません。

## 考えたこと

基本的には、元の記事に沿った形で構築していったわけですが、その前に以下の2点ができないか考えて見ました。

1. LoadBalancer使ったらHTTPS環境も一発で手に入らないかな？
2. CloudFormationでできないかな？

ですが、両方しないことにしました。
Classic Load Balancerであればssh接続をつなげてくれるので、それでできそう！と思ったのですが、ヘルスチェックはHTTPで200を返す必要があるようでした。
開発環境なので常にHTTPのプロセスを起動しているわけではないし、仮にNginxを常時起動しておくのであれば、Nginxにsslをやってもらえばいいわけで。
LoadBalancerを使うのであればCloudFormationで宣言的に環境構築できたほうが楽そうでしたが、今回はEC2とRoute53の設定のみで、なので元記事に倣う形にしました。

HTTPS込みの開発環境をEC2にNginx入れずに作る、もっといい方法があれば、教えて欲しいです。

なので[EC2をスーパー快適な開発環境として使う](https://qiita.com/uraura/items/35d25f8ccf849f7fd1d7)にならって、LambdaでSpot Fleet RequestでEC2を起動し、そのIPアドレスをRoute53に紐付ける形にしました。
元記事を前提に構築していくので、先にそちらを確認してもらうと理解がスムーズかもしれません。

## 構築手順

長々と書きましたが、ようやく構築手順です。

1. IAMの作成
2. その他の準備
3. 初期化スクリプトの作成
4. 一旦SpotFleetRequestをしてみる
5. Lambdaの登録

僕は普段からUbuntuなのでUbuntuで構築しました。またPythonは書けないので、LambdaはJavaScriptで作っています。

### 1. IAMの作成
LambdaからRequest Spot FleetしてEC2を起動するIAMユーザという4名の登場人物がいるので、4つ作る必要があります。

1. Lambdaロール
2. Spot Fleetのロール
3. EC2に付与するロール
4. Lambdaを実行するIAMユーザ

非常に申し訳ないのですが、ここは適当に作ってしまったので、適当にしか伝えられないです。

#### Lambdaロール
これはSpot Fleet Requestを管理コンソールから作る時に必要なものを指摘してくれます。
それを元にLambda用のロールを作ればいいのですが、EC2のInstanceProfileを選択して付与するために、そのlist権限が追加で必要でした。

#### Spot Fleetのロール
以下のポリシーをつけたロールを作成します

- AmazonEC2SpotFleetTaggingRole

#### EC2に付与するロール
これは自由に作りますが、初期化スクリプトの中でEC2とRoute53を操作しているので、それぞれのFullAccessでもつけておけばいいと思います。
あとは開発で使うAWSリソースへのアクセス権限をバンバンつけておきます。

#### Lambdaを実行するIAMユーザ
これもLambdaのロールを基準に色々つけておいたほうが楽でした。
デバッグに使うので、適度に色々つけて置いたほうがよさそうです。
lambdaに付与するロールを選択するために、Roleのlist権限が必要でした。

### 2. その他の準備
以下の2つを作っておきます。
- セキュリティグループ
- アタッチするEBS

EBSは同一のアベイラビリティゾーンに無いとダメみたいなので、作ったアベイラビリティゾーンはメモしておきます。
セキュリティグループはFireWallの代わりになるものなので、sshポートとhttpポートなどを開けておきます。

またRoute53にドメインの登録が必要です。
この記事の範囲外なので省きますが、Freenomなどを使えば無料のドメインが手にはいります。

### 3. 初期化スクリプトの作成
以下のリポジトリにまとめました。
https://github.com/motojouya/ec2-develop

基本的には、元記事に記載されていることをそのまま実行するスクリプトです。
[sshの設定](https://github.com/motojouya/ec2-develop/blob/master/sshd_config.tmpl)だけ違うPortにしたり、パスワードログイン禁止にしたり、sshdを再起動したりしています。
最低限の設定というやつですね、足りてないかもですが。

あとは最初に作られていたubuntuユーザを削除しています。
新しく作ったユーザのshellの設定関連は`/etc/skel`にあるファイルをコピーしてくればよいようです。
アタッチしたEBS内のディレクトリなので、適当なタイミングでコピーしておけば問題ありません。

それと1点ハマったのが、インスタンスタイプです。

```shell
aws ec2 attach-volume --volume-id $volume_id --instance-id $instance_id --device /dev/xvdb --region $region
aws ec2 wait volume-in-use --volume-ids $volume_id
until [ -e /dev/nvme1n1 ]; do
    sleep 1
done
mkdir /home/$username
# mkfs -t ext4 /dev/nvme1n1
mount /dev/nvme1n1 /home/$username
```

上記のスクリプトでおかしいのは`/dev/xvdb`にEBSをアタッチしているはずなのに、マウントは`/dev/nvme1n1`からしています。
あまり詳しく調べてないので、そこら辺の事情はわかっていないのですが、新しい世代のインスタンスタイプだと、こうなるようです。
Nitroインスタンスと呼ばれるもののようです。tなら3から、mなら5からが該当します。
Spot Fleet Requestでは、複数のインスタンスタイプを候補として指定できますが、Nitroとそうでないインスタンスタイプを混ぜるとスクリプトの内容を分岐させる必要がありそうです。
僕はそんなの面倒なので、すべてNitroインスタンスに統一することにしました。

#### aptのリポジトリの修正
あまり調べられてないのですが、aws上のaptリポジトリを参照してるのかな？以下のスクリプトがなくても動いていたのですが、ある日突然ダメになりました。
なのでaptリポジトリを`apt update`の前に修正して、普通の？aptのリポジトリを参照するようにします。
[この](https://www.saintsouth.net/blog/aws-ec2-ubuntu-rudimentary-trouble-shooting-memo/)ブログを参考にさせてもらいました。

```ssh
cp -p /etc/apt/sources.list /etc/apt/sources.list.bak
sed -i 's/ap-northeast-1\.ec2\.//g' /etc/apt/sources.list
```


### 4. 一旦SpotFleetRequestをしてみる
最初に自分の欲しい条件でSpotFleetRequestを作りました。
そのためのJSONを自分で作るのはしんどいですが、管理コンソールからJSONをダウンロードできるので、少し手をくわえてやれば簡単に作れます。
また1で作ったSpot FleetのロールとEC2に付与するロール、2で作ったセキュリティグループはここで指定してJSONに含めておきます。

### 5. Lambdaの登録
3で作ったJSONと2で作ったスクリプトを呼び出してSpot Fleet RequestをするJavaScriptコードです。

```JavaScript
const AWS = require('aws-sdk');

AWS.config.update({region: 'ap-northeast-1'});

exports.handler = (event, context, callback) => {

  console.log(event);

  const params = {
    DryRun: false,
    SpotFleetRequestConfig: getSpotFleetRequestConfig(),
  };

  console.log(params);

  new AWS.EC2().requestSpotFleet(params, (err, data) => {
    if (err) {
      console.log(err, 'spot fleet request failed');
      callback(err);
    }
    console.log(data, 'spot fleet request succeed');
    callback(null, data);
  });
};

const getSpotFleetRequestConfig = () => {

  const userdata = `#!/bin/sh
curl https://raw.githubusercontent.com/motojouya/ec2-develop/master/init.sh | bash -s -- ap-northeast-1 user_id user_name password ssh_port dns_host_zone_id domain volume_id
`;
  const initShell = new Buffer(userdata).toString('base64');

  const instanceSetting = {
    "ImageId": "ami-xxxxxxxxxxxxxxxxx",
    "SubnetId": "subnet-xxxxxxxx",
    "KeyName": "xxxxxxxxxx",
    "UserData": initShell,
    "IamInstanceProfile": {
      "Arn": "arn:aws:iam::999999999999:instance-profile/xxxxxxxxxxxxxx"
    },
    "BlockDeviceMappings": [
      {
        "DeviceName": "/dev/sda1",
        "Ebs": {
          "DeleteOnTermination": true,
          "VolumeType": "gp2",
          "VolumeSize": 8,
          "SnapshotId": "snap-xxxxxxxxxxxxxxxxx"
        }
      }
    ],
    "SecurityGroups": [
      {
        "GroupId": "sg-xxxxxxxxxxxxxxxxx"
      }
    ]
  };

  const launchSpecifications = [
    {
      "InstanceType": "xx.xxxxx",
      "SpotPrice": "0.999",
    },
    {
      "InstanceType": "xx.xxxxx",
      "SpotPrice": "0.999",
    },
  ].map(item => Object.assign(Object.assign({}, instanceSetting), item));

  const validFrom = new Date();
  const validUntil = new Date();
  validUntil.setHours(validFrom.getHours() + 3);

  return {
    "IamFleetRole": "arn:aws:iam::999999999999:role/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "AllocationStrategy": "lowestPrice",
    "TargetCapacity": 1,
    "SpotPrice": "0.999",
    "ValidFrom": validFrom.toISOString(),
    "ValidUntil": validUntil.toISOString(),
    "TerminateInstancesWithExpiration": true,
    "LaunchSpecifications": launchSpecifications,
    "Type": "request"
  };
};
```

という感じです。
ハマったのはuserdataに指定するスクリプトは、シェバンがついていないとダメなようです。
それとrootで実行されるので、スクリプト中もsudoは不要になります。
ワンライナーでかきたかったのですが、複数行にしました。

スクリプトをGithubに置いてしまったので、値は全て引数に渡しています。
あとはLambdaを実行して少し待てば、環境が出来上がります。
僕は決まった時間に使うわけではないので、スマホで管理コンソールにログインして、任意のタイミングでLambda実行できればいいので完了です。

### (追記)ローカルからアクセス
`.ssh/config`には以下の設定を入れておきます。
毎回フィンガープリントが変わるので、アクセス時にknown_hostsに記録しないようにすると便利です。こいつを入れておかないと、毎回`ssh-keygen -R`する必要があります。あとはXフォワーディングなどしてクリップボードを使えるようにしておきます。

```config
Host name
  User ubuntu
  IdentityFile ~/.ssh/secret-key.pem
  HostName your.domain.com
  Port 22
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  ForwardX11 yes
```

## まとめ
ほぼ初めてAWSを触ったのですが、IAMやLambda、EC2への理解が深まりました。
新人研修とかでやりたかったなーと思ったり。
でもvimやemacsな人でないと恩恵少ないですかね。

