---
layout: post
title: "複数のパーティション表に対するPMOPとDMLの同時実行中の高いライブラリ・キャッシュ・ロック待機"
excerpt: "同じテーブルに対してパーティション保守操作（PMOP）と同時にDMLが実行される場合、高い「library cache lock」と「library cache load lock」待機が観察されました。"
date: 2025-07-22 17:00:00 +0800
categories: [Oracle, Database]
tags: [Library Cache Lock, library cache load lock, High Waits, oracle]
image: /assets/images/posts/High-Library-Cache-Lock-Waits.jpg
---

## 問題  

同じ表に対してパーティション保守操作（PMOP）と同時にDMLが実行される場合、高い「library cache lock」と「library cache load lock」待機が観察されました。このケースでは、同じ表に対して約1秒ごとに新しいパーティションが追加され別のパーティションが削除されながら、同時にDMLも実行されていました。同様のワークロードが他のパーティション表に対しても同時に行われていました。  
AWRレポートは以下のような状態を示していました：  

トップ10フォアグラウンド・イベント（合計待機時間順）  

|イベント                            |待機回数     |合計待機時間(秒)      |平均待機(ms)      |DB時間%      |待機クラス|
|:----|----:|----:|----:|----:|----:|
|library cache lock              |64,604                   |144.2K            |2232.7            |75.6     |コンカレンシー|
|library cache load lock         |14,231                    |12.3K            |860.92             |6.4     |コンカレンシー|
|DB CPU                          |                         |8156.9               |4.3             |                   |
|db file sequential read     |16,734,388                   |5672.9              |0.34             |3.0        |ユーザーI/O|
|row cache lock              |14,571,514                   |4442                |0.30             |2.3     |コンカレンシー|
|cursor: pin S wait on X          |2,278                   |2167.6            |951.53             |1.1     |コンカレンシー|
|gc cr disk read              |8,135,284                   |1524.6              |0.19              |.8         |クラスタ|
|enq: IV - contention           |423,859                      |271              |0.64              |.1           |その他|
|SQL*Net message from dblink     |38,133                    |202.7              |5.31              |.1         |ネットワーク|
|Disk file operations I/O    |14,810,701                    |140.5              |0.01              |.1        |ユーザーI/O|

## 解決策  
Oracleは、パーティション表に対するPMOPとDMLの混在を推奨しません。これはPMOPがカーソル無効化を引き起こすためです。同じパーティション表でPMOPと重いDMLを混在させる影響を軽減するには、以下の対策を実施してください。。  

1. AWRのトップ10イベントリストに「cursor: pin S wait on X」待機も表示される場合、以下のバグ修正パッチの適用を検討してください：  
パッチ 14380605 : HIGH LIBRARY CACHE LOCK,CURSOR: PIN S WAIT ON X AND LIBRARY CACHE: MUTEX X  
パッチ 23003919 : LIBRARY CACHE LOCK & CURSOR: PIN S WAIT ON X DUE TO INVALIDATION PARALLEL QUERY  

2. パーティション作成直後に挿入を行う場合、以下のクエリの影響を軽減するため`deferred_segment_creation = false`を設定してください：  
```sql
select pctfree_stg, pctused_stg, size_stg,initial_stg, next_stg,
minext_stg,
  maxext_stg, maxsiz_stg, lobret_stg,mintim_stg, pctinc_stg, initra_stg,
  maxtra_stg, optimal_stg, maxins_stg,frlins_stg, flags_stg, bfp_stg,
enc_stg,
  cmpflag_stg, cmplvl_stg,imcflag_stg
from
 deferred_stg$ where obj# =:1
```
理由：  
セグメント作成直後に挿入する場合、遅延セグメント作成はメリットがなくコストのみが発生します。Xライブラリ・キャッシュ・ロックが2回取得されます（パーティション追加DDL時と挿入時のセグメント作成時）。遅延セグメント作成を無効にすると、DDL時の1回のXロック取得でパーティション追加とセグメント作成の両方が行われます。  

ライブラリ・キャッシュ・ロード時のエクステント・スキャンを削減するため：  
(a) 全パーティションの統計情報を収集する  
理由：  
統計情報のない多数のパーティションが存在すると、パーティション内のブロック数を推定するためのエクステント・スキャンが発生し、ライブラリ・キャッシュ・ロードが低速化します。すべてのパーティションで統計情報を収集することで（他の理由でも推奨されます）、これらのエクステント・スキャンを排除できます。これによりindpart$およびtabpart$クエリのCPU時間が改善されます。  

同じ表で複数のパーティションを削除/追加する場合、12cの新機能である「単一DDL文での複数パーティション操作」を検討してください。これにより、パーティションごとではなくDDL文1回あたり1回のみ行キャッシュ・オブジェクトの無効化が発生します。  

```
関連ドキュメント 1530075.1 - 間隔パーティション表使用時の高い「library cache lock」待機（11.2.0.4未満向け）も参照してください。
```
