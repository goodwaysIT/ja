---
layout: post
title: "ORACLE 19c データベースアーキテクチャと運用管理ガイド"
excerpt: "DB2からOracleデータベースへの製品移行に伴い、ますます多くのOracleデータベースが導入されています。Oracle 12cおよび19cデータベースは導入時期が早く、本番環境での運用期間も長いため、使用過程で多くの課題に直面してきました。Oracleデータベースの安定性を向上させ、ビジネスの健全かつ継続的な運用を確保するため、現行の製品と今後のアーキテクチャの方向性を踏まえ、以下のようなアドバイスを提供します。"
date: 2025-06-03 11:30:00 +0800
categories: [Oracle, データベース]
tags: [データベースメンテナンス,データベースデプロイメント,データベース最適化,Oracle,Leo.Wang]
image: /assets/images/posts/ORACLE-19C-Database-Architecture-and-Operation-and-Maintenance-Guide.jpg
author: Leo.Wang
---

## 一、	概要  
### 1.1	プロジェクトの背景

DB2からOracleデータベースへの製品移行に伴い、ますます多くのOracleデータベースが導入されています。Oracle 12cおよび19cデータベースは導入時期が早く、本番環境での運用期間も長いため、使用過程で多くの課題に直面してきました。Oracleデータベースの安定性を向上させ、ビジネスの健全かつ継続的な運用を確保するため、現行の製品と今後のアーキテクチャの方向性を踏まえ、以下のようなアドバイスを提供します。  
<TABLE>
<TR>
<TD align="left">
<FONT><strong>19C版データベース運用保守システム：</strong></FONT>
</TD>
</TR>
<TR>
<TD align="left">
<img src="https://goodwaysit.github.io/ja/assets/images/database/db_architecture_ja.png" style="float:left;" />
</TD>
</TR>
</TABLE>


### 1.2	ハードウェア環境
<TABLE>
<TR>
<TD align="left">
<FONT><strong>ORACLE Exadata(X6-2、X7-2、X8-2) + PCサーバー(複数台)</strong></FONT>
</TD>
</TR>
<TR>
<TD align="left">
<img src="https://goodwaysit.github.io/en/assets/images/database/exadata.JPG" style="float:left;" />
</TD>
</TR>
</TABLE>

### 1.3	ソフトウェア環境  
<TABLE>
<TR>
<TD align="left">
<img src="https://goodwaysit.github.io/ja/assets/images/database/version_ja.JPG" style="float:left;" />
</TD>
</TR>
</TABLE>

## 二、	多次元分析とアドバイス
### 2.1	バージョン
現行のデータベースリストによると、主に使用されているデータベースバージョンは12.2.0.1で、RUパッチの適用状況も2017年時点で停滞しています。Oracle社が提供するデータベースバージョンのサポートサイクルによると、12.2バージョンの標準サポートは2020年3月末で終了しています。より適切な製品サポートを提供するため、今後のデータベースバージョン選定では、19Cバージョンの採用を推奨します。

