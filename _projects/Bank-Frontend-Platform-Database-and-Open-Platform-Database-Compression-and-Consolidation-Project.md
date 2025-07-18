---
layout: page
title: "銀行フロントエンド基盤データベースとオープンプラットフォームデータベース圧縮・統合プロジェクト"
description: ""
excerpt: "複数のスキーマ、アプリケーション、またはデータベースがターゲットシステム上でホストされている場合、ハードウェアリソースの使用率は、統合と圧縮によって改善でき、アイドル時間サイクルを最小限に抑えることができます。その結果、より少ないリソースで同じ出力を達成でき、コスト削減につながります。"
order: 8
image: /assets/images/case-studies/Bank-Frontend-Platform-Database-and-Open-Platform-Database-Compression-and-Consolidation-Project.svg
tags: ["Oracle", "Oracle Exadata", "Database Migration","Resource compression"]
---

### プロジェクト背景  
複数のスキーマ、アプリケーション、またはデータベースがターゲットシステム上でホストされている場合、ハードウェアリソースの使用率は、統合と圧縮によって改善でき、アイドル時間サイクルを最小限に抑えることができます。その結果、より少ないリソースで同じ出力を達成でき、コスト削減を実現します。

### プロジェクト目標  
圧縮と統合の目的は、ビジネスの重要性、データベースバージョン、日次取引量、メモリ使用量、データベースサイズ、データ同期、災害復旧要件に基づいてOracleデータベースを分類・圧縮することです。これにより、リスクを制御可能にしつつ、オフピーク時間帯のリソース活用を最大化します。  
計画では、3セットのOracle Exadataデータベースと10セットのPCサーバーデータベース（主にメモリ圧縮に焦点を当て、CPUとストレージリソースは部分的に制約あり）の圧縮・統合を含みます。5セットのPCサーバーの再利用を見込んでいます。  

### プロジェクト作業内容  
データベース圧縮・統合計画は3段階で構成されます：データベースリソース圧縮、データベース移行、PCサーバー再利用。  
フェーズ 1: データベースリソース圧縮  -- 既存および再利用可能なデータベースリソースの評価。  
フェーズ 2: データベース移行          -- フェーズ1の評価結果に基づく段階的移行・統合計画の策定と実行。  
フェーズ 3: PCサーバー再利用          -- データベース移行/統合完了および安定稼働後（約1ヶ月）のPCサーバー再利用。  

### プロジェクト実施状況  
Oracle Exadataデータベース3セットとPCサーバーデータベース10セットに対する圧縮・統合作業が完了しました（主にメモリ圧縮を実施、CPUとストレージリソースは比較的余裕あり）PCサーバー5セットの回収を見込んでいます。
1.  **Exadata 3セットのリソース圧縮**  
    -- Exadata上の現行リソース使用状況を確認。  
    -- Exadata上で統合対象データベースを圧縮。  
2.  **PCサーバー5台のリソース圧縮**  
    -- PCサーバ上の現行リソース使用状況を確認。  
    -- PCサーバー上で統合対象データベースを圧縮。  
3.  **データベース移行と統合**  
    -- PCサーバーからExadataデータベースへの移行・統合。  
    -- PCサーバーからPCサーバーデータベースへの移行・統合。  
4.  **PCサーバー再利用**  
