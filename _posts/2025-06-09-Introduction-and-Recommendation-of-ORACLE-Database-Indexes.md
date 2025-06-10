---
layout: post
title: "ORACLEデータベースインデックスの紹介と推奨"
excerpt: "このドキュメントでは、ORACLEデータベースのパーティション表に関連付けられた索引の分類、適用シナリオ、推奨される使用法について、Q&A形式でご紹介します。"
date: 2025-06-09 21:08:00 +0800
categories: [Oracle, Database]
tags: [Database maintenance, Database deployment,Database optimization, oracle]
image: /assets/images/posts/Description-and-recommendation-of-ORACLE-database-index.jpg
---

このドキュメントは、Q&A形式を通じて、ORACLEデータベースのパーティションテーブルに関連するインデックスの分類、適用シナリオ、および推奨される使用方法を紹介することを目的としています。  
パーティションテーブル上にパーティションインデックスを作成するかグローバルインデックスを作成するかという一般的な懸念については、以下のQ&Aを参照してください:

*  パーティションテーブルのインデックスの分類は？
*  上記3種類のインデックスの長所と短所は？
*  パーティションインデックスの分類は？
*  上記3種類のパーティションインデックスの長所と短所は？
*  ローカルインデックスのパフォーマンスへの影響は？
*  グローバルインデックスのパフォーマンスへの影響は？
*  なぜパーティションテーブルのインデックスがUNUSABLEになるのか？
*  パーティションテーブルのメンテナンス方法
*  作成するインデックスの種類の選択方法
*  構文例

パーティションテーブル上に **Global Nonpartitioned** インデックスを作成する場合、定期的なパーティションメンテナンス（例：月次パーティションテーブルで毎月初めに3ヶ月以上前のパーティションを削除）中にインデックス無効化などの問題が発生する可能性があります。

"update index"句を使用すると、グローバルインデックスを同時に更新し、無効化を回避できます。ただし、インデックスのメンテナンスには大量のブロック変更が伴い、パフォーマンス問題のリスクがあります（Parallel DML操作で競合が発生し、"enq : TX - index contention" や "gc" 待機イベントが発生する可能性があります）。

12c以降、"**Global Index Delayed Maintenance**"機能が導入され、"drop partition"時の負荷が軽減されました。ただし、"Global Index Delayed Maintenance JOB"実行時に大規模な集中処理が発生し、パフォーマンス問題を引き起こす可能性があります。

私たちは複数の顧客で上記の問題に遭遇しました。実践的な経験に基づき、システムの安定性（**グローバル**インデックスメンテナンス中の他のOLTP処理への影響回避）を考慮し、**より効率的なローカルインデックス**の設計と、**グローバル**インデックスを使用しないようにSQL文の'WHERE'句を修正することを推奨します。

## パーティションテーブルインデックスに関するよくある質問:

### 1. パーティションテーブルのインデックスの分類は？

パーティションテーブルに関連するインデックスは、主に **グローバルインデックス** と **ローカルインデックス** に分類され、以下のように定義されます:

**グローバルインデックス:**  
"Global Index"とも呼ばれ、複数のテーブルパーティションを指すインデックスキーを含むインデックスです。  
グローバルインデックスには **グローバルパーティションインデックス** (Global Partitioned index) と **グローバル非パーティションインデックス** (Global Nonpartitioned index) が含まれます。

**Global Partitioned Index**  
![](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/img/vldbg007.gif)

**Global Nonpartitioned index**  
![](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/img/vldbg006.gif)

**ローカルインデックス:**  
**Local indexes** または **Local Partitioned Indexes** とも呼ばれ、テーブルと同じ列でパーティション化され、同じ数のパーティションとパーティション境界を持つインデックスです。したがって、各インデックスパーティションは単一の基盤となるテーブルパーティションに関連付けられており、インデックスパーティション内のすべてのキーは、対応する単一のテーブルパーティションに格納された行のみを参照します。

**Local Partitioned Index**  
![](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/img/vldbg003.gif)

### 2. 上記3種類のインデックスの長所と短所は？

一般的に、テーブルがパーティションテーブルとして設計される主な理由は2つです:
1. テーブルデータへのアクセス時にスキャンが必要な基盤データ（またはレコード）の量を減らす（パーティション剪定 - "Partition pruning"）。
2. 履歴データメンテナンス中の負荷を軽減する（例：定期的な履歴データの削除）。

