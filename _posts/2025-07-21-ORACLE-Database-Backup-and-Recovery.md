---
layout: post
title: "ORACLE データベースのバックアップとリカバリ（マルチシナリオ）"
excerpt: "ハードウェア障害、アプリケーションロジックエラー、人的操作ミス、ウイルス、ハッカー攻撃などによるデータ損失を防止するために、バックアップの有効性を確保します。
問題発生時には、関連データを熟練かつ迅速に復旧し、データ損失リスクを最小限に抑えます。"
date: 2025-07-21 17:00:00 +0800
categories: [Oracle, Database]
tags: [データベースバックアップ, ADG, データベースリカバリ, 災害復旧, 論理レプリケーション, 物理レプリケーション, Oracle]
image: /assets/images/posts/ORACLE-Database-Backup-and-Recovery.jpg
---

## 1. 目的  
ハードウェア障害、アプリケーションロジックエラー、人的操作ミス、ウイルス、ハッカー攻撃などによるデータ損失を防止するために、バックアップの有効性を確保します。  
また、問題発生時には、関連データを熟練かつ迅速に復旧し、データ損失リスクを最小限に抑えるます。データ損失ゼロ（Recovery Point Objective, RPO=0）を目指し、データ復旧時間（Recovery Time Objective, RTO）を可能な限り短縮します。  

## 2. 一般的なORACLEデータベースのバックアップ方法  
物理バックアップ (rman/alter tablespace begin backup/フラッシュバック/コピー停止中)  
論理バックアップ (exp/expdp/sqlloader/create table as/rename table など)  

## 3. 一般的なORACLEデータベースのリカバリシナリオ  
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

| リカバリ方法         | 前提条件                          | リカバリ時間 | リカバリ複雑度 | データ完全性 |
|:------------------------|:---------------------------------------|:--------------|:--------------------|:---------------|
| rman バックアップ      | 有効な rman バックアップ              | 一般        | 一般              | 完全       |
| expdp バックアップ     | 有効な expdp バックアップ             | 一般        | 簡単                  | ほぼ完全   |
| フラッシュバック スタンバイ | スタンバイDB + フラッシュバック有効    | 最速          | 一般              | 完全       |
| 遅延スタンバイ     | スタンバイDB + 遅延適用設定           | 高速          | 一般              | 完全       |

### 3.1 ORACLEフラッシュバックを使用した高速リカバリ  
(事前に関連機能/設定の有効化が必要です。「緊急データリカバリシナリオ」参照)  
フラッシュバック非対応オブジェクトタイプ:  
```SQL
- クラスタの一部であるテーブル  
- マテリアライズド・ビュー  
- Advanced Queuing テーブル  
- 静的データディクショナリテーブル  
- システムテーブル  
- テーブルのパーティション  
- リモートテーブル (データベースリンク経由)  
```

フラッシュバック非対応 DDL 操作:  
```SQL
- ALTER TABLE ... DROP COLUMN  
- ALTER TABLE ... DROP PARTITION  
- CREATE CLUSTER  
- TRUNCATE TABLE  
- ALTER TABLE ... MOVE  
- ALTER TABLE ... ADD PARTITION  
- ALTER TABLE ... SPLIT PARTITION  
- ALTER TABLE ... DISABLE / ENABLE PRIMARY KEY  
```

公式リファレンス:  
https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/autonomous-oracle-flashback.html  
Configure flashback database (Doc ID 249319.1)  
DDL, Editions and Flashback (Doc ID 2780613.1)  
Restrictions on Flashback Table Feature (Doc ID 270535.1)  
How To Flashback Primary Database In Standby Configuration (Doc ID 728374.1)  

### 3.2 物理データベースリカバリ (「緊急データリカバリシナリオ」参照)  
RMANを使用した復元とリカバリ（ローカルまたはリモート）です。通常はデータベース全体のリカバリに使用します。リカバリ速度はバックアップネットワーク帯域幅、ディスク/テープI/Oに依存します。  

### 3.3 論理データベースリカバリ (より簡易、手順は省略)
exp/expdp/sqlloader/create table as/rename table を使用した論理バックアップから、imp/impdp/sqlloader/insert into で復元します（より柔軟）。

