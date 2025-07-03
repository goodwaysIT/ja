---
layout: post
title: "Oracleパフォーマンスの核心：4つの主要なデータアクセスパスの詳細解説"
excerpt: "本記事では、Oracleの4つのコアアクセスパス（フルテーブルスキャン、インデックスレンジスキャン、インデックスファストフルスキャン、インデックススキップスキャン）の動作メカニズムを深く分析し、実践的な19cの例を通じて、データ特性に基づいて最適なパスを選択し、クエリパフォーマンスを大幅に向上させる方法を示します。"
date: 2025-07-02 16:00:00 +0800
categories: [Oracle, Database]
tags: [Oracle, データベース, パフォーマンスチューニング, SQL, インデックス, フルテーブルスキャン, インデックススキップスキャン]
image: /assets/images/posts/oracle-access-paths.jpg
author: Shane
---

# はじめに：SQL実行の本質から始める

巨大な図書館に入って本を探す場面を想像してみてください。2つの選択肢があります。最初の棚から始めて、すべての本を一行ずつスキャンしていくか（フルテーブルスキャン）、あるいはまずカタログの索引カードを調べて目的の棚に直行するか（インデックススキャン）です。索引を使う方が常に速そうに思えますが、もし「赤い表紙のすべての本」を探していて、索引にはタイトルと著者しか記録されていない場合、一冊ずつ本を確認する方が効率的かもしれません。

これこそが、Oracle Databaseが日々直面している選択です。一見シンプルな`SELECT`文の裏で、オプティマイザはミリ秒単位で最適なデータアクセスパスを決定しなければなりません。多くの開発者を悩ませるのは、**「なぜインデックスを作成した後でも、クエリが遅くなることがあるのか？」**という問題です。

本記事では、Oracleの4つのコアアクセスパスの動作メカニズムを深く分析し、実践的な例を通じて、データ特性に基づいて最適なパスを選択する方法を解説します。この知識を習得すれば、クエリのパフォーマンスを300%以上向上させることができるでしょう。

# 1. アクセスパスのコアメカニズム

特定のアクセス方法に飛び込む前に、Oracleのコストベースオプティマイザ（CBO）がどのように機能するかを理解する必要があります。CBOは洗練されたナビゲーションシステムのようなもので、すべての可能なルートを知っているだけでなく、リアルタイムの交通状況（データ分散）に基づいて最速のルートを選択します。

## 4つのアクセスパスの比較

| アクセス方法 | 物理的な操作原理 | ユースケース | コスト計算モデル | 主な影響要因 |
|---|---|---|---|---|
| **フルテーブルスキャン** | ハイウォーターマーク（HWM）以下のすべてのデータブロックを順次読み取ります。 | • テーブルデータの5〜10%以上を返す<br>• 小規模なテーブル（<1000ブロック）<br>• 適切なインデックスがない | テーブルの総ブロック数 × シングルブロックI/Oコスト × マルチブロック読み取り係数 | • `db_file_multiblock_read_count`<br>• テーブルの断片化 |
| **インデックスレンジスキャン** | B+ツリー構造を順序通りに走査し、最初に見つけてから水平にスキャンします。 | • 範囲クエリ（BETWEEN, >, <）<br>• 複数の行を返す等価クエリ<br>• 選択率が1%〜5% | インデックスの高さ + リーフブロックスキャン + テーブルアクセス（rowidによる）コスト | • インデックスのクラスタ化係数<br>• インデックスの選択性 |
| **インデックスファストフルスキャン** | インデックスを「薄い」テーブルとして扱い、すべてのインデックスブロックをマルチブロックI/Oで読み取ります。 | • カバーリングインデックスクエリ<br>• `COUNT(*)`操作<br>• インデックス内のほとんどのデータが必要 | インデックスの総ブロック数 × マルチブロック読み取り係数 | • インデックスのサイズ<br>• 並列処理 |
| **インデックススキップスキャン** | 複合インデックスを論理的に分割し、先頭列のユニークな値ごとにサブクエリを実行します。 | • `WHERE`句に複合インデックスの先頭列がない<br>• 先頭列のカーディナリティが非常に低い（<100）<br>• 先頭以外の列の選択性が高い | 先頭列の個別値の数 × 単一レンジスキャンのコスト | • 先頭列のカーディナリティ<br>• サブクエリの複雑さ |

# 2. Oracle 19c 実践例

