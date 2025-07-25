---
layout: post
title: "Data Pump：単なる転送役ではなく、OGGアーキテクチャ最適化の鍵"
excerpt: "Oracle GoldenGateのData Pumpは単なるデータ転送ツールではありません。疎結合化、ルーティング、堅牢性・柔軟性・セキュリティの向上を実現できます。"
date: 2025-07-21 09:01:00 +0800
categories: [Oracle, Architecture]
tags: [oracle, goldengate, datapump, dba, best-practices, データポンプ, アーキテクチャ, ベストプラクティス, 高可用性]
author: Shane
---


私のキャリアの中で、繰り返し目にしてきたシナリオがあります。それは、OGG構成を「シンプル」にしようとするチームが、プライマリExtractプロセスでデータを直接リモートサーバーに書き込み、Data Pumpを省略するというものです。最初はシステムが順調に稼働し、チームは達成感を得ます。しかし数ヶ月後、ネットワーク状況が変動したり新たなデータ配信要件が発生したりすると、データパイプライン全体が脆弱になり、遅延や中断が発生し、最終的にはアーキテクチャ全体の大規模な見直しが必要になります。

多くのエンジニアにとって、Data Pumpはオプションの「中間役」に見えます。特にシンプルなポイントツーポイント同期では、追加するのが無駄に感じられるでしょう。

今日は、「Pumpは役に立たない」という誤解を解きたいと思います。Data Pumpは単なるデータムーバーではありません。それは洗練されたアーキテクチャーバッファーであり、柔軟なデータルーティングハブであり、付加価値のあるセキュリティオフィサーです。これを無視することは、堅牢でスケーラブル、かつ安全なOGGデータアーキテクチャを構築する上で重要なステップを見逃すことを意味します。

## 1. コアバリュー: 突破不可能な「ファイアウォール」を構築

**「データキャプチャ」と「データ転送」という2つのコア責務を完全に分離します。** これこそがData Pumpの最も重要で、かつ見落とされがちな価値です。

*   **Pumpなしの脆弱なアーキテクチャ**: プライマリExtractプロセスは2つの役割を担います。データベースのRedo/Archiveログからの変更を取得する（CPU集約的）と、データをリモートTrailファイルに送信する（ネットワークI/O集約的）です。

    **アーキテクチャ図（Pumpなし）**:
![No Pump Architecture]({{ '/assets/images/ogg/ogg-no-pump-architecture.svg' || relative_url }})

    致命的な欠陥：**ネットワークの不調がデータキャプチャプロセスに直接影響します。**ソースとターゲット間のネットワークが不安定または切断されると、プライマリExtractはリモートTrailファイルへの書き込みができず、Redoログの消費が遅れ、ログが蓄積し、最悪の場合ソースDBの通常運用にも影響を及ぼします。

*   **Pumpありの堅牢なアーキテクチャ**: Pumpを導入することで役割が明確になります：
    *   **プライマリExtract**: データベースからのデータ取得と**ローカル**Trailファイルへの書き込みだけに集中し、ネットワーク障害の影響を受けません。
    *   **Data Pump**: ローカルTrailファイルからデータを読み取り、リモート側へのネットワーク転送を担当します。

    **アーキテクチャ図（Pumpあり）**:
![With Pump Architecture]({{ '/assets/images/ogg/ogg-with-pump-architecture.svg' || relative_url }})

    Data Pumpは「ファイアウォール」として機能し、ネットワークの不確実性をExtractプロセスから隔離します。ネットワークが数日間ダウンしても、ローカルディスク容量が十分であればExtractはデータ取得を継続できます。ネットワークが復旧すれば、Pumpが中断点から再開し、バックログを迅速に送信します。これこそが**真の堅牢性**です。

## 2. コアバリュー: 複雑なネットワークトポロジーも簡単に対応

ビジネス要件は進化します。今日はA→Bの同期、明日はA→BとC、その翌日はDやEがAに統合される必要が出てきます。Data Pumpはこうしたアーキテクチャの進化に比類なき柔軟性を提供します。