## 4. データリカバリの前提条件  
バックアップファイル/メディアの継続的な可用性 (ごみ箱ファイル、フル/増分/差分バックアップ、アーカイブログ、オンラインログなど)。

## 5. データベース/テーブル/削除データの高速リカバリ前提条件  
### 5.1 Oracleフラッシュバックの設定と使用  
注: Oracleフラッシュバックを使用すると、ポイントインタイムメディアリカバリなしで、データベース/オブジェクト/トランザクション/行の過去の状態を表示/復元できます。  
1）フラッシュバックデータベースの有効化 (スタンバイでのみ有効化可能)  
2）フラッシュリカバリ領域を十分な大きさに設定  
3）UNDO_RETENTIONパラメータを適切に設定 (フラッシュバッククエリ/テーブル用)  

### 5.2 ADGの設定と遅延ログ適用の有効化  
ADG遅延適用設定:
```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DELAY 5 DISCONNECT FROM SESSION; -- (DELAY 5 = 5分遅延)
```
(ここでの delay 5 は、ログを5分遅延して適用することを意味する)  

または  
log_archive_dest_nパラメータを変更し "DELAY=" を使用。例: DELAY=5 (分単位)、5分の遅延を意味する。  
```sql
ALTER SYSTEM SET log_archive_dest_2='service=standby reopen=60 lgwr async delay=5 valid_for=(online_logfiles,primary_role) db_unique_name=standby' SCOPE=BOTH;
```

## 6. データ損失の防止
1) アーカイブログモードで運用  
2) コントロールファイルの多重化  
3) 定期的なバックアップの実施  
4) 削除前のデータバックアップ (一貫性を確保。バックアップ前にアプリを切断)  
5) 全てのDB変更に対して検証済みロールバック計画を策定。実行前に開発/ビジネスチームの承認を確認。  
6) 様々なシナリオでのクロスホストリカバリ訓練を定期的に実施し、バックアップを検証し習熟度を向上。  
7) 単一テーブルの論理バックアップに Expdp/Create Table As/Rename Table を使用。  
8) 大規模変更の事前設定: ADG遅延適用、フラッシュバックデータベース（リストアポイント作成）、またはDB全体バックアップ。  

## 7. 10のバックアップとリカバリのベストプラクティス  
### 7.1 ブロックチェックの有効化  
最小限のパフォーマンスオーバーヘッドでデータベースブロック破損を早期検出。  
```SQL
SQL> ALTER SYSTEM SET db_block_checking = TRUE SCOPE=BOTH;
```

### 7.2 RMAN増分バックアップのブロック変更追跡を有効化 (10g+)  
変更されたブロックを追跡し、増分バックアップ時に変更されていないデータの読み取りを回避。  
```SQL
SQL> ALTER DATABASE ENABLE BLOCK CHANGE TRACKING USING FILE '/u01/oradata/ora1/change_tracking.f';
```

### 7.3 リドロググループ/メンバーのミラー化 & アーカイブログの複数場所保存  
ログ損失/破損時の冗長性を確保。  
```SQL
SQL> ALTER SYSTEM SET log_archive_dest_2='location=/new/location/archive2' SCOPE=BOTH;
SQL> ALTER DATABASE ADD LOGFILE MEMBER '/new/location/redo21.log' TO GROUP 1;
```

### 7.4 RMANバックアップで "CHECK LOGICAL" を使用  
物理チェックサムを超えたデータブロックの論理破損をチェック。  
```SQL
RMAN> BACKUP CHECK LOGICAL DATABASE PLUS ARCHIVELOG DELETE INPUT;
```

### 7.5 バックアップのテスト  
復元せずにバックアップの整合性を検証。  
```SQL
RMAN> RESTORE VALIDATE DATABASE;
```

### 7.6 各データファイルを別々のバックアップピースに保存 (RMAN)  
部分復元時、RMANは必要なデータファイル/アーカイブログを取得するためにバックアップピース全体を読み取る必要があります。したがって、バックアップピースが小さいほど、復元は高速化されます。特に大規模データベースのテープバックアップや単一/少数ファイルの復元時に有効です。  
しかし、filespersetの値が小さいと作成されるバックアップピースが増え、バックアップ性能が低下しメンテナンス時間が増加します。これらの要因と復元時間要件を考慮する必要があります。  
```SQL
RMAN> BACKUP DATABASE FILESPERSET 1 PLUS ARCHIVELOG DELETE INPUT;
```

