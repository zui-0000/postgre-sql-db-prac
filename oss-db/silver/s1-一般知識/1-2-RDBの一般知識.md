# S1.2 リレーショナルデータベースに関する一般知識

- 重要度: **4** / 50
- 公式の説明: 「リレーショナルデータベースの基本概念、一般的知識を問う」
- 実習環境: db（練習用のデータベースを作って確かめる）

## 出題範囲

### 主要な知識範囲

- リレーショナルデータモデルの基本概念
- データベース管理システムの役割
- SQL に関する一般知識
- SQLの 分類 (DDL / DML / DCL)
- データベースの設計と正規化

### 重要な用語、コマンド、パラメータなど

（公式の記載なし）

## 学習メモ

このノートの SQL は、練習用のデータベースで実際に動かして確かめた。同じことを試すときは、次のようにデータベースを作ってから実行し、終わったら削除する。

```sh
docker compose exec db psql -U postgres -c "CREATE DATABASE practice"
docker compose exec db psql -U postgres -d practice
# 試し終わったら
docker compose exec db psql -U postgres -c "DROP DATABASE practice"
```

### リレーショナルデータモデルの基本概念

**リレーショナルデータモデル**は、1970年に Edgar F. Codd が論文「A Relational Model of Data for Large Shared Data Banks」で提案した。データを「リレーション（表）」の集まりとして扱う。

| リレーショナルモデルの用語 | SQL での呼び方 | 意味 |
| --- | --- | --- |
| リレーション | テーブル（表） | タプルの集合 |
| タプル | 行（レコード） | 属性の値の組 |
| 属性 | 列（カラム） | 名前とデータ型を持つ要素 |
| ドメイン | データ型 | 属性が取りうる値の集合 |

- リレーションは「集合」なので、行に順番はない。公式ドキュメントにも「SQL はテーブル内の行の順序を一切保証しない」とある。順番が必要なら `ORDER BY` を付ける。
- PostgreSQL の用語集では、テーブルだけでなく、ビュー、シーケンス、インデックス、マテリアライズドビュー、クエリの結果もリレーションに含めている。
- テーブルはデータベースにまとめられる。1つの PostgreSQL サーバーが管理するデータベースの集まりを、**データベースクラスタ**と呼ぶ（[S2.1](../s2-運用管理/2-1-インストール方法.md) で詳しく学ぶ）。

TypeScript で例えると、リレーションは「同じ形のオブジェクトを集めた `Set`」に近い。どの要素も同じプロパティ（属性）を持ち、各プロパティには型（ドメイン）があり、要素に順番はない。

#### ほかのデータモデル

リレーショナルモデルが登場する前から、データを整理するモデルはあった。

| モデル | 関係の表し方 | 多対多 | 代表例 |
| --- | --- | --- | --- |
| 階層型 | 親子の木構造。**子の親は1つだけ** | 苦手（同じデータを重複して持つしかない） | IBM の IMS |
| ネットワーク型 | 階層型を拡張し、**子が複数の親を持てる** | 表せる | CODASYL 系 |
| リレーショナル型 | **値（外部キー）で関係を表す** | 中間テーブルで表せる | PostgreSQL、Oracle Database |
| オブジェクト指向型 | オブジェクトとして格納する | — | — |

**階層型データベース**は、1960年代に IBM が開発した。データにたどり着くには木を上からたどる経路をプログラム側が意識する必要があり、構造を変えるとプログラムも書き直しになる。この「アプリケーションの実装に依存する」点が理由で、リレーショナルモデルに主流の座を譲った。

ただし、使われなくなったわけではない。

- 非常に高い性能と可用性が求められる、銀行、医療、通信などの分野では今も使われている。
- 身近な例として、PostgreSQL の公式ドキュメントは「Unix 系 OS のファイルとディレクトリは階層型データベースの例」と書いている。ほかに Windows のレジストリ、DNS、LDAP も階層構造である。
- 1990年代後半に XML が登場して、階層構造でデータを持つ考え方が見直された。

リレーショナルモデルの発想の転換は、**関係を「経路」ではなく「値」で表した**ことにある。そのため、問い合わせのたびに好きな組み合わせで結合できる。

リレーショナルデータベースでも、**自己参照の外部キー**を使えば木構造を表せる。たどるときは再帰クエリ（`WITH RECURSIVE`）を使う。

```sql
CREATE TABLE categories (
  id int PRIMARY KEY,
  name text NOT NULL,
  parent_id int REFERENCES categories   -- 自分自身を参照する
);

WITH RECURSIVE tree AS (
  SELECT id, name, parent_id, 1 AS depth, name AS path
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.name, c.parent_id, t.depth + 1, t.path || ' > ' || c.name
  FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT depth, repeat('  ', depth - 1) || name AS tree, path FROM tree ORDER BY path;
```

