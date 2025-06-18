---
layout: post
title: "Oracle データベースにおける統計情報の収集方法"
excerpt: "このホワイト ペーパーでは、Oracle データベースでよく見られるシナリオについて、統計をいつどのように収集するかについて詳しく説明します。"
date: 2025-06-16 17:51:00 +0800
categories: [Oracle, Database]
tags: [Database maintenance, Database deployment,Database optimization, oracle]
image: /assets/images/posts/Best-Practices-for-Gathering-Optimizer-Statistics-with-Oracle-Database.jpg
---

**はじめに**   
Oracleにおける統計情報の収集には、自動統計情報収集を使用することが推奨されます。既に確立された手動による統計情報収集手順がある場合は、そちらを使用することもできます。いずれの方法を選択する場合でも、まずデフォルトのグローバル設定がニーズを満たしているかどうかを検討する必要があります。ほとんどの場合、デフォルト設定で問題ありませんが、変更が必要な場合は`SET_GLOBAL_PREFS`で変更できます。変更後、必要に応じてDBMS_STATSの「設定優先度」プロシージャを使用して、グローバルデフォルトをオーバーライドできます。

例えば、増分統計を必要とするテーブルや特定のヒストグラムセットを必要とするテーブルには`SET_TABLE_PREFS`を使用します。
この方法により、統計情報を収集する方法を宣言できるため、個々の「統計収集」操作ごとにパラメータを調整する必要がなくなります。`gather table/schema/database stats`のデフォルトパラメータを自由に使用でき、選択した統計ポリシーが確実に適用されます。さらに、自動統計情報収集と手動統計情報収集を自由に切り替えることが可能になります。

本文では、この戦略の実装方法について説明します。

**自動統計情報収集**  
Oracleデータベースは、統計情報が欠落している、または「陳腐化」（期限切れ）しているデータベースオブジェクトの統計情報を収集します。これは、事前に定義されたメンテナンスウィンドウ中に実行される自動タスクによって行われます。Oracleは内部で統計情報を必要とするデータベースオブジェクトを優先順位付けするため、更新された統計情報を最も必要とするオブジェクトが最初に処理されます。

自動統計情報収集ジョブは`DBMS_STATS.GATHER_DATABASE_STATS_JOB_PROC`プロシージャを使用します。このプロシージャは、他の`DBMS_STATS.GATHER_*_STATS`プロシージャと同じデフォルトパラメータ値を使用します。デフォルト設定はほとんどの場合十分です。ただし、統計情報収集パラメータのデフォルト値を変更する必要が生じることがあり、これは`DBMS_STATS.SET_*_PREF`プロシージャを使用して実現できます。  
パラメータ値は、可能な限り最小のスコープで変更する必要があり、理想的にはオブジェクトごとに変更します。たとえば、特定のテーブルの陳腐化しきい値を変更して（デフォルトの10%ではなくテーブル内の行のわずか5%が変更された場合に統計情報が陳腐化したと見なされるようにする場合）、`DBMS_STATS.SET_TABLE_PREFS`プロシージャを使用して、その1つのテーブルの`STALE_PERCENT`テーブルプリファレンスを変更できます。デフォルト値を最小スコープで変更することで、手動で管理する必要がある非デフォルトパラメータ値の量を制限します。  
以下は、SALESテーブルの`STALE_PERCENT`を5%に変更する例です：  
```sql
exec dbms_stats.set_table_prefs(user,'SALES','STALE_PERCENT','5')
```
設定されたプリファレンスを確認するには、`DBMS_STATS.GET_PREFS`関数を使用できます。この関数は3つの引数を取ります：パラメータ名、スキーマ名、テーブル名です：  
```sql
select dbms_stats.get_prefs('STALE_PERCENT',user,'SALES') stale_percent from dual;

STALE_PERCENT
-------------
5
```