### 7.7 RMANカタログ/コントロールファイルのメンテナンス
**保持ポリシーを慎重に選択。**  
テープ保持ポリシーおよびバックアップリカバリポリシーを満たすようにします。カタログを使用しない場合、CONTROL_FILE_RECORD_KEEP_TIMEパラメータが保持ポリシーと一致することを確認します。
```SQL
SQL> ALTER SYSTEM SET control_file_record_keep_time=21 SCOPE=BOTH; -- レコードを21日間保持
```
これにより、バックアップレコードがコントロールファイルに21日間保持されます。  
詳細は以下を参照:  
Note 461125.1 How to ensure that backup metadata is retained in the controlfile when setting a retention policy and an RMAN catalog is NOT used.  

**以下のカタログメンテナンスコマンドを定期的に実行する必要があります。**  
理由: Delete Obsolete は保持ポリシー外のバックアップを削除します。  
期限切れバックアップを削除しないと、カタログが肥大化しパフォーマンス問題が発生します。  
```SQL
RMAN> DELETE OBSOLETE;
```

理由: Crosscheck はカタログ/コントロールファイルと物理バックアップの一致を確認します。  
バックアップが失われた場合、このコマンドはバックアップピースを "EXPIRED" に設定する。リカバリ開始時にこのバックアップは使用されず、より古いバックアップが使用されます。カタログ/コントロールファイル内の期限切れバックアップを削除するには Delete Expired コマンドを使用します。  
```SQL
RMAN> CROSSCHECK BACKUP;
RMAN> DELETE EXPIRED BACKUP;
```

### 7.8 コントロールファイル損失の防止  
現在のバックアップ中ではなく、バックアップ終了時にコントロールファイルバックアップを実行し、常に最新のコントロールファイルを確保します。  
```SQL
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
```

### 7.9 リカバリのテスト  
理由: 実際のリカバリを実行せずにリカバリプロセスを学習でき、データファイルの再復元を回避できます。  
```SQL
SQL> RECOVER DATABASE TEST;
```

### 7.10 アーカイブログバックアップでの "DELETE ALL INPUT" の回避  
理由: "delete all input" は1つのアーカイブディレクトリのアーカイブログをバックアップ後、異なるアーカイブディレクトリのすべてのアーカイブログコピーを削除します。一方 "delete input" は、バックアップされたディレクトリ内のアーカイブログのみを削除します。次回のバックアップではアーカイブディレクトリ2のログとアーカイブディレクトリ1の新しいログをバックアップし、バックアップ済みログをすべて削除します。これにより、最後のバックアップ以降のアーカイブログ（バックアップ済みログを含む）がアーカイブディレクトリ2に保持され、最後のバックアップ前の2つのコピーが維持されます。  
```SQL
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT; -- 正しいアプローチ
```
参照 MOS: Top 10 Backup and Recovery Best Practices (Doc ID 388422.1)  

## 8. 緊急データリカバリシナリオ  
**定期的なクロスホストリカバリ訓練を推奨!**  
### 8.1 ブロック破損の検出とリカバリ  
rman-validate コマンドに "check logical" 句を付けて、物理的および論理的な破損についてデータベースを検証します。  
以下の例は全データファイルの検証方法:  
```sql
$ rman target / nocatalog
```
データベース全体チェック:  
```SQL
RMAN> RUN {
       ALLOCATE CHANNEL d1 TYPE DISK;
       BACKUP CHECK LOGICAL VALIDATE DATABASE;
       RELEASE CHANNEL d1;
     }
```

特定のデータファイルチェック:  
```sql
RMAN> RUN {
       ALLOCATE CHANNEL d1 TYPE DISK;
       BACKUP CHECK LOGICAL VALIDATE DATAFILE 1;
       RELEASE CHANNEL d1;
     }
```

破損ブロックの表示:  
```sql
SQL> SELECT * FROM V$DATABASE_BLOCK_CORRUPTION;
```

