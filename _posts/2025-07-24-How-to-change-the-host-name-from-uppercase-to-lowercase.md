---
layout: post
title: "RAC環境でホスト名を大文字から小文字に変更する方法"
excerpt: "opatch lsinventoryが1つのノードを大文字、もう1つのノードを小文字で表示する問題"
date: 2025-07-24 15:00:00 +0800
categories: [Oracle, Database]
tags: [uppercase hostname, oracle]
image: /assets/images/posts/How-to-change-the-host-name-from-uppercase-to-lowercase.jpg
---

## 目的  
1. opatch lsinventoryが1つのノードを大文字（不正確）、もう1つのノードを小文字（正確）で表示します。
$ opatch lsinventory

```
Oracle Grid Infrastructure 11g 11.2.0.4.0
There are 1 product(s) installed in this Oracle Home.
There are no Interim patches installed in this Oracle Home.

Rac system comprising of multiple nodes
Local node = rac1
Remote node = RAC2 <<<<< 大文字表記、誤り

```

2. 中央インベントリでは、ホスト名が大文字で表示されています。
$ cat /u01/app/oraInventory/ContentsXML/inventory.xml  

```
...

<HOME NAME="Ora11g_gridinfrahome1" LOC="/apps/oracle/product/11.2.0.4.GRD" TYPE="O" IDX="1" CRS="true">
  <NODE_LIST>
     <NODE NAME="rac1"/>
     <NODE NAME="RAC2"/>                           <<<<< Upper-case, incorrect one.
  </NODE_LIST>
</HOME>

...

```

## 解決方法  
特定の ORACLE_HOME でホスト名を大文字から小文字に変更するには、以下の手順を実行します：
1. 各ノードで以下を実行（ソフトウェア所有者として）。大文字表記を修正:  
```
$ $ORACLE_HOME/oui/bin/runInstaller -updateNodeList ORACLE_HOME="/apps/oracle/product/11.2.0.4.GRD" "CLUSTER_NODES={rac1,rac2}" -silent -local
```  

2. 変更を確認するため以下を再実行:  
```
$ opatch lsinventory  
$ cat oraInventory/ContentsXML/inventory.xml
```