```text
 depth |    tree    |         path
-------+------------+----------------------
     1 | 食品       | 食品
     2 |   果物     | 食品 > 果物
     3 |     みかん | 食品 > 果物 > みかん
     3 |     りんご | 食品 > 果物 > りんご
     2 |   野菜     | 食品 > 野菜
```

さらに PostgreSQL は、JSON/JSONB 型で階層構造のデータを列に持てる（S3.1 の 06 で学ぶ）。「階層型かリレーショナルか」を選ぶ話ではなく、リレーショナルデータベースの中に階層構造を持てる。

#### キー

| キー | 説明 |
| --- | --- |
| スーパーキー | その値の組み合わせで、行を一意に特定できる属性の集合。余分な属性が含まれていてもよい |
| 候補キー | 行を一意に特定できる、**最小の**属性の集合。どの属性を取り除いても一意にならなくなる |
| 主キー | 候補キーの中から1つ選んだもの。**1つのテーブルに1つだけ**で、**NULL を含められない** |
| 外部キー | 別の（まれに同じ）テーブルの行を参照する列。参照先に存在しない値は入れられない |

候補キーと主キーの関係:

- 候補キーは1つのテーブルに複数あってよい。その中から1つを選んだものが主キーになる。
- たとえば社員テーブルで「社員番号」と「メールアドレス」がどちらも一意なら、どちらも候補キー。社員番号を主キーにしたら、メールアドレスには `UNIQUE` 制約と `NOT NULL` 制約を付けて一意性を保つ。
- PostgreSQL の用語集では、主キーは「一意性制約の特別な場合で、NULL を含まないことも保証するもの」と説明されている。
- 主キーは複数の列を組み合わせて作ることもできる（**複合主キー**。下の `order_items` の `(order_id, product_id)` がその例）。

制約の動きを実機で確かめた結果:

```sql
INSERT INTO customers VALUES (NULL, '高橋');    -- 主キーに NULL
INSERT INTO customers VALUES (10, '高橋');      -- 主キーが重複
INSERT INTO orders VALUES (4, '2026-09-03', 99); -- 存在しない顧客を参照
```

```text
ERROR:  null value in column "customer_id" of relation "customers" violates not-null constraint
ERROR:  duplicate key value violates unique constraint "customers_pkey"
ERROR:  insert or update on table "orders" violates foreign key constraint "orders_customer_id_fkey"
```

一方で、`UNIQUE` 制約の列には **NULL を複数入れられる**。NULL どうしは「等しい」とはみなされないため。

```sql
CREATE TABLE unique_demo (email text UNIQUE);
INSERT INTO unique_demo VALUES (NULL), (NULL);   -- エラーにならない（2行入る）
```

#### 関係演算

リレーションに対する操作を**関係演算**と呼ぶ。演算の結果もリレーションになる。

| 演算 | 意味 | SQL での書き方 |
| --- | --- | --- |
| 選択 | 条件に合う**行**を取り出す | `WHERE` |
| 射影 | 指定した**列**だけを取り出す | `SELECT` の列リスト（重複を除くなら `DISTINCT`） |
| 結合 | 条件に合う行どうしをつなげる | `JOIN` |
| 和 | どちらかのリレーションにある行 | `UNION` |
| 差 | 1つ目にあって、2つ目にない行 | `EXCEPT` |
| 共通部分 | 両方にある行 | `INTERSECT` |
| 直積 | 2つのリレーションの、すべての行の組み合わせ | `CROSS JOIN` |
| 商 | ある集合の値をすべて持つ行を取り出す | SQL に直接の構文はない |

- 「選択は行、射影は列」と覚える。
- 「積」という言葉は、資料によって共通部分を指すことも、直積を指すこともある。どちらの意味なのかは文脈で判断する。

実機で確かめた例（テーブルは下の「正規化」で作ったもの）:

```sql
-- 選択: 単価が 100 以上の商品
SELECT * FROM products WHERE unit_price >= 100;          -- りんご だけ

-- 差: 全顧客から、9/1 に注文した顧客を引く
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders WHERE order_date = '2026-09-01';   -- 20 だけ

-- 直積: 顧客 2人 × 商品 2種類
SELECT c.customer_name, p.product_name FROM customers c CROSS JOIN products p;   -- 4行
```

### データベース管理システム（DBMS）の役割

**DBMS** の定義は「ユーザーがデータベースを定義し、作成し、維持し、アクセスを制御できるようにするソフトウェアシステム」。

#### データベースと DBMS は別物

| 用語 | 意味 | 例 |
| --- | --- | --- |
| データベース | 保存されたデータそのもの | 顧客や注文のデータ |
| DBMS | そのデータを管理するソフトウェア | PostgreSQL、MySQL、Oracle Database |

**PostgreSQL は DBMS を組み込んだソフトウェアではなく、PostgreSQL そのものが DBMS**（その中でもリレーショナルデータベース管理システム、RDBMS）である。1つの PostgreSQL が複数のデータベースを管理する。試験では「データベース」と「DBMS」を入れ替えた選択肢が出ることがあるので、区別しておく。

