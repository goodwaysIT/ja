---
layout: post
title: "Oracleメモリ管理の解読：SGAとPGAの最適化設定実践ガイド"
excerpt: "この記事は、多くのDBAが抱える核心的な課題、すなわちSGAとPGAチューニングに関する混乱に取り組みます。この混乱はリソースの無駄遣いやORA-04031のような重大なエラーを引き起こしがちです。本稿ではSGAとPGAの内部動作を解明し、推測から正確な設定へと移行するための体系的でデータ駆動型の手法を提供します。"
date: 2025-08-28 09:30:00 +0800
categories: [Oracle, パフォーマンスチューニング, メモリ管理]
tags: [Oracle, DBA, SGA, PGA, ORA-04031, パフォーマンスチューニング, v$sga_target_advice, v$pga_target_advice, AMM, ASMM]
author: Shane
image: /assets/images/posts/oracle_execution_plan_overview.svg
---

Oracleデータベースのパフォーマンスは、メモリ設定に大きく依存します。しかし、SGAとPGAはブラックボックスのようで、大きくしすぎるとリソースの無駄を恐れ、小さすぎるとクラッシュを心配します。あなたも`ORA-04031`エラーに直面し、途方に暮れたことはありませんか？心配は無用です。このメモリ探求の旅は、あなたを混乱から導き出します。本稿では、SGAとPGAの内部原理から説き起こし、最終的には推測に頼らない、定量的でデータ駆動型な最適化手法を解説します。

### I. SGAとPGA：データベースの心臓と脳

Oracleの世界では、メモリはパフォーマンスの生命線です。すべてのデータ処理、SQL解析、トランザクション記録はメモリの支えなしには成り立ちません。その中で最も重要な役割を担うのが、SGA（System Global Area）とPGA（Program Global Area）です。

#### 1. SGA（システムグローバル領域）：共有のセントラルハブ

SGAは、Oracleインスタンスの起動時に割り当てられる巨大な共有メモリ領域で、すべてのデータベースプロセスがアクセスできます。これは、企業のセントラルオペレーションホールに例えることができ、様々な部署（プロセス）がここでリソースを共有し、協調して作業します。

*   **概念解読 -> 原理分析**
    *   **データベースバッファキャッシュ (Buffer Cache)**: SGAの中で最も大きく、最も重要な部分です。これは、先ほどの「オペレーションホール」で最も忙しい「公共閲覧室」にあたります。ユーザーがデータを読み取る必要がある場合、Oracleはまずここを探します。データが見つかれば（キャッシュヒット）、メモリから直接返され、高コストなディスクI/Oを回避できます。見つからなければ、ディスクからデータを読み取り、将来の使用に備えてバッファキャッシュに格納します。適切に構成されたバッファキャッシュは、読み書き性能を大幅に向上させることができます。
    *   **共有プール (Shared Pool)**: これは企業の「中央会議室兼アーカイブ室」です。主に2つのものをキャッシュします。SQLおよびPL/SQLコードの実行計画（ライブラリキャッシュ）とデータディクショナリ情報（データディクショナリキャッシュ）です。SQL文が発行されると、Oracleはまず共有プール内で全く同じSQLの実行計画がないか探します。もしあれば、それを再利用し、再解析と最適化のプロセスを省略します。このプロセスは「ソフトパース」と呼ばれます。断片化が深刻、あるいは領域が不足している共有プールは、大量の「ハードパース」を引き起こし、CPUを著しく消費します。
    *   **ログバッファ (Log Buffer)**: これは企業の「発送前文書一時保管場所」です。すべてのDML操作（INSERT, UPDATE, DELETE）によって生成されたREDO情報は、まずこのバッファに高速に書き込まれ、その後LGWRプロセスによってREDOログファイルに一括で書き込まれます。通常、サイズはそれほど大きくする必要はありませんが、DML処理が頻繁に行われる高並行性システムでは、適切なログバッファが不可欠です。
    *   **ラージプール (Large Pool)**: これはオプションの「多目的ホール」で、RMANのバックアップリカバリやパラレルクエリなど、大きな連続したメモリ領域を必要とする操作のために予約されています。これを共有プールから分離することで、共有プールの断片化を効果的に防ぐことができます。

*   **チューニング実践/問題診断**
    SGAを十分に大きくすればパフォーマンスが必ず向上するというは、よくある誤解です。しかし、大きすぎるSGAは物理メモリを無駄にするだけでなく、管理オーバーヘッドによって悪影響を及ぼす可能性もあります。私たちはデータに基づいて意思決定を行う必要があります。

