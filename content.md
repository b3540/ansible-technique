# オランダ的
# インフラ
# コントロール
# テクニック
# (ansible)

---

## 自己紹介
## 中村 勲

- 前職:IaaSの設計・開発
- 現在:インフラ全般
- Python, Java, JS(node.js,angular.js), C#(Unity)

```
生まれは函館で帯広に引っ越し、それから小樽に行き、
稚内でロシア人とともに厳しい冬を越し、函館に戻る。
就職を期に埼玉に引っ越して、沼津の山奥に移り、
大自然を満喫した後に、神奈川に移住。
そして、現在は東京で都会人のフリをしています。
```

---

## オランダ的とは

- サッカー
- ロッベンのクロスをスナイデルがダイレクトボレーで合わせるような、
サイドから崩す手法
- 参考:アルゼンチン的 -> 中央から崩す手法

<!-- .slide: data-background="https://s3.amazonaws.com/hakim-static/reveal-js/reveal-parallax-1.jpg" -->


---

## インフラ改善もサイド攻撃

- 中央突破は難しくても、サイドからの崩しで簡単に
  - 中央突破の例: インフラを全部コード化
  - サイド攻撃の例: 本当に困っている箇所だけコード化

---

## 発表の流れ

1. (前半戦)２つの簡単なインフラ改善ケースから、
日々のストレスを高速に解消していく方法を考える
最近は開発者もインフラを何らかの形で触る機会が多いので、

2. (後半戦)改善テクニック、改善例

---

## 利用ツールについて

- chefでもcapistranoでもfabricでも、psshでも、シェルスクリプトでも使い慣れたものでも、実現できれば良い
- 使い慣れたものがなければansibleで

---

# 前半戦

---

## 前提

### インフラ管理はめんどう

- コマンド忘れた
- 社員が増えた
- サーバが重くなった
- 下手にいじると壊れそう
- サーバ3台ぐらいから手でやるのはしんどい

---

## 本日の改善対象タスク
### 心得:時間をかけずに

- ケース1.サーバのユーザ管理
- ケース2.メンテナンス画面

---

## サーバのユーザ管理

### ありがちなケース

- サーバ数台~数十台
  - 手でやるのはつらい
  - インフラをコード化すると楽と聞いているけど、既存の環境をいじるのは怖い
  - useraddのオプションなんだっけ？(たまにしか実行しない)
- ユーザ管理の箇所だけコード化してみる

---

### ansibleを使った例

```
$ ansible-playbook -i hosts user.yml
$
$ cat user.yml
- hosts: appserver
  tasks:
  - user: name="nakamura" group="developer"
  - user: name="suzuki"   group="admin"
$
$ cat hosts
[appserver]
server1
server2
```

---

## シェルスクリプトとの優位性

- 途中でグループを変えたくor消したくなった時
- オプション変えるだけで対応可能

---

## 更なる改善

### コマンドを叩くこと自体がめんどう

- サーバログインはあまりしないので、コマンド忘れる
  - コマンドを叩かないで実行したい
- 解決案:設定変更して、pushしたら自動的にansibleが実行
  - CIツールを利用: Jenkins, Wercker, CircleCI
- image

---

## 更なる更なる改善

### 課題:設定ファイル変更自体がめんどう

目指す姿:hubot useradd nakamuraで全サーバにユーザ追加


1. 設定外出し(変数部分をjson化)
1. 設定ファイルをコントロールするコマンド作成(デモ)
1. chatbot or Jenkinsから実行

```
$ cat userlist.json
{
  "userlist":[
  { "name":"nakamura","uid":"123","group":"developer" },
  { "name":"suzuki","uid":"123","group":"admin",  "groups":"wheel" }
  ]
}
```

- ansibleを挟むことで、実装省力化
- どこまでやるかは、懐事情と相談

---

### ケース2. メンテナンス画面

- nginxの設定とメンテナンスページを書き換える
  - 課題が2つ


