---
title: VHDL 分周回路の穴
id: 236
categories:
  - ソフトウェア
date: 2016-05-19 20:22:03
tags:
---

ちょっと職場でVHDL取得で学習しててはまった話。
コードが見苦しいのは見逃してください・・・

50MHzを分周して1Hzを求め、8つのLEDを2進数表現でカウントしようとした。

* * *

&nbsp;
> process (CLK) begin
> if (CLK' event and CLK = '1') then
> CNT &lt;= CNT + 1;
> if (CNT = 49999999) then
> led_out &lt;= led_out + '1';
> end if;
> end if;
> end process;
> 
> LED &lt;= led_out;

* * *

&nbsp;

一見問題なさそうに見える。ただ、実際動かすと1秒と200msくらいかかってしまう。
精度・・・？いや、Spartan3でそんなに処理が重くなるとは思えないし・・・
もしかして・・・と、CNTを明示的に0にしてあげると・・・

* * *

&nbsp;
> process (CLK) begin
> if (CLK' event and CLK = '1') then
> CNT &lt;= CNT + 1;
> if (CNT = 49999999) then
> led_out &lt;= led_out + '1';
> <span style="color: #ff0000;">CNT &lt;= 0;　--追加</span>
> end if;
> end if;
> end process;
> 
> LED &lt;= led_out;

* * *

&nbsp;

これで正しく1secでカウントアップできるようになった。
明示的に初期化するの大事。