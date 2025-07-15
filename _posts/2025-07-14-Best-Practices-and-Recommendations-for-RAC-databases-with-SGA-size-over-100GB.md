---
layout: post
title: "SGA サイズが 100 GB を超える RAC データベースのベスト プラクティスと推奨事項"
excerpt: "Oracle Real Application Clusters（RAC）データベースのユーザー向けに、インスタンスごとに非常に大規模なSGA（例：100GB）を使用する環境におけるベストプラクティスと推奨事項を提供することです（※RACではクラスタ全体で均一なSGAサイズが前提となります）。本記事は、OracleのグローバルなRAC顧客ベースでの実績に基づき作成・維持されています。"
date: 2025-07-14 15:00:00 +0800
categories: [Oracle, Database]
tags: [Database maintenance, Database deployment,Database optimization, oracle]
image: /assets/images/posts/Best-Practices-and-Recommendations-for-RAC-databases-with-SGA-size-over-100GB.jpg
---

## 適用対象  
- Oracle Database Cloud Exadata Service - バージョンN/A & 以降  
- Oracle Database Cloud Service - バージョンN/A & 以降  
- Oracle Database - Enterprise Edition - バージョン 11.2.0.3 & 以降  
- Oracle Database Backup Service - バージョンN/A & 以降
- Oracle Database Cloud Schema Service - バージョンN/A & 以降  
*このドキュメントの情報はすべてのプラットフォームに適用されます。*  

---

## 目的  
本記事の目的は、インスタンスごとに非常に大きなSGA（例：100GB）を使用するOracle Real Application Clusters（RAC）データベースのユーザーにベストプラクティスと推奨事項を提供することです（RACはクラスタ全体で均一サイズのSGAを前提としていることに注意）。本記事は、OracleのグローバルなRAC顧客ベースでの経験に基づいて作成・維持されています。  

この記事は、Oracleの公式ドキュメントセットに取って代わるものではなく、むしろそれを補完するものです。記事で明確に扱われていない疑問点については、Oracleの公式ドキュメントを読み、理解し、参照することが不可欠です。

すべての推奨事項は、お客様の運用グループによって慎重に検討されるべきであり、関連するリスクに対して得られる可能性のある利益が実装を正当化する場合にのみ実装されるべきです。リスク評価は、システム、アプリケーション、およびビジネス環境に関する詳細な知識がなければ行うことはできません。

お客様の環境はそれぞれ異なります。そのため、Oracle Database、特にOracle RACの実装を成功させるには、テスト環境での検証が不可欠です。Oracleサポートは、本記事の推奨事項が効果を発揮する大規模なSGAのベースラインを100GBと定めていますが、これは単なる基準であり、これより小さいSGAでも同様の恩恵を受けられる可能性があります。したがって、本記事の推奨事項を本番環境に適用する前に、対象となる本番環境のレプリカであるテスト環境で徹底的にテストおよび検証し、負の影響がないことを確認することが極めて重要です。

---

## 適用範囲  
- 本記事はすべての新規および既存のRAC実装に適用されます。  
- ここに記載されているパラメータのほとんどはRACデータベース専用であるため、RACデータベースのみを対象としています。  

---

## 詳細  
本記事で提示する推奨事項は、1TBおよび2.6TBのSGAを備えたデータベースでの作業経験から得られた結果であることに注意してください。しかし、100GBや300GBのSGAを備えたデータベースもこれらの推奨事項の恩恵を受けています。  

また、18.1以降では一部の推奨事項が削除されていますので、推奨事項がご使用のデータベースに適用可能かどうかを確認してください。  

> **注意:**  
> ORAchk 18.2以降を使用して、大規模SGAデータベース（本MOSドキュメントに記載されているもの）の適切な設定を検証できます。チェックはORachk 18.2で利用可能ですが、最新の情報を確実に得るために、常に[ドキュメント1268927.2](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=1268927.2)経由で入手可能なORachkの最新バージョンを使用することを推奨します。  
> 最新のAHFをダウンロードしてください。[Autonomous Health Framework (AHF) - Including TFA and ORAchk/EXAchk Document 2550798.1](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=2550798.1)を参照  

