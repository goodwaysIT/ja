---
layout: post
title: "統計情報収集を推奨しないケース"
excerpt: "オプティマイザが最適なプランを選択するには正確な統計情報が必要ですが、統計情報の収集が困難、コストが高すぎる、またはタイムリーに完了できないシナリオも存在し、代替戦略が必要になることがあります。"
date: 2025-07-08 15:00:00 +0800
categories: [Oracle, Database]
tags: [Database maintenance, Database deployment,Database optimization, oracle]
image: /assets/images/posts/When-Not-to-Gather-Statistics.jpg
---

オプティマイザが最適なプランを選択するには正確な統計情報が必要ですが、統計情報の収集が困難、コストが高すぎる、またはタイムリーに完了できないシナリオも存在し、代替戦略が必要になることがあります。  

### 揮発性テーブル (Volatile Tables)  
揮発性テーブルとは、データ量が時間とともに劇的に変化するテーブルです。例えば、注文キュー用のテーブルは、1日の始めには空です。時間が経ち注文が発生するとテーブルにデータが溜まっていきます。各注文が処理されるとテーブルから削除されるため、1日の終わりには再び空になります。
このようなテーブルの統計情報メンテナンスを自動統計情報収集ジョブに依存している場合、ジョブが夜間に実行される時点ではテーブルが空であるため、統計情報は常にテーブルが空であることを示すことになります。しかし、日中にはテーブルに数十万行のデータが存在している可能性があります。
このようなケースでは、日中にテーブルにデータが存在する状態で代表的な統計情報セットを収集し、それをロックする方が良いでしょう。統計情報をロックすることで、自動統計情報収集タスクがそれらを上書きするのを防ぎます。あるいは、このようなテーブルでは動的サンプリング (dynamic sampling) に依存する方法もあります。オプティマイザはSQL文のコンパイル中に動的サンプリングを使用し、文を最適化する前にテーブルの基本的な統計情報を収集します。動的サンプリングで収集される統計情報は、DBMS_STATSパッケージを使用して収集される統計情報ほど高品質で完全ではありませんが、ほとんどのケースでは十分なはずです。  

### グローバル一時テーブル (Global Temporary Tables)  
グローバル一時テーブル (GTT) は、アプリケーションコンテキストで中間結果を保存するためによく使用されます。グローバル一時テーブルは、その定義を適切な権限を持つ全ユーザーとシステム全体で共有しますが、データの内容は常にセッション固有 (session-private) です。テーブルにデータが挿入されない限り、物理的な記憶域は割り当てられません。グローバル一時テーブルは、トランザクション固有（コミット時に行を削除）またはセッション固有（コミット時に行を保持）にできます。  

### 統計情報の収集  
トランザクション固有のテーブルで統計情報を収集すると、テーブルの切り捨て (truncation) が発生します。一方、行を保持するグローバル一時テーブルで統計情報を収集することは可能ですが、以前のリリースでは必ずしも良い方法ではありませんでした。GTTを使用するすべてのセッションが単一の統計情報セットを共有する必要があったため、多くのシステムは動的統計情報 (dynamic statistics) に依存していました。  
しかし、Oracle Database 18c では、GTTを使用する各セッションごとに別々の統計情報セットを持つことが可能になりました。  
GTTの統計情報共有は、新しいDBMS_STATSプリファレンス `GLOBAL_TEMP_TABLE_STATS` を使用して制御されます。  
デフォルトではこのプリファレンスは `SESSION` に設定されており、GTTにアクセスする各セッションが独自の統計情報セットを持つことを意味します。オプティマイザはまずセッション統計情報を使用しようとしますが、セッション統計情報が存在しない場合は、共有統計情報を使用します。  

***GTTの統計情報を共有しないデフォルトの動作から、統計情報の共有を強制するように変更する例：***  
```SQL
SQL> --グローバル一時テーブルの作成
SQL> Create Global Temporary table TG (col1 number);
Table created.
SQL> --TGのテーブルプリファレンスを取得
SQL> select dbms_stats.get_prefs('GLOBAL_TEMP_STATS','SH','TG') from dual;
DBMS_STATS.GET_PREFS('GLOBAL_TEMP_TABLE_STATS','SH','TG')
------------------------------------------------------------------------------------
SESSION

SQL> --TGのテーブルプリファレンスをSHAREDに変更
SQL> BEGIN
  2        dbms_stats.set_table_prefs('SH','TG','GLOBAL_TEMP_TABLE_STATS','SHARED');
  3      END;
  4      /
PL/SQL procedure successfully completed.

SQL> -- TGのテーブルプリファレンスを取得
SQL> select dbms_stats.get_prefs('GLOBAL_TEMP_STATS','SH','TG') from dual;
DBMS_STATS.GET_PREFS('GLOBAL_TEMP_TABLE_STATS','SH','TG')
------------------------------------------------------------------------------------
SHARED
```
Oracle Database 11g からアップグレードした場合、データベースアプリケーションがGTTのセッション統計情報を活用するように変更されていないならば、DBMS_STATSプリファレンス GLOBAL_TEMP_TABLE_STATS を SHARED に設定することで、アップグレード前の環境とGTTの動作を一貫させておきたいと思うかもしれません（少なくともアプリケーションが更新されるまでは）。  
ダイレクトパス操作 (direct path operation) を使用して（コミット時に行を保持する）GTTにデータを投入する場合、オンライン統計情報収集 (online statistics gathering) によりセッションレベルの統計情報が自動的に作成されます。これにより、追加の統計情報収集コマンドを実行する必要性がなくなり、他のセッションが使用する統計情報に影響を与えません。  

ダイレクトパス操作を使用してGTTにデータを投入すると、セッションレベルの統計情報が自動的に収集される例：  

```sql
SQL> Create global temporary Table SALES2(
          PROD_ID	NUMBER(6),
          CUST_ID	NUMBER,
          TIME_ID	DATE,
          CHANNEL_ID	CHAR(1),
          PROMO_ID	NUMBER(6),
          QUANTITY_SOLD NUMBER(3),
          AMOUNT_SOLD   NUMBER(10,2));
Table created.
SQL>
SQL> insert /*+ APPEND */ into sales2 select * from sales;
254720 rows created.
SQL> commit;
Commit complete.
SQL>
SQL> Select column_name,num_distinct,num_nulls
          From user_tab_col_statistics Where table_name='SALES2';
COLUMN_NAME              NUM_DISTINCT  NUM_NULLS
-------------------------     -----------------  --------------
PROD_ID                        766                    0
CUST_ID                        630                    0
TIME_ID                        620                    0
CHANNEL_ID                       5                    0
PROMO_ID                       116                    0
QUANTITY_SOLD                  44                     0
AMOUNT_SOLD                    583                    0
```

### 中間作業テーブル (Intermediate Work Tables)  
中間作業テーブルは、通常、ELTプロセスの一部や複雑なトランザクションの一部として見られます。これらのテーブルは一度だけ書き込まれ、一度読み取られた後、切り捨て (truncate) または削除されます。このようなケースでは、統計情報はたった一度しか使用されないため、統計情報を収集するコストがそのメリットを上回ってしまいます。代わりに、これらのケースでは動的サンプリングを使用すべきです。永続的な中間作業テーブルについては、自動統計情報収集タスクがそれらの統計情報を収集しようとするのを防ぐために、統計情報をロックすることをお勧めします。  
