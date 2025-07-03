---
layout: post
title: "オプティマイザ統計情報の品質保証"
excerpt: "本論文では、Oracleデータベースで最も一般的なシナリオにおいて、いつ・どのように統計情報を収集すべきか詳細に議論します。"
date: 2025-07-03 11:00:00 +0800
categories: [Oracle, Database]
tags: [Database maintenance, Database deployment,Database optimization, oracle]
image: /assets/images/posts/Assuring-the-Quality-of-Optimizer-Statistics.jpg
---

## オプティマイザ統計情報の品質保証
高品質な統計情報は、最適なSQL実行計画を生成するために不可欠です。しかし、統計情報の品質が低い場合があり、この事実が気付かれない可能性があります。例えば、古い「継承された」システムでは、データベース管理者が理解していないスクリプトが使用されていることがあり、当然のことながら変更を躊躇する傾向があります。しかし、Oracleは統計収集機能を継続的に強化しているため、ベストプラクティスの推奨事項が軽視される可能性があります。  

これらの理由から、Oracle Database 18cでは「オプティマイザ統計アドバイザ」というアドバイザが導入され、データベース内の統計情報の品質向上を支援します。この診断ソフトウェアは、データディクショナリ内の情報を分析し、統計情報の品質を評価し、統計情報がどのように収集されているかを調査します。品質の低い統計や欠落している統計を報告し、これらの問題を解決するための推奨事項を生成します。  

その動作原理は、潜在的な問題を発見するためにベストプラクティスルールを適用することです。これらの問題は一連の「検出結果」として報告され、それらは特定の「推奨事項」につながります。推奨事項は「対応」（即時、またはデータベース管理者が実行する自動生成スクリプト経由）を使用して自動的に実装できます。  

アドバイザタスクはメンテナンス・ウィンドウで自動的に実行されますが、オンデマンドでも実行可能です。アドバイザが生成するHTMLまたはテキストレポートはいつでも閲覧でき、対応はいつでも実施できます。  

アドバイザタスクはデータを収集し、データディクショナリに保存します。これはオプティマイザ統計情報と統計収集情報（既にデータディクショナリに保持されている）の分析を行うため、パフォーマンスへの影響が低い操作です。アプリケーションスキーマオブジェクトに格納されたデータの二次分析は行いません。

タスクが完了すると、レポートはHTMLまたはテキスト形式で生成でき、対応（SQL）スクリプトも作成できます。

自動タスクによって生成されたレポートを表示するのは簡単です：
```sql
select dbms_stats.report_advisor_task('auto_stats_advisor_task') as report from dual;
あるいは、ADVISOR権限を持つユーザーは、以下の3ステップのプロセスを使用してタスクを手動で実行し、結果をレポートできます：

sql
DECLARE
   tname   VARCHAR2(32767) := 'demo';   -- タスク名
BEGIN
   tname := dbms_stats.create_advisor_task(tname);
END;
/
DECLARE
   tname   VARCHAR2(32767) := 'demo';   -- タスク名
   ename   VARCHAR2(32767) := NULL;     -- 実行名
BEGIN
   ename := dbms_stats.execute_advisor_task(tname);
END;
/
SELECT dbms_stats.report_advisor_task('demo') AS report
FROM dual;
```

アドバイザが生成した対応は即時に実装できます：

```SQL
DECLARE
    tname            VARCHAR2 (32767) := 'demo'; -- タスク名
    impl_result      CLOB;                       -- 実装レポート
BEGIN
    impl_result := dbms_stats.implement_advisor_task(tname);
END;
/
```

さらに、Oracle Database 18c Real Application Testingには、SQLパフォーマンス・アドバイザのクイックチェックなど、有用なパフォーマンス保証機能が含まれています。
