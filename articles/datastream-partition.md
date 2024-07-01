---
title: "Datastreamで作成されたテーブルをパーティション化するスクリプトを作った"
emoji: "🗂️"
type: "tech"
topics: ["Data", "BigQuery", "Datastream"]
published_at: 2023-11-27
published: true
publication_name: "wed_engineering"
---

WED株式会社でデータエンジニアをしている[ryuya-matsunawa](https://github.com/ryuya-matsunawa)です。

この記事ではBigQueryで作成したテーブルをパーティション化するスクリプトについて紹介します。

## はじめに
このスクリプトを作った背景は、Datastreamで作られるBigQueryのテーブルはパーティション化されておらず、巨大なレコード数になるとクエリの実行に時間がかかるため、パーティション化する必要があったためです。
（本当はGCP側で設定できると嬉しい）


## スクリプト
スクリプトは以下の手順で処理を行います。

1. テーブルのスキーマを取得
2. Datastreamで作成されたテーブルを削除
3. テーブルをパーティション化して作成

詳しく説明していきます。

まず、以下のようにテーブルのスキーマを取得します。

```python
table = client.get_table(f"{dataset}.{table_name}")
schema = table.schema
```

bigquery.Tableの中にはschemaというプロパティがあり、これを取得することでテーブルのスキーマを取得することができます。

次に、Datastreamで作成されたテーブルを削除します。

```python
client.delete_table(f"{dataset}.{table_name}")
```

最後に、テーブルをパーティション化して作成します。

```python
new_table = bigquery.Table(f"{project_id}.{dataset}.{table_name}", schema=schema)
if primary_key:
  new_table.clustering_fields = ["id"]
new_table.time_partitioning = bigquery.TimePartitioning(
  type_=bigquery.TimePartitioningType.DAY,
  field=f"{partition_key}",
  expiration_ms=None,
)

client.create_table(new_table)

if primary_key:
  query = f"""
  ALTER TABLE {project_id}.{dataset}.{table_name}
  ADD PRIMARY KEY (id) NOT ENFORCED;
  ALTER TABLE {project_id}.{dataset}.{table_name}
  SET OPTIONS (
      max_staleness = INTERVAL '0-0 0 0:15:0' YEAR TO SECOND
  );
  """

  client.query(query).result()
```

最初に取得したスキーマにパーティションを追加してテーブルを作成します。

Datastreamで作成されるテーブルはクラスター化されるため、クラスタリングを有効にしておきます。

テーブルを作成後、主キーの設定などを追加しています。

## まとめ
Datastreamで作成されたテーブルをパーティション化するスクリプトを紹介しました。

スクリプト自体は単純ですが、一度Datastreamを止めないといけないので、GCP側で設定できるようになると嬉しいです。
