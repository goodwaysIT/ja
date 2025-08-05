---
layout: post
title: "ORA-04030: Oracle GoldenGate使用時のプロセスメモリ不足エラー（<nn>バイトの割り当て失敗）"
excerpt: "この問題は、SYSTEM.LOGMNR_LOG$ テーブルに大量のエントリが存在することが原因です。SYSTEM.LOGMNR_LOG$ テーブルは、Oracle GoldenGate Extract プロセスおよび Streams Capture プロセスに必要なアーカイブ済みログファイルを格納しています。"
date: 2025-08-04 15:00:00 +0800
categories: [Oracle, Database, GoldenGate]
tags: [ORA-04030, プロセスメモリ不足, SYSTEM.LOGMNR_LOG$, GoldenGate, oracle]
image: /assets/images/posts/ORA-04030-Out-Of-Process-Memory-When-Using-Oracle-GoldenGate.jpg
---

## 現象  
データベースアラートログに以下のメモリエラーが記録されます:  
```
Errors in file /u01/app/oracle/diag/rdbms/oggdb/oggdb1/trace/oggdb1_ppa7_120312.trc (incident=52058):
ORA-04030: out of process memory when trying to allocate bytes (,)
Use ADRCI or Support Workbench to package the incident.
See Note 411.1 at My Oracle Support for error and packaging details.
2024-08-29T05:58:18.611120+08:00
Errors in file /u01/app/oracle/diag/rdbms/oggdb/oggdb1/trace/oggdb1_ora_67839.trc (incident=52033):
ORA-04030: out of process memory when trying to allocate 278587224 bytes (kxs-heap-w,krvxlogact)
Use ADRCI or Support Workbench to package the incident.
See Note 411.1 at My Oracle Support for error and packaging details.
```

データベーストレースファイルにも以下の内容が出力されます:  
```
*** 2024-08-28T15:58:23.231573+08:00
*** SESSION ID:(291.42918) 2024-08-28T15:58:23.231582+08:00
*** CLIENT ID:() 2024-08-28T15:58:23.231587+08:00
*** SERVICE NAME:(SYS$USERS) 2024-08-28T15:58:23.231591+08:00
*** MODULE NAME:(emagent_SQL_rac_database) 2024-08-28T15:58:23.231595+08:00
*** ACTION NAME:(streams_latency_throughput) 2024-08-28T15:58:23.231600+08:00
*** CLIENT DRIVER:(jdbcthin) 2024-08-28T15:58:23.231604+08:00

[TOC00000]
Jump to table of contents
Dump continued from file: /u01/app/oracle/diag/rdbms/oggdb/oggdb1/trace/oggdb1_ora_67839.trc
[TOC00001]
ORA-04030: out of process memory when trying to allocate 278615672 bytes (kxs-heap-w,krvxlogact)

[TOC00001-END]
[TOC00002]
========= Dump for incident 51807 (ORA 4030) ========

*** 2024-08-28T15:58:23.231929+08:00
dbkedDefDump(): Starting incident default dumps (flags=0x2, level=3, mask=0x0)
[TOC00003]
----- Current SQL Statement for this session (sql_id=7sy4dzsndmh0k) -----
SELECT streams_name, streams_type, streams_latency, total_messages
FROM (
select capture_name streams_name, 'capture' streams_type , (available_message_create_time-capture_message_create_time)*86400 streams_latency, nvl(total_messages_enqueued,0) total_messages
from gv$streams_capture
union all
:
WHERE apc.apply_name = apr.apply_name AND apr.apply_name = aps.apply_name
) WHERE EXISTS
(SELECT 1 FROM v$database WHERE database_role IN ('PRIMARY', 'LOGICAL STANDBY'))
[TOC00003-END]

[TOC00004]
ksedst <- dbkedDefDump <- ksedmp <- dbgexPhaseII <- dbgexProcessError
<- dbgePostErrorKGE <- 1767 <- dbkePostKGE_kgsf <- kgereml <- kxfpProcessError
<- 1786 <- kxfpProcessMsg <- kxfpqidqr <- kxfpqdqr <- kxfxgs
<- kxfxcw <- qerpxFetch <- rwsfcd <- rwsfcd <- qeruaFetch
<- qervwFetch <- qerflFetch <- opifch2 <- kpoal8 <- opiodr
<- ttcpip <- opitsk <- opiino <- opiodr <- opidrv

[TOC00004-END]

```

## 変更内容  
Oracle Grid Control経由でGoldenGateまたはStreams監視機能を使用しています。  