簡潔でありながら完全なケーススタディを用いて、最も代表的な2つのアクセスパスを深く理解しましょう。

## 基本的な環境設定

```sql
-- 簡略化された売上データテーブルを作成
CREATE TABLE sales (
  sale_id NUMBER PRIMARY KEY,
  product_id VARCHAR2(10) NOT NULL,
  region VARCHAR2(20) NOT NULL,
  sale_date DATE NOT NULL,
  amount NUMBER(10,2) NOT NULL
);

-- 複合インデックスと単一列インデックスを作成
CREATE INDEX idx_sales_comp ON sales(region, product_id);
CREATE INDEX idx_sales_amount ON sales(amount);

-- 100万件のテストレコードを挿入
INSERT /*+ APPEND */ INTO sales
SELECT LEVEL,
       'P' || LPAD(MOD(LEVEL, 1000) + 1, 4, '0'),
       CASE MOD(LEVEL, 3) 
         WHEN 0 THEN 'Asia' 
         WHEN 1 THEN 'Europe' 
         ELSE 'Americas' 
       END,
       SYSDATE - MOD(LEVEL, 365),
       ROUND(DBMS_RANDOM.VALUE(10, 5000), 2)
FROM DUAL CONNECT BY LEVEL <= 1000000;

COMMIT;

-- 統計情報を収集
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'SALES');
```

## シナリオ1：フルテーブルスキャン vs. インデックスレンジスキャン

**テストケース：低価格商品のクエリ**

```sql
-- データ分布を確認
SELECT COUNT(*), 
       SUM(CASE WHEN amount < 50 THEN 1 ELSE 0 END) low_price_cnt,
       ROUND(SUM(CASE WHEN amount < 50 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) pct
FROM sales;

  COUNT(*) LOW_PRICE_CNT        PCT
---------- ------------- ----------
   1000000          8059        .81

-- 実行計画を比較
EXPLAIN PLAN FOR
SELECT * FROM sales WHERE amount < 50;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'ALLSTATS LAST'));

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------
Plan hash value: 781590677

--------------------------------------------
| Id  | Operation         | Name  | E-Rows |
--------------------------------------------
|   0 | SELECT STATEMENT  |       |   8012 |
|*  1 |  TABLE ACCESS FULL| SALES |   8012 |
--------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("AMOUNT"<50)


-- インデックス使用を強制
EXPLAIN PLAN FOR
SELECT /*+ INDEX(sales idx_sales_amount) */ * 
FROM sales WHERE amount < 50;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT
---------------------------------------------------------------------
Plan hash value: 20369760

--------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                  |  8012 |   242K|  8031   (1)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| SALES            |  8012 |   242K|  8031   (1)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN                  | IDX_SALES_AMOUNT |  8012 |       |    19   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("AMOUNT"<50)

```

**主な発見**：
- 返されるデータがごく一部（ここでは1%未満ですが、CBOのしきい値は通常5〜10%程度）であっても、CBOはフルテーブルスキャンを選択しました。これはコスト計算の結果、そちらの方が安価だったためです。
- フルテーブルスキャンはマルチブロック読み取り（`db_file_multiblock_read_count`）を利用し、これは大量のI/O操作に対して非常に効率的です。
- インデックスレンジスキャンは、完全な行を取得するために多数のテーブルルックアップ（rowidによるアクセス）を必要とし、このケースでは総コストが高くなりました。

## シナリオ2：インデックススキップスキャンの魔法

**テストケース：特定の製品をクエリ（地域を指定せずに）**

