---
layout: post
title: "Oracle GoldenGate：各バージョンにおける重要な設定"
excerpt: "Oracle GoldenGate (OGG) を様々なバージョン (10g, 11g, 12c, 19c) で成功裏にデプロイするために不可欠な、Oracleデータベースの重要な設定について深く掘り下げます。8年以上の実プロジェクト経験に基づき、一般的な落とし穴を防ぎ、データの一貫性を確保します。"
date: 2025-06-05 16:00:00 +0800
categories: [Oracle, GoldenGate]
tags: [oracle, goldengate, データベース設定, データレプリケーション, ベストプラクティス, oracle 10g, oracle 11g, oracle 12c, oracle 19c, マルチテナント, pdb, cdb, サプリメンタルロギング, アーカイブログ, 強制ロギング]
# image: /assets/images/posts/your-image-here.jpg #
---

Oracle GoldenGate (OGG) は、異種データベース間のデータレプリケーションおよびリアルタイムデータ統合ツールとして、その強力な機能が業界で広く認識されています。しかし、OGGのパフォーマンスとデータ整合性を発揮するためには、基盤となる[Oracleデータベースの正しい設定]({{ site.baseurl }}/services/oracle-database-administration/)が不可欠です。OGGプロジェクトの実装と運用における8年以上の経験を通じて、私は国営銀行のオフサイト災害復旧や通信事業者のビジネスレポート抽出など、Oracle 10g、11g、12c、および19cデータベースバージョンを利用する複数のコアシステムに深く関わってきました。これらのプロジェクト経験から、データベースレベルでの主要な設定がOGGプロジェクト成功の礎であり、さまざまなデータベースバージョン間の微妙な違いがプロジェクトの遅延や失敗の原因となることが多いことが明らかになりました。この記事では、豊富な実務経験を活用し、これらの主要なOracleデータベースバージョンでOGGをデプロイする際に必要な重要な設定と、その背後にあるロジックについて詳細な分析を提供することを目的としています。

# 1 主要設定と普遍的な原則

データベースのバージョンがどのように進化しても、データベース側でのOGG設定の主要な目的は常に同じです。それは、Extractプロセスがすべてのトランザクション変更を正確、効率的、かつ継続的にキャプチャし、データの一貫性を保証することです。以下は、バージョンを超えて共通する主要な設定項目と、OGGにとってのそれらの重要な役割です。

1.  **ARCHIVELOGモード**: OGG Extractプロセスは、データベースのREDOログを読み取ることによってデータの変更をキャプチャします。データベースがARCHIVELOGモードでない場合、古いREDOログが上書きされ、Extractが履歴の変更を取得できなくなり、データの損失やプロセスの中断につながる可能性があります。これはOGGの正常な運用の基本です。
2.  **FORCE LOGGINGモード**: すべてのデータベース操作（ダイレクトパスインサートのようなNOLOGGINGモードの操作を含む）について、REDOログの生成を強制するモードです。OGG によるデータキャプチャの完全性を保証するために極めて重要です。
3.  **サプリメンタルロギング**:
    *   **最小サプリメンタルロギング**: データベースレベルでの最小要件であり、特に主キーや一意キーのないテーブルにおいて、OGGが変更された行を一意に識別するのに十分な列情報がREDOログに含まれることを保証します。OGGのログマイニングにとって不可欠です。
    *   **主キー/一意キーサプリメンタルロギング**: OGGはデフォルトで、ターゲットテーブルの行を特定し更新するために主キーまたは一意キーを使用します。これを有効にすると、主キーまたは一意キーが変更されたときに、REDOログにすべての主キー列の値が含まれるようになり、キーベースの行特定が維持されます。
    *   **全列サプリメンタルロギング**: テーブルに主キーや一意キーがない場合、またはCDC（変更データキャプチャ）シナリオですべての列への変更を追跡する必要がある場合、このオプションはデータの一貫性を確保するための「万能薬」です。パフォーマンスのいくらかの犠牲を払ってでも、REDOログにすべての列の新旧両方の値が含まれることを保証します。
