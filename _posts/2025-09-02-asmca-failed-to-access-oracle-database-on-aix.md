---
layout: post
title: "ORACLE ASMCAコマンドがAIX 7.2でアクセスに失敗"
excerpt: "提供された情報に基づき、AIXオペレーティングシステムでasmcaコマンドを実行した際に、未処理の例外エラーメッセージが発生したことを確認しました。この問題はAIX環境に特有であり、BUG 34585151で調査されており、IBM JVMの設定に関連していると考えられています。"
date: 2025-09-02 15:00:00 +0800
categories: [Oracle, Database, RAC]
tags: [error_value=00000000, Can not use asmca, asmca failed to access oracle, rac, oracle]
image: /assets/images/posts/asmca-failed-to-access-oracle-database-on-aix.jpg
---

## 問題  
対象バージョン: 19.17.0.0.0, IBM AIX on POWER Systems (64-bit)  

asmcaがアクセスに失敗しました  
```
ERROR
-----------------------
unhandled exception
Type=segmentation error vmstate=0x00040000
J9Generic_signal_number=00000018 signal_name=000000b error_value=00000000 signal_code=00000033
.....
-----stack backtrace----
sdbgrfbibf_io_block_file
...

JVMDUMP039I Processing dump event "gpf", detail "" xxxxxx
JVMDUMP032I JVM requested System dump using xxxxxx

STEPS
-----------------------
Run asmca

BUSINESS IMPACT
-----------------------
Can not use asmca
```

## 原因  
提供された情報に基づき、AIXオペレーティングシステムでasmcaコマンドを実行した際に、未処理の例外エラーメッセージが発生したことを確認しました。  
この問題はAIX環境に特有であり、BUG 34585151で調査されており、IBM JVMの設定に関連していると考えられています。  
このバグによれば：  
これはIBM JVMの設定に依存します。OSスレッドのデフォルトのスタックサイズは256Kですが、JDBC OCIDriverを使用して接続するスレッドには小さすぎます。その結果、接続エラーが繰り返し発生するとJVMがクラッシュします。  

## 解決策  
### 1.asmcaバイナリファイルのバックアップ:  
```
cp <GI_HOME>/bin/asmca <GI_HOME>/bin/asmca.bkp  
```

### 2.asmcaファイルを手動で修正し、ファイルの最後の行をコメントアウトし、以下の新しい行を追加します:  
```
vi <GI_HOME>/bin/asmca  
### exec $JRE_DIR/bin/java $JRE_OPTIONS -classpath $CLASSPATH oracle.sysman.assistants.usmca.Usmca $ARGUMENTS ◄◄◄ This line was commented.  
exec $JRE_DIR/bin/java -Xss4m -Xmso1m $JRE_OPTIONS -classpath $CLASSPATH oracle.sysman.assistants.usmca.Usmca $ARGUMENTS ◄◄◄ Add this line  
```
or  
```
vi <GI_HOME>/bin/asmca  
### JRE_OPTIONS="${JRE_OPTIONS} -Dsun.java2d.font.DisableAlgorithmicStyles=true -DDISPLAY=$DISPLAY -DIGNORE_PREREQS=$IGNORE_PREREQS -DJDBC_PROTOCOL=thin -mx128m $DEBUG_STRING"  
◄◄◄ Commented the line above and replace it with the following line.  
JRE_OPTIONS="${JRE_OPTIONS} -Dsun.java2d.font.DisableAlgorithmicStyles=true -DDISPLAY=$DISPLAY -DIGNORE_PREREQS=$IGNORE_PREREQS -DJDBC_PROTOCOL=thin -Xss4m -mx128m $DEBUG_STRING"  
```

### 3.asmca を再度実行します。

## 参考情報  
BUG 34585151 - AIX64-19.17 : ROOT.SH FAILED ON FIRST NODE, UNHANDLED EXCEPTION, CLSRSC-612: FAILED TO CREATE BACKUP DISK GROUP  