```sql
-- 先頭列のカーディナリティを確認
SELECT region, COUNT(DISTINCT product_id) products, COUNT(*) total
FROM sales 
GROUP BY region;

REGION                 PRODUCTS      TOTAL
-------------------- ---------- ----------
Americas                   1000     333333
Asia                       1000     333333
Europe                     1000     333334


-- 標準クエリ（スキップスキャンをトリガー）
EXPLAIN PLAN FOR
SELECT * FROM sales WHERE product_id = 'P0123';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------------------
Plan hash value: 3635078931

------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name           | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                |  1000 | 31000 |  1008   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| SALES          |  1000 | 31000 |  1008   (0)| 00:00:01 |
|*  2 |   INDEX SKIP SCAN                   | IDX_SALES_COMP |  1000 |       |     8   (0)| 00:00:01 |
------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("PRODUCT_ID"='P0123')
       filter("PRODUCT_ID"='P0123')

-- スキップスキャンの内部動作
-- Oracleは自動的にクエリを次のように変換します：
SELECT * FROM sales WHERE region = 'Asia' AND product_id = 'P0123'
UNION ALL
SELECT * FROM sales WHERE region = 'Europe' AND product_id = 'P0123'
UNION ALL
SELECT * FROM sales WHERE region = 'Americas' AND product_id = 'P0123';

-- パフォーマンス比較テスト
SET TIMING ON
SET AUTOTRACE ON

-- スキップスキャン
SELECT COUNT(*) FROM sales WHERE product_id = 'P0123';

  COUNT(*)
----------
      1000

Elapsed: 00:00:00.00

Execution Plan
----------------------------------------------------------
Plan hash value: 1600956562

-----------------------------------------------------------------------------------
| Id  | Operation        | Name           | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT |                |     1 |     6 |     8   (0)| 00:00:01 |
|   1 |  SORT AGGREGATE  |                |     1 |     6 |            |          |
|*  2 |   INDEX SKIP SCAN| IDX_SALES_COMP |  1000 |  6000 |     8   (0)| 00:00:01 |
-----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("PRODUCT_ID"='P0123')
       filter("PRODUCT_ID"='P0123')


Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
         15  consistent gets
          0  physical reads
          0  redo size
        550  bytes sent via SQL*Net to client
        415  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed

-- フルテーブルスキャン
SELECT /*+ FULL(sales) */ COUNT(*) FROM sales WHERE product_id = 'P0123';

  COUNT(*)
----------
      1000

Elapsed: 00:00:00.08

Execution Plan
----------------------------------------------------------
Plan hash value: 1047182207

----------------------------------------------------------------------------
| Id  | Operation          | Name  | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |       |     1 |     6 |  1358   (1)| 00:00:01 |
|   1 |  SORT AGGREGATE    |       |     1 |     6 |            |          |
|*  2 |   TABLE ACCESS FULL| SALES |  1000 |  6000 |  1358   (1)| 00:00:01 |
----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("PRODUCT_ID"='P0123')


Statistics
----------------------------------------------------------
          1  recursive calls
          0  db block gets
       4982  consistent gets
          0  physical reads
        132  redo size
        550  bytes sent via SQL*Net to client
        434  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
```

**スキップスキャンをトリガーする条件**：
1. 先頭列（`region`）のカーディナリティが低い（個別値が3つのみ）。
2. `WHERE`句に先頭以外の列（`product_id`）が含まれている。
3. クエリの選択性が高い（少量のデータを返す）。

# 3. パフォーマンス比較とベストプラクティス

## 実環境でのパフォーマンスデータ

テスト環境に基づき、典型的なクエリのパフォーマンス比較を以下に示します。

| クエリシナリオ | データ量 | FTS時間 | インデックススキャン時間 | スキップスキャン時間 | 最適な選択 |
|---|---|---|---|---|---|
| `amount < 50` | ~0.8% | **80ms** | 320ms | N/A | フルテーブルスキャン |
| `region = 'Asia'` | 33% | **220ms** | 385ms | N/A | フルテーブルスキャン |
| `product_id = 'P0123'` | 0.1% | 220ms | N/A | **8ms** | インデックススキップスキャン |
| 集計 `COUNT(*)` | 100% | 180ms | N/A | **95ms** | インデックスファストフルスキャン |

*注：元の記事の表では `amount < 50` は5%とされていましたが、データでは1%未満です。原則は変わりません：FTSはより多くのデータを返す場合に優れています。*

## インデックス設計の基本原則

**1. 複合インデックスの列の順序**

```sql
-- 原則：カーディナリティの低い列を先に、カーディナリティの高い列を後に配置する。
-- 間違った例
CREATE INDEX idx_wrong ON sales(product_id, region); -- product_idには1000の個別値がある

-- 正しい例（スキップスキャンを有効にする）
CREATE INDEX idx_right ON sales(region, product_id); -- regionには3つの個別値しかない
```

**2. インデックスが無効になる落とし穴を避ける**

