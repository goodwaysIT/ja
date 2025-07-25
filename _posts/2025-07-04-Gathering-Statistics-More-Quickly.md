---
layout: post
title: "統計情報をより迅速に収集する方法"
excerpt: "データ量が増加しメンテナンス期間が短縮される中、統計情報をタイムリーに収集することはこれまで以上に重要です。Oracle では、統計収集処理の並列化から、実際にデータを収集するのではなく統計を生成する方法まで、さまざまな方法を提供しています。"
date: 2025-07-04 11:00:00 +0800
categories: [Oracle, Database]
tags: [Database maintenance, Database deployment,Database optimization, oracle]
image: /assets/images/posts/Gathering-Statistics-More-Quickly.jpg
---

データ量が増加しメンテナンス期間が短縮される中、統計情報をタイムリーに収集することはこれまで以上に重要です。Oracle では、統計収集処理の並列化から、実際にデータを収集するのではなく統計を生成する方法まで、さまざまな方法を提供しています。

**並列処理の活用**  
統計収集ではいくつかの方法で並列処理を活用できます  
» DEGREEパラメータの使用  
»同時統計情報収集統計情報の同時収集
» DEGREEと同時収集の組み合わせ  

**DEGREEパラメータの使用**  
DBMS_STATSのDEGREEパラメータは、統計収集に使用される並列実行プロセスの数を制御します。  
デフォルトでは、Oracleはデータディクショナリ内のテーブル属性として指定された並列サーバープロセス数（並列度）と同じ数を使用します。Oracleデータベース内のすべてのテーブルは、デフォルトでこの属性が1に設定されています。大規模なテーブルの統計収集を高速化するために、このパラメータを明示的に設定すると有効です。  

あるいは、DEGREEをAUTO_DEGREEに設定します。Oracleはオブジェクトのサイズに基づいて、統計収集に使用すべき適切な並列サーバープロセス数を自動的に判断します。この値は、小規模オブジェクトの場合は1（直列実行）から、大規模オブジェクトの場合はDEFAULT_DEGREE（PARALLEL_THREADS_PER_CPU X CPU_COUNT）までの範囲で設定されます。

注意点として、パーティションテーブルに対して DEGREE を設定すると、各パーティションに対して複数の並列サーバープロセスが使用されますが、異なるパーティションに対して統計が同時に収集されるわけではありません。統計は各パーティションごとに順番に収集されます。

**並行統計収集**  
並行統計収集により、スキーマ（またはデータベース）の複数のテーブルや、テーブル内の複数の（サブ）パーティションに対して統計を同時に収集できます。複数のテーブルと（サブ）パーティションに対する統計収集を並行して行うことで、Oracle はマルチプロセッサ環境をフルに活用でき、統計収集にかかる全体の時間を短縮できます。

並行統計収集はグローバルプリファレンスCONCURRENTによって制御され、MANUAL、AUTOMATIC、ALL、OFFに設定できます。デフォルトはOFFです。CONCURRENTが有効な場合、Oracle は Oracle Job Scheduler と Advanced Queuing コンポーネントを使用して、複数の統計収集ジョブを並行して作成および管理します。

CONCURRENT が MANUAL または ALL に設定された場合、DBMS_STATS.GATHER_TABLE_STATS をパーティションテーブルで呼び出すと、Oracle はテーブル内の各（サブ）パーティションに対して個別の統計収集ジョブを作成します。これらのジョブが並行して実行される数と、キューに入る数は、使用可能なジョブキュープロセス数（JOB_QUEUE_PROCESSES 初期化パラメータ、RAC 環境での各ノード）およびシステムリソースに基づきます。現在実行中のジョブが完了すると、さらにジョブがキューから取り出されて実行され、すべての（サブ）パーティションの統計が収集されるまで続きます。
  