4.  **データベースパラメータ**:
    *   **`ENABLE_GOLDENGATE_REPLICATION`**: このパラメータは、Oracle 11.2.0.3以降の統合キャプチャモードにとって重要です。データベースにOGGの統合キャプチャプロセスが実行中であることを通知し、データベースが必要な最適化とログ管理を実行できるようにします。
    *   **`STREAM_POOL_SIZE`**: このメモリプールは、主にLogMinerコンポーネントを含むOGGの統合キャプチャモードをサポートします。適切な設定により、ExtractプロセスがトランザクションとLCRを処理するための十分なメモリを確保し、パフォーマンスのボトルネックやプロセスの中断を回避します。
5.  **OGGユーザー権限**: Extractプロセスは、データベースへの接続、データディクショナリのクエリ、REDOログへのアクセス、およびサプリメンタルロギングの管理を行うために、特定のデータベース権限を必要とします。これにより、セキュリティと機能性の両方が保証されます。
6.  **12c/19c PDB の特性**: マルチテナントアーキテクチャでは、OGG設定はCDB（コンテナデータベース）レベルとPDB（プラガブルデータベース）レベルを区別する必要があります。これは、OGGが特定のビジネスデータベースに正しく接続して監視できるかどうかに直接影響します。

# 2 主要なデータベース設定の実践的分析

このセクションでは、さまざまなOracleバージョンにおける主要なデータベース設定の実践的な操作と考慮事項について掘り下げ、これらの設定がOGGにどのように影響するかに焦点を当てます。

## A. ARCHIVELOGモードとFORCE LOGGINGモード

ARCHIVELOGモードは、OGGがデータ変更をキャプチャするための基本です。FORCE LOGGINGモードは、（NOLOGGING属性を使用するものを含む）すべてのデータベース操作がREDOログを生成することを保証し、データ損失を防ぎます。

**10g/11g環境での操作**

*   **ARCHIVELOGモードの確認**: `SELECT LOG_MODE FROM V$DATABASE;`
*   **ARCHIVELOGモードの設定**:
```sql
SHUTDOWN IMMEDIATE;
-- RAC環境の場合、データベースを停止: srvctl stop database -d <databasename>
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

## B. サプリメンタルロギング設定

サプリメンタルロギングは、OGG ExtractプロセスがLCR（論理変更レコード）を正しく解析し、ターゲット行レコードを特定するための鍵となります。これにより、REDOログにOGGが必要とするすべての列情報が含まれることが保証されます。

**10g/11g環境での操作**:

*   **データベースレベルの最小サプリメンタルロギングの有効化**:

```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
-- 確認
SELECT SUPPLEMENTAL_LOG_DATA_MIN FROM V$DATABASE; -- YESであるべき
```
    
*   **主キー/一意キーサプリメンタルロギングの有効化**:

```sql
-- データベースレベル
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY, UNIQUE INDEX) COLUMNS;
-- 確認
SELECT SUPPLEMENTAL_LOG_DATA_PK, SUPPLEMENTAL_LOG_DATA_UI FROM V$DATABASE; -- YES, YESであるべき