#### なぜ DBMS が必要なのか

データをファイルに自分で保存する場合（TypeScript で `fs.writeFile` を使って JSON ファイルに保存する場合など）と比べると、DBMS が引き受けている仕事がわかる。

| 困ること | ファイルに自分で保存する場合 | DBMS がやってくれること |
| --- | --- | --- |
| 同時に書き込む | 2人が同時に書くと、片方の変更が消える | 同時実行制御。誰がどの順で書いても壊れない |
| 途中で落ちる | 書き込みの途中でプロセスが止まると、ファイルが壊れる | トランザクション。すべて反映するか、何もなかったことにするか |
| 障害から戻す | 壊れたら戻せない | WAL やバックアップから復旧できる |
| 目的のデータを探す | 全件を順に見ていくしかない | インデックスを使って速く探せる |
| データの正しさを保つ | アプリ側で毎回チェックする | 制約（主キー、外部キー、`CHECK`）で DBMS が拒否する |
| 見せる相手を選ぶ | ファイルを読めれば全部見えてしまう | ロールと権限で、見せる範囲を制御できる |

DBMS は「データを正しく、安全に、速く、複数人で共有する」ための仕組みを、まとめて引き受けるソフトウェアである。

#### DBMS が備えるべき機能

Codd は、汎用の DBMS が備えるべき機能として次のものを挙げている。

| DBMS の機能 | PostgreSQL での例 | 関連する章 |
| --- | --- | --- |
| データの格納、検索、更新 | `SELECT`、`INSERT`、`UPDATE`、`DELETE` | S3.1 |
| データを説明するカタログ（メタデータ） | システムカタログ、`information_schema` | S2.5 |
| トランザクションと同時実行の制御 | `BEGIN` / `COMMIT`、MVCC、ロック | S3.3 |
| 障害からの復旧 | WAL、PITR | S2.4 |
| アクセスの認可 | ロール、`GRANT` / `REVOKE` | S2.5 |
| 離れた場所からのアクセス | TCP/IP での接続、`pg_hba.conf` | S2.3 |
| 制約によるデータの整合性の維持 | 主キー、外部キー、`CHECK` 制約 | S3.1 |

このほかに、インポートやエクスポート、監視などのツールも提供する（PostgreSQL なら `COPY` や `pg_dump`）。公式の例題には「DBMS の機能の説明として誤っているもの」を選ぶテーマがあるので、それぞれの機能が何をするのかを説明できるようにしておく。

#### トランザクション管理と同時実行制御の違い

DBMS の機能一覧では「トランザクションと同時実行の制御」とまとめて書かれることが多いが、この2つは別の機能である。混同しやすいので整理しておく。

| | トランザクション管理 | 同時実行制御 |
| --- | --- | --- |
| 何を管理するか | **1つのまとまり**（どこからどこまでが1単位か） | **複数のトランザクションが同時に動いたときの相互作用** |
| 守る ACID の性質 | 原子性（A）と永続性（D） | **独立性（I）** |
| 代表的な仕組み | `COMMIT`、`ROLLBACK`、WAL | **MVCC、ロック、トランザクション分離レベル** |
| 一言でいうと | 「全部やるか、全部やらないか」 | 「他人の途中経過を見せない、壊させない」 |

同時実行制御は「トランザクションとトランザクションの間」を調整する仕組みなので、トランザクションという単位が土台になる。ただし、**「トランザクションを開始することが、同時実行の制御そのもの」ではない**。

PostgreSQL は自動コミットなので、`BEGIN` を書かなくても1つの SQL 文が1つのトランザクションとして実行される。つまり `BEGIN` を書くかどうかに関係なく、同時実行制御は常に働いている。こちらが選べるのは制御の**強さ**で、その手段が次の2つ。

- トランザクション分離レベルを選ぶ（`SET TRANSACTION ISOLATION LEVEL`）
- 明示的にロックを取る（`LOCK TABLE`、`SELECT ... FOR UPDATE`）

実際に2つの接続を同時に動かして確かめた結果は、[S3.3 トランザクションの概念](../s3-開発SQL/3-3-トランザクション.md)に書いた。

#### SQL で操作できる機能と、できない機能

DBMS に指示を出す窓口は SQL だけではない。OS のコマンドと設定ファイルも窓口になる。

| DBMS の機能 | SQL で操作する | SQL 以外で操作する |
| --- | --- | --- |
| データの検索、追加、更新、削除 | DML（`SELECT`、`INSERT`、`UPDATE`、`DELETE`） | — |
| テーブルなどの定義 | DDL（`CREATE`、`ALTER`、`DROP`） | — |
| 権限の制御 | DCL（`GRANT`、`REVOKE`） | — |
| トランザクション | `BEGIN`、`COMMIT`、`ROLLBACK` | MVCC は裏側で自動的に働く |
| 障害からの復旧 | — | WAL への書き込みは自動。バックアップは `pg_dump` や `pg_basebackup`、リカバリは設定ファイルとサーバーの起動 |
| 接続の受け付けと認証 | — | `postgresql.conf`、`pg_hba.conf` という設定ファイル |
| サーバーの起動と停止 | — | `pg_ctl` という OS のコマンド |
| データベースクラスタの作成 | — | `initdb` という OS のコマンド |