> 以下は、各バージョンの製品サポート ライフサイクルの図です。  
> ![support timelines](https://goodwaysit.github.io/en/assets/images/database/timelines.jpg#pic_left)

Oracle データベース製品のライフサイクル サポート ポリシーによれば、上図に示すように、次のことがわかります。  

> バージョン12.2.0.1の標準サポート期間は2020年3月31日に終了し、2022年3月31日まで延長されます（延長サポートは別途購入する必要があります）  
> バージョン19Cの標準サポート期間は2024年4月30日に終了し、2027年4月30日まで延長されます（延長サポートは別途購入する必要があります）  
> 19Cのサポート期間は2027年半ばに終了し、12C製品ファミリーの中では長期サポート版です。  

### 2.2	アーキテクチャ
#### 2.2.1	データベースアーキテクチャ
<TABLE width="500">
<TR>
<TD align="left">
<FONT><strong>NON-CDBアーキテクチャ</strong></FONT>
</TD>
<TD align="left">
<FONT><strong>CDBアーキテクチャ</strong></FONT>
</TD>
</TR>
<TR>
<TD align="left">
<img src="https://goodwaysit.github.io/en/assets/images/database/non-cdb.jpg" style="float:left;" />
</TD>
<TD align="left">
<img src="https://goodwaysit.github.io/en/assets/images/database/cdb.jpg" style="float:left;" />
</TD>
</TR>
</TABLE>

**NON-CDB:**  
大規模企業では、数百から数千ものデータベースを管理する必要に迫られることがあります。通常、これらのデータベースは複数の物理サーバー上で動作し、異なるプラットフォームで運用されているケースも少なくありません。しかし、CPU数をはじめとするハードウェア技術の進化により、現代のサーバーはより高い負荷を処理できるようになりました。これは逆に、1つのデータベースがサーバーリソースのごく一部しか消費せず、多くのハードウェアと人的リソースが無駄になります。 DBA チームは、各データベースのSGA、データベース ファイル、アカウント、セキュリティなどを個別に管理する必要があります。

**CDB:**  
1. 各アプリケーションは独自のPDBを保有可能
* アプリケーションの変更なしで運用可能
* PDBのコピーによる迅速な環境構築や移行が可能
* プラグイン方式のため、高い移植性を実現


2. CDBレベルでの一般的な業務
* パッチ適用、アップグレード、HA 設定、バックアップなど、個々のPDBごとに操作する必要がなく、CDBに対する操作がすべてのPDBに反映される。
* きめ細かな管理


3. メモリとバックグラウンドプロセスの共有化
* 各DBが従来のように独自のSGA/PGAとバックグラウンドプロセスを持つ必要がないため、メモリを節約でき、各サーバーでより多くのアプリケーションを実行可能

**現状：**  
現在、本番環境で運用されているデータベースのほとんどはNON-CDB（非コンテナ型）アーキテクチャを採用しており、同一物理サーバー上で複数のインスタンスが稼働しているケースが多く見られます。これらのデータベースは規模が小さく、リソースが分散しているため、非効率な状態です。

**アドバイス：**  
CDB アーキテクチャは、比較的小規模で分散したデータベースに特に適しており、単一のデータベースと単一のインスタンスによって発生するリソースの無駄を回避します。Oracle 20C以降では、NON-CDBアーキテクチャのサポートがデフォルトで終了し、CDBアーキテクチャのみが選択可能となっています。今後のデータベース展開においては、OracleのCDBソフトウェアアーキテクチャに適応した設計を採用することを強く推奨します。これにより、ハードウェアリソースの効率的な活用が可能となるだけでなく、運用保守の効率も大幅に向上するでしょう。

#### 2.2.2	Maximum Availability Architecture (MAA)
Oracle Maximum Availability Architecture (MAA) は、長年にわたる高可用性技術のノウハウとベストプラクティスを集約したソリューションです。 MAA の目的は、コストと複雑さを最小限に抑えながら、最高の高可用性アーキテクチャを実現することです。

* MAA ベスト プラクティスは、Oracle Database、Oracle Application Server、Oracle Applications、および Grid Control を対象としています。
* MAA は、さまざまなビジネスニーズを考慮して、これらのベストプラクティスを広く適用できるようにします。
* MAA は低コストのサーバーとストレージを使用します。
* MAA は、新しい Oracle バージョンと機能とともに進化し続けます。
* MAA はハードウェアやオペレーティング システムに依存しません。  
> ![MMA](https://goodwaysit.github.io/en/assets/images/database/mma.jpg#pic_left)  

**現状：**  
既存の環境では、コアデータベースに MAA アーキテクチャに準拠した ADG が装備されています。

**アドバイス：**  
一部データベースにADGが未導入のため、ハードウェアリソースに余裕がある場合、ADG構成への移行を推奨します。

### 2.3	環境ベースライン
データベースの安定運用を確保するため、ベストプラクティスに基づき、パラメータ設定およびパッチ適用のベースラインを策定することを推奨します。これにより、今後のデータベース環境構築において統一的なガイドラインを提供でき、安定した運用基盤を確立できます。

パラメータとパッチ ベースラインについては、時間をかけて詳細な検討が必要不可欠です。負荷テストを通じてパラメータの性能評価を実施することもお勧めします。

#### 2.3.1	19C標準インストールマニュアル
**現状：**  
19C向けの包括的なインストールマニュアルが整備されていません。

**アドバイス：**  
I19Cインストールマニュアルを整備し、さまざまなプラットフォーム、RAC構成、単独構成、およびシングルインスタンスのシナリオをカバーします。

#### 2.3.2	パラメータベースライン  
パラメータベースラインは、データベースのソフトウェアアーキテクチャに基づいて、NON-CDBとCDBに細分化する必要があります。
パラメータベースラインには、以下のポイントを含めるべきです。  
*	NON-CDBおよびCDBの２種類のベースライン標準
*	CDBアーキテクチャにおけるPDBリソース割り当てと分離
*	CDBアーキテクチャのメモリ構成（SGA/PGA）
*	DRMおよびACS関連機能
*	新機能に関するパラメータ


**現状：**  
19Cパラメータ分析は完了しました。  

**アドバイス：**  
特定のCDBアーキテクチャに基づいて、CDBアーキテクチャのパラメータ推奨をさらに改善します。  

#### 2.3.3	パッチ
OSプラットフォームやデータベースのバージョンに応じて、世界中のOracleユーザーの使用経験を踏まえ、最適なパッチをまとめることで、プログラム上のバグを最大限回避することが可能です
過去の障害のうち、2件は古いRU（Release Update）パッチが原因で発生しました。詳細は以下の通りです。  

**Bug 27162390 - RAC LMS Process Hits ORA-600 [kclantilock_17] Error and Instance Crashes (Doc ID 27162390.8)**  
> ![Bug 27162390](https://goodwaysit.github.io/en/assets/images/database/Bug_27162390.jpg#pic_left)

**Bug 28681153 - ORA-600: [qosdexpstatread: expcnt mismatch] (Doc ID 28681153.8)**  
> ![Bug 28681153](https://goodwaysit.github.io/en/assets/images/database/Bug_28681153.jpg#pic_left)

**現状：**  
バージョン 12.2: 詳細なパッチ分析は実施されておらず、本番環境のパッチは2017年8月時点のまま更新されていません。  
19C バージョン: 19.7 ベースのパッチ分析が最近完了しました。

**アドバイス：**  
バージョン 19C は分析済み、バージョン 12.2 では詳細なパッチ分析と適用を推奨します。

### 2.4	アプリケーションテストとSQL監査  
アップグレードであっても、日常的なアプリケーションバージョンの変更であっても、アプリケーションのSQL文を確認する必要があります。これは、製品による監査または人的監査で実現できます。

##### 現状：   
最近の障害は、制御されていないアプリケーションSQLが多くの本番環境の障害を引き起こしていることも示しています。

**例：**  
* Exadata ログイン遅延  
**問題現象:**    
DB にはバインド変数を使用しないステートメントが多数あり、その結果、バージョン数が異常に高くなり、頻繁にリロードが発生します。例:   
`select zno from branch where zno = '982052'`  
**原因分析:**  
SQL開発仕様の問題 - バインド変数を使用していない  

* テーブルスペースを拡張できない  
**問題現象:**  
ビジネステーブル(INFO_TAB)のデータを挿入できません。警告ログで、DATA_TBSがテーブルに必要なスペースを拡張できなかったことが判明しました。  
**原因分析:**  
SQL 開発が不適切で、断片化率が高く、頻繁な挿入・削除操作が存在しています。一時的な操作に対してテンポラリテーブルやtruncate技術が使用されていない。

* RAC Hang問題  
**問題現象:**  
トップイベントは「cr request retry」です  
**current sql:**  
`select a.CUsT CODE,
b.PRO_TYPE,
b.type,
from app_info a
inner join (
SELECT a.CUST CODE,
CASE
WHEN SUBSTR(a.PRO_TYPE,1,4)='2012' THEN
'锟斤拷锟斤'`;  
`sys@clmdb> select * from cwm.remp_info where rownum < 2;`  
**原因分析:**  
SQLの記述が不適切：条件を指定しないクエリ文；不要なテーブル結合；フルテーブルスキャン

#### アドバイス：  
1. SQLの問題を事前に回避するために、SQL審査製品を導入し、アプリケーションリリース前にSQL監査を実施します。  
2. プログラムリリース前にDBAによる審査を通過させます（他部門との連携が必要）。  

### 2.5	安定性とパフォーマンス評価
データベースの安定運用を確保するには、必要な監視とメンテナンスが必要です。
監視およびメンテナンスのシナリオをいくつか以下に示します。  

**監視:**
*	オブジェクトレベル：表領域、コア テーブルの断片化率、インデックスの断片化率、インデックススプリットなど
* SQLレベル：TOP SQL、実行計画の変更、ハードパースなど
*	メモリプール：Shared Pool，Buffer Cache

**メンテナンス：**  
適切なデータクリーンアップ計画を策定し、データベースのスリム化を行い、データベースが最適な状態で稼働するように確認します。

**現状：**  
既存の監視にはORACLE EM とサードパーティの監視が含まれており、重点監視項目が十分にカバーされていません。

**アドバイス：**  
1. 関連監視項目を改善します。 （優先度：高）  
2. 包括的なデータクリーニングおよびスリム化計画を策定します。 (優先度：低)。

### 2.6	操作・保守マニュアルおよび緊急時マニュアル  
> 日常的な運用保守および緊急事態に対応するため、通常の点検マニュアルに加え、標準的な運用保守手順と緊急処置手順を記載したマニュアルの整備が必要:  
> ![maintenance](https://goodwaysit.github.io/ja/assets/images/database/maintenance_ja.jpg#pic_left)  

**現状：**  
システム点検チェックリストおよび通常運用マニュアルは既に存在します。

**アドバイス：**  
緊急対応マニュアルを整備し、継続的に内容を改善します。

### 2.7	ベースラインの改善  
日々運用が継続されるにつれ、パラメータやパッチに関するベースラインは継続的に改善されていきます。ベースラインメンテナンスを実装するには、専任の担当者またはプロセスを設定するが必要です。

### 2.8	バックアップ
ORACLE ZDLRA オンラインバックアップ + テープバックアップによるオフラインストレージ  
バックアップ アーキテクチャは比較的整備されているため、調整は必要ありません。

### 2.9	その他のアドバイス
**Serviceの設定：**  
一部のシステムではバッチ実行中に GC 待機時間が長く、単一ノードで負荷要件を満たせる場合、GC 競合を回避するために、Serviceを使用して1つのノードで実行することをお勧めします。  

# 付録: CDB におけるのリソース分離パラメータの設定例  
> リソース制御のバージョン管理レベル比較:  
> ![resource management](https://goodwaysit.github.io/en/assets/images/database/resource.jpg)  

### ケース1：某銀行  

#### PDB CPUリソース管理  
CDBパラメータでリソースマネージャを設定する場合、CPUリソース割り当てを強制実行するためには、CDBレベルで「RESOURCE_MANAGER_PLAN」パラメータに「DEFAULT_CDB_PLAN」を設定する必要があります。  
（1）Oracle 12.2以降、CPU使用はPDBのCPU_COUNTカウントによって制限されます。
（2）Oracle 18.1以降、システムはPDBのCPU_COUNTに基づきCPUスケジューリングシェアを自動設定します。  
> ![task plans](https://goodwaysit.github.io/en/assets/images/database/plan.jpg)

*	autotask：
```bash
shares: -1  
utilization_limit: 90  
parallel_server_limit: 100  
```
***shares = -1 は、自動メンテナンスタスクがシステムリソースの20%を使用することを示します。***  
***v$rsrcmgrmetric_historyはリソースの割当と使用状況を記録します。***

*	default_pdb_directive：
```bash
new_shares: 1
utilization_limit: 100
parallel_server_limit: 100
```  
***注:*** Shares=1  
DEFAULT_PDB_DIRECTIVE は作成されるPDBに全てのリソースを割り当てます。  
CPU_COUNT によるリソース割当が制限され、Sharesのデフォルト値は1となります  

***参照ドキュメント:***  
How to Provision PDBs, based on CPU_COUNT Doc ID 2326708.1  

#### PDBメモリリソース管理  
**機能概要:**  
複数の PDB を使用すると、必然的にリソースの競合が発生します。 Oracle 12.2 は、各リソースの使用を効果的に制御・調整できます。  
> PDBメモリ管理に必要なパラメータ設定:  
> ![resource limit](https://goodwaysit.github.io/ja/assets/images/database/resource_limit_ja.JPG#pic_left)

**パラメータ注釈:**  
*	PDB: SGA_TARGET  PDBメモリ最大使用量パラメータ  
    * CDBパラメータ設定より小さい  

*	PDB: DB_CACHE_SIZE PBDデータキャッシュ。このパラメータを設定すると、メモリが「盗まれ」なくなります。  
    * ASMMメモリ管理モードでは、データキャッシュの最小設定は20%SGA～30%SGAです。  

*	PDB:SHARED_POOL_SIZE PDB共有プールキャッシュ。このパラメータを設定すると、メモリが「盗まれ」なくなります。  
    * ASMMメモリ管理モードでは、共有プールの最小設定は20%SGA～30%SGAです。  
    * 共有プール優先原則: <50%* SGA_TARGETである場合、共有プールメモリを確保します。  

*	PDB:PGA_AGGREGATE_LIMIT、[2G , sessions*3M]  
    * PDBでのこのパラメータ設定は、CDBでの設定値より小さくする必要があります。  
    * PDBでのこのパラメータ設定は、PGA_AGGREGATE_TARGETの2倍以上にする必要があります。  

*	PDB:PGA_AGGREGATE_TARGET、[< PGA_AGGREGATE_LIMIT/2]  
    * PDBでのこのパラメータ設定は、CDBでの設定値より小さくする必要があります  
    * PDBでのこのパラメータ設定は、PDB PGA_AGGREGATE_LIMIT*50%未満にする必要があります  

***参照ドキュメント:***  
How to Control and Monitor the Memory Usage (Both SGA and PGA) Among the PDBs in Mutitenant Database- 12.2 New Feature (Doc ID 2170772.1)  How To Deal With "SGA: allocation forcing component growth" Wait Events (Doc ID 1270867.1)  

#### PDB I/O リソース管理  
PDB レベルの I/O 使用制御:  
*	PDB: MAX_IOPS、PDBの1秒あたりの最大IO要求数、単位：回、動的に変更可能  
*	PDB: MAX_MBPS、PDBの1秒あたりの最大IO要求数、単位：M、動的に変更可能  
```sql  
SQL> alter system set MAX_IOPS=1000;   --負荷テストにおける最大値
SQL> alter system set MAX_MBPS=500;   --負荷テストにおける最大値
```  
***参照ドキュメント:***   
I/O Rate Limits for PDBs 12.2 New feature . (Doc ID 2164827.1)  
It is recommended to set these parameters when IO performance problems occur.    


### ケース2：あるオペレーター  

#### ターゲット  
* 通常の運用状況では、UAT ライブラリ、STANDBY ライブラリ、および DR ライブラリに対する CDB レベルのリソース割り当ては制限します。
* 特殊な状況では、本番環境内のPDBレベルでのリソース割り当てを制限します。


#### CDBレベルのIORM   
* I/O リソース管理 (IORM) は、複数のデータベースと、データベース内のワークロードが Oracle Exadata System Software の I/O リソースを共有する方法を管理するためのツールです。   
* 「dbplan」を設定し、「share」、「limit」、「flashcachesize」でデータベース間のリソース割当を管理します。  
`share`- データベースの相対的な優先順位を指定します。  
`limit`- データベースの最大ディスク使用率を指定します。これは「パフォーマンスに応じた支払い」のユースケースに最適ですが、ワークロード間の公平性を実現するために使用しません。  
`flashcachesize`- データベースに割り当てるフラッシュキャッシュの固定サイズを指定します。  
> ![resource profile](https://goodwaysit.github.io/ja/assets/images/database/profile_ja.JPG#pic_left)

#### PDBレベルでのリソース管理  
CPUと並列クエリを制限するために、3階層のPDBリソース計画を設計しています。  

|ゴールド シルバー ブロンズ 計画               |    Share            |    ut limit            |    parallel limit |
|:----                                      |    :----            |    :----               |    :----          |
|GOLD                                       |    8                |    100                 |    100            |
|SILVER                                     |    4                |    40                  |    40             |
|BRONZE                                     |    2                |    20                  |    20             |
|                                           |                     |                        |                   |

#### CDBとPDBのCPU制御  
CDBとPDBでのCPU制御は、cpu_countを変更することによっても実現できます。  

#### PDBレベルでのI/O制御  
以下のパラメータでPDBレベルのI/Oを動的に制御します：  
* `MAX_IOPS`  
* `MAX_MBPS`  
PDBレベルでのメモリリソース制御  

## まとめ
*	小規模データベースや独立データベースによる冗長なバックグラウンドプロセス問題を回避するため、CDB アーキテクチャを通じていくつかの散在する小規模データベースを統合することを推奨します。コアデータベースについては、NON-CDBであってもCDBであっても、専用データベースとして運用することを推奨します。
*	19C標準インストールマニュアルを整備し、各プラットフォーム、RAC、単独および単一インスタンスのシナリオをカバーします
*	19C CDB環境のパラメータベースラインをさらに整備します。負荷テストとPDBリソース分離を組み合わせ、多角的なテストを行う必要があります
*	12.2データベースバージョンに関しては、RUが古いため、アップグレードを推奨します。既知のバグを回避するために、アップグレード前な十分なテストが必要です。
*	SQL監査を継続的に推進し、SQLによるパフォーマンスリスクを回避します。
*	関連する監視の整備：
    * オブジェクトレベル：コアテーブルの断片化率、インデックス断片化率、インデックススプリットなど
    * SQLレベル：TOP SQL、ハードパース、実行計画変更の有無、ハードパースなど
    * メモリプール：Shared Pool、Buffer Cacheなど
*	Serviceを設定します。単一ノードで業務が運用可能な場合はServiceを利用して単一ノードで業務を実行し、バッチ処理時のGC待機問題を回避します。
*	19c運用マニュアル（特にPDB環境向け）を整備します。
