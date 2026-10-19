---
title: "dbt のスタースキーマに Genie Agent を載せて、ベンチマーク 10 問で確かめるまでの手戻り"
emoji: "🧞"
type: "tech"
topics: ["databricks", "genie", "dbt", "unitycatalog", "datamodeling"]
published: false
---

dbt で作ったスタースキーマに、自然言語で質問できる Databricks の Genie Agent (旧称 Genie Space) を載せた。対象は架空の雑貨 EC を題材にした学習用データで、注文ヘッダーと注文明細のファクト 2 表に、顧客・商品・日付のディメンション 3 表を加えた計 5 表だ。

作業は 2026-10-10 から翌日にかけて行い、ベンチマーク 10 問がすべて正解になるまでに 4 回つまずいた。手で 1 度通した手順は、dbt の manifest から Agent の定義を組み立てるスクリプトと、ベンチマークを実行するスクリプトにまとめ直した。

## dbt の説明が Agent に届いていなかった

Genie は Unity Catalog (以下 UC) の表と列のコメントを、質問を SQL に直すときの文脈に使う。dbt の YAML には、各表の粒度、SCD Type 2 の有効期間、売上の定義を書いてあった。ところが UC 上のコメントは空だった。dbt は既定では YAML の `description` を UC に書き込まない。