出題範囲は、この分かれ方にそのまま対応している。

| 出題分野 | 割合 | 内容 |
| --- | ---: | --- |
| S2 運用管理 | 52% | SQL 以外の部分。`initdb`、`pg_ctl`、設定ファイル、バックアップのコマンド |
| S3 開発/SQL | 32% | SQL の部分。DDL、DML、DCL、トランザクション |

SQL を書けるだけでは PostgreSQL を運用できないので、S2 のほうが割合が大きい。学習環境で db（SQL を試す）と sandbox（`initdb` や `pg_ctl` を手で実行する）を分けているのも、この違いに合わせたもの（[環境構築](../../../docs/01-環境構築.md)）。

SQL が実行されるまでの流れ:

```text
psql（クライアント）
  │  SQL を送る
  ▼
postgres（サーバープロセス）＝ DBMS の本体
  │  ・SQL を解析して実行計画を立てる
  │  ・権限を確認する（DCL で設定した内容）
  │  ・トランザクションと同時実行を制御する
  │  ・WAL に記録してからデータを書く
  ▼
データファイル（データベースクラスタ）
```

`pg_ctl` や `initdb` は、この postgres プロセスやデータファイルそのものを操作するコマンドで、SQL とは別のルートから DBMS を扱っている。

#### ACID 特性

トランザクションが満たすべき4つの性質。同時に処理が動いていても、エラーや電源断が起きても、データが正しい状態を保つためのもの。

| 性質 | 意味 | PostgreSQL での仕組み |
| --- | --- | --- |
| 原子性（Atomicity） | トランザクションの操作は、すべて成功するか、すべて取り消されるかのどちらか。障害が起きても、途中の結果は見えない | `ROLLBACK`、WAL |
| 一貫性（Consistency） | データが常に整合性制約を満たしている。コミットの時点で違反が残っていれば、ロールバックされる | 各種の制約 |
| 独立性（Isolation） | コミットする前の変更は、同時に動いているほかのトランザクションからは見えない | MVCC、トランザクション分離レベル |
| 永続性（Durability） | コミットした変更は、障害やクラッシュが起きても失われない | WAL |

原子性を実機で確かめた結果（顧客は最初2人）:

```sql
BEGIN;
INSERT INTO customers VALUES (30, '高橋');
SELECT count(*) FROM customers;   -- 3
ROLLBACK;
SELECT count(*) FROM customers;   -- 2（INSERT が取り消された）
```

日本語訳は資料によって揺れるので、どちらの言葉でも意味がわかるようにしておく。

| 英語 | よくある訳 |
| --- | --- |
| Atomicity | 原子性、不可分性 |
| Consistency | 一貫性、整合性 |
| Isolation | 独立性、分離性、隔離性 |
| Durability | 永続性、耐久性 |

混同しやすいところ:

- **原子性と一貫性**: 原子性は「全部やるか、全部やらないか」という**実行のまとまり**の話。一貫性は「制約に違反したデータを残さない」という**データの正しさ**の話。
- **独立性は「同時に動いているほかのトランザクションから見えない」という意味**。自分のトランザクションの中では、自分の変更は見える（上の実機の結果で、`ROLLBACK` する前の `SELECT` が 3 件を返しているのがそれ）。
- **永続性はコミットしたあとの話**。コミットしていない変更が消えるのは、永続性ではなく原子性が働いた結果。

短く覚えるなら、**A は「まとまり」、C は「正しさ」、I は「他人から見えない」、D は「消えない」**。

### SQL に関する一般知識

#### 成り立ちと標準化

- 1970年代前半に、IBM の Donald D. Chamberlin と Raymond F. Boyce が、Codd のリレーショナルモデルをもとに開発した。IBM の System R という DBMS 用だった。
- 最初の名前は **SEQUEL**（Structured English Query Language）。商標の問題があったので、**SQL** に短くした。
- **1986年に ANSI**、**1987年に ISO** が標準として採用した。規格の正式名称は **ISO/IEC 9075「Database Language SQL」**。
- 改訂の歴史: SQL-92 → SQL:1999 → SQL:2003 → SQL:2006 → SQL:2008 → SQL:2011 → SQL:2016 → **SQL:2023**（最新）。新しい版が出ると、古い版は置き換えられる。

#### PostgreSQL と SQL 標準

- SQL:1999 以降、規格の機能は「**Core**」（準拠する実装が必ず備える機能）と、オプションの機能に分かれている。
- PostgreSQL は、伝統的な機能や常識に反しない範囲で、最新の規格（SQL:2023）への準拠を目指している。
- SQL:2023 の Core の必須機能 177 個のうち、PostgreSQL は少なくとも 170 個に準拠している。なお、Core に完全準拠していると主張する DBMS は、公式ドキュメントの執筆時点では1つもない。