**DBMS_STATSプリファレンスの設定**  
上述のように、必要に応じて自動統計情報収集の動作を変更するために、特定のオブジェクトやスキーマを対象にDBMS_STATSプリファレンスを設定することが可能です。個々の`DBMS_STATS.GATHER_*_STATS`コマンドに特定の非デフォルトパラメータ値を指定することもできますが、推奨されるアプローチは、「ターゲット型」の`DBMS_STATS.SET_*_PREFS`プロシージャを使用して、必要に応じてデフォルトをオーバーライドすることです。  

パラメータのオーバーライドは、以下のプロシージャのいずれかを使用して、テーブル、スキーマ、データベース、またはグローバルレベルで指定できます（`AUTOSTATS_TARGET`と`CONCURRENT`はグローバルレベルでのみ変更可能であることに注意）：  
SET_TABLE_PREFS  
SET_SCHEMA_PREFS  
SET_DATABASE_PREFS  
SET_GLOBAL_PREFS  

従来、よく上書きされるプリファレンスとしては、`ESTIMATE_PERCENT`（サンプリングする行の割合を制御するため）と`METHOD_OPT`（ヒストグラム作成を制御するため）がありました。ただし、後述する理由により、`ESTIMATE_PERCENT`は現在ではデフォルト値のまま使用するのが望ましいとされています。

`SET_TABLE_PREFS`プロシージャは、指定されたテーブルのみに対して、`DBMS_STATS.GATHER_*_STATS`プロシージャで使用されるパラメータのデフォルト値を変更できます。

`SET_SCHEMA_PREFS`プロシージャは、指定されたスキーマ内のすべての既存テーブルに対して、`DBMS_STATS.GATHER_*_STATS`プロシージャで使用されるパラメータのデフォルト値を変更できます。このプロシージャは実際には、指定されたスキーマ内の各テーブルに対して`SET_TABLE_PREFS`プロシージャを呼び出します。`SET_TABLE_PREFS`を使用するため、このプロシージャを実行した後に作成された新しいオブジェクトには影響しません。新しいオブジェクトは、すべてのパラメータに対してグローバルプリファレンス値を取得します。

SET_DATABASE_PREFSプロシージャは、データベース内のすべてのユーザー定義スキーマに対して、DBMS_STATS.GATHER_*_STATSプロシージャで使用されるパラメータのデフォルト値を変更できます。このプロシージャは実際には、各ユーザー定義スキーマの各テーブルに対してSET_TABLE_PREFSプロシージャを呼び出します。SET_TABLE_PREFSを使用するため、このプロシージャを実行した後に作成された新しいオブジェクトには影響しません。新しいオブジェクトは、すべてのパラメータに対してグローバルプリファレンス値を取得します。ADD_SYSパラメータをTRUEに設定することで、Oracle所有のスキーマ（sys、systemなど）を含めることも可能です。

SET_GLOBAL_PREFSプロシージャは、既存のテーブルプリファレンスを持たないデータベース内の任意のオブジェクトに対して、`DBMS_STATS.GATHER_*_STATS`プロシージャで使用されるパラメータのデフォルト値を変更できます。各パラメータは、テーブルプリファレンスが設定されていない場合、またはGATHER_*_STATSコマンドでパラメータが明示的に設定されていない場合、グローバル設定をデフォルトとします。このプロシージャによる変更は、実行後に作成された新しいオブジェクトに影響します。新しいオブジェクトは、すべてのパラメータに対してGLOBAL_PREFS値を取得します。

DBMS_STATS.GATHER_*_STATSプロシージャと自動統計情報収集タスクは、以下の優先順位ルールに従ってパラメータ値を決定します：コマンドで明示的に設定されたパラメータ値が他のすべてを上書きします。コマンドでパラメータが設定されていない場合、テーブルレベルのプリファレンスを確認します。テーブルプリファレンスが設定されていない場合、グローバルプリファレンスを使用します。  

DBMS_STATS.GATHER_*_STATSのパラメータ値の優先順位：  
GATHER_%_STATS パラメータ > テーブルプリファレンス > グローバルプリファレンス

