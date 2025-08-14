---
layout: post
title: "OGG `SHOWSYNTAX` パラメータの詳細解説：SQL生成から問題解決への活用"
excerpt: "`SHOWSYNTAX` はOGGで最も強力な対話型デバッグツールの一つです。本記事では、その原理、ReportやTRACEとの違いを深く掘り下げ、実践的な演習を通じて、データ変換エラーやパフォーマンスのボトルネックを正確に特定し、Replicatの問題を解決する方法を解説します。"
date: 2025-08-14 09:30:00 +0800
categories: [Oracle GoldenGate, Troubleshooting]
tags: [ogg, showsyntax, replicat, sql debugging, performance tuning, data synchronization, interactive debug, オラクルゴールデンゲート, レプリカ, SQLデバッグ, パフォーマンスチューニング, データ同期, インタラクティブデバッグ, データベース, データ分析, データエンジニアリング, データ管理]
author: Shane
image: /assets/images/posts/oracle_goldengate_showsyntax_flowchart.svg
---

私が Oracle GoldenGate を用いたデータ同期プロジェクトに携わってきた中で、繰り返し直面した課題のひとつは、Replicat プロセスが遅延したり ABEND した際、レポートファイルに明確なエラーメッセージが見当たらないという状況です。その結果、データが密かに欠落したり不整合が発生し、業務ユーザーから苦情が寄せられ、DBA やデータエンジニアはログを途方に暮れてしまいます。
このような問題に直面したとき、Replicatの内部を直視できるツールが必要です。そのツールが `SHOWSYNTAX` です。これは単なるログパラメータではなく、強力な対話型SQLデバッガです。本記事を読み終えたときには、`SHOWSYNTAX`を使いこなし、そうした厄介な Replicat の問題を正確に特定し、解決できるようになっているでしょう。

## 1. SHOWSYNTAX：対話型SQLインスペクタ

`SHOWSYNTAX` の価値を理解するためには、まず他のデバッグツールとの違いを明確にする必要があります。

*   **レポートファイルとの比較**：レポートファイル（`.rpt`）は、Replicatプロセスの開始・停止、遅延状況、統計情報、最終的なエラー結果などを記録します。これは何が起きたかは教えてくれますが、なぜ起きたのかまではほとんど教えてくれません。適用の失敗を報告することはあっても、その失敗を引き起こした、変換やマッピング後の原因であるSQL文は表示しません。

*   **TRACE / TRACE2 との比較**：`TRACE` パラメータは非常に強力で、OGG内部の関数呼び出し、メモリ割り当て、詳細な処理ステップを記録できます。しかしその反面、膨大で一般ユーザーには解読困難なログを生成します。`SHOWSYNTAX` が関心を持つのはただ一つ、**Replicatが最終的にターゲットデータベースにコミットしようとしているSQL文が一体どのようなものか**ということです。

`SHOWSYNTAX` の中核的な動作原理は、Replicatの自動化されたプロセスを中断させ、各DML文（INSERT, UPDATE, DELETE）がターゲットデータベースに適用される前に、まずその全文をコマンドライン画面に表示し、実行を一時停止してあなたの指示を待つ、というものです。これにより、データ変換、列マッピング、関数計算の最終結果が期待通りか、一つ一つ確認する貴重な機会が得られます。

### `SHOWSYNTAX` のワークフロー

より直感的に理解するために、`SHOWSYNTAX` のワークフローを視覚化してみましょう。

![GoldenGate SHOWSYNTAX Workflow]({{ '/assets/images/ogg/ogg-showsyntax-workflow.svg' | relative_url }})

この図は、`SHOWSYNTAX` がReplicatのSQL生成とターゲットデータベースへの適用との間の重要なポイントでどのように割り込み、我々に強力なデバッグ能力を与えてくれるかを明確に示しています。

## 2. `SHOWSYNTAX` の設定と使用方法

理論はここまでにして、早速実践に移りましょう。OGG 19c環境をベースに、`SHOWSYNTAX` を用いた完全なデバッグプロセスを実演します。

### ステップ1：Replicatパラメータファイルの変更

まず、Replicatのパラメータファイル（`.prm`）に `SHOWSYNTAX` パラメータを追加する必要があります。

