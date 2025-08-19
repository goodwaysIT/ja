---
layout: post
title: "Oracle SQL実行計画：見積もりから実践までの詳細解析とシナリオ別活用法"
excerpt: "見積もり実行計画と実際の計画の混同は、チューニングの失敗に繋がります。本稿では、EXPLAIN PLANからAWRレポートに至るまで、Oracle SQL実行計画を取得するための各種手法を体系的に解説し、開発、ライブ環境のトラブルシューティング、過去の性能分析といった各シナリオで最適なツールを選択するための明確なフレームワークを提供します。"
date: 2025-08-18 15:02:00 +0900
categories: [Oracle Database, パフォーマンスチューニング]
tags: [oracle, 実行計画, sqlチューニング, パフォーマンスチューニング, dba, explain plan, dbms_xplan, v$sql, awr, tkprof, sql最適化, オラクル, データベース]
author: Shane
image: /assets/images/posts/oracle_execution_plan_overview.svg
---


データベースのパフォーマンスチューニングにおいて、SQL実行計画の分析はパフォーマンスボトルネックを特定し解決するための核心的なプロセスです。しかし、開発者やDBAが取得した実行計画が、本番環境でのSQLの実際の実行パスではない可能性があるという点が、よくある課題です。「見積もり計画」と「実際の計画」を混同することは、しばしば最適化作業を誤った方向へ導きます。

本稿では、OracleデータベースでSQL実行計画を取得するための様々な主要手法を体系的に整理します。開発段階の理論的な予測から、オンライン環境での実際の軌跡追跡まで、各手法の違い、適用シナリオ、限界を深く掘り下げ、異なる状況で最適なツールを選択するための意思決定フレームワークを提供します。

### I. 基礎環境の準備 (Oracle 19c)

すべての例の再現性を確保するため、まず標準的なテスト環境を構築します。明確で一貫性のあるサンドボックスは、効果的な技術検証の基盤となります。

```sql
-- 1. tuserスキーマにサンプルテーブルを作成
CREATE TABLE t_users (
    id           NUMBER(10) NOT NULL,
    username     VARCHAR2(50) NOT NULL,
    status       VARCHAR2(10) DEFAULT 'ACTIVE' NOT NULL,
    created_date DATE NOT NULL
);

-- 2. 主キーと補助インデックスを定義
ALTER TABLE t_users ADD CONSTRAINT pk_users PRIMARY KEY (id);
CREATE INDEX idx_users_status ON t_users(status);

-- 3. テストデータを投入
BEGIN
    FOR i IN 1..100000 LOOP
        INSERT INTO t_users (id, username, created_date) 
        VALUES (i, 'user_' || i, SYSDATE - MOD(i, 365));
    END LOOP;
    -- データの偏りを作成
    UPDATE t_users SET status = 'INACTIVE' WHERE MOD(id, 10) = 0;
    COMMIT;
END;
/

-- 4. 統計情報を収集 (オプティマイザの意思決定の根拠)
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(ownname => 'TUSER', tabname => 'T_USERS');
END;
/
```
環境の準備が整いました。次に、実行計画の取得技術を段階的に探っていきます。

### II. 基礎編：開発段階での「理論的な見積もり」

SQLのコーディングと初期レビューの段階では、実際にクエリを実行してリソースを消費することなく、SQLの理論的な実行パス、特にインデックスの利用状況を迅速に評価することが主な要件となります。

#### 1. `EXPLAIN PLAN`: オプティマイザの静的サンドボックス

`EXPLAIN PLAN`は、実行計画を取得するための最も基本的で迅速なコマンドです。これはSQLを実行せず、コストベースオプティマイザ（CBO）に既存のオブジェクト統計情報に基づいて見積もり実行計画を生成するよう要求するだけです。

このプロセスは地図上で運転ルートを計画することに例えられます。既知の地図情報（テーブルとインデックスの統計）に基づいて最適なルートを提示しますが、実際の交通状況（システム負荷、データキャッシュ）を予測することはできません。