#### 2. PGA（プログラムグローバル領域）：独立したプライベートオフィス

SGAの「共有」という性質とは異なり、PGAは各サーバープロセスに割り当てられるプライベートなメモリ領域であり、互いに干渉しません。これは、会社が各従業員に割り当てる独立したオフィスと考えることができます。

*   **概念解読 -> 原理分析**
    SQLがソート（`ORDER BY`, `GROUP BY`）やハッシュ結合（Hash Joins）を行う必要がある場合、十分なPGAメモリが割り当てられていれば、これらの操作はメモリ内で効率的に完了します（Optimal Execution）。PGAが不足している場合、Oracleはディスク上の一時表領域を使用せざるを得ず、「ワンパス実行」や「マルチパス実行」となり、パフォーマンスが急激に低下します。

*   **チューニング実践/問題診断**
    PGAは大きければ大きいほど良いというのもよくある誤解です。しかし、PGAの総消費量は`pga_aggregate_target`に同時実行プロセス数を掛けたものであることを忘れてはなりません。制御不能なPGA設定は、サーバーの全メモリを簡単に使い果たしてしまう可能性があります。パフォーマンスとリソース消費のバランスを見つける必要があります。

### II. 自動管理か手動か？AMM vs ASMM

Oracleは、DBAを煩雑な手動パラメータ調整から解放するため、主に2つの自動メモリ管理メカニズムを提供しています。

*   **自動メモリ管理 (AMM - Automatic Memory Management)**: `MEMORY_TARGET`と`MEMORY_MAX_TARGET`パラメータを設定することで有効になります。OracleはSGAとPGAの間で動的にメモリ割り当てを調整します。これは「完全自動運転」のようなもので、シンプルで便利であり、ワークロードが比較的安定しているシステムに適しています。
*   **自動共有メモリ管理 (ASMM - Automatic Shared Memory Management)**: `SGA_TARGET`と`PGA_AGGREGATE_TARGET`を設定することで有効になります。OracleはSGAの内部コンポーネント（バッファキャッシュ、共有プールなど）間でのみ動的に調整し、PGAのサイズは`PGA_AGGREGATE_TARGET`によって独立して制御されます。これは「運転支援システム」に似ており、DBAがSGAとPGAの総量をより強力に制御できます。

**私の個人的な経験では、ほとんどの本番環境でASMMの使用を推奨しています。** なぜなら、安定性と柔軟性のバランスが良く、AMMで極端な負荷下で発生しうるSGAとPGA間の激しいメモリの「奪い合い」を避け、より予測可能なパフォーマンスを提供するためです。

*   **実践デモ**：現在のメモリ管理モードを確認する方法は？

```sql
-- Oracle 19c
-- MEMORY_TARGET, SGA_TARGET, PGA_AGGREGATE_TARGETの値を確認
SHOW PARAMETER TARGET;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
archive_lag_target                   integer     0
db_big_table_cache_percent_target    string      0
db_flashback_retention_target        integer     720
fast_start_io_target                 integer     0
fast_start_mttr_target               integer     0
memory_max_target                    big integer 0
memory_target                        big integer 0
parallel_servers_target              integer     768
pga_aggregate_target                 big integer 10G
sga_target                           big integer 40G
target_pdbs                          integer     0

-- 論理的な判断:
-- 1. MEMORY_TARGETが0以外の値に設定されている場合、AMMモードです。
-- 2. MEMORY_TARGETが0で、SGA_TARGETが0以外の値に設定されている場合、ASMMモードです。
-- 3. 3つすべてが0の場合、手動管理モードです（非推奨）。
```

### III. データ駆動：Advisorビューに答えを求めよう

データベースの最適化において、感覚や推測は無価値です。Oracleは、インスタンス起動後の実際のワークロードデータに基づき、正確なチューニング推奨を提供する強力な「Advisorビュー」群を提供しています。

#### 1. SGAターゲット・アドバイザ (`v$sga_target_advice`)

このビューは、異なる`SGA_TARGET`サイズがデータベース全体の物理読み取り（ひいてはDB Time）に与える影響を予測します。

*   **実践デモ**