```sql
-- repabc.prm
REPLICAT repabc
-- ターゲットデータベース接続情報
USERID ogguser@targetdb, PASSWORD your_password
-- エラー発生時はデフォルトで異常終了
REPERROR DEFAULT, ABEND

-- SHOWSYNTAXを有効化し、LOBデータを最大1MBまで表示
SHOWSYNTAX INCLUDELOB 1MB

-- discardファイルの定義
DISCARDFILE ./dirrpt/repabc.dsc, APPEND
-- ソーステーブルからターゲットテーブルへのマッピング定義
MAP tuser.t1, TARGET tuser.t2;
```

**主要パラメータの解説**:
*   `SHOWSYNTAX`: 中核となるパラメータで、対話型デバッグモードを起動します。
*   `[APPLY | NOAPPLY]`: このサブオプションは非常に重要です。`APPLY`（デフォルト値）は、画面にSQLを表示した後、続行を選択するとそのSQLがターゲットデータベースに適用されることを意味します。一方 `NOAPPLY` は、SQLを表示するだけで、データベースには一切適用せず、破棄ファイルにも書き込みません。これは純粋な「読み取り専用」の調査モードであり、本番環境でのデバッグ時に強く推奨されます。
*   `INCLUDELOB [max_bytes | ALL]`: デフォルトでは、`SHOWSYNTAX` はLOB、XML、UDT型列の完全な内容を表示せず、`<LOB data>` のようなプレースホルダを表示します。`INCLUDELOB` を使用すると、これらのデータを表示できます。最大バイト数（例：`1M`, `10K`）を指定するか、`ALL` を使用して全内容を表示できます。

### ステップ2：コマンドラインからReplicatを起動

これはよくある「落とし穴」です：**`SHOWSYNTAX` を設定したReplicatプロセスはGGSCIからは起動できません**。GGSCIで `START REPLICAT repabc` を試みると、即座に `OGG-01991` エラーが発生します。

正しい方法は、オペレーティングシステムのコマンドライン（シェル）を開き、OGGのインストールディレクトリに移動し、直接Replicatプログラムを実行することです。

```bash
$ ./replicat paramfile dirprm/repabc.prm
```

### ステップ3：対話的なSQLの調査

ソース側でデータ変更が発生すると、起動したReplicatプロセスは画面に適用対象の最初のSQL文を表示し、一時停止します。

```
2024-12-02 17:58:44 INFO OGG-06510 Using the following key columns for target table TUSER.T2: ID.

INSERT INTO "TUSER"."T2" ("ID", "NAME", "DESCRIPTION", "CLOB_DATA", "CATE") VALUES
('1','zhang', 'YkilH...', 'NATPA0...', TO_TIMESTAMP('2024-12-02 17:00:40...'))
Statement length: 4,203

(S)top display, (K)eep displaying (default):
```
この時点で、2つの選択肢があります：
*   **Enterキーを押す（デフォルトはK）**: "Keep Displaying"。現在のSQL文を実行し、次のSQL文の表示を続けます。
*   **Sを入力してEnterキーを押す**: "Stop Display"。現在のSQL文を実行した後、対話モードを終了し、Replicatが自動的なバッチ処理を再開するようにします。これにより、SQLは逐一表示されなくなります。

この方法で、問題のあるSQLを見つけるまで、Replicatの動作を一つずつ調査できます。

## 3. 典型的な利用シーンとトラブルシューティングのアプローチ

`SHOWSYNTAX` は以下のシナリオで大きな力を発揮します：

### シーン1：複雑なデータ変換と `COLMAP` ロジックのデバッグ

**課題**: `MAP` 文で複雑な `@IF`, `@CASE`, `@STRCAT` などの列変換関数を使用した場合、ターゲット側のデータが期待通りでないと、ソースデータの問題なのか、関数ロジックの誤りなのかを判断するのが困難です。

**アプローチ**:
1. Replicatパラメータファイルに `SHOWSYNTAX NOAPPLY` を追加します。`NOAPPLY` オプションにより、デバッグ作業がターゲットデータを汚染しないことを保証できます。
2. コマンドラインからReplicatを起動します。
3. 複雑な変換を含む `INSERT` や `UPDATE` 文が画面に表示されたら、`VALUES (...)` 部分や `SET` 句の値を注意深く確認します。
4. **例**：`COLMAP (USEDEFAULTS, T_STATUS = @CASE(S_CODE, 'A', 'Active', 'I', 'Inactive', 'Unknown'))` と設定した場合、`SHOWSYNTAX` は直接 `UPDATE "TARGET_TABLE" SET "T_STATUS" = 'Active' WHERE ...` と表示します。これにより、`S_CODE` の値が正しく変換されたかどうかを一目で確認できます。

### シーン2：Replicatのパフォーマンス問題の診断