#### 宣言型の言語

SQL は**宣言型**の言語で、**集合**を単位に操作する。「どんなデータが欲しいか」だけを書き、「どうやって取り出すか」（インデックスを使うか、どの順番で表を結合するかなど）は DBMS が決める。

TypeScript で `for` 文を書いて、配列から条件に合う要素を1つずつ探すのは**手続き型**の書き方。SQL では「この条件の行が欲しい」とだけ書けばよい。

### SQL の分類

| 分類 | 正式名称 | 役割 | 主な文 |
| --- | --- | --- | --- |
| DDL | Data Definition Language（データ定義言語） | テーブルなどのオブジェクトを定義する | `CREATE`、`ALTER`、`DROP` |
| DML | Data Manipulation Language（データ操作言語） | データを検索、追加、更新、削除する | `SELECT`、`INSERT`、`UPDATE`、`DELETE` |
| DCL | Data Control Language（データ制御言語） | 権限などを制御する | `GRANT`、`REVOKE` |

#### 分類は資料によって違う

どの文をどの分類に入れるかは、資料によって揺れがある。

| 文 | よくある分類 | 別の分類の例 |
| --- | --- | --- |
| `SELECT` | DML | DQL（Data Query Language）として分ける（英語版 Wikipedia） |
| `COMMIT`、`ROLLBACK` | DCL に含める（日本語の解説記事に見られる） | 「トランザクション制御文」として分ける（Oracle の公式ドキュメント）。TCL（Transaction Control Language）と呼ぶこともある |
| `GRANT`、`REVOKE` | DCL | DDL に含める（Oracle の公式ドキュメント） |

試験では、次の対応は確実に押さえる。`SELECT` や `COMMIT` のように揺れがあるものは、選択肢の中でもっとも適切なものを選ぶ。

- `CREATE`、`ALTER`、`DROP` → DDL
- `INSERT`、`UPDATE`、`DELETE` → DML
- `GRANT`、`REVOKE` → DCL

DCL を実機で確かめた結果:

```sql
CREATE ROLE reader;
GRANT SELECT ON products TO reader;
SET ROLE reader;
SELECT product_name FROM products;   -- 読める
SELECT * FROM customers;             -- ERROR:  permission denied for table customers
RESET ROLE;
```

### データベースの設計と正規化

#### 設計の流れ

| 段階 | 内容 |
| --- | --- |
| 概念設計 | 業務で扱う対象と、その関係を整理する。E-R 図などで表す。もっとも抽象的な段階 |
| 論理設計 | 概念設計の内容を、テーブル、列、キーに落とし込む。正規化もここで行う。特定の DBMS には依存しない |
| 物理設計 | 使う DBMS に合わせて、データ型、インデックス、格納場所などを決める |

#### E-R モデル

**E-R モデル**（Entity-Relationship モデル）は、1976年に Peter Chen が提案した、データの構造を表す方法。

| 要素 | 意味 | 例 |
| --- | --- | --- |
| エンティティ | 独立して存在し、一意に識別できる「もの」 | 顧客、商品、注文 |
| リレーションシップ | エンティティどうしの関係。文にしたときの動詞にあたる | 顧客が注文する |
| 属性 | エンティティやリレーションシップが持つ情報 | 顧客名、注文日 |

エンティティどうしの対応の数（**カーディナリティ**）には、1対1、1対多、多対多がある。

- 顧客と注文は **1対多**（1人の顧客が複数の注文をする）。
- 注文と商品は **多対多**（1つの注文に複数の商品、1つの商品が複数の注文に含まれる）。多対多はテーブルで直接は表せないので、**中間テーブル**（下の `order_items`）を作って、2つの1対多に分ける。

#### 正規化の目的

**正規化**は、データの重複を減らして整合性を高めるために、テーブルを分割すること。Codd が1970年に第1正規形、1971年に第2正規形と第3正規形を定義し、1974年に Codd と Raymond F. Boyce がボイス・コッド正規形を定義した。

正規化していないと、データを変更するときに次の**異常**が起きる。

| 異常 | 内容 | 下の例（1つの表に全部入れた場合） |
| --- | --- | --- |
| 追加時異常 | ある事実を、ほかの情報がないと登録できない | まだ注文されていない商品を登録できない（主キーの注文番号が決まらない） |
| 更新時異常 | 同じ情報が複数の行にあり、一部の行だけ更新すると矛盾する | 顧客名を1行だけ書き換えると、同じ顧客に2つの名前ができる |
| 削除時異常 | ある事実を削除すると、別の事実まで消えてしまう | 鈴木さんの唯一の注文を削除すると、鈴木さんという顧客の情報も消える |

#### 関数従属

「A の値が決まると、B の値が1つに決まる」とき、**B は A に関数従属する**といい、`A → B` と書く。たとえば「顧客番号 → 顧客名」。正規形の条件は、この関数従属を使って定義される。

