---
layout: post
title: "SCANを使用したJDBC接続がORA-12516で失敗する問題"
excerpt: "スキャンリスナーログとVIPリスナーログに基づくと、TNS-12516エラーは7月31日から発生していることが確認された。お客様は何も変更を行っていないと述べている。TNS-12516エラーはスキャンリスナーログに記録されているが、VIPリスナーログでは問題は見られない。"
date: 2025-08-06 15:00:00 +0800
categories: [Oracle, Database]
tags: [TNS-12516, ORA-03137, JDBC Driver, SCAN Fail, oracle]
image: /assets/images/posts/JDBC-Connections-Using-SCAN-Fail-With-ORA-12516.jpg
---

## 症状  
バージョン: 19.7.0.0.0, RDBMS  
listener_scan.log:  
```
2024-07-31T02:00:00.006290+08:00
31-JUL-2024 02:00:00 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=jdbc)(USER=ptdb))(SERVICE_NAME=ptdb)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.12.27)(PORT=47898)) * establish * ptdb * 12516
TNS-12516: TNS:listener could not find available handler with matching protocol stack
31-JUL-2024 02:00:00 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=jdbc)(USER=ptdb))(SERVICE_NAME=ptdb)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.12.27)(PORT=47900)) * establish * ptdb * 12516
TNS-12516: TNS:listener could not find available handler with matching protocol stack
2024-07-31T02:00:01.010650+08:00
31-JUL-2024 02:00:01 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=jdbc)(USER=ptdb))(SERVICE_NAME=ptdb)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.12.27)(PORT=47910)) * establish * ptdb * 12516
TNS-12516: TNS:listener could not find available handler with matching protocol stack
31-JUL-2024 02:00:01 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=jdbc)(USER=ptdb))(SERVICE_NAME=ptdb)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.12.27)(PORT=47912)) * establish * ptdb * 12516
TNS-12516: TNS:listener could not find available handler with matching protocol stack
....................

06-AUG-2024 04:06:37 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=jdbc)(USER=ptdb))(SERVICE_NAME=ptdb)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.12.31)(PORT=52314)) * establish * ptdb * 12516
TNS-12516: TNS:listener could not find available handler with matching protocol stack
06-AUG-2024 04:06:37 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=jdbc)(USER=ptdb))(SERVICE_NAME=ptdb)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.12.31)(PORT=52316)) * establish * ptdb * 12516
TNS-12516: TNS:listener could not find available handler with matching protocol stack
06-AUG-2024 04:06:37 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=jdbc)(USER=ptdb))(SERVICE_NAME=ptdb)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.12.31)(PORT=52320)) * establish * ptdb * 12516
TNS-12516: TNS:listener could not find available handler with matching protocol stack
2024-08-06T04:06:38.265467+08:00

```

## 原因  
スキャンリスナーログとVIPリスナーログに基づくと、TNS-12516エラーは7月31日から発生していることが確認された。  
お客様は何も変更を行っていないと述べている。  
TNS-12516エラーはスキャンリスナーログに記録されているが、VIPリスナーログでは問題は見られない。  
エラーはPDB: `ptdb` に集中している。  
スキャンリスナーログでは、他のPDBへの接続は正常であることを示している。  
スキャンリスナーログには、PDB: `ptdb` への接続が成功しているケースも散見される。  
主要なアプリケーションサーバーのIPアドレスは: 192.168.12.27/31 である。  
データベースの `PROCESS` パラメータも `SESSION` パラメータも最大制限に達していない。  
お客様はアプリケーションサーバーとデータベースサーバーの間にロードバランサー機器を使用している。  

使用されているJDBCドライバーのバージョンは: 11.2.0.4 である。  
このバージョンは比較的古く、*JDBC Connections Using SCAN Fail With ORA-12516 Or ORA-12520* (Doc ID 1555793.1) のような問題を引き起こすことが知られている。  

加えて、環境内のalert.logにも以下のようなエラーが含まれている:  
`ORA-03137: クライアントからの不正なTTCパケットが拒否されました: [3146] [94] [] [] [] [] [] []`。  
このエラーは *ORA-03137: malformed TTC packet from client rejected: [3146] [94] [] [] [] [] [] [] While Using JDBC 12.2.0.1* (Doc ID 2519886.1) に関連している。  
この問題を解決するためにもJDBCバージョンのアップグレードが必要である。  

## 解決策  
JDBCバージョンを19c以降にアップグレードする。  

参照:  
https://www.oracle.com/database/technologies/appdev/jdbc-downloads.html  
Oracle Database 19c (19.24.0.0) JDBC Driver & UCP Downloads - 長期リリース <<<<<<<<<<<<<  

## 参照資料  
ORA-03137: クライアントからの不正なTTCパケットが拒否されました: [3146] [94] [] [] [] [] [] [] (JDBC 12.2.0.1 使用時) (Doc ID 2519886.1)  
SCANを使用したJDBC接続がORA-12516またはORA-12520で失敗する (Doc ID 1555793.1)  
