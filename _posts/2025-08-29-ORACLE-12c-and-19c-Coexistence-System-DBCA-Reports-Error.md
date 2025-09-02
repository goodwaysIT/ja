---
layout: post
title: "Oracle 12cおよび19cの共存システムでDBCAエラー報告"
excerpt: "現在のシステムは、12cと19cの共存システムです。クラスタは19C、データベースは19cと12cです。ソフトウェアは異なるユーザー配下にあります：19c（ORACLEユーザー）、12c（oracle2ユーザー）。現在、12c環境でDBCAを実行してデータベースを作成する際に、以下のエラーが報告されます：DBT-00007 ユーザーに適切な書き込み権限がありません。"
date: 2025-08-29 15:00:00 +0800
categories: [Oracle, Database, RAC]
tags: [PRVF-7595, PRVG-2043, DBT-00007, rac, oracle]
image: /assets/images/posts/ORACLE-12c-and-19c-Coexistence-System-DBCA-Reports-Error.jpg
---

## 問題  
現在のシステムは、12cと19cの共存システムです。クラスタは19C、データベースは19cと12cです。ソフトウェアは異なるユーザー配下にあります：19c（ORACLEユーザー）、12c（oracle2ユーザー）。  
現在、12c環境でDBCAを実行してデータベースを作成する際に、以下のエラーが報告されます：  
```
DBT-00007 Users does not have the appropiate write privileges .  
```
その後、CRSチェックで以下のエラーが報告されました：  
```
PRVF-7595 : CRS status check cannot be performed on node "xxxx"
PRVG-2043 : Command "/u01/app/19.7.0.0/grid/bin/crs_stat -t " failed on node "xxxx"
```

## 原因  
```
Check the permissions of files like `$ORACLE_BASE/cfgtoollogs/dbca`
node-a:~ # id -a oracle2
uid=54323(oracle2) gid=54421(oinstall) groups=54322(dba),54327(asmdba),54421(oinstall)
node-a:~ # su - oracle2
oracle2@node-a:/home/oracle2$ echo $ORACLE_BASE
/u01/app/oracle
oracle2@node-a:/home/oracle2$ ls -la $ORACLE_BASE/cfgtoollogs
total 0
drwxr-x--- 4 oracle oinstall 34 May 22 14:46 .
drwxrwxr-x 10 oracle oinstall 126 May 22 15:09 ..
drwxr-x--- 3 oracle oinstall 98 Jun 14 14:29 dbca
drwxr-x--- 9 oracle oinstall 249 May 22 15:21 sqlpatch
node-a:~ # id -a grid
uid=54322(grid) gid=54421(oinstall) groups=54327(asmdba),54328(asmoper),54329(asmadmin),54330(racdba),54421(oinstall)
node-a:~ # id -a oracle
uid=54321(oracle) gid=54421(oinstall) groups=54322(dba),54323(oper),54324(backupdba),54325(dgdba),54326(kmdba),54327(asmdba),54328(asmoper),54330(racdba),54421(oinstall)
node-a:~ # su - grid
grid@node-a:/home/grid$ echo $ORACLE_BASE
/u01/app/grid
grid@node-a:/home/grid$ ls -la $ORACLE_BASE/cfgtoollogs
ls: cannot access '$ORACLE_BASE/cfgtoollogs': No such file or directory
grid@node-a:/home/grid$ ls -la $ORACLE_BASE/cfgtoollogs
total 0
drwxrwxr-x 9 grid oinstall 102 May 10 15:08 .
drwxrwxr-x 8 grid oinstall 97 May 11 10:13 ..
drwxr-x--- 2 grid oinstall 298 May 11 16:03 asmca
drwxr-x--- 4 grid oinstall 36 May 10 15:21 dbca
drwxrwxr-x 2 grid oinstall 156 May 10 15:23 mgmtca
drwxrwxr-x 2 grid oinstall 6 May 10 14:52 mgmtua
drwxr-x--- 2 grid oinstall 142 May 10 15:04 netca
drwxrwxr-x 2 grid oinstall 6 May 10 14:52 restca
drwxr-x--- 6 grid oinstall 174 May 10 15:22 sqlpatch
grid@node-a:/home/grid$ exit
logout
node-a:~ # su - oracle
oracle@node-a:/home/oracle$ echo $ORACLE_BASE
/u01/app/oracle
oracle@node-a:/home/oracle$ ls -la $ORACLE_BASE/cfgtoollogs
total 0
drwxr-x--- 4 oracle oinstall 34 May 22 14:46 .
drwxrwxr-x 10 oracle oinstall 126 May 22 15:09 ..
drwxr-x--- 3 oracle oinstall 98 Jun 14 14:29 dbca
drwxr-x--- 9 oracle oinstall 249 May 22 15:21 sqlpatch
```
CRSチェックで以下のエラーが報告されました：  
```
PRVF-7595 : CRS status check cannot be performed on node "node-b" - Cause: Could not verify the status of CRS on the node indicated using ''crsctl check''. - Action: Ensure the ability to communicate with the specified node. Make sure that Clusterware daemons are running using ''ps'' command. Make sure that the Clusterware stack is up.
PRVG-2043 : Command "/u01/app/19.7.0.0/grid/bin/crs_stat -t " failed on node "node-b" and produced the following output: /u01/app/19.7.0.0/grid/bin/crs_stat.bin: error while loading shared libraries: libocr12.so: cannot open shared object file: No such file or directory - Cause: An executed command failed. - Action: Respond based on the failing command and the reported results.
```
CRS CHECK中のエラーは、12c DBと19c Gridの間に互換性のないコマンドがあることを示しています。  

## 解決策  
1.すべてのノードでCRSが確実に実行されていることを確認してください。  
2.すべてのノードでCRSが正常に実行されている場合は、このエラーを無視してください。  
3.インストーラーの実行を続行してください。  
または、以下の回避策を使用できます：  
$ export ORA_DISABLED_CVU_CHECKS=TASKCRSINTEGRITY,TASKNTP,STASKRESOLVCONFINTEGRITY  
$ echo ORA_DISABLED_CVU_CHECKS  

/u01 のユーザーと権限をリセットしてください。  
例：  
```
chown -R oracle2:oinstall /u01
chmod -R 775 /u01
```

## 参考情報  
DBCA Database Creation Fails With Error PRCR-1154 Due to Wrong Ownership/Permission (Doc ID 2165706.1)  

