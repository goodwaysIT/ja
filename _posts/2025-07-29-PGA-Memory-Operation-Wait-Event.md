---
layout: post
title: "PGAメモリ操作待機イベント：異なるテーブルデータ量による事象"
excerpt: "SQL実行がACTIVE状態で滞留（SQLにCONNECT BY/regexp_substrを含む）。待機時間がPGAメモリ操作に起因していた。"
date: 2025-07-29 15:00:00 +0800
categories: [Oracle, Database]
tags: [PGA Memory Operation, Wait Event, High Waits, oracle]
image: /assets/images/posts/PGA-Memory-Operation-Wait-Event.jpg
---

## 事象説明
SQL実行がACTIVE状態で滞留（SQLにCONNECT BY / regexp_substrを含む）。待機時間がPGAメモリ操作に起因していた。  
このSQLはテスト環境（Oracle 19.3）では正常に実行されたが、本番環境（Oracle 19.7）では実行に失敗した。  

## SQLHCレポート

```
Description
--------------
SQLHC for c9333k2g237qx

Relevant Information Collection
---------------------------------------
SQL Text
SELECT DISTINCT REGEXP_SUBSTR(T.TASK_FRORGS, '[^,]+', 1, LEVEL) AS AA FROM EARLY_CONFIG_SP_TASK T
WHERE T.TASK_START_DATE = :B3
AND T.TASK_FRORGS = :B2
AND T.TASK_BATCH_ID = :B1
CONNECT BY REGEXP_SUBSTR(T.TASK_FRORGS, '[^,]+', 1, LEVEL) IS NOT NULL
AND PRIOR T.TASK_FRORGS = T.TASK_FRORGS
AND PRIOR SYS_GUID() IS NOT NULL

Current Plans Summary (GV$SQL)

Execution Plans performance metrics for c9333k2g237qx while still in memory. Plans ordered by average elapsed time.
# Plan HV Avg Avg Avg Avg Avg Avg Avg Avg Avg Avg Avg Avg Total Total Total Total Total Total Min Max Min Max First Load Last Load Last Active
Elapsed CPU IO Conc Appl Clus PLSQL Java Buffer Disk Direct Rows Execs Fetch Loads Inval Parse Child Cost Cost Opt Env HV Opt Env HV
Time Time Time Time Time Time Time Time Gets Reads Writes Proc Calls Cursors
(secs) (secs) (secs) (secs) (secs) (secs) (secs) (secs)
1 753884331 116074.806 116019.226 0 0 0 0 0 0 6 0 0 0 1 1 1 0 1 1 4 3228039672 2024-06-23/03:00:00 2024-06-23/03:00:00 2024-06-24/11:14:34

Active Session History by Plan Line (GV$ACTIVE_SESSION_HISTORY)

Snapshots counts per Plan Line and Wait Event for c9333k2g237qx.
This section includes data captured by AWR.
Available on 11g or higher..
# Plan Plan Plan Plan Plan Plan Session Wait Event Curr Curr Snaps
Hash Line Operation Options Object Object State Class Obj Object Count
Value ID Owner Name ID Name
1 753884331 2 FILTER ON CPU 2
2 753884331 3 CONNECT BY WITHOUT FILTERING (UNIQUE) ON CPU 110905. 《----- Top count
```

## 追加分析手順
SQLID c9333k2g237qx のSQLHC情報分析結果：当該SQLは長時間実行されていたが、時間はCPU Timeに費やされ、長期間のPGAメモリ操作待機イベントは確認されなかった。  
問題は実行計画、テーブルサイズ、統計情報、渡されるバインディング変数の差異にある可能性がある。テスト環境での実行状況比較が必要。以下情報を提供ください。  

1. テスト環境（Oracle 19.3）で当該SQLのSQLHCを収集  
2. テスト環境と本番環境の両方で、当該SQLの10046トレースと10053トレースを収集  
```SQL
SQL> connect executing_user
SQL> alter session set max_dump_file_size = UNLIMITED;
SQL> alter session set timed_statistics=true;
SQL> alter session set events '10053 trace name context forever, level 2';
SQL> alter session set events '10046 trace name context forever, level 12';
```

事象再現SQLを実行  
実行時間が非常に長い場合、10～20分後に強制中断  
```SQL
SQL> alter session set events '10046 trace name context off';
SQL> alter session set events '10053 trace name context off';
```

トレースファイルを以下で特定  
```SQL
SQL> select * from v$diag_info where NAME in ('Default Trace File', 'Diag Trace')
```

3. SQL実行時間帯の本番環境DBアラートログ

4. テスト環境と本番環境で以下クエリを実行し結果を提供
```SQL
spool /tmp/test or product.html
set markup html on
set time on
alter session set nls_date_format='dd-mon-yyyy hh24:mi:ss';
SELECT * FROM DBA_TAB_STATISTICS where TABLE_NAME='EARLY_CONFIG_SP_TASK';
SELECT * FROM DBA_IND_STATISTICS where TABLE_NAME='EARLY_CONFIG_SP_TASK';
SELECT * FROM DBA_TAB_COL_STATISTICS where TABLE_NAME='EARLY_CONFIG_SP_TASK';
SELECT * FROM DBA_OPTSTAT_OPERATIONS where START_TIME >=to_date('2024-06-17 00:00:00','yyyy-mm-dd hh24:mi:ss') and END_TIME <= to_date('2024-06-24 01:00:00','yyyy-mm-dd hh24:mi:ss') order by START_TIME;
SELECT * FROM DBA_TAB_STATS_HISTORY where TABLE_NAME='EARLY_CONFIG_SP_TASK';
set markup html off
spool off
```

## 問題概要
追加分析で判明：テスト環境(19.3)と本番環境(19.7)でデータ量が異なっていた。  
同一データ量ではテスト環境でもSQL実行失敗。開発側がSQL修正（新たなフィルタ条件追加）後、正常動作した。  
