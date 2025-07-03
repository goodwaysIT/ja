---
layout: post
title: "オプティマイザ統計情報の品質保証"
excerpt: "本論文では、Oracleデータベースで最も一般的なシナリオにおいて、いつ・どのように統計情報を収集すべきか詳細に説明します。"
date: 2025-07-03 11:00:00 +0800
categories: [Oracle, Database]
tags: [Database maintenance, Database deployment,Database optimization, oracle]
image: /assets/images/posts/Assuring-the-Quality-of-Optimizer-Statistics.jpg
---

## オプティマイザ統計情報の品質保証
最適なSQL実行計画を生成するためには、高品質な統計情報が不可欠です。しかし、統計情報の品質が低い場合でも、それが見過ごされることがあります。たとえば、古い「継承システム」では、現在のデータベース管理者が内容を把握していないスクリプトが使用されているケースもあり、それらを変更することに対して慎重になるのは当然です。しかし、Oracleは統計収集機能を継続的に強化しており、最新のベストプラクティスが無視される可能性もあります。 

これらの理由から、Oracle Database 18c では「オプティマイザ統計アドバイザ」という機能が導入され、データベース内の統計情報の品質向上を支援します。この診断ツールはデータディクショナリの情報を分析し、統計の品質を評価し、統計情報がどのように収集されているかを調査します。統計の欠如や不適切な統計に関するレポートを生成し、それらの問題を解決するための推奨事項を提示します。

その原理は、ベストプラクティスに基づく一連のルールを適用することで、潜在的な問題を検出するというものです。検出された問題は「検出結果」として報告され、それに対応する「推奨事項」が示されます。推奨事項は「アクション」として自動的に実行することも可能であり、すぐに適用するか、データベース管理者が実行するためのスクリプトとして生成することもできます。

アドバイザタスクはメンテナンス・ウィンドウで自動的に実行されますが、オンデマンドでの手動実行も可能です。アドバイザが生成するHTMLまたはテキストレポートはいつでも閲覧でき、推奨アクションも随時実施できます。  

アドバイザタスクはデータディクショナリ内で情報を収集・保存します。この処理は、オプティマイザ統計情報と統計収集情報（既にデータディクショナリに保持されているもの）を分析するだけであるため、パフォーマンスへの影響は最小限です。アプリケーションスキーマオブジェクトに格納されたデータに対する二次分析は行いません。

タスクが完了すると、HTMLまたはテキスト形式でレポートを生成でき、アクション用のSQLスクリプトも作成可能です。

自動タスクによって生成されたレポートを確認するのは簡単です：
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