最初は YAML の説明を Agent 側の表と列の説明へ手で写した。これでは dbt 側を直すたびに写し直しが要る。そこで [dbt の persist_docs](https://docs.getdbt.com/reference/resource-configs/persist_docs) を有効にし、モデルの作成時に UC へ書き込ませた。

```yaml
+persist_docs:
  relation: true
  columns: true
```

コメントはモデルを `CREATE OR REPLACE` で作り直すときに入る。設定を足しただけでは既存の表は空のままなので、YAML を変えたらモデルを再実行する。表の所有者がサービスプリンシパルの場合は、その資格情報で作り直す必要がある。OAuth M2M のプロファイルでは `databricks auth token` でトークンを取れなかった。代わりに Python SDK の `Config(profile=...).authenticate()` を使った。

## カタログの MANAGE では表を読めなかった

次に詰まったのは権限だ。Agent に質問すると SQL は生成されるが、その SQL の実行が失敗した。

```
INSUFFICIENT_PERMISSIONS ... USE CATALOG
```

このとき対象のカタログに付けていた権限は `MANAGE` だった。`MANAGE` だけでは表の中身を読めず、`USE CATALOG`、`USE SCHEMA`、表の `SELECT` が別に要る。

Genie Agent では、権限の確認がどの資格情報で行われるかにも注意がいる。SQL Warehouse には Agent の作者の資格情報が埋め込まれる。一方、データを読めるかどうかは質問した本人の UC 権限で判定される。作者と利用者の両方に、対象の表の `SELECT` を付けておく。

## Bundle の deploy は 3 回止まった

UI で作った Agent は定義が手元に残らず、別の環境で同じものを作り直せない。そこで [Declarative Automation Bundles のリソース](https://docs.databricks.com/aws/en/dev-tools/bundles/resources) (`genie_spaces`) として、定義ファイルをリポジトリで管理することにした。deploy は次の順に 3 回止まった。

1. Bundle が作るスキーマのために、カタログの `CREATE_SCHEMA` 権限が足りなかった
2. `mode: development` の target では Agent の置き場所が `<root_path>/resources` になる。このフォルダが無く、更新が 404 (`Tree node ... does not exist`) で止まった。`databricks workspace mkdirs` でフォルダを作ってから再実行した
3. `bundle plan` が `lineage mismatch in state files` を出した。ローカルとリモートの state を比べ、実在しないリソースを記録していた側を別名で退避してから plan し直した

先に UI で作った Agent がある場合、そのまま deploy すると Agent が二重に作られる。`databricks bundle deployment bind <key> <SPACE_ID> --target <t>` で既存の Agent と結び付けてから deploy する。

定義ファイル (serialized space) は Bundle の変数を展開しない。表の識別子はカタログ名とスキーマ名まで固定で書くことになる。UI で手直ししたら `databricks genie get-space <ID> --include-serialized-space` で書き出し、定義ファイルへ戻す運用にした。

## 丸めの指示が 1 問を不正解にした

ここまでで Agent は質問に答えるようになった。[Genie のベンチマーク](https://docs.databricks.com/aws/en/genie/benchmarks)は、質問と正解 SQL の組を登録し、Agent の生成 SQL の結果と突き合わせる機能だ。10 問を登録して実行した結果は 9/10 だった。外れたのは「平均注文金額は？」で、正解 SQL は丸めない 1 列を返す。

原因は全体向けの文章の指示にあった。指示には次の 1 行を書いていた。

```
* 金額の単位は円。金額は整数に丸めて表示し、件数・金額には集計期間を添える。
```

指示の文面は「表示し」だったが、Genie はこれを SQL の段階で守り、`ROUND` と集計期間の列を足した。結果の形が正解 SQL と違うため、判定は BAD になった。判定理由は `RESULT_MISSING_COLUMNS` で、値の誤りではなく列の不一致として扱われた。

指示を次のように直し、回答例として登録していた SQL からも `ROUND` を外した。

```
* 金額の単位は円。SQL では金額を丸めず、集計期間の列も足さない。要約文で金額を整数に丸め、集計期間を書き添える。
```

再実行すると 10/10 になった。表示の整え方は要約文で行い、SQL の結果は正解 SQL と同じ形に保つ。割合のように SQL の段階で丸めたい値がある場合は、ベンチマークの正解 SQL も同じ形にそろえる。

[Genie のベストプラクティス](https://docs.databricks.com/aws/en/genie/best-practices)には、文章の指示は SQL 式と回答例で表せないときの最後の手段にせよ、とある。今回の 1 問は、その文章の指示が SQL にまで効いてしまった例だった。

## 手順をスクリプトにまとめ直す

4 つの手戻りは、どれも次に別のスタースキーマで Agent を作るときに再び起こり得る。そこで手順を 2 つのスクリプトに分け、コーディングエージェントに読ませる Genie 用の skill に同梱した。

1 つ目は、dbt の manifest と短い spec (JSON) から Agent の定義ファイルを組み立てるスクリプトだ。manifest からは表、説明、dbt の `relationships` テストを取り、テストが定義する外部キーを結合関係にする。日付キーのように `relationships` テストの無い結合は、spec の `extra_joins` に書く。指標 (measures)、絞り込み (filters)、回答例、ベンチマークも spec に置く。

初版では、`relationships` テスト由来の結合と `extra_joins` に書いた結合が重なり、結合関係が 8 本になった。手作業版は 6 本で、重複を除く処理を足して 6 本に一致させた。表 5、指標 5、回答例 6、ベンチマーク 10 も手作業版と一致している。

2 つ目はベンチマーク用のスクリプトで、2 つのサブコマンドを持つ。`check-sql` は回答例とベンチマークの正解 SQL を実データで実行する。生成した定義の SQL 16 本はすべて実行に成功した。`run` はベンチマークを実行して完了を待ち、BAD の判定理由、正解 SQL、生成 SQL を並べて出す。実際の Agent で実行した結果は 10/10 だった。

## 未確認のこと

スクリプトで生成した定義ファイルは、まだ `create-space` に渡して作成していない。形式は手作業版と同じで、手作業版は API が受け付けた。BAD が出たときの表示も、10/10 だったこの実行では通っていない。表示する項目名は、同じ日に手作業で結果を分析したときの名前 (`assessment_reasons` など) に合わせてある。

対象も dbt の manifest に限られる。Dataform など、ほかの変換ツールの定義からは組み立てられない。

次に別のスタースキーマで Agent を作るときは、権限と UC コメントを先に確かめる。文章の指示を書く前に、表示の整え方を要約文の側に置けているかも見直す。