### シナリオ1: データファンアウト

単一のデータソースから複数のターゲットにデータを配信する必要がある場合、Pumpなしではソース側で複数のExtractを動かす必要があり、リソースの無駄や管理の複雑化を招きます。

Pumpを使えば簡単です。1つのExtractがローカルTrailを作成し、複数のPumpプロセスがそれぞれ異なる宛先にデータを送信します。

**コード例（`pump_b.prm` & `pump_c.prm`）**:

```ini
-- Pump to B (pump_b.prm)
EXTRACT pump_b
EXTTRAIL ./dirdat/lt
RMTHOST server_b, MGRPORT 7809
RMTTRAIL /ogg/dirdat/rb
TABLE hr.*;
```

```ini
-- Pump to C (pump_c.prm)
EXTRACT pump_c
EXTTRAIL ./dirdat/lt
RMTHOST server_c, MGRPORT 7809
RMTTRAIL /ogg/dirdat/rc
TABLE hr.*;
```

**アーキテクチャ図（ファンアウト）**:
![Fan-Out Architecture]({{'/assets/images/ogg/ogg-fan-out-architecture-with-data-pump.svg' || relative_url }})

### シナリオ2: データ統合

複数のソースから中央ターゲットにデータを統合する場合、各ソースでExtractとPumpを構成し、すべてのデータをターゲット側の同じReplicatに送信します。ここでPumpは「支流が本流に合流する」役割を果たします。

## 3. コアバリュー: パフォーマンスとセキュリティの「ビュッフェ」

デカップリングやルーティングが「メインディッシュ」だとすれば、Pumpは同期効率やセキュリティを大幅に高める多くの「デザート」も提供します。

### パフォーマンス向上: ネットワーク圧縮

リージョン間やパブリックネットワークでの同期では帯域幅が貴重です。PumpのパラメータファイルでTCP/IP圧縮を有効にすれば、転送データ量を大幅に削減できます。

```ini
-- In pump prm file
EXTRACT pump_to_remote
RMTHOST remote_server, MGRPORT 7809, COMPRESS
RMTTRAIL /ogg/dirdat/rt
...
```
`COMPRESS`パラメータを追加するだけで、OGGは送信前にデータを圧縮し、受信側で解凍します。テキストデータでは3:1以上の圧縮率を実現したこともあり、非常に効果的です。

### セキュリティ: 暗号化通信

信頼できないネットワークを通過する場合、セキュリティは最重要です。Pumpを使えば暗号化も簡単です。

```ini
-- In pump prm file
EXTRACT pump_to_cloud
RMTHOST cloud_server, MGRPORT 7809, ENCRYPT AES128
RMTTRAIL /ogg/dirdat/rt
...
```
`ENCRYPT`パラメータとウォレットの設定により、転送中のデータを暗号化でき、パケットが傍受されても内容は解読されません。

## まとめ: 

| 機能 | Pumpなし（直接Extract） | Pumpあり（推奨） |
| :--- | :--- | :--- |
| **堅牢性** | 低い—ネットワーク障害がソースに影響 | **高い**—キャプチャ/転送を分離し、ネットワーク障害を隔離 |
| **柔軟性** | 非常に低い、拡張困難 | **非常に高い**—ファンアウトや統合が容易 |
| **ソース負荷** | 高い、Extractが多重作業 | **低い**—Extractはキャプチャに専念 |
| **ネットワーク最適化** | なし | **圧縮**で帯域節約 |
| **データセキュリティ** | なし | **暗号化**でデータ保護 |

**本番環境では必ずData Pumpを使いましょう。**

最もシンプルなポイントツーポイント同期であっても、Pumpの設定に数分余分にかけるだけで、将来の拡張性・安定性・セキュリティ・パフォーマンス最適化の余地が大きく広がります。

控えめなData Pumpを過小評価しないでください。それはオプションの付属品ではなく、堅牢なOGGアーキテクチャの礎石です。 