V$DATABASE_BLOCK_CORRUPTIONビューで見つかった影響を受ける全オブジェクトをチェック:  
```sql
RMAN> RUN {
      ALLOCATE CHANNEL d1 TYPE DISK;
      BLOCKRECOVER CORRUPTION LIST;
      RELEASE CHANNEL d1;
     }
```

または Data Recovery Advisor (DRA) を使用:  
```sql
RMAN> VALIDATE CHECK LOGICAL DATABASE;
RMAN> LIST FAILURE;
RMAN> ADVISE FAILURE;
RMAN> REPAIR FAILURE PREVIEW;
RMAN> REPAIR FAILURE NOPROMPT;
```
参照: Quick guide RMAN corrupt block recover steps (Doc ID 1428823.1)  

### 8.2 DDL(Drop Table)/DML(Insert/Delete/Update) 単一テーブルリカバリ (非パーティション表/単一パーティション)  
**注: undo_retention: undoデータの有効期限を表します。システムデフォルトは900で、15分に相当します。**  
**しかし、この時間内にundoデータが有効であることを保証する前提は、undo表領域に十分な空きがあることです。**  
**undo領域が満杯で新規トランザクションが実行されると、undoデータの有効期限に関係なく上書きされます。**  
**undo領域に余裕があれば、指定時間を超えても上書きされていない限りundoデータは残存するため、フラッシュバック可能です（しかし、テーブルレコードの追加/削除/変更前に alter table table_name enable row movement を実行する必要があります。つまり行移動を許可しないと、指定時間後、上書きされていなくてもフラッシュバック不可となり ora-01466 エラーが発生します）。**  

#### 1) 削除行の復元
a) 現在のセッションの時間フォーマットを変更
```sql
ALTER SESSION SET nls_date_format = 'yyyy-mm-dd hh24:mi:ss';
```

b) 現在のシステム時間を確認  
```sql
SELECT SYSDATE FROM DUAL;
```

c) フラッシュバッククエリで誤削除前のテーブルデータを確認  
```sql
SELECT * FROM test AS OF TIMESTAMP TO_TIMESTAMP('2018-02-15 13:38:05','yyyy-mm-dd hh24:mi:ss');
```

d) フラッシュバッククエリを使用したデータ復元  
```sql
-- オプション1: バックアップテーブル作成
CREATE TABLE test_old AS
SELECT * FROM test AS OF TIMESTAMP TO_TIMESTAMP('2018-02-15 13:38:05','yyyy-mm-dd hh24:mi:ss');

-- オプション2: 削除行を挿入戻し
INSERT INTO test
SELECT * FROM test AS OF TIMESTAMP TO_TIMESTAMP('2018-02-15 13:38:05','yyyy-mm-dd hh24:mi:ss');

-- オプション3: フラッシュバックテーブル (行移動要)
ALTER TABLE test ENABLE ROW MOVEMENT;
FLASHBACK TABLE test TO SCN 11111;
FLASHBACK TABLE test TO TIMESTAMP TO_TIMESTAMP('2013/06/23 19:17:00','yyyy/mm/dd hh24:mi:ss');
```

さらに、フラッシュバックバージョンクエリを使用すると、過去の一定期間におけるレコードの変更履歴を確認できます。  
```sql
col versions_xid for a16 heading 'XID'
col versions_startscn for 99999999 heading 'Vsn|Start|Scn'
col versions_endscn for 99999999 heading 'Vsn|End|Scn'
col versions_operation for a12 heading 'Operation'
select versions_xid,versions_startscn,versions_endscn,
decode(
versions_operation,
'I','Insert',
'U','Update',
'D','Delete','Original') Operation,
id,name
from test2
VERSIONS BETWEEN SCN MINVALUE AND MAXVALUE
where id=1;
```

フラッシュバックトランザクションクエリ機能を使用して、トランザクションによる全変更を確認することも可能です。  
同じXIDは同じトランザクションを示します。  
```sql
select xid,operation,commit_scn,undo_sql
from flashback_transaction_query
where xid in (
select  versions_xid
from test4
versions between scn minvalue and maxvalue
);
```

#### 2) 削除テーブルの復元 (非パーティション)  
```SQL
flashback table test to before drop;
select * from test;
```

