---
layout: post
title: "Replicatを安易にABENDさせない：REPERRORで実現する細粒度エラーハンドリング"
excerpt: "本記事ではOGGのREPERRORパラメータを詳細解説し、ジレンマから脱する方法を紹介します。DISCARDとTRANSACTIONオプションを使用することで、耐障害性があり監査可能な細粒度エラーハンドリング戦略を構築し、システムの高可用性とデータ一貫性の完璧なバランスを実現できます。"
date: 2025-07-28 18:02:00 +0800
categories: [Oracle GoldenGate, Error Handling]
tags: [ogg, replicat, reperror, error handling, discard, abend, オラクルゴールデンゲート, エラーハンドリング, クロスプラットフォーム移行, エンディアン]
author: Shane
image: /assets/images/posts/ogg-reperror-banner.svg
---

データ同期プロセス中に、OGG Replicatプロセスは様々なデータベースエラー（例えばORA-00001一意性制約違反など）により異常終了（ABEND）することがよくあります。これはデータ同期を中断させ、遅延の蓄積を招くだけでなく、ビジネスの継続性にも影響を与える可能性があります。この時、頭に浮かぶ選択肢は2つしかないでしょう。1つは、ターゲット側で競合するデータを手動で処理してプロセスを再起動すること。もう1つは、パラメータファイルに REPERROR (DEFAULT, IGNORE) の1行を追加して、重要なデータが失われないよう祈ることです。

これは、数えきれないほど多くのOGG運用エンジニアが直面してきたジレンマです。私たちは「システムの利用不能」と「データの信頼性の低下」という2つの間で選択を迫られているように感じます。

しかし、実はそうではありません。今日は、OGGのエラーハンドリングにおける「第三の道」——REPERRORパラメータの使い方をご紹介します。この記事を読めば、Replicatが「軽微な問題」をうまく処理しながら、「重大な問題」にはすぐにアラートを出せる、堅牢で監査可能なエラーハンドリングシステムの構築方法を理解していることでしょう。

### 1. 困難の根源：デフォルト動作の「脆弱性」

まず理解すべきは、Replicatのデフォルト動作が実は「安全第一」の設計であるということです。Replicatは非常に「正直」で、自身が解決できない問題に遭遇すると、まず停止（ABEND）し、問題を報告します。この設計は、予期しないデータの問題が見逃されることを防ぐためのものです。

しかし、複雑な本番環境では、多くのエラーは予期可能であり、時には「良性」です。例えば：

*   **ORA-00001 (一意性制約違反)**: ターゲット側でデータのアーカイブやクリーンアップを行った際に、一部のデータを誤って削除した後、再度挿入された場合に発生する可能性がある。
*   **ORA-01403 (データが見つかりません)**: 双方向同期において、A側であるレコードが削除され、その削除操作がB側に同期されたとする。しかし、同時にB側でも手動で同じレコードが削除された場合、A側からの削除操作がB側に到達したとき、削除対象のデータが見つからずにORA-01403が発生する。

これらのエラーに対して同期全体のリンクを停止することは、コストに見合うものではありません。

**エラーハンドリングフローの概念図**
![Replicat Error Handling Flow]({{ '/assets/images/ogg/ogg-reperror-error-handling-flow.svg' | relative_url }})

### 2. `REPERROR`：万能対応から精密対応へ

`REPERROR`パラメータの強力さは、**特定のエラーコード**に基づいて、**具体的な処理動作**を定義できる点にある。ここでは、その主な使い方を見ていきましょう。

#### `IGNORE` vs `DISCARD`：履歴を残すか否か？

これは`REPERROR`の最も基本的で重要な2つのオプションだ。

*   **`IGNORE`**: 最も「乱暴な」処理方法だ。Replicatに「このエラーに遭遇したら、見なかったことにして、この操作をスキップし、次に進め」と指示する。
    *   **利点**: シンプルで直接的。
    *   **致命的な欠点**: **データが静かに失われます**。どの操作が無視されたのかを追跡するログや記録が一切残りません。これはデータ整合性の悪夢の始まりであり、本番環境での IGNORE の使用は強く避けるべきです。

*   **`DISCARD`**: これは私が推奨する「ゴールドスタンダード」です。Replicat に対して、「このエラーが発生したら、その操作もスキップしてよいが、エラーの詳細、SQL 文、トランザクション情報など、操作の完全な情報を Discard ファイルにそのまま記録するように」と指示するものです。
    *   **利点**: プロセスの継続的な稼働を確保しつつ、**完全な監査トラッキング**が可能になります。定期的に Discard ファイルを確認し、破棄された操作を分析・手動修復することで、最終的なデータ整合性を確保できます。
      
#### Discardファイルの設定

`DISCARD`を使用するには、まずReplicatのパラメータファイルで破棄ファイルの場所とルールを定義する必要があります。

