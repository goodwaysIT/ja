---
layout: post
title: "OGG for Big Dataを使用したOracle CDCからKafkaへのストリーミング：設定ガイド"
excerpt: "Oracle GoldenGate for Big Dataを使用して、OracleデータベースからKafkaへリアルタイムの変更データキャプチャ（CDC）をストリーミングするためのエンドツーエンドの設定ガイドを学びます。この記事では、アーキテクチャ、主要なパラメータ設定、およびベストプラクティスについて詳しく説明します。"
date: 2025-07-31 14:05:00 +0800
categories: [Oracle GoldenGate, Big Data]
tags: [ogg, big data, kafka, cdc, data integration, json, handler]
author: Shane
image: /assets/images/posts/ogg-for-bigdata-kafka-banner.svg
---

現代のデータアーキテクチャでは、ビジネスの俊敏性にとってデータ統合の速度が重要です。従来のバッチ処理（T+1）は、リアルタイム分析やイベント駆動型マイクロサービスのようなユースケースには不十分なことがよくあります。標準的なアプローチは、データベースの変更データキャプチャ（CDC）とKafkaのようなリアルタイムメッセージングプラットフォームを組み合わせて、効率的なデータパイプラインを構築することです。

Oracle、DB2、PostgreSQLなどのさまざまなデータベースからKafkaへのリアルタイムデータストリームを構築するために、Oracle GoldenGate（OGG）for Big Dataは広く使用されている安定したツールです。

しかし、OGG for Big Dataには大きな学習曲線があります。その設定、特にReplicatプロパティファイルは、従来のOGGとは大きく異なり、多くのパラメータを含んでいるため、新規ユーザーにとっては困難な場合があります。

このガイドでは、明確で再現可能なエンドツーエンドの設定プロセスを提供します。OracleからKafkaへのデータパイプラインを設定するための主要なパラメータをカバーし、OGG for Big Dataの動作メカニズムを説明します。

### 1. アーキテクチャ概要：OGG for Big DataはどのようにKafkaと通信するか

手を汚す前に、まずこのアーキテクチャのワークフローを理解する必要があります。これは従来のOGGアーキテクチャと類似点がありますが、根本的に異なります。

**アーキテクチャ図：**
![OGG for Big Data to Kafka Architecture]({{ '/assets/images/ogg/ogg-bigdata-kafka-architecture.svg' | relative_url }})

1.  **ソース（Oracle）**：従来のOGGと同様に、ソースでプライマリExtractプロセス（通常はIntegrated Extract）を設定して、データベースのRedoログをキャプチャし、ローカルTrailファイルを生成します。
2.  **データ転送**：ソースからOGG for Big Dataサーバー上のリモートTrailファイル（`rt`）にローカルTrailファイル（`lt`）を転送するために、引き続きData Pumpプロセスを使用することを強くお勧めします。これにより、アーキテクチャの分離と堅牢性が確保されます。
3.  **ターゲット（OGG for Big Data + Kafka）**：**これが核心的な違いです**。もはやSQLを直接適用する従来のReplicatプロセスは使用しません。代わりに、特別な**Big Data Replicat**を使用します。このReplicatはどのターゲットデータベースにも接続せず、「**ハンドラ**」をロードすることで外部システムと対話します。このシナリオでは、このハンドラは**Kafkaハンドラ**です。
    *   **Big Data Replicat**：その主な責任は、Trailファイルからデータレコードを読み取ることです。
    *   **Kafkaハンドラ**：Replicatから渡されたデータレコードを受け取り、指定された形式（例：JSON）でKafkaメッセージに変換し、標準のKafka Producer APIを介して指定されたトピックに送信する責任があります。

本質的に、**KafkaハンドラはOGG ReplicatとKafkaクラスタ間のコネクタとして機能します**。

### 2. ハンズオンラボ：エンドツーエンド設定手順（OGG 19c、Kafka 3.7.2）

さあ、袖をまくって作業に取り掛かりましょう。ソースのExtractとPumpはすでに設定済みで、データはOGG for Big Dataサーバーの`./dirdat/rt`ディレクトリに継続的に転送されていると仮定します。

#### ステップ1：Big Data Replicatプロセスの設定

OGG for Big DataのGGSCIで、`ADD REPLICAT`コマンドを使用してターゲットプロセスを追加します。

```sh
-- OGG for Big DataのGGSCIで実行
ADD REPLICAT repkafka, EXTTRAIL ./dirdat/rt
```
*   `ADD REPLICAT repkafka`：役割を明確に定義する`repkafka`という名前のReplicatプロセスを追加します。
*   `EXTTRAIL ./dirdat/rt`：データソースがPumpによって転送されたリモートTrailファイルであることを指定します。

#### ステップ2：Replicatパラメータファイル（`repkafka.prm`）の作成

このパラメータファイルは非常にシンプルです。その主な目的は、Replicatにどのハンドラをロードするか、そしてその設定ファイルがどこにあるかを伝えることです。

```ini
-- repkafka Replicatプロセスのパラメータファイル
REPLICAT repkafka
-- Crucial! TARGETDB LIBFILE libggjava.so SET property=...
-- It tells OGG this is a Java application, with specific configuration in kafka.properties
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafka.properties
-- Use the MAP statement to define the tables to be processed, this is a best practice
MAP source_schema.source_table, TARGET source_schema.source_table;
```
*   **Note**: In a Big Data context, the `TARGET` clause in the `MAP` statement is usually ignored because the real "target" (like a Kafka Topic) is defined in the `.properties` file. However, maintaining the full `MAP ... TARGET ...` structure is a good habit.

