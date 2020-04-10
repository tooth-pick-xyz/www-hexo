---
title: prometheusとgrafanaで良い感じにdockerコンテナとホストを監視する
date: 2017-05-14 21:39:27
toc: true
categories:
 - サーバ
tags:
 - docker
 - prometheus
 - grafana
 - cadvisor
---

夏コミの原稿が行き詰まっていますが僕は元気です。  

Dockerの環境も良い感じになったし、そろそろまともにちゃんとサーバとDockerホスト監視しようぜ！！ってことでprometheusとgrafanaで良い感じに可視化します。

![](/img/2017/grafana.jpg)

<!-- more -->

# 使うコンテナ
 - [prom/prometheus](https://hub.docker.com/r/prom/prometheus/)  
   prometheusはみんなだいすきSoundCloudが中心となってGo言語で開発されている監視ツールです。  
   有名なZabbixなどはpush型の監視ですが、prometheusはpull型のシステムです。設定でサービスディスカバリを記述してあげると、あとは良い感じにデータを拾って来てくれます。
 - [prom/node-exporter](https://hub.docker.com/r/prom/node-exporter/)  
   node-exporterはホストの状態を取得するexporterです。
 - [google/cadvisor](https://github.com/google/cadvisor)  
   cadvisorがgoogleが開発している、dokerコンテナの情報を取得するexporterです。  
   これ単品でもdokcerコンテナを監視することは出来ますが、今回はデータをprometheusで一括監視するために使用します。
 - [grafana/grafana](https://github.com/grafana/grafana-docker)  
   prometheusからの情報を簡単で良い感じに視覚化してくれるツールです。

# 環境構築
dokcer-composeで一気に環境構築します。  
想定している環境としては、フロントエンドが[jwilder/nginx-proxy:alpine
](https://github.com/jwilder/nginx-proxy)と[JrCs/docker-letsencrypt-nginx-proxy-companion
](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)で構築されている環境を前提としています。

今回、grafanaのデータを監視するサブドメインはprometheus.example.com、prometheusを直接弄れるドメインはprometheus.prometheus.example.comとしました。  
またgrafana自体にはログイン機能が付いていますが、prometheus自体にはついていないためnginx-proxyでBASIC認証をかけます。  
cAdvisorについては、今回外部から不可視の状態としました。  

## コンテナの立ち上げ
dokcer-composeを使用して一気にコンテナを立ち上げます。  
適当に./prometheus
dokcer-compose.ymlの記述は以下の通りです。(めっちゃ長いです)  
``` dokcer-compose.yml
prometheus:
   image: prom/prometheus
   container_name: prometheus
   mem_limit: 256mb
   restart: always
   volumes:
     - ./prometheus.yml:/etc/prometheus/prometheus.yml
   expose:
     - 9090
   links:
     - cadvisor
     - node_exporter
   environment:
     VIRTUAL_HOST: prometheus.prometheus.example.com
     LETSENCRYPT_HOST: prometheus.prometheus.example.com
     LETSENCRYPT_EMAIL: hoge@example.com
cadvisor:
   image: google/cadvisor:latest
   container_name: cadvisor
   mem_limit: 128mb
   restart: always
   volumes:
     - /:/rootfs:ro
     - /var/run:/var/run:rw
     - /sys:/sys:ro
     - /var/lib/docker/:/var/lib/docker:ro
   expose:
     - 8080
node_exporter:
   image: prom/node-exporter
   container_name: node_exporter
   mem_limit: 128mb
   restart: always
   volumes:
     - /proc:/host/proc:ro
     - /sys:/host/sys:ro
     - /:/rootfs:ro
   expose:
     - 9100
grafana:
   image: grafana/grafana
   container_name: grafana
   mem_limit: 256mb
   restart: always
   expose:
     - 3000
   links:
     - prometheus
     - cadvisor
     - node_exporter
   environment:
     GF_SECURITY_ADMIN_USER: yourname
     GF_SECURITY_ADMIN_PASSWORD: password
     GF_USERS_ALLOW_SIGN_UP: "false"
     GF_USERS_ALLOW_ORG_CREATE: "false"
     VIRTUAL_HOST: prometheus.example.com
     LETSENCRYPT_HOST: prometheus.example.com
     LETSENCRYPT_EMAIL: hoge@example.com
```
CAdviserですが、要求するvolumes設定が危険な内容です、ドキュメントの指示に従い安易にrwを与えずに、必要最低限のroを与えます。  
またVPSのメモリが若干厳しいため各コンテナにメモリ制限を与えています。

次にprometheusでdiscoveryするデータの設定をします。  
prometheus自体に対しては、ループバックアドレスで良いですが、他のデータに関しては別のこんてなとなるため、docker-compose.ymlないで指定しているlinksで指定します。そうすることでdockerが自動でアドレスを解決します。
```prometheus.yml
global:
  scrape_interval: 15s
  external_labels:
    monitor: "monitor"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["127.0.0.1:9090"]
  - job_name: "node"
    static_configs:
      - targets: ["node_exporter:9100"]
        labels:
          group: "docker-host"
  - job_name: "docker"
    static_configs:
      - targets: ["cadvisor:8080"]
        labels:
          group: "docker-container"
```

最後にprometheusのBASIC認証設定です。  
jwilder/nginx-proxyの説明にあるように、設定します。私の設定の場合、jwilder/nginx-proxyを../rproxy/htpasswdに保管するようにしています。  
起動時にprometheusのコンテナ起動時に渡すenvironmentのVIRTUAL_HOST名を自動で読み込むように出来ています。  
ですので、prometheus.prometheus.example.comというドメインを指定した場合、同様の名前の設定ファイルを指定のディレクトリに保存します。  
以下のコマンドでhtpasswdを作成します。   
`htpasswd -c -b /etc/httpd/conf/.htpasswd user password`  


# コンテナの起動
prometheusのdocker-compose.ymlを保管したディレクトリ内で以下のコマンドで実行します。  
`sudo docker-compose up`  

## prometheusの確認
ログがずらーっと流れてきますので、エラーがなさそうであればprometheus.prometheus.example.comにアクセスし、メニューのStatus > Targetsからちゃんと指定したデータが監視出来てるか確認します。  
![](/img/2017/targets.png)  

## grafnaの設定
prometheus.example.comへアクセスしdocker-compose.yml内で設定した`GF_SECURITY_ADMIN_USER`と`GF_SECURITY_ADMIN_PASSWORD`でログインします。  
grafanaのDataSourceを以下の画像のように入力すればデータが取得出来るはずです。  
![grafanaのインポート設定](/img/2017/grafana_data.png) 

## grafanaのDashboadのImport
grafanaのサイトには、良い感じにグラフを出力してくれるDashboardの設定が公開されています。  
New dashboard > Importに395と22をImportすることで、dockerとNodeを良い感じに監視しやすいグラフが設定されます。
![nodeのダッシュボード](/img/2017/node.png)
![dockerのダッシュボード](/img/2017/docker.png)

動作が一通り確認出来たところで、コンソールからtrl+Cで一度dockerコンテナを止め、バックグランドで操作させるために`docker-compose -up -d`でバックグラウンドで動作するようにします。  
これで全ての設定が完了です。お疲れさまでした。


いかがでしたでしょうか。dokcerで監視環境を構築することで、簡単にprometheusとgrafanaで良い感じにdockerコンテナとホストを監視する環境が整いました。  
dockerコンテナを監視するのはprometheusが流行りつつあるものの、未だにzabbixがメインじゃないかと思います。  
私の昔のサーバではmuninで構築していましたが、今後prometheusとgrafanaが主流になると面白いですね。  