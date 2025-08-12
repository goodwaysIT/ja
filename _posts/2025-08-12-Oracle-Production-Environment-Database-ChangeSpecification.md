---
layout: post
title: "Oracle本番環境データベース変更仕様"
excerpt: "すべての本番環境変更は、監査可能・ロールバック可能・追跡可能という三原則に従わなければなりません。スクリプト実行後、ログは自動的にDBAチームへメール送信する必要があります。"
date: 2025-08-12 10:00:00 +0800
categories: [Oracle, Database]
tags: [Rollback Plan Design, Specification, oracle]
image: /assets/images/posts/Oracle-Production-Environment-Database-ChangeSpecification.jpg
---

## I. 変更プロセス仕様  
### 前提条件  

すべての変更は作業指示システムによる承認が必要です（設計文書添付必須）  
実行者はDBA認定資格を保有していなければなりません  
変更ウィンドウ：業務ピーク時間外（例：23:00-5:00）  
スクリプト命名規則：  
[プロジェクト番号]_[オブジェクト種別]_[機能]_[日付].sql  
例：PROJ123_TBL_CreateOrderTbl_20230811.sql  

## II. オブジェクト作成仕様  
### 1. ユーザーと表領域  
```sql
-- ユーザー作成（デフォルト表領域の紐付け必須）
CREATE USER ops_user IDENTIFIED BY "S3cureP@ss#2023"
  DEFAULT TABLESPACE ops_data
  QUOTA UNLIMITED ON ops_data
  ACCOUNT UNLOCK;

-- 表領域作成（BIGFILEモード指定必須）
CREATE BIGFILE TABLESPACE ops_data
  DATAFILE '+DATA' SIZE 10G AUTOEXTEND ON NEXT 1G;
```
### 2. テーブルと索引  
```sql
-- テーブル作成例（コメントとストレージパラメータ必須）
CREATE TABLE order_details (
  order_id   NUMBER(12)   PRIMARY KEY,
  product_id VARCHAR2(20) NOT NULL,
  amount     NUMBER(16,2) DEFAULT 0
) TABLESPACE ops_data
  PCTFREE 10 PCTUSED 80
  COMPRESS FOR OLTP;  -- 高度な圧縮を有効化

COMMENT ON TABLE order_details IS '注文明細テーブル';
COMMENT ON COLUMN order_details.order_id IS '注文固有ID';

-- 索引命名規則：IDX_[テーブル略称]_[カラム名]
CREATE INDEX IDX_ORDETAILS_PRODID ON order_details(product_id)
  TABLESPACE ops_index
  PARALLEL 4 NOLOGGING;  -- 並列処理で作成を高速化
```

### 3. ビューとシーケンス  
```sql
-- ビューはOR REPLACEとFORCEオプション必須
CREATE OR REPLACE FORCE VIEW v_active_orders AS
SELECT /*+ INDEX(o IDX_ORD_STATUS) */
       order_id, product_id
FROM orders o
WHERE status = 'ACTIVE'
WITH READ ONLY;  -- 読み取り専用を強制

-- シーケンスはギャップ防止のためNOCACHE指定必須
CREATE SEQUENCE seq_order_id
  START WITH 1000000
  INCREMENT BY 1
  NOCACHE NOCYCLE;
```
RAC環境推奨設定: CACHE+NOORDER。order属性未指定時、RACはデフォルトでキャッシュ値20に設定します。高頻度使用シーケンスはキャッシュを1000~2000に調整します。

## III. スクリプトコーディング仕様  
要件	例/説明  
文字セット	AL32UTF8必須  
ファイルエンコーディング	UNIX形式、BOMなしUTF-8  
トランザクション制御	DMLは明示的なCOMMIT/ROLLBACK必須  
バインド変数	ハードコーディング禁止（SQLインジェクション防止）  
正しいDML例：  
```sql
BEGIN
  UPDATE accounts SET balance = balance - :amt WHERE id = :acc_id;
  -- 例外処理の記述必須
  EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    RAISE;
END;
/
COMMIT;
```

## IV. ロールバック計画設計  
### 1. DDLロールバック（構造変更）  
```sql
-- 元操作：カラム追加
ALTER TABLE orders ADD (discount NUMBER(5,2));

-- ロールバックスクリプト（事前生成必須）
ALTER TABLE orders DROP COLUMN discount;
```

### 2. DMLロールバック（データ変更）  
```sql
-- 元操作：データ修正
UPDATE employees SET salary = salary * 1.1
WHERE dept_id = 'IT';

-- ロールバック計画（Flashback Query使用）
UPDATE employees e
SET e.salary = (
  SELECT salary
  FROM employees AS OF TIMESTAMP SYSDATE - 1/24  -- 1時間前
  WHERE employee_id = e.employee_id
)
WHERE dept_id = 'IT';
```

### 3. オブジェクト削除ロールバック  
```sql
-- 削除前バックアップ（標準操作）
CREATE TABLE orders_bak_20230811 AS
SELECT * FROM orders;

-- ロールバックコマンド
RENAME orders_bak_20230811 TO orders;
```

## V. 緊急時対応  
### 誤削除データ復旧  
```sql
-- Flashback Table使用（行移動の有効化が必要）
ALTER TABLE orders ENABLE ROW MOVEMENT;
FLASHBACK TABLE orders TO TIMESTAMP (SYSTIMESTAMP - INTERVAL '15' MINUTE);
```

### パフォーマンスロールバック計画  
索引変更後の性能劣化時：  
```sql
-- 高速索引ロールバック
DROP INDEX new_index_name FORCE;
CREATE INDEX old_index_name ... NOLOGGING PARALLEL 8;  -- 並列処理で再構築
```

## VI. バージョン管理要件  
プロジェクトディレクトリ構造：  
├── scripts  
│ ├── deploy # デプロイスクリプト  
│ ├── rollback # 対応ロールバックスクリプト  
│ └── logs # 実行ログ（タイムスタンプ付き）  
└── docs  
└── ER_diagram.pdf # 設計文書  

重要原則：すべての本番環境変更は「監査可能、ロールバック可能、追跡可能」の3原則に従わなければなりません。スクリプト実行後、ログは自動的にDBAチームへメール送信する必要があります。  