```sql
-- SGAサイズ調整による効果を予測
-- ESTD_DB_TIMEは推定DB時間、ESTD_PHYSICAL_READSは推定物理読み取り
SELECT
    sga_size,
    sga_size_factor,
    estd_db_time,
    estd_physical_reads
FROM
    v$sga_target_advice
ORDER BY
    sga_size_factor;

  SGA_SIZE SGA_SIZE_FACTOR ESTD_DB_TIME ESTD_PHYSICAL_READS
---------- --------------- ------------ -------------------
     15360            .375     57884978          9.8523E+10
     20480              .5     40363977          4.5410E+10
     25600            .625     31660469          1.8920E+10
     30720             .75     29475772          1.2152E+10
     35840            .875     28610035          9647829746
     40960               1     27139096          5163132691
     46080           1.125     26354776          2766922809
     51200            1.25     26270645          2397242508
     56320           1.375     26251648          2397242508
     61440             1.5     26248934          2162836284
     66560           1.625     26248934          1770438200
     71680            1.75     26246220          1345512379
     76800           1.875     26243506          1181324760
     81920               2     26243506          1181324760

14 rows selected.
	
```
**結果の解釈**：`SGA_SIZE_FACTOR`が1の行に注目してください。これが現在の設定です。`SGA_SIZE`を増やす（`SGA_SIZE_FACTOR` > 1）ことで`ESTD_DB_TIME`が大幅に減少するかどうかを確認します。SGAを2倍にしてもDB Timeが1%しか減少しない場合、その投資対効果は低いです。逆に、SGAを20%減らすとDB Timeが急増する場合、現在のSGA設定は不十分かもしれません。

#### 2. PGAターゲット・アドバイザ (`v$pga_target_advice`)

このビューは`PGA_AGGREGATE_TARGET`設定の妥当性を評価するために使用されます。

*   **実践デモ**

```sql
-- PGAサイズ設定の評価
-- ESTD_PGA_CACHE_HIT_PERCENTAGE: 推定PGAキャッシュヒット率
-- ESTD_OVERALLOC_COUNT: 推定PGA過剰割り当て回数
SELECT
    pga_target_for_estimate / 1024 / 1024 AS target_mb,
    pga_target_factor,
    advice_status,
    estd_pga_cache_hit_percentage,
    estd_overalloc_count
FROM
    v$pga_target_advice;

 TARGET_MB PGA_TARGET_FACTOR ADV ESTD_PGA_CACHE_HIT_PERCENTAGE ESTD_OVERALLOC_COUNT
---------- ----------------- --- ----------------------------- --------------------
      1280              .125 ON                             53              1694600
      2560               .25 ON                             53              1694587
      5120                .5 ON                             55              1521626
      7680               .75 ON                             66              1036977
     10240                 1 ON                             99               127685
     12288               1.2 ON                            100                  461
     14336               1.4 ON                            100                  141
     16384               1.6 ON                            100                    4
     18432               1.8 ON                            100                    0
     20480                 2 ON                            100                    0
     30720                 3 ON                            100                    0
     40960                 4 ON                            100                    0
     61440                 6 ON                            100                    0
     81920                 8 ON                            100                    0

14 rows selected.
	
```

**結果の解釈**：目標は`ESTD_PGA_CACHE_HIT_PERCENTAGE`をできるだけ100%に近づけ、同時に`ESTD_OVERALLOC_COUNT`（PGAメモリの過剰割り当て。システムの不安定化を招く可能性あり）を0に保つことです。現在の設定（`PGA_TARGET_FACTOR`=1）のヒット率が低い場合、より大きなファクタを持つ行の推奨値を参考に`PGA_AGGREGATE_TARGET`を増やすことを検討します。

#### 3. その他の重要なビュー

*   **`v$pgastat`**: PGAメモリのリアルタイムな使用状況スナップショットを提供します。

```sql
-- PGAメモリ使用統計の確認
SELECT name, value/1024/1024 as value_mb, unit FROM v$pgastat;

NAME                                      VALUE_MB UNIT
--------------------------------------- ---------- -------
aggregate PGA target parameter               10240 bytes
aggregate PGA auto target                      640 bytes
global memory bound                     1023.99414 bytes
total PGA inuse                         11122.7998 bytes
total PGA allocated                      11959.834 bytes
maximum PGA allocated                   18023.3662 bytes
total freeable PGA memory                  501.375 bytes
MGA allocated (under PGA)                     1236 bytes
maximum MGA allocated                         1236 bytes
process count                            .00057888
max processes count                     .000677109
PGA memory freed back to OS              716819546 bytes
total PGA used for auto workareas           .96875 bytes
maximum PGA used for auto workareas     6710.36914 bytes
total PGA used for manual workareas              0 bytes
maximum PGA used for manual workareas     28.21875 bytes
over allocation count                   .401830673
bytes processed                          294570439 bytes
extra bytes read/written                3046394.24 bytes
cache hit percentage                    .000094385 percent
recompute count (total)                 6.36017418

21 rows selected.

-- 主要な指標：
-- 'aggregate PGA target parameter': 現在のPGA_AGGREGATE_TARGET設定
-- 'total PGA allocated': 現在割り当てられているPGA総サイズ
-- 'maximum PGA allocated': インスタンス起動後のPGA割り当てのピーク値
```
このビューは、PGAのピーク使用量がターゲット値に近づいているか、あるいは超えているかを判断するのに役立ちます。