```ini
-- Replicatパラメータファイル (rep_main.prm)
REPLICAT rep_main
USERIDALIAS ogg_tgt_alias DOMAIN OracleGoldenGate

-- Discardファイルを定義。ファイル名は自動的にプロセス名とシーケンス番号が付加される
-- PURGEONDAYS 7 は7日分のDiscardファイルを保持し、その後自動的にクリーンアップ
-- MEGABYTES 100 は各Discardファイルの最大サイズを100MBに設定
DISCARDFILE ./dirrpt/rep_main.dsc, APPEND, PURGEONDAYS 7, MEGABYTES 100

-- 私たちのエラーハンドリング戦略
-- ORA-00001エラーに遭遇した場合、DISCARD操作を実行する
REPERROR (ORA-00001, DISCARD)

-- MAP文
MAP sales.*, TARGET sales.*;
```
設定が完了すると、Replicatが`ORA-00001`エラーに遭遇してもABENDせず、代わりに`./dirrpt/`ディレクトリに以下のような内容の`.dsc`ファイルが生成されます。
```
Oracle GoldenGate Delivery for Oracle process started, group REP_MAIN...
...
2023-10-27 16:30:15  WARNING OGG-01154  Oracle GoldenGate Delivery for Oracle, rep_main.prm:  SQL error 1 mapping SALES.ORDERS to SALES.ORDERS.
OCI Error ORA-00001: unique constraint (SALES.PK_ORDERS) violated
...
-- ここにエラーを引き起こした完全なSQL文がリストされる
INSERT INTO "SALES"."ORDERS" ("ORDER_ID", "CUSTOMER_ID", "ORDER_DATE", "AMOUNT")
VALUES (1001, 205, TO_DATE('2023-10-27 16:30:00', 'YYYY-MM-DD HH24:MI:SS'), 599.99);
```
このファイルがあれば、問題を追跡し修復するためのすべての証拠が手に入れます。

### 3. スマートな「階層的エラー戦略」の構築

`REPERROR`の真の威力は、多層的で細粒度エラーハンドリング戦略を定義できる点にあります。

*   **「軽微な問題」（予期可能な良性エラー）に対しては、記録して処理を継続。**
*   **「中程度の問題」（データ不整合の可能性を示唆）に対しては、現在のトランザクションを中止するが、プロセスは終了させない。**
*   **「重大な問題」（未知または深刻なエラー）に対しては、直ちにプロセスをABENDさせ、注意を引く。**

`REPERROR`ルールを重ねることで、これを実現できます。Replicatは上から順にルールを照合し、一致するものが見つかれば対応するアクションを実行します。

**本番環境レベルの`REPERROR`設定例：**
```ini
-- Replicatパラメータファイル (rep_prod.prm)
REPLICAT rep_prod
USERIDALIAS ogg_tgt_alias DOMAIN OracleGoldenGate
DISCARDFILE ./dirrpt/rep_prod.dsc, APPEND, PURGEONDAYS 14, MEGABYTES 200

-- === 細粒度エラーハンドリング戦略 ===

-- 1. 一意性キー制約違反（ORA-00001）に対しては、Discardファイルに記録して継続
REPERROR (ORA-00001, DISCARD)

-- 2. 「更新/削除対象行が見つからない」（ORA-01403）は、データ不整合の可能性を示唆する。
--    ここではTRANSACTIONレベルの処理を選択：現在のトランザクションの全操作を破棄し、
--    トランザクション全体をDiscardファイルに書き出し、次のトランザクションから処理を開始する。
--    これはABENDよりもエレガントだ。サービス全体を中断させないからだ。
REPERROR (ORA-01403, TRANSACTION)

-- 3. 明示的に指定されていない他のすべてのデータベースエラー（DEFAULT）に対しては、
--    プロセスを直ちにABENDさせ、DBAの注意を引く。未知の問題が隠蔽されるのを防ぐ。
--    これが最後の安全策だ。
REPERROR (DEFAULT, ABEND)

MAP fin.*, TARGET fin.*;
```
この設定は、階層的な処理思想を体現している。
*   **`DISCARD`**: 操作レベル（Operation-level）の処理。単一のDMLにのみ影響。
*   **`TRANSACTION`**: トランザクションレベル（Transaction-level）の処理。現在のトランザクションの全DMLに影響。
*   **`ABEND`**: プロセスレベル（Process-level）の处理。サービス全体を中断。

### まとめ

`REPERROR`を巧みに活用することで、私たちはReplicatのエラーとの関わり方を根本的に変えることができます。

| 処理方法 | 利点 | 欠点 | 適用シナリオ |
| :--- | :--- | :--- | :--- |
| **ABEND** | 安全、問題が見過ごされない | システム可用性が低い、頻繁な手動介入が必要 | 致命的、未知のエラー |
| **`IGNORE`** | プロセスが中断しない | **データブラックホール、監査不可** | **本番での使用は強く非推奨** |
| **`DISCARD`** | プロセスが中断せず、**完全な監査が可能** | DISCARDファイルの定期的な確認が必要 | 予期可能で許容範囲内の単発エラー |
| **`TRANSACTION`**| 問題のあるトランザクションを隔離、プロセスは中断しない | TRANSACTION 全体のデータが破棄される | 論理的な問題が疑われるエラー |

次回、Replicat がよくあるエラーで停止したとき、`SKIPTRANSACTION`を使ったり、安易に`IGNORE`を指定するのはやめましょう。エラーの性質を見極めたうえで、適切な `REPERROR`ルールを検討してください。