これら2つの目標を達成する上で、**ローカルインデックス** (Local Index)、**グローバルパーティションインデックス** (Global Partitioned index)、**グローバル非パーティションインデックス** (Global Nonpartitioned index) の違いは以下の通りです:

<style>
      .mypic {
            float: left;  
            margin-right: 10px;
      }
      .left-align {
            text-align: left;
    }
    table {
        width: 100%;
        border-collapse: collapse;
    }
    th, td {
        border: 1px solid black;
        padding: 8px;
        text-align: left;
    }
    th {
        background-color: #f2f2f2;
    }
</style>

| | **Local Partitioned Index** | **Global Partitioned index** | **Global Nonpartitioned index** |
| --- | --- | --- | --- |
| **パーティション・プルーニング** | 高 <br> テーブルとインデックスの両方でパーティション剪定が可能 | 中 <br> インデックスでパーティション剪定が可能 | 低 <br> パーティション剪定を達成できない |
| **履歴データメンテナンスコスト** | 低 <br> テーブルとインデックスパーティションを同時にメンテナンス可能、他のパーティションに影響なし | 中 <br> テーブルパーティションのメンテナンスが複数のインデックスパーティションに影響を与える可能 | 高 <br> テーブルパーティションのメンテナンスがインデックス全体に影響を与える可能 |

表が示すように、**ローカルインデックス** (Local Index) は上記2つの目標を達成するための最良の方法です。**グローバル非パーティションインデックス** (Global Nonpartitioned index) はほとんど利点がなく、**検索条件でパーティションキーを指定できない場合にのみ検討すべき**です。**グローバルパーティションインデックス** (Global Partitioned index) は最初の2つの中間的な妥協案です。

したがって、パーティションテーブルの利点を活用するために、可能な限りパーティションインデックス (Partition Index) の使用を推奨します。

### 3. パーティションインデックスの分類は？

**パーティションキー** と **インデックス列** の関係に基づいて、パーティションインデックスはさらに **ローカル接頭辞インデックス** (Local Prefixed Index)、**ローカル非接頭辞インデックス** (Local Nonprefixed Index)、**グローバル接頭辞パーティションインデックス** (Global Prefixed Partitioned Index) に分類されます。定義は以下の通りです:

**ローカル接頭辞インデックス:** インデックス列の左接頭辞でパーティション化されたローカルインデックスで、パーティションキーがインデックスキーに含まれています。
ローカル接頭辞インデックスは一意でも非一意でもかまいません。

**例:**  
'emp'テーブルとそのローカルインデックス'ix1'が'deptno'列でパーティション化されている場合、インデックス'ix1'が列(deptno, other_columns)で定義されている場合、これは **ローカル接頭辞インデックス** です。  

**Local Prefixed Index**  
![](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/img/vldbg019.gif)

**ローカル非接頭辞インデックス:** インデックス列の左接頭辞でパーティション化されていない、またはインデックスキーにパーティションキーが含まれていないローカルインデックス。
パーティションキーがインデックスキーのサブセットでない限り、一意のローカル非接頭辞インデックスは不可能です。

**例:**  
'checks'テーブルとそのローカルインデックス'ix3'が'chkdate'列でパーティション化されている場合、インデックス'ix3'が列(acctno)で定義されている場合、これは **ローカル非接頭辞インデックス** です。   

**Local Nonprefixed Index**  
![](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/img/vldbg018.gif)

**グローバルパーティションインデックス:**  
グローバルインデックスが基盤となるテーブルとは異なる方法でパーティション化されている場合、**グローバルパーティションインデックス** (Global Partitioned Index) と呼ばれます。グローバルパーティションインデックスは接頭辞付きインデックスのみをサポートしており、インデックスのパーティションキーはインデックス列の左接頭辞です。Oracleデータベースは非接頭辞グローバルパーティションインデックスをサポートしていません。  

グローバル接頭辞パーティションインデックスは一意でも非一意でもかまいません。非パーティションインデックスはグローバル接頭辞非パーティションインデックスと見なされます。