-- テーブルレベル (GGSCIでのADD TRANDATA schema.table_nameと同様)
ALTER TABLE schema.table_name ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY, UNIQUE INDEX) COLUMNS;
```

*   **PK/UIのない特定のテーブルに対する全列サプリメンタルロギングの有効化**: GGSCIで`ADD TRANDATA schema.table_name ALLCOLS`を実行することを推奨します。内部的には、これは`ALTER TABLE schema.table_name ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;`のようなSQLコマンドを自動的に実行します。

**12c/19c環境での操作**:

*   **CDBレベル**: CDBルートでは、最小サプリメンタルロギングのみを有効にする必要があります。
```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
```

*   **PDBレベル**: ターゲットPDBに接続後、PDBレベルのサプリメンタルロギング設定を実行します。

```sql
ALTER SESSION SET CONTAINER = pdb_name;
-- 主キー/一意キーサプリメンタルロギングの有効化 (データベースまたはテーブルレベル)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY, UNIQUE INDEX) COLUMNS;
ALTER TABLE schema.table_name ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY, UNIQUE INDEX) COLUMNS;
-- PK/UIのない特定のテーブルについては、PDB内でGGSCIコマンドを介して全列サプリメンタルロギングを有効化
-- GGSCI: ADD TRANDATA schema.table_name ALLCOLS
-- GGSCI: ADD SCHEMATRANDATA schema ALLCOLS 
```

**経験共有**:

ある銀行のOGG移行プロジェクト（**11gコアシステム**から**11g災害復旧システム**への移行）で、チームは複雑な問題に遭遇しました。OGG Extractプロセスは正常に開始されましたが、特定のテーブルに対するUPDATE操作後、Replicatプロセスが`OGG-00868 No unique key found`や`OGG-01403 No row found for LCR update operation`のようなエラーを報告しました。慎重な調査の結果、問題のあるテーブルには主キーも一意キーもないことが判明しました。データベースレベルの最小サプリメンタルロギングは有効になっていましたが、これらのキーなしテーブルに対して`ADD TRANDATA`による`ALLCOLS`が有効になっていませんでした。Extractはログ内でRID（Row Identifier）しかキャプチャできませんでしたが、一意のビジネスキーがないため、ターゲットで行が移動または圧縮されたときにReplicatが正確に行を照合できませんでした。解決策は、主キーのないすべてのテーブルに対して緊急に`ADD TRANDATA schema.table_name ALLCOLS`コマンドを実行することでした。Extractを再起動した後、問題はすぐに解決しました。

**教訓**: 主キー/一意キーなしテーブルに対しては、たとえデータベースレベルのサプリメンタルロギングが有効になっていても、別途全列サプリメンタルロギングを有効にすることが必須です。これがデータの一貫性を確保するための「最低ライン」です。　

## C. OGGデータベースユーザーと権限

OGGプロセスは、データベースに接続し、データディクショナリのクエリやREDOログの読み取りなど、一連の権限を保持する特定のデータベースユーザーIDを必要とし、スムーズなデータキャプチャとレプリケーションを保証します。

*   **10g/11g環境での操作 (非CDB)**:

```sql
CREATE USER ogguser IDENTIFIED BY ogguser_password;
GRANT CONNECT, RESOURCE, CREATE SESSION, ALTER SESSION TO ogguser;
GRANT SELECT ANY DICTIONARY TO ogguser;
GRANT SELECT ANY TABLE TO ogguser; -- 本番環境では、特定のスキーマまたはテーブルに制限することを推奨
GRANT ALTER ANY TABLE TO ogguser; -- DDLレプリケーションが必要な場合
GRANT SELECT ON DBA_CLUSTERS TO ogguser;
GRANT FLASHBACK ANY TABLE TO ogguser;
GRANT EXECUTE ON DBMS_FLASHBACK TO ogguser;
GRANT UNLIMITED TABLESPACE TO ogguser;
-- 11.2.0.3以降では、DBMS_GOLDENGATE_AUTHパッケージを使用して権限付与を簡素化することを強く推奨します:
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('OGGUSER', 'capture', grant_select_privileges=>true);
```

*   **12c/19c環境での操作 (CDB/PDB)**: マルチテナントアーキテクチャにおけるOGGユーザー権限管理はより複雑で、OGGプロセスの接続ターゲットとレプリケーション範囲に依存します。Extractプロセスは通常、CDBのREDOログにアクセスする必要があるため、CDBレベルの共通ユーザーを必要とします。Replicatプロセスは、より頻繁にPDBレベルのローカルユーザーを使用します。

**CDB共通ユーザー (CDB内のすべてのPDBをレプリケートする場合)**: CDBルートで作成し、権限を付与します。ユーザー名は`C##`で始まる必要があります。
```sql
CREATE USER C##ggadmin IDENTIFIED BY <password>
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp CONTAINER=ALL;
GRANT CONNECT, RESOURCE TO C##ggadmin CONTAINER=ALL;
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('C##ggadmin', 'capture', container=>'all');
-- 注意: このユーザーはCONTAINER=ALLを介してすべてのPDBをカバーし、C##プレフィックスを使用する必要があります。
```

