---
layout: post
title: "異なるテーブルデータ量による PGA メモリ操作の待機イベント"
excerpt: "CONNECT BY／regexp_substrを含むSQL文の実行が、PGAメモリ操作による待機時間でACTIVE状態で滞留しました。"
date: 2025-07-29 15:00:00 +0800
categories: [Oracle, Database]
tags: [PGA Memory Operation, Wait Event, High Waits, oracle]
image: /assets/images/posts/PGA-Memory-Operation-Wait-Event.jpg
---

## イベントの説明
CONNECT BY／regexp_substrを含むSQL文の実行が、PGAメモリ操作による待機時間でACTIVE状態で滞留しました。。  
このSQLはテスト環境（Oracle 19.3）では正常に実行されましたが、本番環境（Oracle 19.7）では実行が停止しました。

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

## 後続の分析手順

SQL ID c9333k2g237qxに対するSQLHC情報の分析を通じて、当該SQLは長時間実行されていたことが判明しましたが、時間の大部分はCPU Timeに費やされており、長時間にわたるPGAメモリ操作に関する待機イベントは観測されませんでした。問題の原因は、実行計画、テーブルサイズ、統計情報、またはバインド変数の違いにある可能性があります。テスト環境における実行状況と比較する必要があります。以下の情報を提供ください。
1. テスト環境（Oracle 19.3）で当該SQLのSQLHCを収集  
2. テスト環境と本番環境の両方で、当該SQLの10046トレースと10053トレースを収集  
```SQL
SQL> connect executing_user
SQL> alter session set max_dump_file_size = UNLIMITED;
SQL> alter session set timed_statistics=true;
SQL> alter session set events '10053 trace name context forever, level 2';
SQL> alter session set events '10046 trace name context forever, level 12';
```

問題を再現するためにステートメントを実行します。
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

## まとめ
後続の分析により、テスト環境（19.3）と本番環境（19.7）でデータ量が異なっていたことが判明しました。
同じデータを使用した場合でも、テスト環境ではSQLが正常に実行されませんでした。その後、開発チームがSQLに新たなフィルタ条件を追加して修正したことで、SQLは正常に実行されるようになりました。