**使用方法：**
```sql
-- 1. 対象SQLの実行計画を生成
EXPLAIN PLAN FOR
SELECT * FROM t_users WHERE status = 'INACTIVE';

-- 2. 計画表からフォーマットして出力
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);


PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
Plan hash value: 616708042

-----------------------------------------------------------------------------
| Id  | Operation         | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |         | 50000 |  1513K|   137   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| T_USERS | 50000 |  1513K|   137   (1)| 00:00:01 |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------

   1 - filter("STATUS"='INACTIVE')

13 rows selected.
```

**重要な考慮事項：**
`EXPLAIN PLAN`の結果は参考価値が高いですが、最終的な結論と見なすべきではありません。よくある落とし穴として、セッションの環境パラメータ（NLS設定など）が本番環境と異なるために生成された計画が実際の計画と食い違い、誤った判断を招くことがあります。**したがって、SQLの迅速で予備的な構造チェックに最も適しています。**

#### 2. `SET AUTOTRACE` (SQL*Plus): 実行と分析の一体化

SQL*Plusのようなコマンドライン環境では、`AUTOTRACE`コマンドがSQLの実行と、その計画および統計情報の表示を組み合わせる便利な方法を提供します。

この方法は、計画されたルートを実際に走行した後に、走行レポートを確認するようなものです。レポートには、燃料消費量（論理読み取り）や走行距離（物理読み取り）などの主要なパフォーマンス指標が含まれます。

**使用方法：**
```sql
-- 問い合わせ結果を表示せず、実行計画と統計情報のみ表示
SET AUTOTRACE TRACEONLY EXPLAIN STATISTICS;

-- 対象クエリを実行
SELECT * FROM t_users WHERE id = 12345;

Execution Plan
----------------------------------------------------------
Plan hash value: 4006063161

----------------------------------------------------------------------------------------
| Id  | Operation                   | Name     | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |          |     1 |    31 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| T_USERS  |     1 |    31 |     2   (0)| 00:00:01 |
|*  2 |   INDEX UNIQUE SCAN         | PK_USERS |     1 |       |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("ID"=12345)


Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
          3  consistent gets
          0  physical reads
          0  redo size
        668  bytes sent via SQL*Net to client
        389  bytes received via SQL*Net from client
          1  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed

-- AUTOTRACEをオフにする
SET AUTOTRACE OFF;
```
`AUTOTRACE`の利点は、実際の実行時統計情報を提供することですが、その前提条件として**SQLの実行が完了するのを待つ必要があります**。長時間実行されるSQLにはこの方法は適していません。

### III. 中級編：オンライン上の「実際の軌跡」の診断

本番環境でパフォーマンス問題をトラブルシューティングする際には、SQLの**現在または直近の実際の実行計画**を取得することが極めて重要です。この実際の実行情報はOracleの共有プール（Shared Pool）に格納されています。

#### `DBMS_XPLAN.DISPLAY_CURSOR`: 精密診断のための「キラーツール」

`DBMS_XPLAN.DISPLAY_CURSOR`は、オンラインのSQLパフォーマンス問題を診断するための最良のツールです。これは共有プールから特定のSQLの実際の実行計画を抽出し、正確な実行時統計情報（実際の返却行数、実際の実行時間など）を付加して表示します。

これは、車のイベントデータレコーダー（EDR）からデータを取得するようなもので、SQL実行の各ステップとそのステップごとの実際の成果と消費時間を忠実に記録します。見積もり行数（E-Rows）と実際の行数（A-Rows）の大きな乖離は、しばしばオプティマイザの見積もりミスを特定し、パフォーマンス問題を発見する鍵となります。

**手順：**

* **対象SQLの`SQL_ID`を特定する**:

