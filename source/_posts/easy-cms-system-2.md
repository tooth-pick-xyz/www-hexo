---
title: dockerとgitlabを使ったお手軽CMS環境構築　その2
date: 2017-03-25 15:00:17
categories:
 - サーバ
tags:
 - docker
 - gitlab
 - hexo
 - Cloud9
---

![](/img/2017/docker_hexo_gitlab.png)

前回の記事[dockerとgitlabを使ったお手軽CMS環境構築　その1](https://www.tooth-pick.xyz/2017/03/25/easy-cms-system-1/)の続きの記事です。  
前回は構築する概要等を説明しましたが、その2からは実際のシステムの更新に入っていきたいと思います。


# webフロントエンドの作成
今回はwebフロントエンド用のコンテナとして、2つのコンテナを使用しました。
- [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy)
- [JrCs/docker-letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)  

この2つのコンテナを使用することでバックエンドで構築したwebサーバをdocker.sock経由で感知し、nginx-proxyがリバースプロキシ設定を自動的に設定します。   docker-letsencrypt-nginx-proxy-companionはLet's EncryptでのSSL証明書を自動発行、SSL通信化を自動で行えます。  
立ち上げておけばいいだけなのでとっても便利ですね👏  

<!-- more -->

### docker-compose.ymlを作成
適当な場所にディレクトリを作成、ディレクトリ内にdocker-compose.ymlを作成し、以下の様に記述します。

```docker-compose.yml
nginx-proxy:
  image: jwilder/nginx-proxy
  privileged: true
  restart: always
  ports:
    - 80:80
    - 443:443
  volumes:
    - ./docker-compose.d/certs:/etc/nginx/certs:ro
    - ./docker-compose.d/htpasswd:/etc/nginx/htpasswd
    - /etc/nginx/vhost.d
    - /usr/share/nginx/html
    - /var/run/docker.sock:/tmp/docker.sock:ro

letsencrypt-nginx-proxy-companion:
  image: jrcs/letsencrypt-nginx-proxy-companion
  privileged: true
  restart: always
  volumes:
    - ./docker-compose.d/certs:/etc/nginx/certs:rw
    - /var/run/docker.sock:/var/run/docker.sock:ro
  volumes_from:
    - nginx-proxy
```

保存後にdocker-compose.ymlを保存したディレクトリ内で `docker-compose run`を実行し、実際にエラーが起きずに実行出来たか確認します。  
問題無い場合は`docker-compose run -d`でバックグラウンド動作する様にしましょう。  
docker-compose.yml内で`restart=always`を指定しているので、サーバをホスト再起動後も自動でコンテナが起動されます。

# サーバ移行先のブログ更新環境(Cloud9)のコンテナ作成
今回はCMSをwordpressをHexoに移行したため、その更新環境を整える必要があります。  
Hexoの更新環境をローカル上に構築することも可能ですが __外出先でも更新する__ ことを考え、Cloud9上に更新します。
## Cloud9とは  
[Cloud9](https://c9.io/)は無料で2GBのディスク、512MBのメモリ、1Cpuを使用出来ます。  
本来はCloud9が提供しているサービスを使用できますが、今回は大きい画像を使用することを考え自前でdockerコンテナを建てることにしました。  
画像をあまり利用しない記事を書く場合は、わざわざ自宅サーバ内にコンテナを作成せずにCloud9が提供しているサービスをそのまま使用してしまっても十分です。 
 
### docker-compose.ymlを作成
Cloud9も公式でdokcerコンテナが提供されていますが、今回はこちらを参考に構築しました。  
[Cloud9をIDCFクラウドで使う - Part1: Docker Composeでインストール #idcf](http://qiita.com/masato/items/9f1ee60b895a03fffc46)  
　1. まず適当にディレクトリを作成します。`cloud9`とかでディレクトリ名を作ると良いでしょう  
　2. Dockerfileを作成します

``` Dockerfile
FROM node:7
MAINTAINER changeme <changeme@exsample.com>

RUN apt-get update && apt-get install -y vim

RUN git clone https://github.com/c9/core.git /cloud9 && \
    cd /cloud9 && ./scripts/install-sdk.sh

RUN npm install hexo -g
RUN npm install hexo-cli -g

WORKDIR /workspace
```

　3. docker-compose.ymlを作成します   

``` dokcer-compose.yml
app:
  build: .
  restart: always
  ports:
    - 80
    - 9000:9000
  volumes:
    - ./workspace:/workspace
    - /change/dir/ssh/id_ecdsa:/root/.ssh/id_ecdsa:ro
    - /change/dir/docker/.ssh/config:/root/.ssh/config:ro
    - ~/.gitconfig:/root/.gitconfig
    - /etc/localtime:/etc/localtime:ro
  environment:
    VIRTUAL_HOST: cloud9.your.domain
    HTTPS_METHOD: noredirect
    LETSENCRYPT_HOST: cloud9.your.domain
    LETSENCRYPT_EMAIL: changeme@test.com
  command: node /cloud9/server.js --port 80 -w /workspace -l 0.0.0.0 --auth user:password
```

- .yml内の`VIRTUAL_HOST:`、`LETSENCRYPT_HOST:`、`LETSENCRYPT_EMAIL:`、`--auth user:password`は適時変更してください。  
- `HTTPS_METHOD: noredirect`、`LETSENCRYPT_HOST:`、`LETSENCRYPT_EMAIL:`は
  [JrCs/docker-letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)  の設定です。 github上のreadmeを参考に設定してください。 

　4. `docker-compose up`でコンテナを起動します。https://cloud9.your.domain にアクセスし、上手く起動している場合は`docker-compose up -d`を指定して、バックグラウンドで実行する様にしましょう。


これで、docker上にwebフロントエンドとCloud9のコンテナを作成できました。  
次回はCloud9上でHexoの環境を整え、サイトを公開出来る状態までを公開したいと考えています。  
平日に入って仕舞うので、ちょっと続けての更新は無理そうですが・・・