*   **`v$sga_dynamic_components`**: ASMMモード下でのSGA各コンポーネントのサイズ変化を示します。

```sql
-- SGA各コンポーネントの動的リサイズ状況を確認
SELECT
    component,
    current_size / 1024 / 1024      AS current_size_mb,
    min_size / 1024 / 1024          AS min_size_mb,
    max_size / 1024 / 1024          AS max_size_mb,
    last_oper_type || ' ' || last_oper_mode AS last_operation
FROM
    v$sga_dynamic_components
WHERE
    current_size > 0;


COMPONENT                CURRENT_SIZE_MB MIN_SIZE_MB MAX_SIZE_MB LAST_OPERATION
------------------------ --------------- ----------- ----------- ----------------
shared pool                         7680        6912        8704 GROW DEFERRED
large pool                           640         640         640 STATIC
java pool                            128         128         128 STATIC
streams pool                         128         128         256 SHRINK DEFERRED
DEFAULT buffer cache               32128       30976       32896 SHRINK DEFERRED
Shared IO Pool                       128         128         128 STATIC

6 rows selected.	
```
これは、ASMMが内部でどのようにメモリを再配置しているかを理解するのに役立ちます。

### IV. 定番の障害診断：ORA-04031の分析

`ORA-04031: unable to allocate X bytes of shared memory` は、DBAにとって最も厄介なエラーの一つです。これは通常、共有プール（Shared Pool）の領域不足または深刻な断片化を指し示します。

*   **根本原因分析**:
    1.  **共有プールが小さすぎる**: `SGA_TARGET`の不適切な設定により、現在のSQL解析やデータディクショナリのキャッシュ要求に対して共有プールに割り当てられるメモリが不足している。
    2.  **深刻な断片化**: バインド変数を使用しないSQL（ハードパース）が大量に存在すると、共有不可能な実行計画が多数生成されます。これらは急速に共有プールを埋め尽くし、断片化させるため、合計の空き領域が十分であっても、新しいメモリ要求を満たすための十分な大きさの*連続した*領域が見つからなくなります。
    3.  **バグまたはメモリリーク**: まれに、Oracleのバグが原因である可能性があります。

*   **特定と解決策**:
    1.  **応急処置**: `ALTER SYSTEM FLUSH SHARED_POOL;` は一時的に問題を緩和できますが、これは対症療法にすぎません。キャッシュされているすべての実行計画がクリアされるため、その後しばらくシステムパフォーマンスが不安定になります。
    2.  **恒久的な解決策**:
        *   **バインド変数の使用を徹底**: これが最も重要です。アプリケーションコードを見直し、すべてのSQLでバインド変数を使用するようにし、ハードパースを根本から減らします。`v$sqlarea`を照会し、`EXECUTIONS`が低いにもかかわらず`VERSION_COUNT`が多いSQLを特定できます。
        *   **適切な設定**: `v$sga_target_advice`の推奨に基づき、`SGA_TARGET`のサイズを適切に増やします。
        *   **カーソルキャッシュの使用**: `SESSION_CACHED_CURSORS`パラメータを設定することで、セッションレベルのカーソルをキャッシュし、ソフトパースのオーバーヘッドを削減できます。

### まとめ

Oracleのメモリ管理は不可解なものではありません。その核心は、SGAとPGAの協調メカニズムを理解し、経験駆動からデータ駆動へと移行することにあります。

*   **理解が基礎**: バッファキャッシュ、共有プール、PGAそれぞれの役割を明確に理解することが、正しい意思決定の前提です。
*   **Advisorは科学的な手法**: 推測をやめ、`v$sga_target_advice`や`v$pga_target_advice`などのビューを使いこなし、Oracleにデータに基づいた最適化の提案をさせましょう。
*   **問題解決が目的**: `ORA-04031`などの典型的な問題を起点として逆追跡し、システムのメモリ使用におけるボトルネックや非効率な点を見つけ出しましょう。

メモリの最適化は、一度きりのタスクではなく、監視、分析、調整を繰り返す継続的なプロセスです。この記事で学んだ手法を日々の運用に取り入れることで、あなたのデータベースはより安定した堅牢なパフォーマンスで応えてくれるでしょう。