```sql
-- SQLテキストの一部を使用してv$sqlビューからSQL_IDを検索
SELECT sql_id, child_number, sql_text 
FROM v$sql 
WHERE sql_text LIKE 'SELECT /* BAD_SQL */%';

SQL_ID        CHILD_NUMBER SQL_TEXT
------------- ------------ ----------------------------------------------------------------------------------------
fanswvakttff4            0 SELECT /* BAD_SQL */ * FROM t_users WHERE TRIM(status) = 'INACTIVE'
fanswvakttff4            1 SELECT /* BAD_SQL */ * FROM t_users WHERE TRIM(status) = 'INACTIVE'

```

* **実際の実行計画を抽出して表示する**:

```sql
-- 見つかったsql_idが 'fanswvakttff4' の場合
-- 'ALLSTATS LAST' パラメータは最後の実行に関する完全な実行時統計情報を取得するために使用
SQL> set line 300 pages 999 long 999999
SQL> SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('fanswvakttff4', null, 'ALLSTATS LAST'));

PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------
SQL_ID  fanswvakttff4, child number 0
-------------------------------------
SELECT /* BAD_SQL */ * FROM t_users WHERE TRIM(status) = 'INACTIVE'

Plan hash value: 616708042

----------------------------------------------
| Id  | Operation         | Name    | E-Rows |
----------------------------------------------
|   0 | SELECT STATEMENT  |         |        |
|*  1 |  TABLE ACCESS FULL| T_USERS |   1000 |
----------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter(TRIM("STATUS")='INACTIVE')

Note
-----
   - Warning: basic plan statistics not available. These are only collected when:
       * hint 'gather_plan_statistics' is used for the statement or
       * parameter 'statistics_level' is set to 'ALL', at session or system level

SQL_ID  fanswvakttff4, child number 1
-------------------------------------
SELECT /* BAD_SQL */ * FROM t_users WHERE TRIM(status) = 'INACTIVE'

Plan hash value: 616708042

----------------------------------------------
| Id  | Operation         | Name    | E-Rows |
----------------------------------------------
|   0 | SELECT STATEMENT  |         |        |
|*  1 |  TABLE ACCESS FULL| T_USERS |  10000 |
----------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter(TRIM("STATUS")='INACTIVE')

Note
-----
   - statistics feedback used for this statement
   - Warning: basic plan statistics not available. These are only collected when:
       * hint 'gather_plan_statistics' is used for the statement or
       * parameter 'statistics_level' is set to 'ALL', at session or system level


49 rows selected.

```

**ケーススタディ：インデックス列への関数使用によるフルテーブルスキャン**

不適切な関数使用によりインデックスが無効になる典型的なシナリオを考察します。

```sql
-- 悪しき実践: インデックス列に関数を使用すると、CBOがインデックスを利用できなくなる
SELECT /* BAD_SQL */ * FROM t_users WHERE TRIM(status) = 'INACTIVE';
```
`DISPLAY_CURSOR`でその実行計画を確認すると、`TABLE ACCESS FULL`（フルテーブルスキャン）を実行していることがわかります。さらに、`A-Rows`（実際の行数）と`E-Rows`（見積もり行数）の間に大きな乖離が見られる場合があります。

```sql
-- ベストプラクティス: 述語内のインデックス列を元の状態のままにする
SELECT /* GOOD_SQL */ * FROM t_users WHERE status = 'INACTIVE';
```
一方、最適化されたSQLの実行計画は正しく`INDEX RANGE SCAN`を使用します。両者の`A-Rows`と`A-Time`（各操作の実際の所要時間）を比較することで、パフォーマンスの違いは一目瞭然となります。

### IV. 上級編：詳細な「フォレンジック」分析

根が深く複雑なパフォーマンス問題に対しては、より低レベルな分析ツールを用いて徹底的な診断を行う必要があります。

#### 1. SQL Trace (10046イベント) & TKPROF