---

### **init.oraパラメータ:**  
a. **`_lm_sync_timeout`** を `1200` に設定  
   *データベース 12.2 以下でのみ有効*  
   再構成およびDRM中のタイムアウトを防止します。静的パラメータ。ローリング再起動が可能。  

b. **`shared_pool_size`** を SGA総サイズの **15%以上** に設定  
   *動的パラメータ*  
   例：1TB SGAの場合、共有プールを150GB以上に設定。  

c. **`_gc_policy_minimum`** を `15000` に設定  
   *動的パラメータ*  
   **注意:**  
   - DRMが無効（`_gc_policy_time=0`）の場合、不要です。  
   - DRMの無効化は動的パラメータ `_lm_drm_disable` で行います（`_gc_policy_time` ではありません）。  
   - デフォルト値: 23c、19c DBRU JUL '23、19c ADB (Bug 34729755) で15000。  

d. **`_lm_tickets`** を `5000` に設定  
   *データベース 12.2 以下でのみ有効*  
   デフォルト=1000。静的パラメータ。増加時のローリング再起動は可能。減少時にはコールド再起動が必要な場合あり。  

e. **`gcs_server_processes`** を **デフォルトLMSプロセス数の2倍** に設定  
   *データベース 12.2 以下でのみ有効*  
   *静的パラメータ。ローリング再起動が可能*  
   **要件:**  
   - デフォルトLMS数 = f(CPU/コア数) (Oracle Databaseリファレンスガイド参照)。  
   - サーバー上のすべてのDBにまたがるLMSプロセスの合計は **CPU/コア総数未満でなければならない** ([Doc 558185.1](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=558185.1) 参照)。  
   *修正は12.2.0.1 JUL 2018 RU+に含まれる。*  

f. **`TARGET_PDBS`** を **CDB内で計画しているPDBの数** に設定  
   *12.2以降で有効*  
   *(シードPDBおよびルートCDBは除く)*  
   大規模な`sga_target`でのデフォルト値はパフォーマンス/エビクション問題を引き起こす可能性あり ([Doc 2644243.1](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=2644243.1) 参照)。  

---

### **HugePages/Large Pages:**  
- 大規模SGAを伴う**Linuxでは必須**。  
- 可能な場合は**すべてのプラットフォームで推奨**。  

---

### **推奨パッチ:**  
**11.2.0.3+:**  
11.2.0.3.5 DB PSU 以降を適用。  

**11.2.0.4 バグ修正:**  
- BUG 12747740 - RAC PERF: NODE JOIN RECONFIGURATION (PCMREPLAY) DOES NOT SCALE WITH MORE LMS'S  
- BUG 14193240 - LMS SIGNALED ORA-600[KGHLKREM1] DURING BEEHIVE LOAD  
- BUG 16392068 - MSGQ: LMSO HITS ORA-600 [KJBMPOCR:DSB]  
- BUG 17232014 - INITIAL ALLOCATION FOR KJBR6KJBL ARE TOO LOW W/ LARGE CACHES DUE TO UB4 OVERFLOW  
- BUG 17257445 - RAC PERF: DRM OPTIMIZATION (BUG 14558880) SHOULD ALSO WORK FOR RECONFIGURATION  
- BUG 17314971 - RAC PERF: RM/PT LATCH REDUCTION FOR RCFG (17257445) SHOULD BE ENABLED FOR SYNC7  

**SGA >4TB (Linux):**  
- BUG 18780342 - LINUX SUPPORT FOR > 4TB SGA  

---

## 参照資料  
- [NOTE:558185.1 - LMS and Real Time Priority in Oracle RAC 10g and 11g](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=558185.1)  
- [NOTE:1392248.1 - Auto-Adjustment of LMS Process Priority in Oracle RAC with 11.2.0.3 and later](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=1392248.1)  
- [NOTE:2550798.1 - Autonomous Health Framework (AHF) - Including TFA and ORAchk/EXAchk](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=2550798.1)  
- [NOTE:2644243.1 - Performance Issues when using PDBs with Oracle RAC 19c and 18c](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=riz393vca_119&id=2644243.1)  