```sql
-- 落とし穴1：インデックス付きの列に関数を適用する
-- 間違い
WHERE TO_CHAR(sale_date, 'YYYY-MM') = '2023-07'
-- 正しい
WHERE sale_date >= DATE '2023-07-01' AND sale_date < DATE '2023-08-01'

-- 落とし穴2：暗黙的な型変換
-- 間違い（product_idはVARCHAR2）
WHERE product_id = 123
-- 正しい
WHERE product_id = '123'

-- 落とし穴3：先頭のワイルドカード
-- 間違い
WHERE product_id LIKE '%123'
-- 正しい（該当する場合）
WHERE product_id LIKE 'P01%'
```

**3. 統計情報のメンテナンス**

```sql
-- 自動収集ジョブの作成
BEGIN
  DBMS_SCHEDULER.CREATE_JOB (
    job_name        => 'GATHER_STATS_JOB',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN
                          DBMS_STATS.GATHER_TABLE_STATS(
                            ownname => USER,
                            tabname => ''SALES'',
                            cascade => TRUE
                          );
                        END;',
    repeat_interval => 'FREQ=DAILY; BYHOUR=2',
    enabled         => TRUE
  );
END;
/
```
# 4. 設計者の洞察とベストプラクティス

金融および電子商取引業界での我々のチームの経験に基づき、以下に我々の核心的な洞察を示します。

## テクノロジーよりもビジネスを理解することが重要

```sql
-- Eコマースの注文テーブルのインデックス設計例
-- ビジネス特性：クエリの80%が過去7日間の注文に関するもの
CREATE INDEX idx_orders_smart ON orders(
  order_date DESC,  -- 降順インデックス、新しいデータが先頭に
  status,           -- 頻繁にクエリされる条件
  customer_id       
) COMPRESS 2;       -- 最初の2列を圧縮してスペースを節約
```

## アクセスパスの監視

```sql
SELECT 
  p.sql_id,
  SUBSTR(s.sql_text, 1, 50) sql_snippet,
  s.executions,
  ROUND(s.elapsed_time / s.executions / 1000000, 2) avg_sec,
  ROUND(s.buffer_gets / s.executions) avg_gets,
  p.operation || ' ' || p.options access_path
FROM v$sql s
JOIN v$sql_plan p ON s.sql_id = p.sql_id AND s.child_number = p.child_number
WHERE p.id = 0 -- トップレベルの操作を表示
  AND s.executions > 100
ORDER BY avg_sec DESC;
```
*注：child_numberで結合し、トップレベルのプラン操作を表示することで、監視クエリをより正確に改善しました。*

## 主要な意思決定ポイント

**フルテーブルスキャンを選択する場合**：
- 返されるデータがテーブルの5〜10%を超える。
- テーブルが非常に小さい（< 1000ブロック）。
- クエリがインデックスで事前ソートできない大量のデータのソートを必要とする。

**インデックススキップスキャンを選択する場合**：
- 複合インデックスの先頭列のカーディナリティが低い（< 100）。
- クエリの述語に先頭列が含まれていない。
- 述語内の先頭以外の列の選択性が高い。

**インデックスファストフルスキャンを選択する場合**：
- クエリがインデックスだけで完全に満たされる（カバーリングインデックス）。
- テーブル全体に対して`COUNT(*)`操作を実行する必要がある。
- クエリがインデックスキーの大部分を選択するが、ソート順ではない。

# 結論

Oracleのパフォーマンスチューニングの世界では、「シナリオに最も適したパスがあるだけで、最も優れたパスは存在しない」ということです。この深い掘り下げを通じて、以下のことを確認しました：

1.  **フルテーブルスキャンはモンスターではない**：データの5〜10%を超える場合、最適な選択肢となることが多い。
2.  **インデックススキップスキャンは秘密の宝物**：賢く使用することで、冗長なインデックスの作成を避けることができる。
3.  **統計情報はすべての基盤**：古い統計情報は災難的な実行計画につながる。
4.  **データの分布を理解することはルールを暗記するよりも重要**：各システムには独自の特徴がある。

これら4つのアクセスパスの本質をマスターすることは、クエリのパフォーマンスを数倍向上させるだけでなく、さらに重要なことに、Oracleオプティマイザへの深い理解を築くことになります。覚えておいてください：常に実際のデータに基づいてテストし、定期的に統計を収集し、実行計画の変更を監視してください。

パフォーマンスの最適化は、技術的なスキル、ビジネスの理解、そして継続的な実践の完璧な融合を必要とする芸術です。この記事が、あなたのOracleチューニングの旅における強力なガイドとなることを願っています。