関数従属には3つの種類があり、どの正規形で取り除くかが決まっている。

| 種類 | 意味 | 例（主キーは `(order_id, product_id)`） | 解消する正規形 |
| --- | --- | --- | --- |
| **完全関数従属** | 候補キー**全体**で決まり、その**一部では決まらない** | `(order_id, product_id)` → `quantity` | これが目指す状態（2NF の条件） |
| **部分関数従属** | 候補キーの**一部だけ**で決まってしまう | `order_id` → `customer_name`、`product_id` → `product_name` | 第2正規形 |
| **推移関数従属** | **非キー属性を経由**して決まる | `order_id` → `customer_id` → `customer_name` | 第3正規形 |

完全関数従属と部分関数従属は反対の関係にある。「キー全体が必要か、一部で足りてしまうか」の違い。

関数従属は、データを数えれば判定できる。左の値でグループ化して、右の値が何種類あるかを数え、**最大が 1 なら決まっている**と言える。

```sql
-- order_id だけで customer_name が決まるか（1 なら決まる）
SELECT max(c) FROM (
  SELECT count(DISTINCT customer_name) AS c FROM order_items_1nf GROUP BY order_id
) t;
```

実際に数えた結果:

```text
 order_id → customer_name | product_id → product_name | customer_id → customer_name
--------------------------+---------------------------+-----------------------------
                        1 |                         1 |                           1

 order_id → quantity | product_id → quantity | (order_id, product_id) → quantity
---------------------+-----------------------+-----------------------------------
                   2 |                     2 |                                 1
```

- 顧客名と商品名は、主キーの一部だけで決まっている（**部分関数従属**）。
- 数量は主キーの一部では決まらず、全体で決まっている（**完全関数従属**）。
- `customer_id → customer_name` も成立していて、これが推移関数従属の中間になる。

実際のデータで見ると分かりやすい。

```text
 order_id | product_id | customer_name | quantity
----------+------------+---------------+----------
        1 |        101 | 佐藤          |        3
        1 |        102 | 佐藤          |        5
```

同じ `order_id = 1` でも数量は 3 と 5 で違うが、顧客名はどちらも「佐藤」。だから顧客名は `order_id` だけで決まり、数量は両方そろって初めて決まる。

```text
【第1正規形】主キー = (order_id, product_id)

  (order_id, product_id) ─→ quantity                    ← 完全関数従属（正しい状態）
   order_id ─→ order_date, customer_id, customer_name   ← 部分関数従属（2NF 違反）
   product_id ─→ product_name, unit_price               ← 部分関数従属（2NF 違反）

        ↓ 部分関数従属を取り除く

【第2正規形】orders(order_id, order_date, customer_id, customer_name)

   order_id ─→ customer_id ─→ customer_name             ← 推移関数従属（3NF 違反）
                 非キー属性を経由している

        ↓ 推移関数従属を取り除く → 第3正規形
```

判定のポイント:

- **部分関数従属は、複合キーのときにしか起こらない。** 主キーが1列なら「一部」が存在しないので、1NF を満たした時点で 2NF も満たす。
- 推移関数従属で問題なのは、`order_id → customer_name` という関係そのものではなく、**間に非キー属性の `customer_id` が挟まっている**こと。
- 取り除く順番は、繰り返し項目（1NF）→ 部分関数従属（2NF）→ 推移関数従属（3NF）。

#### 正規形

| 正規形 | 条件 |
| --- | --- |
| 非正規形 | 繰り返し項目がある（1つの行の1つの項目に、複数の値が入っている） |
| 第1正規形（1NF） | 各項目に値が1つだけ入っている |
| 第2正規形（2NF） | 1NF を満たし、候補キーに含まれない属性が、候補キーの**一部**ではなく**全体**に関数従属している（**部分関数従属**がない） |
| 第3正規形（3NF） | 2NF を満たし、候補キーに含まれない属性が、ほかの候補キーに含まれない属性を通して決まることがない（**推移的関数従属**がない） |
| ボイス・コッド正規形（BCNF） | 自明でない関数従属の左辺（決定する側）が、すべてスーパーキーになっている。3NF をさらに厳しくしたもの |

#### 正規化の例

非正規形: 1つの注文の行に、複数の商品がまとめて入っている。

```text
注文番号 | 注文日     | 顧客番号 | 顧客名 | 商品（商品番号, 商品名, 単価, 数量）
1        | 2026-09-01 | 10       | 佐藤   | (101, りんご, 150, 3), (102, みかん, 80, 5)
2        | 2026-09-02 | 10       | 佐藤   | (101, りんご, 150, 1)
3        | 2026-09-02 | 20       | 鈴木   | (102, みかん, 80, 2)
```

第1正規形: 商品ごとに行を分けて、1つの項目に値が1つだけ入るようにする。主キーは `(order_id, product_id)` の組み合わせになる。