**CDB共通ユーザー (特定のPDBのみをレプリケートする場合)**: CDBルートで作成しますが、権限付与時にターゲットPDBを指定します。
```sql
CREATE USER C##ggadmin IDENTIFIED BY <password>
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp; -- 注意: ここではCONTAINER=ALLなし
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('C##ggadmin', 'capture', container=>'<PDB_NAME>');
-- 注意: 権限は指定されたPDBに制限されます。CONTAINER=ALLは使用できません。
```

**PDBレベルのローカルユーザー (補助操作用、例: Replicatプロセス)**: 補助操作（シーケンスレプリケーション、ハートビートテーブル、特定のPDBに接続するReplicatなど）にのみ使用されます。各PDBにローカルユーザーを作成する必要があります。
```sql
ALTER SESSION SET CONTAINER = <PDB_NAME>;
CREATE USER ogguser_local IDENTIFIED BY ogguser_password;
GRANT CONNECT, RESOURCE, CREATE SESSION, ALTER SESSION TO ogguser_local;
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('ogguser_local', 'capture', grant_select_privileges=>true); 
-- ここでの'capture'権限は一般的なテンプレートです。Replicatは実際には'apply'権限が必要な場合があります。
-- Replicatの場合、以下の権限が必要になることがあります:
-- GRANT CREATE ANY TABLE, ALTER ANY TABLE, DROP ANY TABLE TO ogguser_local; (ReplicatがDDL操作を必要とする場合)
-- GRANT UNLIMITED TABLESPACE TO ogguser_local;
```

**経験共有**:

*   ある**通信事業者**の**12c本番環境**アップグレードプロジェクトにおいて、OGG Extractプロセスは特定のPDBに接続するよう設定されていました。しかし、起動後、頻繁に`OGG-00664`（データベース初期化時の致命的エラー）や`OGG-01031`（権限不足）が報告されました。初期調査の結果、OGGユーザーはCDB共通ユーザー（C##OGG）であり、CDBルートでほとんどの権限が付与されていました。問題の原因は、Extractが実際にPDBに接続した際、PDB内のコンテキストを正しく認識できなかったか、あるいは特定のPDB内部ビューへのアクセス権限が不足していたことでした。
*   解決策は、CDB/PDB環境におけるOGGユーザーのベストプラクティスを再評価し、OGGプロセスの接続ターゲット（Extractの場合はCDBルート、Replicat/ユーティリティの場合は特定のPDB）およびレプリケーション範囲に基づいて、作成と権限付与の方法を調整することでした。ExtractプロセスのCDB共通ユーザーが`CONTAINER=ALL`権限を持ち、`DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE`を正しく使用していることを確認しました。

## D. データベースパラメータ設定

*   **`ENABLE_GOLDENGATE_REPLICATION`**
Oracle 11.2.0.3以降の統合キャプチャモードでは、このパラメータをTRUEに設定する必要があります。これにより、データベースがOGGと統合され、必要なログキャプチャの最適化が提供されます。
**各バージョンでの操作**：CDBルートで設定し、通常はすべてのPDBに影響します。
```sql
ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION=TRUE SCOPE=BOTH;
-- 検証
SHOW PARAMETER ENABLE_GOLDENGATE_REPLICATION;
```