## 8.3 Truncate Table/Drop Table パーティションリカバリ (論理バックアップなし)
#### 1) 優先方法: ADGスタンバイ側で事前に遅延適用を設定し、誤削除から復元します。  
ADG遅延適用設定方法:  
```SQL
alter database recover managed standby database delay 5 disconnect from session;（delay 5 は5分遅延してログ適用）
```  

または  
log_archive_dest_nパラメータを変更し "DELAY=" を使用します。例: DELAY=5 (分)、5分遅延を意味します。  
```SQL
alter system set log_archive_dest_2='service=standby reopen=60 lgwr async delay=5 valid_for=(online_logfiles,primary_role) db_unique_name=standby' scope=both;
```

ADG遅延スタンバイ機能によるデータ復元します。遅延スタンバイはスタンバイログ適用の遅延を意図的に設定し、プライマリDBの操作が即座にスタンバイに適用されるのを防ぎます。これにより、プライマリDBでの誤データ削除時、遅延時間内はスタンバイのデータが削除されないことを保証し、遅延スタンバイDBからデータを復元できます。  
具体的な操作手順:  
a) スタンバイDBのログ適用を即時キャンセル  
```sql
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
```

b) プライマリでログスイッチ:  
```SQL
SQL> ALTER SYSTEM SWITCH LOGFILE;
```

c) スタンバイをマウント:  
```SQL
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
```

d) 誤truncate前の時間点にスタンバイをリカバリ:  
```SQL
RUN {
  SET UNTIL TIME "to_date('2023-10-11 16:45:00','yyyy-mm-dd hh24:mi:ss')";
  RECOVER DATABASE;
}
```

e) スタンバイを読み取り専用でオープン & データ確認:  
```SQL
SQL> ALTER DATABASE OPEN READ ONLY;
SQL> SELECT COUNT(*) FROM test.testdebug; -- データ存在を確認
```

f) スナップショットスタンバイモードでスタンバイDBをオープンしテーブルデータをエクスポート  
```
SQL> ALTER DATABASE CONVERT TO SNAPSHOT STANDBY;
SQL> ALTER DATABASE OPEN;
$ expdp \'/ AS SYSDBA\' DUMPFILE=test.dmp TABLES=test.testdebug DIRECTORY=expdir LOGFILE=test.log
```

g) 物理スタンバイDBに切り替え同期を復元  
```SQL
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
SQL> ALTER DATABASE CONVERT TO PHYSICAL STANDBY;
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP;
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DELAY 120 DISCONNECT FROM SESSION;
```

#### 2) スタンバイデータベースのフラッシュバック (事前にフラッシュバック有効化要)
フラッシュバックでデータベース全体を誤削除前の時点に戻します。この方法の欠点は明らか：データベース全体のデータを同時に戻すため、本番プライマリ環境ではほぼ操作不可能です（1テーブルのために全DBデータを戻せない）。したがって、この方法はバックアップ環境に適する。スタンバイDBでフラッシュバック機能が有効な場合、スタンバイDBでフラッシュバック復元を選択可能です。  
具体的な操作手順:  
a) 最古フラッシュバック時間を確認:  
```SQL
SQL> SELECT * FROM V$FLASHBACK_DATABASE_LOG;
```

b) スタンバイをマウント:  
```SQL
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
```

c) 誤truncate前の時間点にフラッシュバック:  
```SQL
SQL> FLASHBACK DATABASE TO TIMESTAMP TO_TIMESTAMP('2023-10-11 17:45:00','yyyy-mm-dd hh24:mi:ss');
```

d) データベースを読み取り専用でオープンし復元データを確認  
```SQL
SQL> alter database open read only;
SQL> select count(*) from test.testdebug;

  COUNT(*)
----------
    172864
```

e) スナップショットスタンバイモードでスタンバイDBをオープンしテーブルデータをエクスポート  
```SQL
SQL> alter database convert to snapshot standby;
SQL> Alter database open;
```

--expdpでテーブルデータエクスポート  
```SQL
expdp 'userid="/ as sysdba"' dumpfile=test.dmp tables=test.testdebug directory=expdir logfile=test.log
```

f) 物理スタンバイDBに切り替え同期を復元  
```SQL
SQL> shutdown immediate;
SQL> startup mount ;
SQL> alter database convert to physical standby;
```