```sql
CREATE TABLE order_items_1nf (
  order_id int, product_id int, order_date date,
  customer_id int, customer_name text,
  product_name text, unit_price int, quantity int,
  PRIMARY KEY (order_id, product_id)
);
```

```text
 order_id | product_id | order_date | customer_id | customer_name | product_name | unit_price | quantity
----------+------------+------------+-------------+---------------+--------------+------------+----------
        1 |        101 | 2026-09-01 |          10 | 佐藤          | りんご       |        150 |        3
        1 |        102 | 2026-09-01 |          10 | 佐藤          | みかん       |         80 |        5
        2 |        101 | 2026-09-02 |          10 | 佐藤          | りんご       |        150 |        1
        3 |        102 | 2026-09-02 |          20 | 鈴木          | みかん       |         80 |        2
```

この状態で、顧客名を1行だけ書き換えると**更新時異常**が起きる。

```sql
UPDATE order_items_1nf SET customer_name = '佐藤花子'
WHERE order_id = 1 AND product_id = 101;

SELECT DISTINCT customer_id, customer_name FROM order_items_1nf ORDER BY 1, 2;
```

```text
 customer_id | customer_name
-------------+---------------
          10 | 佐藤
          10 | 佐藤花子
          20 | 鈴木
```

顧客番号 10 の顧客に、名前が2つできてしまった。

第2正規形から第3正規形へ、関数従属を見ながら分解する（`*` は主キー）。

```text
【第1正規形】
order_items_1nf (*order_id, *product_id, order_date, customer_id, customer_name,
                 product_name, unit_price, quantity)

  ↓ 部分関数従属を取り除く
    order_id → order_date, customer_id, customer_name   （主キーの一部だけで決まる）
    product_id → product_name, unit_price               （主キーの一部だけで決まる）

【第2正規形】
orders      (*order_id, order_date, customer_id, customer_name)
products    (*product_id, product_name, unit_price)
order_items (*order_id, *product_id, quantity)          （数量だけは主キー全体で決まる）

  ↓ 推移的関数従属を取り除く
    order_id → customer_id → customer_name              （顧客番号を通して決まる）

【第3正規形】
customers   (*customer_id, customer_name)
orders      (*order_id, order_date, customer_id)
products    (*product_id, product_name, unit_price)
order_items (*order_id, *product_id, quantity)
```

第3正規形のテーブルを作る SQL:

```sql
CREATE TABLE customers (customer_id int PRIMARY KEY, customer_name text NOT NULL);
CREATE TABLE products  (product_id int PRIMARY KEY, product_name text NOT NULL, unit_price int NOT NULL);
CREATE TABLE orders (
  order_id int PRIMARY KEY, order_date date NOT NULL,
  customer_id int NOT NULL REFERENCES customers
);
CREATE TABLE order_items (
  order_id int REFERENCES orders, product_id int REFERENCES products,
  quantity int NOT NULL, PRIMARY KEY (order_id, product_id)
);
```

分解したテーブルを**結合**すると、第1正規形の表とまったく同じ結果に戻ることを確かめた。正規化しても情報は失われない。

```sql
SELECT o.order_id, i.product_id, o.order_date, c.customer_id, c.customer_name,
       p.product_name, p.unit_price, i.quantity
FROM order_items i
JOIN orders o    ON o.order_id = i.order_id
JOIN customers c ON c.customer_id = o.customer_id
JOIN products p  ON p.product_id = i.product_id
ORDER BY 1, 2;
```

顧客名は `customers` の1か所にしかないので、書き換えても矛盾は起きない。

#### 非正規化

**非正規化**は、読み取りを速くするために、あえてデータを重複させたり、まとめたりすること。

- 結合が減るので `SELECT` は速くなるが、`INSERT`、`UPDATE`、`DELETE` は遅くなる。
- 重複したデータどうしを、常に一致させておく必要がある。
- 最初から正規化していない設計とは違う。第3正規形などに正規化し、制約を整えたうえで、意図して行うもの。

TypeScript で例えると、Redux の公式ガイドがストアの状態を正規化（id をキーにしてデータを1か所に置く）するよう勧めているのと同じ考え方。一覧表示を速くするために、別の場所にデータのコピーを持たせるのが非正規化にあたる。

## ⚠️ 試験と最新版の違い

- **PG15**: `UNIQUE` 制約に `NULLS NOT DISTINCT` オプションが追加され、NULL どうしを重複とみなすよう指定できるようになった。デフォルトの動き（NULL は複数入れられる）は変わっていない。試験の基準（PG 12〜14）には、このオプションはない。
- **SQL 標準の最新版が違う。** 試験の基準である PG14 の公式ドキュメントでは、最新の規格は SQL:2016 と書かれている。PG18 の公式ドキュメントでは SQL:2023 が最新。「最新の SQL 標準」を問う問題では、いつの時点の話なのかに注意する。なお「Core の必須機能 177 個のうち、少なくとも 170 個に準拠」という数字は、どちらのドキュメントでも同じ。

## 確認問題

