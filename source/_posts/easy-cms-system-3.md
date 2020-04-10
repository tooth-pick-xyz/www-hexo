---
title: dockerとgitlabを使ったお手軽CMS環境構築　その3
date: 2017-04-02 00:19:19
categories:
 - サーバ
tags:
 - docker
 - gitlab
 - hexo
 - Cloud9
---

![](/img/2017/docker_hexo_gitlab.png)

寝過ぎて頭痛いです。もの凄く痛い。バファリン飲んだけど治る気しない。  
[その1](https://www.tooth-pick.xyz/2017/03/25/easy-cms-system-1/)と[その2](https://www.tooth-pick.xyz/2017/03/25/easy-cms-system-2/)の続きです。  
今回はCloud9上Hexoの更新環境を構築してから、Git上にpushするまでです。区切りが悪かったので内容が中途半端な感じ否めない。
Gitlab周りの設定は今回はブログシステムを構築するという都合上、滅茶苦茶省いています。分からない場合はググってください。気が向いたら詳細に書くかも知れない。

<!-- more -->

# Cloud9上でHexoをインストールする
Cloud9の環境が整ったところでHexoの環境を整えます。  

## SSH keyの作成
 Hexoの環境を整える前にgitlabで使用するSSHキーを作成します。  
 今回鍵方式はECDSA 521bit鍵を使用しました。
1. ホスト上の`/change/dir/.ssh/`で`ssh-keygen -t ecdsa -b 521`を実行します。  
認証用のパスワード入力を求められますので、無入力でエンターかパスワードを入力してください。  
秘密鍵と公開鍵が生成されますので、公開鍵をgitlabに登録します。  
gitlabの右上にあるアイコンをクリック、Settingを選択し、SSH Keysという項目があるので、こちらに公開鍵の内容を貼り付けて登録します。
2. 同じフォルダにconfigを作成します。  
``` config
Host github.com
HostName gitlab.com
User [yourname]
IdentityFile ~/.ssh/id_ecdsa
Port 22
```
以上でgitのSSHキー登録は完了です。  

## gitlab上にプロジェクトを作成
Hexoのweb公開データをpushするProjectを新規作成します。  
gitlabにログインし、New Projectをクリック、Project名を適当に入力します。  
Visibility LevelはPublicでも問題無いです。どうせ公開する内容ですし。  
最後にCreate projectで完了です。

## Hexo環境の構築
続いてhexoの環境を構築します。ここからはcloud9上での作業です。  
  hexoの初期化をし、試しに1度デプロイを実施してみます。
   1. cloud9上の環境で`hexo init [blog_name]`を実行します。
   2. `cd [blog_name]`、`npm install hexo-deployer-git --save`を実行します。
   3. 一度`hexo server -p 9000`を実行し、http://server-ip:9000　へアクセスし、正しく表示されることを確認します。  サーバはCtrl+Cで停止します。
   4.  config.ymlにdeploy用の設定を記述します  
   ``` _config.yml
   # Deployment
   ## Docs: https://hexo.io/docs/deployment.html
  deploy:
    type: git
    repo: [repository url]
    branch: master
    message: "Updated: {{ now('YYYY-MM-DD HH:mm:ss') }}"
   ```
   `[repository url]`はgitlab上に作成したprojectからアドレスを取ってきます。  
   ![](/img/2017/hexo_git_project_url.png)
   4. `hexo deploy -g`を実行します。問題無ければgithubかgitlabへpushが行われます。web上で確認しましょう。

以上でHexoの環境構築は完了です。  
次回はwebバックエンドの構築、gitlabへpush後、webhook等の設定を行います。  
次が最後かな・・・？