**例:**  
'emp'テーブルが'deptno'列でパーティション化され、インデックス'ix3'が'empno'列でパーティション化されている場合、インデックス'ix3'が列(empno, other_columns)で定義されている場合、これは **グローバル接頭辞パーティションインデックス** です。    

**Global Prefixed Partitioned Index**  
![](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/img/vldbg020.gif)

### 4. 上記3種類のパーティションインデックスの長所と短所は？
以下はこれら3種類のインデックスの構造比較です:

| | **インデックスとテーブルのパーティションが同一** | **パーティションキーがインデックス列の左接頭辞** | **例: テーブルパーティションキー** | **例: インデックスパーティションキー** | **例: インデックス列** |
| --- | --- | --- | --- | --- | --- |
| **ローカル接頭辞インデックス** <br> 任意のパーティショニング (Range, Hashなど) | はい | はい | A | A | A,B |
| **ローカル非接頭辞インデックス** <br> 任意のパーティショニング (Range, Hashなど) | はい | いいえ | A | A | B,A |
| **グローバル接頭辞インデックス** <br> 範囲パーティショニング | いいえ | はい | A | B | B |

以下はその他の特性の比較です:

| | **一意性を許可** | **管理の難易度** | **OLTPアプリケーション向け** <br> *(オンライントランザクション処理)* | **DSSアプリケーション向け** <br> *(意思決定支援システム)* |
| --- | --- | --- | --- | --- |
| **ローカル接頭辞インデックス** <br> 任意のパーティショニング (Range, Hashなど) | はい | 簡単 | 良い | 良い |
| **ローカル非接頭辞インデックス** <br> 任意のパーティショニング (Range, Hashなど) | はい | 簡単 | 悪い | 良い |
| **グローバル接頭辞インデックス** <br> 範囲パーティショニング | はい | より困難 | 良い | 良くない |

表が示すように、**ローカル接頭辞インデックス** (Local Prefixed Index) は評価されたすべての特性で最高のパフォーマンスを発揮します。他の2種類にはそれぞれ長所と短所があり、使用前に評価が必要です。

したがって、特定の要件がない限り、パーティションインデックスの中では可能な限り **ローカル接頭辞インデックス** (Local Prefixed Index) の使用を推奨します。

### 5. ローカルインデックスのパフォーマンスへの影響は？

*   **システム可用性の向上:** データの無効化や利用不能を引き起こす操作は、影響を受けるパーティションのみに限定されます。
*   **パーティションメンテナンスの簡素化:** テーブルパーティションを移動する場合、関連するローカルインデックスパーティションのみを再構築またはメンテナンスすれば十分です。。
*   **パーティションプルーニング:** パーティション化されたテーブルに対するSQL実行時、‘WHERE’句によって不要なパーティションを自動的に除外することができます。
*   **テーブルスペースのポイントインタイムリカバリ(PITR)の簡素化:** テーブルパーティションまたはサブパーティションを特定の時点にリカバリするには、対応するインデックスエントリも同じ時点にリカバリする必要があります。これを実現するには、ローカルインデックスの使用が唯一の方法となります。

要約すると、アプリケーションが **パーティションキー** または **パーティションキーと他のフィールドの組み合わせ** を使用してデータにアクセスする場合、**ローカル接頭辞インデックスが最適な選択肢**です。

ローカル非接頭辞インデックスを使用する場合、クエリ条件にパーティションキーが含まれていないと、すべてのパーティションをスキャンする必要があります。パーティションキーが含まれている場合にのみパーティション剪定が発生します。   
テーブルT1を例にします（フィールドC1でパーティション化）、フィールド(C2, C3)にローカル非接頭辞インデックス'idx_ind2'を作成:

```SQL
select * from t1 where c2=‘20170609’ and c3=’1’ ;
------ クエリ条件にパーティションキーが含まれない場合、インデックスidx_ind2を使用するがすべてのパーティションをスキャン
```
```
-----------------------------------------------------------------------------------------------------------------------
| Id | Operation                                 | Name      | Rows | Bytes | Cost (%CPU)| Time     | Pstart| Pstop |
-----------------------------------------------------------------------------------------------------------------------
|  0 | SELECT STATEMENT                          |           |   10 |   250 |     2   (0)| 00:00:01 |       |       |
|  1 |  PARTITION RANGE ALL                      |           |   10 |   250 |     2   (0)| 00:00:01 |     1 |    10 |
|  2 |   TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| T1       |   10 |   250 |     2   (0)| 00:00:01 |     1 |    10 |
|* 3 |    INDEX RANGE SCAN                       | IDX_IND2 |   10 |       |     1   (0)| 00:00:01 |     1 |    10 |
-----------------------------------------------------------------------------------------------------------------------
```

