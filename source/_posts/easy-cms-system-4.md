---
title: dockerとgitlabを使ったお手軽CMS環境構築　その4
date: 2017-04-15 21:30:12
categories:
 - サーバ
tags:
 - docker
 - gitlab
 - hexo
 - Cloud9
---

![](/img/2017/docker_hexo_gitlab.png)

仕事がバタバタしたり、土日に所用があって更新が遅れてしまいました。  
dockerとgitlabを使ったお手軽CMS環境構築シリーズの最後です。  
最後はwebバックエンドの構築～webhookサーバ～バックエンドの自動rebuild環境を構築します。
シリーズ一覧はこちら
1. [その1](https://www.tooth-pick.xyz/2017/03/25/easy-cms-system-1/)
2. [その2](https://www.tooth-pick.xyz/2017/03/25/easy-cms-system-2/)
3. [その3](https://www.tooth-pick.xyz/2017/04/02/easy-cms-system-3/)
4. その4　<-いまここ！！

<!-- more -->

# バックエンドの構築
Hexo公開用のバックエンド用Webサーバのコンテナを作成していきます。  
適当な場所にディレクトリを作成して、Dockerfileを作成します
``` Dockerfile
FROM nginx:latest
MAINTAINER tumayouzi <tumayouzi@gmail.com>

RUN apt-get update && \
    apt-get install -y git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir /data
WORKDIR /data

ADD https://www.random.org/strings/?num=16&len=16&digits=on&upperalpha=on&loweralpha=on&unique=on&format=plain&rnd=newuuid

RUN git clone https://gitlab.com/tumayouzi/hexo_website.git

RUN rm -rf /usr/share/nginx/html && \
    mv /data/hexo_website /data/html && \
    mv /data/html/ /usr/share/nginx/

WORKDIR /usr/share/nginx
```

`ADD https://www.random.org/strings/?num=16&len=16&digits=on&upperalpha=on&loweralpha=on&unique=on&format=plain&rnd=newuuid`では、Dcoker runする場合、前回とフォルダの内容が同じハッシュ値である場合、rebuildする場合git cloneもキャッシュを使用してしまいます。  
ハッシュ値が違うファイルが入った場合、それ以降の処理はキャッシュを使用しなくなるので`WORKDIR ~~`までの処理はキャッシュ、それ以降の処理は毎回キャッシュを使用しない処理となります。  
詳しい処理については以下の記事が参考になります。
[Dockerfileを書く時の注意とかコツとかハックとか | kim hirokuni](http://kimh.github.io/blog/jp/docker/gothas-in-writing-dockerfile-jp/)

# webhookサーバのインストール
今回はwebhookサーバとして[captainhook](https://github.com/bketelsen/captainhook)を使用しました。  
captainhookは以下の順序でインストール出来ます。
 - VPSホスト上で `sudo apt update && sudo apt install -y golang`
 - `go get github.com/bketelsen/captainhook`
続いて、captainhook用のconfigを書いてきます。
 - Dockerfileと同じフォルダに適当なフォルダを作成。フォルダ内に以下のファイルを作成。  
 ファイル名は何でも大丈夫です。ここでは`./hook/web.json`としました。
 ``` web.json
 {
    "scripts": [
        {
            "command": "../scripts/web.sh"
        }
    ],
    "allowedNetworks": [
      //ここにwebhookIP許可リストを記述
    ]
 }
 ```  
 
 webhookの許可IPはgitlabの場合はAzureのIPからですので、そのリストを以下から取得し、リストに入力します。かなりの行です。  
 [Azureで使用されるグローバルIPアドレスの範囲を確認することはできますか？ ： Cloud Steady](http://cloudsteady.jp/faq/2061.html/)
 ufwに最初はIP全て打ち込もうとしたのですが、こっちに書いた方がスマートな気がしたのでこういう形にしています。  
 captainhookがtokenに対応していればIPリストでフィルタすること無いんですが・・・力業に近いです。  

# docker起動用のスクリプトを作成  
web.jsonから見て1階層上に./scripts/web.shを作成します。  
やっていることは入らないdocker imageを消した後に、コンテナをRebuildして、古いコンテナをkillしたあとに新しいコンテナを走らせているだけです。
 ``` web.sh
 #!/bin/bash
 echo "Docker <none> image delete..."
 docker images | awk '/<none/{print $3}' | xargs docker rmi

 echo "Docker Container Rebuild..."
 docker build -t website_www /change/dir/

 echo "Docker Container kill..."
 docker rm -f website

 echo "Docker Start..."
 docker run --restart=always -v /etc/localtime:/etc/localtime:ro -e VIRTUAL_HOST=your.domain -e LETSENCRYPT_HOST=your.domain -e LETSENCRYPT_EMAIL=your@email.address --name website -d -p 80 -p 443 website
 ```

# systemctlにcaptainhookサービスを登録
systemctlに自作のサービスを登録します。  
ファイル名は`/etc/systemd/system/captainhook_web.service`とします。
``` captainhook_web.service  
[Unit]
Description = Restart www docker container from git wehook at captainhook.

[Service]
ExecStart = /usr/local/bin/captainhook -listen-addr=0.0.0.0:9000 -echo -configdir /usr/local/share/docker/web/captainhook
Restart = always
Type = simple

[Install]
WantedBy = multi-user.target
```
自作サービスを登録します。
`sudo systemctl enable captainhook_web.service`の1行だけ。  
この時点ではサービスが起動していないので、`sudo systemctl start captainhook_web.service`で起動させます。

これで全ての環境が構築されました。
動作確認方法ですが、
1. Docker単品でbuild&run動作確認
2. Service起動
3. gitlabからpush通知テスト

の順で行うと良いでしょう。

# 課題事項
実は今回構築した環境ですが、少し問題があります。
webhookの応答ですが、gitlabからpush->paptainhookまでが応答を返すのですが、シェルスクリプトが完走するまで応答が帰りません。  
そのため、gitlab側でタイムアウトしてしまい、pushのたびに2回程度通知が来てしまうため、再pushrebuildが行われてしまいます。今のところ回避方法が思い浮かばないので、とりあえず放置していますが・・・  
あと、webhookの受信方法もどうにかしたいですね・・・今はIPフィルタリングしていますが、していたとしてもAzureのサーバからpushされるとそれだけでサーバに負荷を掛けてしまうことが出来ます。うーん・・・


いかがでしたでしょうか。これでHexoを使用したCMSに使った楽々ブログ更新環境の完成です。  
すこし課題事項もありますが「dockerを使用してこんなCMSシステムも構築出来ますよ」というサンプルにはなるかと思います。  
皆さんもお試しあれ。