--再同期  
```SQL
SQL> shutdown immediate;
SQL> startup;
SQL> alter database recover managed standby database using current logfile  disconnect from session;
```

#### 3) RECOVER TABLE (Oracle 12c+ 機能)  
手順:  
テーブル/パーティションを含むバックアップを識別。  
補助領域をチェック。  
リカバリ用補助インスタンスを作成。  
Data Pumpエクスポートダンプを生成。  
(オプション) 復元オブジェクトをインポート。  
(オプション) インポート時にオブジェクト名を変更。  

**例1: 異なるマシンで recover table コマンドを実行し、EBSDB PDB内のtest_partテーブルパーティションp20を復元**  
--- until time 復元時点  
---AUXILIARY DESTINATION 補助インスタンスデータファイルパス  
---DATAPUMP DESTINATION エクスポートdmpファイルパス  
---DATAPUMP FILE エクスポートdmpファイル名  
---NOTABLEIMPORT ターゲットDBにインポートしない  
```sql
rman target sys/oracle@racpdg log /tmp/recover_table.log
RECOVER TABLE "TEST"."TEST_PART":"P_20" OF PLUGGABLE DATABASE ebsdb
UNTIL TIME "to_date('11/09/2022 15:18:11','MM/DD/YYYY HH24:MI:SS')"
AUXILIARY DESTINATION '/u01/app/oracle/auxinstance'
DATAPUMP DESTINATION '/tmp'
DUMP FILE 'test_part_p20.dmp'
NOTABLEIMPORT;
```

**例2: 復元データを直接ターゲットDBにインポート (最速の復元)**  
```sql
RECOVER TABLE "TEST"."TEST_PART":"P_20" OF PLUGGABLE DATABASE ebsdb
UNTIL TIME "to_date('11/09/2022 15:18:11','MM/DD/YYYY HH24:MI:SS')"
AUXILIARY DESTINATION '/u01/app/oracle/auxinstance';
```
---インポート操作を実行。ターゲットDBにインポートされるテーブル名は table名_パーティション名  
```sql
Performing import of tables...
   IMPDP> Master table "SYS"."TSPITR_IMP_lAng_oigD" successfully loaded/unloaded
   IMPDP> Starting "SYS"."TSPITR_IMP_lAng_oigD":
   IMPDP> Processing object type TABLE_EXPORT/TABLE/TABLE
   IMPDP> Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
   IMPDP> . . imported "TEST"."TEST_PART_P_20"                     5.484 KB       1 rows
   IMPDP> Job "SYS"."TSPITR_IMP_lAng_oigD" successfully completed at Sun Nov 13 15:15:50 2022 elapsed 0 00:00:06
Import completed
```

---テーブル空間内の他のテーブルデータに影響なし  
```sql
SQL> select * from "TEST"."TEST_PART_P_20"  ;

        ID INSERTDATE
---------- -------------------
        11 2022-11-09 15:15:25

SQL>  select * from "TEST".test_recover;

INSERTDATE
-------------------
2022-11-09 15:23:08

SQL>
```

**例3: リマップを使用した複数パーティションリカバリ**  
ログシーケンス番号に基づき複数パーティションテーブルを復元し、復元後のテーブル空間とテーブル名をリマップ  
```sql
RECOVER TABLE SH.SALES:SALES_1998, SH.SALES:SALES_1999
    UNTIL SEQUENCE 354
    AUXILIARY DESTINATION '/tmp/oracle/recover'
    REMAP TABLE 'SH'.'SALES':'SALES_1998':'HISTORIC_SALES_1998',
              'SH'.'SALES':'SALES_1999':'HISTORIC_SALES_1999'
    REMAP TABLESPACE 'SALES_TS':'SALES_PRE_2000_TS';
```

**例4: スキーマリマップを使用した複数テーブルリカバリ**  
複数テーブルを復元し、復元後のユーザーとテーブル名をリマップ   
```sql
RECOVER TABLE HR.DEPARTMENTS, SH.CHANNELS
UNTIL TIME 'SYSDATE – 1'
AUXILIARY DESTINATION '/tmp/auxdest'
REMAP TABLE hr.departments:example.new_departments, sh.channels:example.new_channels;
```