```SQL
select * from t1 where c2=‘20170609’ and c3=’1’ and C1=’20160609’；
------ クエリ条件にパーティションキーが含まれる場合、インデックスidx_ind2を使用し単一パーティションのみスキャン
------ (グローバル非パーティションインデックスはパーティションキーが含まれていてもパーティション剪定を達成できない)
```
```
-----------------------------------------------------------------------------------------------------------------------
| Id | Operation                                 | Name      | Rows | Bytes | Cost (%CPU)| Time     | Pstart| Pstop |
-----------------------------------------------------------------------------------------------------------------------
|  0 | SELECT STATEMENT                          |           |   99 |  2475 |     2   (0)| 00:00:01 |       |       |
|  1 |  PARTITION RANGE SINGLE                   |           |   99 |  2475 |     2   (0)| 00:00:01 |    10 |    10 |
|* 2 |   TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| T1       |   99 |  2475 |     2   (0)| 00:00:01 |    10 |    10 |
|* 3 |    INDEX RANGE SCAN                       | IDX_IND2 |  100 |       |     1   (0)| 00:00:01 |    10 |    10 |
-----------------------------------------------------------------------------------------------------------------------
```
### 6. グローバルインデックスのパフォーマンスへの影響は？
高速アクセス、整合性、可用性に有用: OLTPシステムでは、テーブルが1つのキー（例：employees.department_id）でパーティション化されている場合でも、アプリケーションは多くの異なるキー（例：employee_idやjob_id）を使用して複数のパーティションにまたがるデータにアクセスする必要があるかもしれません。グローバルインデックスは、これらのクロスパーティションデータアクセスシナリオで有用です。

管理がより困難: テーブルパーティションのメンテナンス中（例：履歴パーティションの削除）、グローバルインデックスのすべてのパーティションが影響を受ける可能性があります。

高競合時のパフォーマンス向上の可能性: 少数のリーフブロックで高競合が発生するマルチユーザーOLTP環境では、ハッシュインデックスパーティショニングがインデックスパフォーマンスを改善できます。

要約すると、アプリケーションが複数のパーティションにまたがってデータにアクセスする場合（クエリ条件にパーティションキーが含まれない）、アプリケーションの実際のニーズに基づいてグローバルインデックスの使用を検討できます。ただし、グローバルインデックスを使用することは、後続のパーティションメンテナンス操作における関連するメンテナンスコストを受け入れることを意味します。

### 7. なぜパーティションテーブルのインデックスがUNUSABLEになるのか？
デフォルトでは、パーティションテーブルでの多くのテーブルメンテナンス操作は、グローバルインデックスと影響を受けたローカルインデックスパーティションを無効化（UNUSABLEとしてマーク）します。テーブルでパーティションメンテナンスを実行する場合、グローバルインデックスのすべてのパーティションが影響を受けます。その後、インデックス全体または影響を受けた各インデックスパーティションを再構築する必要があります。

操作が影響を受けたインデックスパーティションを利用不可としてマークするかどうかは、パーティションタイプに依存します（例：範囲パーティションテーブルでは、パーティションの追加はグローバルおよび影響を受けたローカルインデックスを利用不可とマークしませんが、ハッシュパーティションテーブルではそうする可能性があります）。

### 8. パーティションテーブルのメンテナンス方法
一般的に、パーティション全体のデータをクリーンアップする必要がある場合、DROP PARTITIONまたはTRUNCATE PARTITIONの使用を推奨します。DELETEを使用してパーティション全体をクリーンアップすることは非効率的であり、後で追加の使用およびメンテナンスコストが発生します。前述のように、パーティションメンテナンス中にインデックスがUNUSABLEになる可能性を考慮する必要があるため、パーティションメンテナンス中に同時にインデックスをメンテナンスする必要があります。