### 問1（選択）

リレーショナルモデルの用語と SQL の用語の対応として、正しいものを1つ選べ。

- A. リレーション ― 列
- B. タプル ― 行
- C. 属性 ― テーブル
- D. ドメイン ― 行

### 問2（全て選択）

主キーについて、正しいものをすべて選べ。

- A. 1つのテーブルに複数定義できる
- B. NULL を含められない
- C. 候補キーの中から選ぶ
- D. 複数の列を組み合わせて定義できる
- E. 主キーに選ばれなかった候補キーは、一意にならない

### 問3（選択）

次の SQL 文のうち、DDL に分類されるものを1つ選べ。

- A. `INSERT`
- B. `SELECT`
- C. `ALTER TABLE`
- D. `DELETE`

### 問4（入力）

`GRANT` で付与した権限を取り消す SQL 文のコマンド名を答えよ。

### 問5（選択）

ACID 特性の「独立性（Isolation）」の説明として、正しいものを1つ選べ。

- A. トランザクションの操作は、すべて成功するか、すべて取り消されるかのどちらかである
- B. コミットした変更は、障害が起きても失われない
- C. コミットする前の変更は、同時に動いているほかのトランザクションからは見えない
- D. データが常に整合性制約を満たしている

### 問6（選択）

第1正規形のテーブルを第2正規形にするときに取り除くものを、1つ選べ。

- A. 繰り返し項目
- B. 部分関数従属
- C. 推移的関数従属
- D. 主キー

### 問7（全て選択）

正規化していない1つのテーブルに、すべての情報を入れた場合に起こりうる異常を、すべて選べ。

- A. 更新時異常
- B. 追加時異常
- C. 削除時異常
- D. 結合時異常

### 問8（選択）

関係演算と SQL の対応として、**誤っている**ものを1つ選べ。

- A. 選択 ― `WHERE`
- B. 射影 ― `SELECT` の列リスト
- C. 差 ― `UNION`
- D. 直積 ― `CROSS JOIN`

### 問9（全て選択）

DBMS が提供する機能として、適切なものをすべて選べ。

- A. トランザクションと同時実行の制御
- B. 障害からの復旧
- C. アクセスの認可
- D. テーブル設計の自動的な正規化
- E. データを説明するカタログ（メタデータ）

### 解答

| 問 | 答え | 解説 |
| --- | --- | --- |
| 1 | B | リレーション＝テーブル、タプル＝行、属性＝列、ドメイン＝データ型 |
| 2 | B、C、D | 主キーは1つのテーブルに1つだけ。主キーに選ばれなかった候補キーも、一意であることに変わりはない |
| 3 | C | `INSERT`、`SELECT`、`DELETE` は DML |
| 4 | `REVOKE` | `GRANT` と `REVOKE` はどちらも DCL |
| 5 | C | A は原子性、B は永続性、D は一貫性 |
| 6 | B | A の繰り返し項目は第1正規形にするとき、C の推移的関数従属は第3正規形にするときに取り除く |
| 7 | A、B、C | 追加時異常、更新時異常、削除時異常の3つ。「結合時異常」という異常はない |
| 8 | C | 差は `EXCEPT`。`UNION` は和 |
| 9 | A、B、C、E | 正規化はテーブルを設計する人が行う作業で、DBMS が自動で行うものではない |

## 参考リンク

- [Concepts（公式ドキュメント 18）](https://www.postgresql.org/docs/18/tutorial-concepts.html)
- [Glossary（公式ドキュメント 18）](https://www.postgresql.org/docs/18/glossary.html)
- [SQL Conformance（公式ドキュメント 18）](https://www.postgresql.org/docs/18/features.html)
- [SQL Conformance（公式ドキュメント 14）](https://www.postgresql.org/docs/14/features.html)
- [様々な種類のSQL文（Oracle Database SQL 言語リファレンス 11g）](https://docs.oracle.com/cd/E16338_01/server.112/b56299/statements_1001.htm)
- [SQLの種類｜DDL、DML、DCL（アナリティクス沖縄）](https://analytics-okinawa.jp/sql/2445/)（`COMMIT` と `ROLLBACK` を DCL に含める例）
- [Normalizing State Shape（Redux）](https://redux.js.org/usage/structuring-reducers/normalizing-state-shape)
- [Relational model（Wikipedia）](https://en.wikipedia.org/wiki/Relational_model)
- [Relational algebra（Wikipedia）](https://en.wikipedia.org/wiki/Relational_algebra)
- [Database（Wikipedia）](https://en.wikipedia.org/wiki/Database)
- [SQL（Wikipedia）](https://en.wikipedia.org/wiki/SQL)
- [Database normalization（Wikipedia）](https://en.wikipedia.org/wiki/Database_normalization)
- [Denormalization（Wikipedia）](https://en.wikipedia.org/wiki/Denormalization)
- [Entity–relationship model（Wikipedia）](https://en.wikipedia.org/wiki/Entity–relationship_model)