1. 設定変更+再読み込みするのがめんどう
2. メンテナンスページの書換がめんどう

```
$ // 以下をサーバ台数分実行
$ vim maintenance-page.html <- メンテページを修正(日付入れたり)
$ vim /etc/nginx/maintenance.conf <- メンテナンスページを表示する設定を入れる
$ nginx -t <- 書き換えたコンフィグファイルに文法ミスはないかのチェック
$ service nginx reload <- 設定反映
```

---

## (課題1)設定変更+再読み込み

1. メンテページ表示の設定変更がめんどう
 - 解決案:nginxの設定で特定の場所に適当なファイルを置くと、メンテナンスページを表示
2. 複数台のサーバにログインしてコマンドを実行がめんどう
 - 解決案:CIツールから叩く

```
// /var/tmp/maintenanceファイルを作成するとメンテモードになる
if (-f /var/tmp/maintenance) {
        set $maintenance 1;
}
```

---

## (課題2)ページ書換がめんどう

- 直接サーバ内のhtmlを書換
- 外部からファイルコピー
- gitからばらまく
  - どの方法もページ書換が必要

- (解決案)可変箇所を変数化して外部から変更

---

## 画面イメージ

<img src="image/jenkins-maintenance.jpg" style="width: 700px; height: 400px;"/>

```
// Jenkinsで以下のシェルを実行
if [ ${confirm} == false ]; then
  exit 1
fi
ansible-playbook -i inventories/dev/hosts --extra-vars "message='${message}'" --tags="on" maintenance.yml

// maintenance.htmlの例
<p>本日のメンテナンス時間は{{ message }}を予定しています</p>
```

### URLが外せるならデモ

---

## 今回の改善方法の仕組み

1. Jenkins
2. chatbot -> Jenkins 
3. command -> Jenkins

### image

---

### 前半戦まとめ

- 小さな改善は時間をかけずスピーディーに
- 小さな改善で効果を示したり、成功体験を作った後に、大きな改善につなげていく

- (もちろん、最初から全て完璧に自動化できれば問題ない)

---

# 後半戦
## 改善テクニック・例

---

ansible（インフラをコード化すると便利なこと）の便利な使い方
処理内容をpartsに分割できる->コピペをしないというプログラミングの考え方
テストもできる
jinja2

```
{parts/fluentd-redshift}
```

アプリサーバとアドミンサーバ
サーバによって必要なミドルウェアの設定は違う

サーバ毎によって設定ファイルを変える必要がある

fluentd

---

## deployもansibleで(会社によってデプロイ方法は千差万別)

ELBの付け外し等簡単に実施可能
(便利な機能が沢山ある

```
- name: Instance De-register
  local_action:
    module: ec2_elb
    instance_id: "{{ ansible_ec2_instance_id }}"
    state: 'absent'
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}
```

---

## インスタンス作成

GUIで聞かれる内容をそのままコード化すれば良い

コード化が気に入れば、Terraformやopsworks、cloudformationを検討

---

# ツール紹介

---

## ec2signal

- サーバの起動停止を開発者でも気軽に
- 開発者に必要な情報だけを見せる+高速絞込

### デモ

---

## iOSアプリ配布ツール

- 社内配布用(Developer Enterprise)ページ自動生成ツール

Jenkinsかコマンドから実行可能
実行すると、静的HTMLを生成する別ツールが動く
コマンドにはコードネームを自動生成する機能付き

### デモ

---

## db schemeの自動更新

---

## ログの可視化
### fluentd+elasticsearch+kibana
mysqlのスロークエリ
アプリケーションエラー
アクセスログ
sensuのイベントログ

---

# おわり

<!-- .slide: data-background="#DDDDDD" -->

---

## 資料について
- プレゼン資料はreveal.js+markdown+github pagesで作成しています。
- [ここ](https://github.com/n10o/ansible-technique)でソースを公開しているので・・・

---
