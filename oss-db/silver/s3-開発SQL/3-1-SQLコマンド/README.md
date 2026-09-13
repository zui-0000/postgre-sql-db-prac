# S3.1 SQLコマンド

- 重要度: **13** / 50（Silver の中項目でいちばん大きい）
- 公式の説明: 「基本的なSQL文およびデータベースの構成要素に関する知識を問う」「レプリケーションの基本機能、種類、特徴などの理解を問う」
- 実習環境: db（07 レプリケーションだけは、サーバーを2台用意する必要がある）

範囲が広いので、7つのファイルに分けている。公式の用語は、下の表のとおりに振り分けた。

| ファイル | 主要な知識範囲 | 重要な用語、コマンド、パラメータなど | 状態 |
| --- | --- | --- | --- |
| [01 データ型](01-データ型.md) | データ型 | `INTEGER`、`SMALLINT`、`BIGINT`、`NUMERIC`、`DECIMAL`、`REAL`、`DOUBLE PRECISION`、`CHAR`、`CHARACTER`、`VARCHAR`、`CHARACTER VARYING`、`TEXT`、`BOOLEAN`、`DATE`、`TIME`、`TIMESTAMP`、`INTERVAL`、`SERIAL`、`BIGSERIAL`、`BYTEA`、`NULL` | 未着手 |
| [02 テーブルと制約](02-テーブルと制約.md) | テーブル定義 | `CREATE/ALTER/DROP TABLE`、`PRIMARY KEY`、`FOREIGN KEY`、`REFERENCES`、`UNIQUE`、`NOT NULL`、`CHECK`、`DEFAULT`、`GENERATED (AS IDENTITY)` | 未着手 |
| [03 DMLとSELECT](03-DMLとSELECT.md) | SELECT 文、INSERT 文、UPDATE 文、DELETE 文 | `SELECT/INSERT/UPDATE/DELETE`、`FROM`、`JOIN`、`WHERE`、`INTO`、`VALUES`、`SET`、`LIMIT`、`OFFSET`、`ORDER BY`、`DISTINCT`、`GROUP BY`、`HAVING`、`EXISTS`、`IN`、`NOT` | 未着手 |
| [04 DBオブジェクト](04-DBオブジェクト.md) | インデックス、ビュー、マテリアライズドビュー、トリガー、シーケンス、スキーマ、テーブルスペース、関数定義 / プロシージャ定義、PL/pgSQL | `CREATE/ALTER/DROP INDEX/VIEW/MATERIALIZED VIEW/TRIGGER/SCHEMA/SEQUENCE/TABLESPACE/FUNCTION/PROCEDURE`、`CALL` | 未着手 |
| [05 パーティション](05-パーティション.md) | パーティション | `CREATE TABLE PARTITION BY/OF`、`ALTER TABLE ATTACH/DETACH PARTITION` | 未着手 |
| [06 JSON](06-JSON.md) | （該当なし） | `JSON`、`JSONB`、JSON PATH | 未着手 |
| [07 レプリケーション](07-レプリケーション.md) | ストリーミングレプリケーション、ロジカルレプリケーション | `CREATE PUBLICATION/SUBSCRIPTION` | 未着手 |