DBMS_STATS.GATHER_DATABASE_STATS、DBMS_STATS.GATHER_SCHEMA_STATS、または DBMS_STATS.GATHER_DICTIONARY_STATS を使用して統計情報を収集する場合、Oracle は非パーティションテーブルごとに個別の統計収集ジョブを作成し、パーティションテーブルについては各（サブ）パーティションごとにジョブを作成します。また、パーティションテーブルごとに、それらの（サブ）パーティションジョブを管理するコーディネータジョブも作成されます。データベースは可能な限り多くのジョブを並行実行し、残りのジョブは現在のジョブが完了するまでキューに入れられます。

しかし、デッドロックの可能性を防ぐために、複数のパーティションテーブルを同時に処理することはできません。したがって、あるパーティションテーブルに対するジョブが実行中の場合、スキーマ（またはデータベース、ディクショナリ）内の他のパーティションテーブルは、現在の処理が完了するまで待機されます。非パーティションテーブルにはこのような制限はありません。

個々の統計収集ジョブも、DEGREEパラメータが指定されていれば並列実行を活用できます。  

また、テーブル、パーティション、サブパーティションが非常に小さい、あるいは空である場合、データベースはジョブ管理のオーバーヘッドを削減するために、他の小さいオブジェクトとまとめて1つのジョブとして処理することがあります。

**並行統計収集の設定**  
統計情報収集の並列処理設定はデフォルトで無効になっています。以下のようにして有効にできます：
```sql
exec dbms_stats.set_global_prefs('concurrent', 'all')
```
通常の統計収集に必要な権限に加えて、追加権限も必要です。  
ユーザーは以下のジョブスケジューラおよびAQ（Advanced Queuing）権限を持っている必要があります：
» CREATE JOB  
» MANAGE SCHEDULER  
» MANAGE ANY QUEUE  

ジョブスケジューラは内部テーブルとビューをSYSAUX表領域に格納するため、SYSAUX表領域はオンラインである必要があります。最後に、JOB_QUEUE_PROCESSESパラメータは、統計収集プロセスに利用可能（または割り当て済み）なすべてのシステムリソースを完全に活用するように設定する必要があります。並列実行を使用しない場合、JOB_QUEUE_PROCESSESをCPUコア総数の2倍（RAC環境ではノードごとのパラメータ）に設定すべきです。このパラメータはセッションレベル（ALTER SESSION）ではなく、システム全体（ALTER SYSTEM または init.ora ファイル）で適用するようにしてください。

並列実行を利用して並列統計情報収集を行う場合、以下の設定を無効化する必要があります：

```sql
ALTER SYSTEM SET parallel_adaptive_multi_user=false;
```
リソースマネージャーは、例えば以下のようにして有効化する必要があります：

```sql
ALTER SYSTEM SET resource_manager_plan = 'DEFAULT_PLAN';
```
また、並列ステートメントキューイングを有効にすることを推奨します。これにはリソースマネージャーの有効化と、コンシューマーグループ「OTHER_GROUPS」でキューイングが有効化された一時リソースプランの作成が必要です。デフォルトでは、リソースマネージャーはメンテナンス期間中のみ有効化されます。以下のスクリプトは、一時リソースプラン（pqq_test）を作成し、このプランでリソースマネージャーを有効にする方法を示しています。  

```sql
-- DBA権限を持つユーザーとして接続
begin
dbms_resource_manager.create_pending_area();
dbms_resource_manager.create_plan('pqq_test', 'pqq_test');
dbms_resource_manager.create_plan_directive(
'pqq_test',
'OTHER_GROUPS',
'pqq用OTHER_GROUPSディレクティブ',
parallel_target_percentage => 90);
dbms_resource_manager.submit_pending_area();
end;
/
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'pqq_test' SID='*';
```
自動統計情報収集タスクで並列処理を活用する場合、CONCURRENTをAUTOMATICまたはALLに設定してください。新しいORA$AUTOTASKコンシューマーグループが、メンテナンスウィンドウ中に使用されるリソースマネージャープランに追加され、並列統計情報収集がシステムリソースを使いすぎないように保証されます。
