---
layout: page
title: "ORACLE Databaseバックアップ最適化・アップグレード"
description: "本記事では、主に、バックアップ/リカバリ時間の長さ、アーカイブログの容量の多さ、バックアップのタイミングの悪さ、バックアップジョブの分散、データ損失のリスク、バックアップ管理の難しさといった問題を解決するために、既存のバックアップ戦略とバックアップアーキテクチャを最適化する方法をお客様がどのように支援しているかについて紹介します。"
excerpt: "金融機関向けに、Oracleデータベースのバックアップ戦略を全面的に最適化・アップグレード。Oracle ZDLRAを導入し、バックアップ時間の大幅な短縮、管理の一元化、そしてリアルタイム保護によるゼロデータ損失の実現を達成した包括的なプロジェクト事例。"
order: 3
image: /assets/images/case-studies/oracle-backup-optimization-architecture.svg
tags: ["Oracle", "Oracle Exadata", "バックアップ戦略の最適化", "集中管理", "高速リカバリ", "ゼロデータ損失"]
---

# ORACLE Databaseバックアップ最適化・アップグレードプロジェクト

![Oracle Database Backup Optimization Architecture]({{ '/assets/images/case-studies/oracle-backup-optimization-architecture.svg' | relative_url }})

## 背景  
お客様の業務運用の発展に伴い、各データベースのデータ量は徐々に増加しています。同時に、新規業務の急成長により、月に約3～8の新しいデータベースがデプロイされています。データベースのバックアップとリカバリは重大な課題に直面しています：  
(1) **バックアップ時間の長期化**：継続的なデータ増加によりバックアップ時間が延長。バックアップ操作は大量のI/OおよびCPUリソースを消費し、データベースパフォーマンスに影響を与えます。  
(2) **リカバリリスク**：現在、REDOログとデータファイルは毎日まとめてバックアップされます。緊急リカバリ時、テープライブラリからのリストアには、バックアップ時点以降に生成されたREDOログの適用が必要です。これらのログ（ローカルディスクに保存）が破損、ストレージ障害による紛失、または誤削除された場合、ゼロデータロスリカバリは不可能になります。  
(3) **アーカイブログのオーバーヘッド**：バックアップにはデータファイルとアーカイブログが含まれます。トランザクション量の多いデータベースは過剰なアーカイブログを生成し、バックアップ時間をさらに延長します。  
(4) **テープベースの低速リカバリ**：1.7 TBのデータベース（例：顧客情報システム）をテープライブラリからリストアし、REDOログを適用するには2時間以上かかります。重要トランザクションデータベースのリカバリ時間延長は、ユーザーエクスペリエンスと組織の評判に影響します。  
(5) **バックアップ検証の欠如**：テープライブラリリソースが限られているため、バックアップ完全性の日次検証ができず、リカバリ時の破損バックアップ検出漏れリスクがあります。  
(6) **手動管理**：バックアップ設定、監視、実行、リカバリは手動スクリプトに依存し、合理化された管理のための集中ツールが不足しています。
要約すると、既存のORACLEデータベースバックアップは、バックアップ/リカバリ時間の延長、ログ喪失リスク、未検証のバックアップ有効性、非効率な管理といった課題に直面しています。本プロジェクトはこれらの課題に対処します。  

## 目標  
ORACLEバックアップシステムをアップグレードし、以下を達成します：  
(1) **バックアップ戦略の最適化**：日次フルバックアップから、初期フルバックアップ＋日次増分バックアップへの移行。  
(2) **ゼロデータロス**：データベースログのリアルタイムバックアップを実装。  
(3) **高速リカバリ**：リカバリ速度1 GB/s（テープベースリストアの2倍以上）を達成し、ダウンタイムを大幅に削減。  
(4) **集中管理**：  
&nbsp;&nbsp;&nbsp;&nbsp; - ORACLE OEMを使用した統一バックアップポリシー設定、記録保持、容量管理。  
&nbsp;&nbsp;&nbsp;&nbsp; - バックアップアプライアンスでのバックアップスクリプト自動生成/実行により、ローカルサーバースクリプティングを不要化。  

## 実施状況  
### 1. 達成目標  
導入後の成果：  
(1) **バックアップ戦略**：初期フルバックアップ＋日次増分バックアップを展開。  
(2) **速度向上**：日次バックアップ/リカバリ性能の顕著な改善。  
(3) **集中管理**：OEMによる統一バックアップポリシー設定、タスクスケジューリング、結果監視。  
(4) **ゼロデータロス（RPO=0）**：  
&nbsp;&nbsp;&nbsp;&nbsp; - ORACLEデータベースアクティビティログのリアルタイムバックアップ機能を有効化。データベースログはリアルタイムでOracle Exadataバックアップマシンに転送・保存され、データベース障害時にはいつでもExadataマシンからリカバリ可能。RPO=0（データ損失なし）を達成。  
&nbsp;&nbsp;&nbsp;&nbsp; - 現在、データベースのゼロデータロス機能は、テスト環境および本番環境で完全にテスト・検証され、機能要件を満足。  

### 2. 未達目標  
**テープ連携**：契約ではバックアップアプライアンスと物理テープライブラリ（TS4500）の連携が必要でしたが、計画で以下のように変更されました：  
&nbsp;&nbsp;&nbsp;&nbsp; - 長期データ保存用に従来型テープバックアップを維持。  
&nbsp;&nbsp;&nbsp;&nbsp; - 新しいバックアップアプライアンスは10GbEでExadata/PCサーバーに接続し、データベースを直接バックアップ。  

### 3. マイルストーン概要  
(1) **フェーズI**：ハードウェア調達、設置、電源投入、ソフトウェア検証を完了。  
(2) **フェーズII**：非本番環境でのバックアップ/リカバリ機能の徹底テストを完了。  
(3) **フェーズIII**：全ORACLEデータベースに最適化バックアップ（増分戦略、高速リカバリ、集中管理）をアップグレードし、安定稼働中。  

### 4. 主要成果  
(1) **本番環境デプロイ**：48のORACLEデータベースで自動化スケジュールバックアップ（初期フル→日次増分）を実装し、日次バックアップ量を削減。  
(2) **性能向上**：  
&nbsp;&nbsp;&nbsp;&nbsp; - 例：統計レポートデータベースのリカバリ速度が **178 MB/s から 475 MB/s に向上**。リカバリ時間が **16分から6分に短縮**。  
(3) **OEM集中管理**：バックアップポリシー設定、タスクカスタマイズ、結果可視化の統一プラットフォームを提供。  