*   **`STREAM_POOL_SIZE`**
ストリームプールは、OGG統合キャプチャモードのコアメモリ領域であり、LCRやメタデータなどを格納するために使用されます。適切な設定により、Extractが効率的に実行され、メモリ不足やパフォーマンスのボトルネックを回避できます。
**12c/19c環境での操作**: Extractプロセスの数に基づいて設定することを推奨します。Integrated Extract プロセス1つにつき少なくとも1GBのメモリを割り当て、さらに追加の Extract プロセスごとに200MB～500MBのバッファを追加、または実際の負荷に基づいて調整するのが良いとされています。
```sql
-- CDBルートで設定し、すべてのPDBに影響します (ExtractがCDBに接続する場合)
ALTER SYSTEM SET STREAM_POOLS_SIZE = 2G SCOPE=BOTH; -- 例、実際のニーズに基づいて調整
-- 検証
SHOW PARAMETER STREAM_POOLS_SIZE;
```

*   **`UNDO_RETENTION`**
ログマイニング中、OGG ExtractプロセスはトランザクションのUNDO情報にアクセスして完全な行イメージを構築する必要がある場合があります。`UNDO_RETENTION`パラメータは、トランザクション情報がUNDOセグメントに保持される時間を決定します。
**各バージョンでの操作**： 実際のトランザクション量とExtractの遅延に基づいて、十分に大きな値を設定します。通常、少なくとも3600秒（1時間）にすることを推奨します。長時間実行されるトランザクションやExtractの遅延が大きいシナリオでは、より大きな値が必要です。
```sql
ALTER SYSTEM SET UNDO_RETENTION = 3600 SCOPE=BOTH; -- 例の値
-- 検証
SHOW PARAMETER UNDO_RETENTION;
```

**経験共有**:

ある**大手保険会社**の**19cコアトランザクションシステム**から**19cデータウェアハウス**へのOGGリアルタイム同期プロジェクトを実施した際、PDB設定後、Extractは正常に起動しました。運用が進む中で新たに作成された PDB に対し、Replicat 側で再び同期遅延や中断が発生するようになりました。調査の結果、新しいPDBが標準に従ってサプリメンタルロギングとOGGユーザーを設定していなかったことに加え、元々の比較的小さな`STREAM_POOL_SIZE`も、データ量とトランザクション量の増加によりボトルネックとなり、Extractがログのバックログを発生させていたことが判明しました。

解決策として、プロジェクトプロセス管理を強化し、新規PDB作成後のOGGデータベース設定を必須チェック項目としました。新PDBが本番環境に移行後、DBAチームは直ちにそのPDB内でOGG関連のサプリメンタルロギングとユーザー権限スクリプトを実行する必要があります。同時に、実際の運用データと監視結果に基づき、トランザクション並行性の高まりに対応するため、`STREAM_POOL_SIZE`を2Gから4Gに増やしました。

**教訓**: 19cは安定していますが、マルチテナント環境でのOGG設定は「一度設定すれば終わり」というものではありません。新PDBは独立したOGGソースとして扱い、**その内部のOGG関連設定を個別に完了させる必要**があります。さらに、`STREAM_POOL_SIZE`のようなパフォーマンスパラメータは、実際の負荷に基づいて動的に調整し最適化する必要があります。

# 3 バージョンの互換性とアップグレード

OGGバージョンとOracle Databaseバージョンの互換性は非常に重要です。一般的に、新しいバージョンは古いバージョンをサポートできますが、その逆はできません。例えば、OGG 19cはOracle 11g/12c/19cに接続してデータを抽出できますが、OGG 12cはOracle 19cの新機能を完全にはサポートしていない可能性があります。そのため、データベースバージョンをアップグレードする際には、OGG の互換性とアップグレードパスを事前に計画することが不可欠です。

**データベースアップグレード時のOGG設定移行のキーポイント**:

1.  **OGGプロセスの停止**: データベースをアップグレードする前に、関連するすべてのExtractおよびReplicatプロセスを停止します。
2.  **新バージョンの互換性検証**: 現在のOGGバージョンがターゲットデータベースバージョンと互換性があるか確認します。
3.  **設定の再検証**: データベースアップグレード後、設定コマンドが同じであっても、OGGが必要とするすべてのデータベースパラメータ（特に`ENABLE_GOLDENGATE_REPLICATION`、`STREAM_POOL_SIZE`）、サプリメンタルロギング、およびOGGユーザー権限が新しいデータベースバージョンで有効であることを再検証することが重要です。11gから12c/19cにアップグレードする場合、PDB設定は特に重要です。
4.  **初期ロード**: 大規模なアップグレードの場合、完全なデータ一貫性を確保するために初期ロードが検討されることがあります。

# 4 まとめ

OGGデプロイメントの成功は、主要なデータベース設定の深い理解と厳密な実行にかかっています。経験豊富なOGGアーキテクトとして、私は数え切れないほどの実際の経験から以下のベストプラクティスとチェックリストを抽出しました。

1.  **「六位一体」の原則**: **ARCHIVELOGモード**、**FORCE LOGGING**、**最小サプリメンタルロギング**、**主キー/一意キーサプリメンタルロギング**、**主キーのないテーブルの全列サプリメンタルロギング**、および**OGGユーザー権限**を常に不可分一体として扱います。どれも欠かすことはできません。
2.  **PDBコンテキスト優先**: Oracle 12c以降のマルチテナント環境では、ほとんどのOGG関連データベース設定とローカルユーザー権限付与がターゲットPDB内で実行されるようにします。新しく作成されたPDBは、独立したOGG関連設定を行う必要があります。
3.  **`ENABLE_GOLDENGATE_REPLICATION`はオンであること**: 統合キャプチャの場合、このパラメータは基本です。CDBまたは関連するPDBレベルでTRUEに設定されていることを確認します。
4.  **`STREAM_POOL_SIZE`の動的調整**: Extractプロセスの数と実際のトランザクション負荷に基づいて、`STREAM_POOL_SIZE`を合理的に設定し（Extractプロセスごとに少なくとも1GB + 200MBを推奨）、本番環境での使用状況を継続的に監視して最適化します。
5.  **十分な`UNDO_RETENTION`の確保**: トランザクション特性とOGGの遅延に基づいて適切な`UNDO_RETENTION`期間を設定し、UNDO情報の損失によるExtractの中断を防ぎます。
6.  **標準化されたスクリプトと自動化**: 標準化されたOGGデータベース設定スクリプトのセットを開発および保守します。新しい環境をデプロイしたり、新しいPDBを作成したりする際には、これらのスクリプトを直接実行して人為的エラーを減らします。
7.  **厳密な事前チェックとテスト**: OGGが本番稼働する前に、基本的な接続テストに加えて、**データ一貫性検証**と**ストレステスト**を実行します。本番環境のトランザクション負荷をシミュレートして、OGGがデータを安定かつ効率的にキャプチャおよびレプリケートできることを確認します。
8.  **詳細なエラーログ分析**: 問題が発生した場合は、OGG独自のエラーメッセージだけに注目するのではなく、データベースのアラートログ（alert.log）やデータベースセッショントレースファイルを深く調査します。多くのOGGの問題はデータベースに起因します。
9.  **定期的なヘルスチェック**: アーカイブログスペース、サプリメンタルロギングステータス、Extractプロセスの遅延などを含め、OGGとデータベースの定期的なヘルスチェックメカニズムを確立し、問題が発生する前に防止します。

OGGは強力なツールですが、万能ではありません。異なるOracleデータベースバージョン間のギャップを埋めるには、基礎となるメカニズムの徹底的な理解と、各設定項目の綿密な実行、そして実際のプロジェクト経験が必要です。そうして初めて、OGGはエンタープライズコアシステムで価値を最大限に発揮できます。
