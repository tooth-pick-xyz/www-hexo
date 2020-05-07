---
title: OnePlus7を買い（譲ってもらい）ました & OnePlus7用FlokoROMを公開しました
date: 2020-05-08 01:00:00
categories:
 - Android
tags:
 - Android
 - FlokoROM
 - OnePlus7
---

だいたいROM関係がおちついたので

## OnePlus7を譲っていただきました。

「新しい端末欲しいなぁ」とは前々から思っていたのですが、元々はxiaomi9 Mi 9Tあたりを狙っていました。  
というのも、欲しい端末の条件として
- フロントに指紋認証がある端末が良い
    - 机において端末をアンロックしたい
        - 背面指紋認証だと端末を持ち上げないといけないのでダメ
        - 次点で側面（電源キー）指紋認証
- そこそこな値段で買える
    - できれば5万くらい
    - ゲームやらないのでSoCはsdm600系でも良い
- カスタムROMが動いている実績がある
    - FlokoROM焼きたいじゃないですか？
    - なのでLineageOSかcrDroidが動く実績がある端末が良い

が条件でした。

## そして時は来た

そう、それは唐突に――――――――――
<!-- more -->

<iframe src="https://mstdn.maud.io/@hota/104031302416695362/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://mstdn.maud.io/embed.js" async="async"></script>

私はこう思ったのでした。  
「このおたく買って1週間でなんで手放したんだ・・・？？」
<iframe src="https://mstdn.maud.io/@hota/104000348608357966/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://mstdn.maud.io/embed.js" async="async"></script>

ということでふと朝通勤途中に気づいてとっさにDMして譲っていただくことになりました。

<iframe src="https://mstdn.maud.io/@tumayouzi/104062765851171536/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://mstdn.maud.io/embed.js" async="async"></script>

## FlokoROMの構築

届いたら即FlokoROMを焼きたかったので、あらかじめROMに必要なソースコードを整えてビルドしておきました。

[FlokoROM-OnePlus](https://github.com/FlokoROM-OnePlus)

因みに元になったデバイスツリー類

基本的にcrDroidに使用している[Hikari-no-Tenshi氏](https://github.com/Hikari-no-Tenshi)のデバイスツリーを参考にしました。

- kernel
    - lineageOSからForkして、dirty hack(主にディスプレイの90Hz動作)を削除した物。
    - そこから[android-linux-stable/msm-4.14](https://github.com/android-linux-stable/msm-4.14)をマージして最新のカーネルまで追従
        - cafコードのマージはOnePlusが独自の改変を入れている箇所が多々あったため、`-s ours`でマージして、cafのコードマージは取り込まずに、OnePlusのコードのままとした
        - Linux kernelで手を入れられている箇所は地道にコンフリクトを解決
        - CrDroidのレポジトリにあるkernelは独自の改変された[neutrinoカーネル](https://github.com/0ctobot/neutrino_kernel_oneplus_sm8150)をforkしたものだったので使わなかった
- device
    - 基本的に[Hikari-no-Tenshi氏](https://github.com/Hikari-no-Tenshi/android_device_oneplus_guacamoleb)のやつのまま
    - [OOSCam(OnePlusオリジナルのカメラ)とギャラリーだけ削除した](https://github.com/FlokoROM-OnePlus/android_device_oneplus_guacamoleb/commit/77ea3138e29612a77e4e4c62bad0afd6f056c272)
        - ライセンス的にアレな物は含まない事がFlokoROMを構築する上で重要なので・・・
        - カメラはgCam使えば良いと思うよ

## はい

<iframe src="https://mstdn.maud.io/@tumayouzi/104062991432367108/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://mstdn.maud.io/embed.js" async="async"></script>

**はい。**
元々ブートローダアンロックされた状態で届いたので、あらかじめビルドしておいた物をOxygenOSはとりあえず初期の確認だけしてとっととFlokoROMを焼きました。

## FlokoROMの公開

1週間くらい使用して特に致命的な問題が無かったのでFlokoROMの管理人ことほた氏に「公式にして良いよ」とお許しをいただいたので公開しました。

- Xda [[ROM][OFFICIAL] FlokoROM v3.0 [10][guacamoleb][OnePlus 7]](https://forum.xda-developers.com/oneplus-7/development/rom-flokorom-v3-0-t4093225)

Pushbulletとか導入しているので、もしOnePlus7でFlokoROMを使う方がいらっしゃったら新しいビルドが公開されると通知が来るので便利です
<a class="pushbullet-subscribe-widget" data-channel="flokorom_oneplus7_release" data-widget="button" data-size="small"></a>
<script type="text/javascript">(function(){var a=document.createElement('script');a.type='text/javascript';a.async=true;a.src='https://widget.pushbullet.com/embed.js';var b=document.getElementsByTagName('script')[0];b.parentNode.insertBefore(a,b);})();</script>

### バグとか

特に大きなバグは無いですが、以下の点がちょっと怪しいです。
- 画面の自動照度調節が明るくなったり暗くなったりするときがある
    - 特に暗いときに発生しやすい。多分センサーの位置かなぁという感じです。暗いところだと自分の液晶の明るさで明るくなったり暗くなったりを繰り返すので・・・
- WiFiで同じSSIDで2.4GHz帯と5GHz帯で使用していると2.4GHz帯を優先して接続してしまう
    - 私も良くわかってないです、なんだろこれ

## とりあえず2週間くらい使ってみて

- 良いところ
    - AMOLEDで綺麗  
        今まで普通の液晶の端末を使っていたので嬉しい。黒が黒く表示される
    - SoCがsm8150でサクサク  
        今まで使用していた端末がSDM630だったので・・・
    - やっぱり指紋認証がフロントにあると嬉しい
- 悪いところ
    - 重い
        ガラス使ってるからでしょうか？なんか重い画面が大きいので余計重く感じます。片手だとそのうち落とす気がする・・・
    - 画面内指紋認証の精度  
        カスタムROM使っちゃってるのでなんとも言えないのですが、時々読み取りに失敗したり、時間かかる時があります。


こんなところかなぁというところです。sm8150とか使っててもスマホゲーとか全くしないので意味ないんですけどねぇなんて。  

OnePlusのカーネルソースコードについては公開してくれているだけでもありがたいのですが、更新方法が多分社内で使っているコードを公開用のフォルダにコピーしてgitで変更纏めてどーんしているようで、どのコードが何を意図して変えられたのかわからないし、1つのとてつもなく大きくなるので💩って感じです。  
公開されているだけでありがたいですが・・・（2度目）

あとはdbrandでスキンシールを頼んで発送されたんですが、コロナの影響があったり、丁度ゴールデンウィークにかかっちゃったりしてまだ手に入れられてないです・・・  
今のところ譲ってもらうときに一緒にもらったケースをつけて我慢しています。
届いたらこれは別の記事でかけたら良いなぁと考えています。

それでは。