ALTER TABLE文でメンテナンス操作のUPDATE INDEXES句を指定します。この句は、メンテナンスDDL文の実行中にデータベースがインデックス（グローバルおよび影響を受けたローカルインデックスパーティションの両方）を更新するように指示します。

グローバルインデックスのみを更新するには、UPDATE GLOBAL INDEXES句を使用します。

以下の操作がUPDATE INDEXES句をサポートしています:

```sql
ADD      PARTITION | SUBPARTITION  
COALESCE PARTITION | SUBPARTITION  
DROP     PARTITION | SUBPARTITION  
EXCHANGE PARTITION | SUBPARTITION  
MERGE    PARTITION | SUBPARTITION  
MOVE     PARTITION | SUBPARTITION  
SPLIT    PARTITION | SUBPARTITION  
TRUNCATE PARTITION | SUBPARTITION  
```

指定"UPDATE INDEXES"時の注目すべき影響:  
パーティションDDL文の実行時間が長くなります。なぜなら、本来ならUNUSABLEとマークされるはずのインデックスが更新されるためです。  
経験則として、パーティションサイズがテーブルサイズの5%未満の場合、インデックスの更新は比較的高速です。  

**注意:**  
12c以降、"Global Index Delayed Maintenance"機能が導入され、DROPおよびTRUNCATEテーブルパーティション操作に関連するグローバルインデックスのメンテナンスが最適化されました。グローバルインデックスのメンテナンスは、DROPまたはTRUNCATE操作とは独立してスケジュールできます。グローバルインデックスのメンテナンスが完了するまで、パーティションテーブルに対するクエリ、DDL、またはDMLはグローバルインデックスの無効なエントリを無視します（この機能は、テーブルパーティションのサブセットにインデックスを作成する機能に依存しているため、テーブルのグローバルインデックス全体を利用不可としてマークする必要がなくなります）。

グローバルインデックスを持つテーブルを更新すると、インデックスはその場で更新されます。インデックスの更新はログに記録され、redoおよびundoレコードが生成されます。対照的に、グローバルインデックス全体を手動で再構築する場合、NOLOGGINGモードで実行できます。さらに、手動再構築はよりコンパクトで効率的なインデックスを作成する可能性があります。

インデックスまたはインデックスパーティションがパーティションメンテナンス操作前に利用不可だった場合、update indexes句が指定されていても操作後も利用不可のままです。