SQL Traceは、SQLセッション内で発生するすべてのデータベースコール、待機イベント、CPU時間などの低レベルな活動を記録し、トレース（.trc）ファイルを生成する強力な診断メカニズムです。これは、SQLの実行プロセスに包括的でミリ秒単位の監視プローブを設置するようなものです。

その後、`TKPROF`ユーティリティを使用して生のトレースファイルをフォーマットし、要約することで、時間の消費に関するあらゆる詳細を正確に明らかにする、非常に可読性の高いパフォーマンス分析レポートを生成します。

**適用シナリオ：**
SQLのパフォーマンスボトルネックがCPU消費ではなく、主に待機イベント（例えば物理I/O待機を示す`db file sequential read`）として現れる場合、SQL Traceが最終的な判断材料となります。時間がどこで「待機」に費やされたかを明確に示します。

**注意：** SQL Traceを有効にすると、ある程度のパフォーマンスオーバーヘッドが発生するため、本番環境での使用は慎重に行うべきであり、通常は難問を解決するための「最後の手段」として使用されます。

#### 2. AWRレポート & `DBMS_XPLAN.DISPLAY_AWR`

自動ワークロードリポジトリ（AWR）は、Oracleに組み込まれたパフォーマンスデータの「ブラックボックス」であり、定期的に（デフォルトでは1時間ごと）データベースの主要なパフォーマンス指標と高負荷SQLのスナップショットを保存します。

過去のパフォーマンス問題（例えば「昨日の午後3時にシステムが遅かった」）を調査する必要がある場合、AWRレポートは遡及分析のための貴重なデータを提供します。レポートの「Top SQL」セクションを分析することで、問題の期間中に最もリソースを消費したSQLを特定できます。そして、`DBMS_XPLAN.DISPLAY_AWR`関数を使用して、そのSQLの特定時点での実行計画を履歴スナップショットから正確に抽出できます。

```sql
-- AWRの履歴から特定のSQL_IDの実行計画を取得
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_AWR('fanswvakttff4'));
```
この方法は、実行計画の急な変更（Plan Flip）によって引き起こされる「SQLのパフォーマンスが良い時と悪い時がある」といった問題の分析に特に有効です。

### まとめと意思決定ガイド

実行計画を取得するためにどのツールを使用するかは、具体的なシナリオと分析目標に依存します。以下に簡潔な意思決定フローを示します。

*   **シナリオ：SQL開発とコードレビュー**
    *   **推奨ツール**：`EXPLAIN PLAN` またはIDE統合ツール。
    *   **目標**：構文、アクセスパス、インデックス使用戦略の正しさを迅速に検証する。

*   **シナリオ：オンラインでのリアルタイムな遅延クエリ診断**
    *   **推奨ツール**：`DBMS_XPLAN.DISPLAY_CURSOR`。
    *   **目標**：実際の実行時統計情報を含む実行計画を取得し、パフォーマンスボトルネックを正確に特定する。

*   **シナリオ：長時間実行されるバッチジョブの分析**
    *   **推奨ツール**：リアルタイムSQLモニタリング（`V$SQL_MONITOR`）。
    *   **目標**：実行の進捗を動的に追跡し、最も時間のかかっているステップを特定する。

*   **シナリオ：過去のパフォーマンスインシデントの事後分析**
    *   **推奨ツール**：AWRレポートと`DBMS_XPLAN.DISPLAY_AWR`の組み合わせ。
    *   **目標**：過去の高負荷SQLとその問題発生時点での実行計画を分析する。

*   **シナリオ：複雑な待機イベント問題の詳細分析**
    *   **推奨ツール**：SQL Trace (10046) と TKPROF。
    *   **目標**：SQLの実行プロセスに対して「フォレンジック」分析を行い、すべての待機とリソース消費を定量化する。

実行計画の取得方法を習得することは、SQL最適化の出発点です。より高度なスキルは、計画の内容を正確に解釈し、その背後にあるオプティマイザの意思決定ロジックを理解することにあります。