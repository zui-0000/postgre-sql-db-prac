# postgre-sql-db-prac

PostgreSQL を学ぶためのリポジトリ。当面は OSS-DB Silver の試験勉強に使う。

## はじめかた

```sh
docker compose up -d --wait   # db と sandbox が起動する
```

| 使いたい環境 | 入り方 | 用途 |
| --- | --- | --- |
| db | `docker compose exec db psql -U postgres` | SQL、権限、VACUUM など。サーバーは起動済み |
| sandbox | `docker compose exec sandbox bash` | `initdb` や `pg_ctl` を自分で実行する。サーバーは起動していない |

sandbox の中では、次のようにしてクラスタを作ってサーバーを起動する。

```sh
initdb -D ~/data
pg_ctl -D ~/data -l ~/server.log start
psql
```

詳しい使い方と、この構成にした理由は [01 環境構築](docs/01-環境構築.md) にまとめている。

## 構成

```text
.
├── compose.yaml        # PostgreSQL 18.6 の db と sandbox
├── docs/               # 試験に依存しないドキュメント
│   └── 01-環境構築.md
└── oss-db/
    └── silver/         # OSS-DB Silver の学習ノート
```

## 学習ノート

- [OSS-DB Silver](oss-db/silver/README.md)
