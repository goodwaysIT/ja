---
layout: page
title: "銀行向けOracleデータベース常駐運用保守プロジェクト"
description: "当社は銀行にエンジニアを常駐させるとともに、専任のシニアエンジニアを指名し、銀行の故障緊急時に対し、タイムリーな対応と問題処理を実施しています。"
excerpt: "当社は銀行にエンジニアを常駐させるとともに、専任のシニアエンジニアを指名し、銀行の故障緊急時に対し、タイムリーな対応と問題処理を実施しています。"
order: 9
image: /assets/images/case-studies/Oracle-On-Site-Operation-and-Maintenance-Project-for-a-Bank.svg
tags: ["Oracle", "保守対象データベース", "常駐サービス","インストールと設定","日常点検","例外処理","パフォーマンス最適化","データ処理","重要業務保障"]
---

# 銀行向けOracleデータベース常駐運用保守プロジェクト  
![Maintenance]({{ '/assets/images/case-studies/Oracle-On-Site-Operation-and-Maintenance-Project-for-a-Bank.svg' | relative_url }})  

## 運用保守業務概要  

### 常駐サービス  
> 銀行にエンジニアを常駐させ、専任のシニアエンジニアを配置し、銀行の障害緊急事態に対応・処理します。  

### 保守対象データベース  
> 合計126基のOracleデータベースを保守対象とします。  

### 保守業務の分類と説明  

- **インストールと設定**  
<!-- -->  
    - 本番/テスト環境DB構築（ORACLE スタンドアロン/RAC/ADG/クライアント、MySQL スタンドアロン/クラスタ、redis スタンドアロン/クラスタ環境）  
    - テーブルスペースの拡張  
    - テスト環境・本番環境データベースのパッチ分析・検証・適用  
    - 標準化されたDBインストール手順書、ADG設定手順書、移行計画書などの作成・提供  
    - DBインフラ構成方案への技術提言  

- **日常点検**  
<!-- -->  
    - 本番データベースの毎朝の点検（Daily Morning Check）の実施  
    - 本番データベースの定期点検（Weekly Inspection）の実施  
    - 要求に応じた重要システムデータベースの詳細点検（Quarterly Inspection）の実施  

- **障害対応**  
<!-- -->  
    - RAC障害の分析と対応  
    - アーカイブ障害の分析と対応  
    - ORAエラーの分析と対応  
    - その他の障害処理など  

- **パフォーマンス最適化**  
<!-- -->  
    - アプリケーションSQLの最適化  
    - データベースパラメータの最適化  
    - ストレージスペースの最適化  
    - テーブル構造の最適化  

- **データ処理**  
<!-- -->  
    - 本番環境・テスト環境データのマスキング  
    - 本番環境・テスト環境データのエクスポートとインポート  
    - データベースのバックアップ、リストア操作など  

- **重要業務保障**  
<!-- -->  
    - 定期的なデータベース変更時の対応保証  
    - 法定休日中の対応保証  
    - データベースアップグレードと移行時の対応保証  
    - 災害復旧切り替え訓練の対応保証  

- **その他サービス**  
<!-- -->  
    - ユーザー、スペース、パスワードポリシー、ストレージ、パラメータ、ログ、統計情報などのデータベース情報収集  
    - 日常点検、バックアップ、データ抽出、比較などのスクリプト作成  
    - その他種類のデータベースのインストール、設定、テスト支援  
    - 関連データベース操作プラットフォーム、マスキングソフトなどのテストおよび設定支援  
    - 週次報告、月次報告、年次報告の作成  

    <style>
          .mypic {
                float: left;  
                margin-right: 10px;
          }
          .left-align {
                text-align: left;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        th, td {
            border: 1px solid black;
            padding: 8px;
            text-align: left;
        }
        th {
            background-color: #f2f2f2;
        }
    </style>

#### カテゴリ別業務量サマリー:  

| **No.** | **作業分類**             | **回数** | **作業内容**                                                                                                                               |
| :-----: | :------------------- | :----------: | :-------------------------------------------------------------------------------------------------------------------------------------------- |
|    1    | **インストールと設定** |    302 回 | 本番/テスト環境DB構築（ORACLE スタンドアロン/RAC/ADG/クライアント、MySQL スタンドアロン/クラスタ、redis スタンドアロン/クラスタ環境）<br>テスト/本番DBへのパッチ分析・検証・適用<br>標準化されたDBインストール手順書、ADG設定手順書、移行計画書などの作成・提供<br>DBインフラ構成方案への技術提言 |
|    2    | **日常点検** |  247 回   | 本番DB毎朝点検（Daily Morning Check）実施<br>本番DB週次点検（Weekly Inspection）実施<br>要求に基づく重要システムDB詳細点検（Quarterly Inspection）実施 |
|    3    | **障害対応** |    226 回 | RAC障害分析対応、アーカイブ障害分析対応、ORAエラー分析対応、その他例外処理など                                                                 |
|    4    | **パフォーマンス最適化** |     22 回 | アプリケーションSQL最適化、DBパラメータ最適化、ストレージスペース最適化、テーブル構造最適化                                                                                |
|    5    | **データ処理**  |    110 回 | 本番/テスト環境データマスキング<br>本番/テスト環境データエクスポート・インポート<br>DBバックアップ、リストア操作など                                 |
|    6    | **重要業務保障** |     27 回 | 定期DB変更時保障、法定休日保障、災害復旧切り替え訓練保障<br>DBアップグレード・移行時保障               |
|    7    | **その他**   |     41 回 | ユーザー/スペース/パスワードポリシー/ストレージ/パラメータ/ログ/統計情報等のDB情報収集<br>日常点検/バックアップ/データ抽出/比較等のスクリプト作成<br>他種DBの導入・設定・テスト支援<br>関連DB操作プラットフォーム/マスキングソフト等のテスト・設定支援<br>週次/月次/年次報告書の作成 |
| **　**  | **合計**            |   **975**    | **　**                                                                                                                                        |
	