## 原因  
問題は、SYSTEM.LOGMNR_LOG$ テーブルに大量のエントリが存在することによって引き起こされます。
SYSTEM.LOGMNR_LOG$は、Oracle GoldenGate ExtractプロセスおよびStreams Captureプロセスに必要なアーカイブログファイルを格納します。    
何らかの理由でアーカイブログファイルがシステムから正しく削除されない場合、このテーブルは増大し、このテーブルを使用するビュー（例えば dba_capture、gv$streams_capture、gv$xstream_capture、gv$goldengate_capture）をクエリする際にエラーが発生する可能性があります。
本ケースでは、Grid Control DBエージェントがGG/Streamsレイテンシスループット統計の一部としてこのクエリを発行しています:  

```
SELECT streams_name, streams_type, streams_latency, total_messages
FROM (
select capture_name streams_name, 'capture' streams_type , (available_message_create_time-capture_message_create_time)*86400 streams_latency, nvl(total_messages_enqueued,0) total_messages
from gv$streams_capture
union all
```

## 解決策  
重要: データベースが 19.17DBRU 以下、または 21c リリースである場合、既知の内部バグ 34115836 の修正が必要です。これが適用されていないと、パージ処理が正しく機能しない可能性があります。このバグ修正は、19.18DBRU および 23.1 以降に含まれています。詳細は Doc ID 34115836.8 を参照してください。  
GoldenGateまたはStreamsによる使用後にアーカイブログファイルが適切にパージされない原因は複数考えられます。以下は両方の問題（アーカイブログファイルのパージとORA-04030エラー）を解決する一般的なアプローチです:  
### 1.以下のクエリを使用して、各GoldenGate Extract（またはStreams Capture）プロセスに関連付けられたアーカイブログファイルの数を確認します:  

```SQL
sqlplus / as sysdba

spool support1.out
col CONSUMER_NAME for a30
set line 200
alter session set nls_date_format = 'dd/mon/rrrr hh24:mi:ss';
select min(FIRST_TIME) from SYSTEM.LOGMNR_LOG$ ;
select max(FIRST_TIME) from SYSTEM.LOGMNR_LOG$ ;
select count(*) from SYSTEM.LOGMNR_LOG$ ;
select count(*), PURGEABLE from dba_registered_archived_log group by PURGEABLE;
select count(*), PURGEABLE , CONSUMER_NAME from dba_registered_archived_log group by PURGEABLE , CONSUMER_NAME order by 2,1;
select count(*), status from v$archived_log group by status ;
select capture_name, status, to_char(APPLIED_SCN), to_char(REQUIRED_CHECKPOINT_SCN) from dba_capture;
exit
```

```
SYS@oggdb1> col CONSUMER_NAME for a30
SYS@oggdb1> set line 200
SYS@oggdb1> alter session set nls_date_format='yyyy-mm-dd hh24:mi:ss';

Session altered.

SYS@oggdb1> select min(first_time),max(first_time) from system.logmnr_log$;

MIN(FIRST_TIME) MAX(FIRST_TIME)
------------------- -------------------
2022-06-21 08:01:45 2024-09-05 15:54:37

SYS@oggdb1> select count(*) from system.logmnr_log$;

COUNT(*)
----------
274717

SYS@oggdb1> select count(*),purgeable from dba_registered_archived_log group by purgeable;

COUNT(*) PUR
---------- ---
1309 NO
273409 YES

SYS@oggdb1> select count(*),purgeable,consumer_name from dba_registered_archived_log group by purgeable,consumer_name order by 2,1;

COUNT(*) PUR CONSUMER_NAME
---------- --- ------------------------------
1309 NO OGG$CAP_IRDCB
273409 YES OGG$CAP_IRDCB

SYS@oggdb1> select count(*),status from v$archived_log group by status;

COUNT(*) S
---------- -
36237 D
131 A

SYS@oggdb1> select capture_name,status,to_char(applied_scn),to_char(required_checkpoint_scn) from dba_capture;

CAPTURE_NAME STATUS TO_CHAR(APPLIED_SCN)
-------------------------------------------------------------------------------------------------------------------------------- -------- ----------------------------------------
TO_CHAR(REQUIRED_CHECKPOINT_SCN)
----------------------------------------
OGG$CAP_IRDCB ENABLED 22465368276
22465368276

SYS@oggdb1> spool off
```

### 2.対象データベース上のすべてのGoldenGate ExtractまたはStreams Captureプロセスを停止します。  