#### ステップ3：Kafkaハンドラプロパティファイル（`kafka.properties`）の作成 - 核心中の核心

このファイルは、設定全体の中で最も重要な部分です。接続の詳細からデータ形式、プロデューサー設定まで、Kafkaハンドラのすべての動作を制御します。

```properties
# file: ./dirprm/kafka.properties

# A. Core Handler Configuration: Specify the use of the Kafka Handler
gg.handler.name=kafka
gg.handler.kafka.type=kafka

# B. Kafka Connection Configuration: Specify your Kafka cluster address
# This is one of the most important configurations and must be accurate
gg.handler.kafka.bootstrap.servers=kakfaserver:9092

# C. Message Formatting Configuration: Define the output message format to Kafka
# We use JSON format and include metadata like operation type and timestamp
gg.handler.kafka.format=json
# gg.handler.kafka.format.metaColumns=true
# gg.handler.kafka.format.insertOpKey=I
# gg.handler.kafka.format.updateOpKey=U
# gg.handler.kafka.format.deleteOpKey=D

# D. Topic Routing Configuration: Decide which Topic the data goes into
# This is a very flexible configuration. Here we use a template to automatically use the table name as the Topic name
gg.handler.kafka.topicMappingTemplate=${tableName}
# If you want to send all table changes to the same Topic, you can write:
# gg.handler.kafka.topicName=my_single_topic

# E. Advanced Kafka Producer Configuration (Production-grade config focusing on low latency)
# These parameters are passed directly to the underlying Kafka Producer
# acks=1: The producer gets an acknowledgment after the leader replica has written the message.
# This setting offers a good balance of throughput and low latency, but is slightly less reliable than acks=all.
gg.handler.kafka.producer.acks=1

# Specify the key and value serializers. For JSON format, the OGG Handler internally converts it to a byte array,
# so using ByteArraySerializer is correct.
gg.handler.kafka.producer.value.serializer=org.apache.kafka.common.serialization.ByteArraySerializer
gg.handler.kafka.producer.key.serializer=org.apache.kafka.common.serialization.ByteArraySerializer

# The amount of time to wait before attempting to reconnect to a failed Kafka host (in milliseconds).
gg.handler.kafka.producer.reconnect.backoff.ms=1000

# The batch size. Here it's set to 16KB, a balanced choice.
gg.handler.kafka.producer.batch.size=16384

# linger.ms=0: The producer will send messages immediately with no waiting.
# This minimizes latency and is suitable for scenarios with extremely high real-time requirements.
gg.handler.kafka.producer.linger.ms=0
```

#### ステップ4：起動と検証

すべての準備が整ったら、Replicatプロセスを開始します。

```sh
-- GGSCIで実行
START repkafka
```

次に、Kafkaサーバーにログインし、コマンドラインツールを使用して対応するトピックを消費します。OracleデータベースからリアルタイムでストリーミングされるJSON形式のデータ変更メッセージが表示されるはずです！

```sh
# Kafkaサーバーで実行
kafka-console-consumer.sh --bootstrap-server kakfaserver:9092 --topic source_table --from-beginning
```

出力メッセージは次のようになります：
```json
{
  "table": "SOURCE_SCHEMA.SOURCE_TABLE",
  "op_type": "I",
  "op_ts": "2025-07-30 16:30:00.123456",
  "current_ts": "2025-07-30 16:30:02.789012",
  "pos": "00000000000000123456",
  "after": {
    "ID": 101,
    "NAME": "John Doe",
    "STATUS": "ACTIVE"
  }
}
```

### 3. 高度なトピックとベストプラクティス

*   **データ形式の選択**：JSONは可読性が高いですが、冗長性があります。Avroは高性能でスキーマの進化を組み込みでサポートしていて、大規模な本番環境に適していますが、Schema Registryのサポートが必要です。シナリオに基づいてトレードオフを比較検討してください。
*   **エラー処理**：Kafkaハンドラには強力なエラー処理メカニズムもあります。メッセージの送信に失敗した場合（例：存在しないトピックや権限不足）、プロセスをABENDさせるか、エラーをログに記録して続行するように設定できます。
*   **Acksパラメータの選択**：`acks=1`（ここで設定）は、低遅延と高スループットの良好なバランスを提供します。ビジネスシナリオで極端なデータ信頼性要件があり、単一のメッセージを失うリスクを許容できない場合は、`acks=all`を選択する必要がありますが、これにより一部の遅延が犠牲になります。

### 結論

このガイドでは、OracleからKafkaへのエンドツーエンドのデータ統合プロセスについて説明しました。OGG for Big Dataの機能は、その**ハンドラメカニズム**と関連する**プロパティ設定ファイル**を中心に展開されます。

このデータパイプラインの主要コンポーネントを確認しましょう：

*   **ソースキャプチャ**：**Integrated Extract**プロセスがOracleからのデータ変更をキャプチャします。
*   **データ転送**：**Data Pump**プロセスがデータ転送とアーキテクチャの分離を処理します。
*   **ターゲット配信**：**Big Data Replicat**が**Kafkaハンドラ**をロードして、メッセージをフォーマットし、Kafkaに送信します。
*   **コアロジック**：**`kafka.properties`**ファイルに、Kafkaハンドラのすべての設定ロジックが含まれています。

このプロセスを習得することで、従来のデータベースから最新のデータプラットフォームへリアルタイムデータをストリーミングできるようになり、これはリアルタイムデータサービスを構築するための基本的なスキルです。