**Oracle Database 12c Release 2では、PREFERENCE_OVERRIDES_PARAMETERという新しいDBMS_STATSプリファレンスが導入されました。**  
このプリファレンスをTRUEに設定すると、プリファレンス設定がDBMS_STATSパラメータ値を上書きできるようになります。たとえば、グローバルプリファレンスESTIMATE_PERCENTがDBMS_STATS.AUTO_SAMPLE_SIZEに設定されている場合、これは既存の手動統計情報収集手順が（10%の固定サンプリング率など）異なるパラメータ設定を使用していても、このベストプラクティスの設定が使用されることになります。  

PREFERENCE_OVERRIDES_PARAMETERプリファレンスを使用した場合の優先順位：  
テーブルプリファレンス > グローバルプリファレンス > GATHER_%_STATS パラメータ  

**ESTIMATE_PERCENT**  
ESTIMATE_PERCENTパラメータは、統計情報の計算に使用される行の割合を決定します。  
テーブル内のすべての行が処理されたとき（100%サンプル）に最も正確な統計情報が収集され、これは計算統計情報（computed statistics）と呼ばれます。
Oracle Database 11gでは、ハッシュベースで決定論的な統計情報を提供する新しいサンプリングアルゴリズムが導入されました。この新しいアプローチは、100%サンプルに近い精度を持ちながら、コストは最大でも10%サンプル相当です。  
この新しいアルゴリズムは、DBMS_STATS.GATHER_*_STATSプロシージャのいずれかでESTIMATE_PERCENTがAUTO_SAMPLE_SIZE（デフォルト）に設定されている場合に使用されます。Oracle Database 11g以前では、DBAは統計情報収集を確実に迅速に行うために、ESTIMATE_PERCENTパラメータを低い値に設定することがよくありました。しかし、詳細なテストなしでは、正確な統計情報を得るためにどのサンプルサイズを使用すべきかを知るのは困難です。  
Oracle Database 11g以降では、ESTIMATE_PERCENTにデフォルトのAUTO_SAMPLE_SIZEを使用することを強く推奨します。なぜなら、新しいOracle Database 12cのヒストグラムタイプ（HYBRIDやTop-Frequency）は、自動サンプルサイズが使用された場合にのみ作成できます。  
多くのシステムにはまだ、手動で推定割合（estimate percent）を設定する古い統計情報収集スクリプトが含まれているため、Oracle Database 12cリリース2にアップグレードする際は、PREFERENCE_OVERRIDES_PARAMETERプリファレンス（上記参照）を使用して自動サンプルサイズの使用を強制することを検討すべきです。  

**METHOD_OPT**  
METHOD_OPTパラメータは、統計収集中にヒストグラムを作成するかどうかを制御します。ヒストグラムは、テーブル列内のデータ分布に関するより詳細な情報を提供するために作成される特別な列統計です。  
METHOD_OPT のデフォルト値（推奨値）は「FOR ALL COLUMNS SIZE AUTO」です。これは、ヒストグラムの作成が有益と判断された列に対してのみ自動的に作成されます。列が等値述語や範囲述語（例：WHERE col1 = 'X' や WHERE col1 BETWEEN 'A' AND 'B'）で使用され、特に列値の分布に偏りがある場合、その列はヒストグラムの候補となります。この情報がディクショナリテーブル SYS.COL_USAGE$ で追跡・保存されているため、オプティマイザはどの列が検索条件で使用されているかを把握しています。  
一部のDBAは、ヒストグラムがいつ、どのように作成されるかを厳密に制御することを好みます。これを実現する推奨アプローチは、SET_TABLE_PREFSを使用して、テーブルごとにどのヒストグラムを作成するかを指定することです。    
たとえば、SALESテーブルにcol1とcol2のみヒストグラムを持たせるように指定する方法は次のとおりです：  

```SQL  
begin
dbms_stats.set_table_prefs(
user,
'SALES',
'method_opt',
'for all columns size 1 for columns size 254 col1 col2');
end;
/
```  

ヒストグラムを持たせる必要がある列（col1とcol2）を指定し、さらにオプティマイザが追加のヒストグラムが有用かどうかを判断できるようにすることも可能です：  
```sql
begin
dbms_stats.set_table_prefs(
user,
'SALES',
'method_opt',
'for all columns size auto for columns size 254 col1 col2');
end;
/
```