**課題**: ReplicatプロセスのLAGが増え続けているが、ターゲットデータベースにはロックや明確なパフォーマンスのボトルネックがなく、レポートファイルにもエラーがありません。

**アプローチ**:

1.  **`WHERE` 句の確認**：私がある案件で遭遇したケースでは、クライアントのReplicatが非常に遅くなっていました。`SHOWSYNTAX` を使用したところ、ソース側の主キーがないテーブルに対する `UPDATE` 操作により、Replicatが生成する `UPDATE` 文の `WHERE` 句にそのテーブルの全列が含まれており、ターゲットデータベースで毎回フルテーブルスキャンが発生していることが判明しました。レポートファイルは決してこれを教えてくれませんが、`SHOWSYNTAX` を使えば見逃すことはありません。
2.  **バッチ処理の中断**：`SHOWSYNTAX` は `BATCHSQL` モードを一時停止させます。これ自体がパフォーマンスを低下させる要因ではありますが、単一の文の構造を観察することで、バッチ処理の最適化を妨げる可能性のある要因（例：異なる種類のDMLが頻繁に混在する）が存在するかどうかを判断できます。

### シーン3：特殊な文字セットとLOBデータの処理

**課題**: データ同期後、ターゲット側で文字化けが発生したり、LOBフィールドの内容が不完全になったりします。

**アプローチ**:
1.  **文字セットの問題**：`SHOWSYNTAX` は、印刷不可能な文字（U+0000からU+001F）を16進数形式（`\xx`）で表示します。出力にこのようなエスケープ文字が多数見られる場合は、ソース側とターゲット側の文字セット設定、およびOGGの `SOURCECHARSET` パラメータを確認する必要があります。
2.  **LOBデータの切り捨て**：LOBデータが同期中に失われた疑いがある場合は、`SHOWSYNTAX INCLUDELOB ALL` を使用して完全なLOB内容を表示し、ソース側と比較することができます。これは、LOBベースのXMLやJSONフィールドの同期をデバッグする際に特に役立ちます。

## 4. アドバイスと注意点

`SHOWSYNTAX` は強力なツールですが、正しく使用する必要があります。

1.  **`NOAPPLY` はあなたのセーフティネット**：データに影響を与えるかどうか不確かな場合は、**常に `SHOWSYNTAX NOAPPLY` を優先して使用してください**。これにより、完全にリスクなくSQLを調査できます。
2.  **これはデバッグツールであり、監視ツールではない**：その対話性とシングルスレッドの特性により、`SHOWSYNTAX` はデータ遅延を著しく増加させます。これは問題診断専用であり、問題が解決したらすぐにパラメータファイルから削除すべきです。
3.  **Coordinated ReplicatとParallel Replicatは非対応**：`SHOWSYNTAX` はCoordinated ReplicatやParallel Replicatモードでは使用できません。これらのシナリオでは、Oracleは関連するデータベース適用プロセスで `sqltrace` を有効にすることを代替案として推奨しています。
4.  **`BATCHSQL` とは排他的**：`SHOWSYNTAX` が実行されている間、`BATCHSQL` のバッチ処理最適化は自動的に一時停止します。デバッグが終了し、`SHOWSYNTAX` なしでReplicatを再起動すると、`BATCHSQL` は自動的に再開されます。

## まとめ

迅速なレビューのために、OGGの主要なデバッグツールを比較してみました。

| ツール | 主な用途 | 使用する状況 |
| :--- | :--- | :--- |
| **`SHOWSYNTAX`** | **対話型SQLインスペクタ** | `MAP`マッピング、データ変換、またはReplicatが生成した特定のSQLに起因するパフォーマンス問題のデバッグ。 |
| **レポートファイル (`.rpt`)** | **高レベルの実行サマリ** | 日常的なヘルスチェック、エラー報告、LAG監視。 |
| **Logdump** | **Trailファイル内容ビューア** | Trailファイル内の生のデータレコードを表示し、データがReplicatに到達する前に分析。 |
| **`TRACE`** | **低レベル内部関数トレーサ** | OGG内部を深く調査し、関数呼び出しやバッファの問題を分析。通常はOracle Supportの指示の下で使用。 |

`SHOWSYNTAX`を使いこなせば、 Replicatを完全に理解できるようになります。コマンドラインでの操作が必要ですが、特定の複雑な Replicat の問題を解決するために得られる明確さと確実性は、他では代えがたいものです。精度が求められる場面では、きっとあなたの助けとなるでしょう。