### 9. 作成するインデックスの種類の選択方法  
インデックスタイプを選択する際は、以下の決定木を参照してください:  
フィールドc1でパーティション化されたテーブルT1  
![](https://goodwaysit.github.io/en/assets/images/database/Decision_Tree.png)

### 10. 構文例
--テスト用テーブル作成:
```sql
DROP TABLE t1;
CREATE TABLE t1(
c1 CHAR (8) NOT NULL
,c2 CHAR (8) NOT NULL
,c3 NUMBER (22,4) NOT NULL
,c4 NUMBER (22,4) NOT NULL
)
PARTITION BY RANGE (c1)
(
PARTITION t_p1 VALUES LESS THAN('20160601')
,PARTITION t_p2 VALUES LESS THAN('20160602')
,PARTITION t_p3 VALUES LESS THAN('20160603')
,PARTITION t_p4 VALUES LESS THAN('20160604')
,PARTITION t_p5 VALUES LESS THAN('20160605')
,PARTITION t_p6 VALUES LESS THAN('20160606')
,PARTITION t_p7 VALUES LESS THAN('20160607')
,PARTITION t_p8 VALUES LESS THAN('20160608')
,PARTITION t_p9 VALUES LESS THAN('20160609')
,PARTITION t_P10 VALUES LESS THAN (MAXVALUE)
);

begin
for i in 0 .. 9 loop
for j in 1 .. 100 loop
insert into t1 values('2016060'||to_char(i) ,'2017060'||to_char(i),10-i,10000-j);
commit;
end loop;
end loop;
end;
/
```

--ローカル接頭辞インデックス:
```sql
create index idx_ind1 on t1 (c1,c3) local;
```

--ローカル非接頭辞インデックス:
```sql
create index idx_ind2 on t1 (c2,c3) local;
```

--グローバル接頭辞範囲パーティションインデックス:
```sql
CREATE INDEX idx_ind3 ON t1(c2,c3,c4)
GLOBAL PARTITION BY RANGE (c2)
(
 PARTITION t_p1 VALUES LESS THAN('20170601')
,PARTITION t_p2 VALUES LESS THAN('20170602')
,PARTITION t_p3 VALUES LESS THAN('20170603')
,PARTITION t_p4 VALUES LESS THAN('20170604')
,PARTITION t_p5 VALUES LESS THAN('20170605')
,PARTITION t_p6 VALUES LESS THAN('20170606')
,PARTITION t_p7 VALUES LESS THAN('20170607')
,PARTITION t_p8 VALUES LESS THAN('20170608')
,PARTITION t_p9 VALUES LESS THAN('20170609')
,PARTITION t_P10 VALUES LESS THAN (MAXVALUE)
);
```

--グローバル接頭辞ハッシュパーティションインデックス:
```sql
create index idx_ind4 on t1 (c1,c2,c3) global partition by hash (c1) partitions 64;  
```

--グローバル非パーティションインデックス:  
```sql
CREATE INDEX idx_ind5 ON t1(c1,c3,c4) GLOBAL;
```

--インデックスステータス確認  
```sql
col owner for a20
col index_name for a20
col index_type for a20
col table_name for a20
col partitioned for a20
set lin 300 pages 999
select OWNER,INDEX_NAME,INDEX_TYPE,TABLE_NAME,PARTITIONED
from dba_indexes
where TABLE_NAME='T1';
```
```
OWNER                INDEX_NAME           INDEX_TYPE           TABLE_NAME           PARTITIONED
-------------------- -------------------- -------------------- -------------------- ----------------
TEST                 IDX_IND1             NORMAL               T1                   YES
TEST                 IDX_IND2             NORMAL               T1                   YES
TEST                 IDX_IND3             NORMAL               T1                   YES
TEST                 IDX_IND4             NORMAL               T1                   YES
TEST                 IDX_IND5             NORMAL               T1                   NO
```
```sql
select owner,index_name,table_name,partitioning_type,
partition_count,partitioning_key_count,locality,alignment
--subpartitioning_type,def_subpartition_count,subpartitioning_key_count
from DBA_PART_INDEXES where TABLE_NAME='T1';
```
```
OWNER INDEX_NAME TABLE_NAME PARTITION PARTITION_COUNT PARTITIONING_KEY_COUNT LOCALI ALIGNMENT
------- ------------ ----------- --------- --------------- ---------------------- ------ ------------
TEST   IDX_IND1    T1         RANGE     10              1                     LOCAL  PREFIXED
TEST   IDX_IND2    T1         RANGE     10              1                     LOCAL  NON_PREFIXED
TEST   IDX_IND3    T1         RANGE     10              1                     GLOBAL PREFIXED
TEST   IDX_IND4    T1         HASH      64              1                     GLOBAL PREFIXED
```
## 参考文献:  
Example of Script to Create Local Prefixed Partitioned Index (Doc ID 165938.1)  
Script to Create Local Non-Prefixed Partitioned Index (Doc ID 166112.1)  
Example of Script to Create a Global Prefixed Partitioned Index (Doc ID 165656.1)  
Intelligent Rewrite with UNION ALL / Table Expansion in the Presence of Unusable local Index Partition or Partial Index (Doc ID 1638318.1)  
Bug 10258337 - Unusable index segment not removed for "ALTER TABLE MOVE" (Doc ID 10258337.8)  
Common Maintenance Commands That Cause Indexes to Become Unusable (Doc ID 165917.1)  
How to Drop/Truncate Multiple Partitions in Oracle 12C (Doc ID 1482264.1)  
Common Questions on Indexes on Partitioned Table (Doc ID 1481609.1)  
How to Create Partial Global/Local Indexes for Partitioned Tables in Oracle 12c (Doc ID 1482460.1)  
11.2 New Feature: Unusable Indexes and Index Partitions Do Not Consume Space (Doc ID 1266458.1)  
How to Create Primary Key Partitioned Indexes (Doc ID 74224.1)  
How Do Indexes Become Unusable? (Doc ID 1054736.6)  
https://docs.oracle.com/en/database/oracle/oracle-database/19/vldbg/index-partitioning.html  