### 3.実際に存在するファイルのみを再カタログ化するためにRMANの"crosscheckコマンド"を使用します:  
注: RMANがバックアップツールでなくても、ディスク上の実際のファイルとデータベース制御ファイルを再同期させるために以下を実行する必要があります。  
例:  

```SQL
rman target / log /tmp/crosscheck.log
rman> crosscheck archivelog all;
text
crosscheck.log
-------------------------------------
Recovery Manager: Release 12.2.0.1.0 - Production on Thu Sep 12 08:52:46 2024

Copyright (c) 1982, 2017, Oracle and/or its affiliates. All rights reserved.

RMAN>
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=219 instance=oggdb1 device type=DISK
validation succeeded for archived log
archived log file name=+RECODG1/oggdb/ARCHIVELOG/2024_09_11/thread_1_seq_562177.719.1179394927 RECID=1014710 STAMP=1179394929
：
archived log file name=+RECODG1/oggdb/ARCHIVELOG/2024_09_12/thread_2_seq_452465.816.1179478241 RECID=1014875 STAMP=1179478242
Crosschecked 168 objects
RMAN>
Recovery Manager complete.
```

### 4.SQL*Plus経由で、対象のGoldenGate ExtractまたはStreams Captureに関連付けられたキャプチャコンポーネントに対してパラメータ「_CHECKPOINT_FORCE」を設定します:  
注: 複数のExtract/Captureが存在する場合、通常はステップ1のクエリでエントリ数が多いプロセス1つでこのタスクを完了できます:  
```SQL
conn / as sysdba
exec dbms_capture_adm.set_parameter('<capture_name>','_CHECKPOINT_FORCE','Y');
```

### 5.次のクエリから取得した APPLIED_SCN と REQUIRED_CHECKPOINT_SCN のうち、最低値を該当する Extract／Capture プロセスの FIRST_SCN に設定します:  
```SQL
select capture_name, status, to_char(APPLIED_SCN), to_char(REQUIRED_CHECKPOINT_SCN) from dba_capture where capture_name = '<capture_name>';
```
注: これは安全な操作であり、以下のプロシージャは正常に完了する必要があります。  

```SQL
conn / as sysdba
BEGIN
DBMS_CAPTURE_ADM.ALTER_CAPTURE(
capture_name => '<capture_name>',
first_scn => &SCN );
END;
/
```
ここで&SCNは、前のクエリのAPPLIED_SCNとREQUIRED_CHECKPOINT_SCN列のうち小さい方のSCN値でなければなりません。  

### 6.GoldenGate Extract / Streams Captureプロセスを正常に再起動します。  

### 7.環境を監視します。基盤となるクリーンアップが完了するまでに1時間程度かかる場合があります。ステップ1と同じクエリを使用してクリーンアップの進捗を監視できます。  
SYSTEM.LOGMNR_LOG$の行数が減少すれば、ORA-04030は発生しなくなります。

### 現在OGGプロセスを停止できない場合、メンテナンス期間中にこの問題に対処することを検討してください。推奨される解決策は以下の通りです:

1. OGG抽出プロセスの再作成  
OGG抽出プロセスを再作成すると、「PURGEABLE」アーカイブログが解放されます。  

2. 蓄積されたアーカイブログの手動クリア（この方法は理論的に導出されたものです。削除前に完全バックアップを必ず実行してください！）:  
1). すべての抽出プロセスを停止します。  
2). SYSTEM.LOGMNR_LOG$テーブルをバックアップします。  
3). パージ可能（purgeableが「YES」）なすべてのアーカイブログを手動で削除します。  

```SQL
DELETE FROM SYSTEM.LOGMNR_LOG$ WHERE (THREAD#, SEQUENCE#, RESETLOGS_CHANGE#, RESET_TIMESTAMP) IN
(SELECT DISTINCT P.THREAD#, P.SEQUENCE#, P.RESETLOGS_CHANGE#, P.RESET_TIMESTAMP AS RESETLOGS_ID FROM SYSTEM.LOGMNR_LOG$ P WHERE BITAND(P.STATUS, 2) = 2 MINUS SELECT DISTINCT Q.THREAD#, Q.SEQUENCE#, Q.RESETLOGS_CHANGE#, Q.RESET_TIMESTAMP AS RESETLOGS_ID FROM SYSTEM.LOGMNR_LOG$ Q WHERE BITAND(Q.STATUS, 2) <> 2) ;
```