METHOD_OPTが'FOR ALL COLUMNS SIZE 1'に設定されている場合、ヒストグラムの作成は無効になります。たとえば、METHOD_OPTのDBMS_STATSグローバルプリファレンスを変更して、デフォルトではヒストグラムが作成されないようにすることができます：

```sql
begin
dbms_stats.set_global_prefs(
'method_opt',
'for all columns size 1');
end;
/
```
不要なヒストグラムは、DBMS_STATS.DELETE_COLUMN_STATSを使用し、col_stat_typeを'HISTOGRAM'に設定することで個別に削除できます。  

**手動統計情報収集**  
既に確立された統計情報収集手順がある場合、または何らかの理由でメインアプリケーションスキーマの自動統計情報収集を無効にしたい場合は、ディクショナリテーブル向けの自動収集は有効のままにすることを推奨します。これは、DBMS_STATS.SET_GLOBAL_PREFSプロシージャを使用して、AUTOSTATS_TARGETパラメータの値をAUTOからORACLEに変更することで実現できます。  
`exec dbms_stats.set_global_prefs('autostats_target','oracle')`  

統計情報を手動で収集するには、PL/SQLのDBMS_STATSパッケージを使用する必要があります。廃止予定のANALYZEコマンドは使用してはいけません。DBMS_STATSパッケージは、DBMS_STATS.GATHER_*_STATSプロシージャ群を提供し、ユーザースキーマオブジェクト・ディクショナリ・固定オブジェクトの統計情報を収集します。理想的には、スキーマ名とオブジェクト名以外のパラメータはすべてデフォルト値のままにすべきです。Oracleが選択するデフォルト値と適応型パラメータ設定はほとんどのケースで十分です：
`exec dbms_stats.gather_table_stats('sh','sales')`  

上述のように、統計情報収集パラメータのデフォルト値を変更する必要が生じた場合は、DBMS_STATS.SET_*_PREFプロシージャを使用して、可能な限り最小のスコープ、理想的にはオブジェクトごとに変更を行ってください。  

**保留統計情報（Pending Statistics）**  
`DBMS_STATS.GATHER_*_STATS`プロシージャのパラメータのデフォルト値を変更する場合、本番環境で変更を行う前に、それらの変更を検証することを強くお勧めします。フルスケールのテスト環境がない場合は、保留統計情報を活用すべきです。保留統計情報では、統計情報は通常のディクショナリテーブルではなく保留テーブルに格納されるため、公開してシステム全体で使用される前に、管理された方法で有効化してテストできます。保留統計情報収集を有効にするには、保留統計情報を作成したいオブジェクトに対して、DBMS_STATS.SET_*_PREFSプロシージャのいずれかを使用して、パラメータPUBLISHの値をTRUE（デフォルト）からFALSEに変更する必要があります。  
以下の例では、SHスキーマのSALESテーブルで保留統計情報を有効にし、その後SALESテーブルの統計情報を収集しています：  
`exec dbms_stats.set_table_prefs('sh','sales','publish','false')`  

通常通りオブジェクトの統計情報を収集します：  
`exec dbms_stats.gather_table_stats('sh','sales')`  

これらのオブジェクトに対して収集された統計情報は、USER_*_PENDING_STATSというディクショナリビューを使用して表示できます。
初期化パラメータOPTIMIZER_USE_PENDING_STATSをTRUEに設定するalter sessionコマンドを発行することで、保留統計情報の使用を有効にできます。保留統計情報を有効にした後、このセッションで実行されるSQLワークロードは、新しい未公開の統計情報を使用します。ワークロードでアクセスされるテーブルに保留統計情報がない場合、オプティマイザは標準のデータディクショナリテーブル内の現在の統計情報を使用します。保留統計情報を検証した後、DBMS_STATS.PUBLISH_PENDING_STATSプロシージャを使用してそれらを公開できます。  
`exec dbms_stats.publish_pending_stats('sh','sales')`  

出典: https://www.oracle.com/docs/tech/database/technical-brief-bp-stats-gather-0218.pdf
