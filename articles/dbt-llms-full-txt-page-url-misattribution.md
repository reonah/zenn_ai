---
title: "dbt の llms-full.txt 検索が、本文の最初のリンクを返していた"
emoji: "🔗"
type: "tech"
topics: ["dbt", "llmstxt", "claudecode", "awk", "python"]
published: false
---

Claude Code の skill として、dbt の公式ドキュメントの全文 (`https://docs.getdbt.com/llms-full.txt`) をキーワードで検索し、該当ページの URL を返すスクリプトを使っている。2026-07-06 に公式の agent skills から取り込んだ版で、2026-09-30 に見直したところ、返ってくる URL がキーワードを含むページではなかった。

## 再現

旧スクリプトで `ad hoc queries` を検索すると、次の 2 件が返った。

```
https://docs.getdbt.com/sql-reference/select.md
https://docs.getdbt.com/best-practices/how-we-structure/4-marts.md
```

llms-full.txt でこの語句を含むページは「About the Discovery API」「SQL CASE WHEN」「SQL ORDER BY」の 3 つで、SELECT のページも marts のページも含まれていない (2026-10-02 に llms-full.txt を取得して確認)。CASE WHEN のページ本文に最初に出てくるリンクが `select.md`、ORDER BY のページ本文に最初に出てくるリンクが `4-marts.md` だった。

## 原因

旧スクリプトは awk でページを区切り、その中の最初のリンクをページの URL とみなしていた。コメントにもそう書いてある。

```awk
# 1. Page delimiter is: --- followed by ###
# 2. After delimiter, first docs.getdbt.com link is the page URL
```

llms-full.txt の各ページは `---` の区切りと `### タイトル` の見出しで始まるが、そのページ自身の URL は書かれていない。本文中の最初のリンクは、たいてい関連する別ページへのリンクになる。

同じ前提から、さらに 2 つの取りこぼしが出ていた。

- ページ URL が見つかるまでの行は、直前のページの URL に帰属する。新しいページの冒頭にある一致は、前のページの結果として返る
- ファイル先頭のページは前に `---` が無いので、URL が空のまま読み飛ばされる。先頭は「About the Discovery API」で、上の検索結果に出てこなかったのはこのためだ

## 対策

URL は索引の llms.txt から取ることにした。llms.txt は 1 ページ 1 行で、タイトルと URL を対で持っている。

```
- [SQL CASE WHEN](https://docs.getdbt.com/sql-reference/case.md): CASE statements allow you to ...
```

スクリプトは標準ライブラリだけの Python に書き直し、既存のシェルの入口からは `exec` で呼ぶ。全文検索の部分は、llms-full.txt の見出しを llms.txt のタイトルで URL に引き直す。

```python
sections = re.split(r'^---\s*$', full, flags=re.M)
for section in sections:
    heading = re.search(r'^### (.+)$', section, re.M)
    if heading and heading[1] in titles:
        recognized += 1
        if matches(section):
            found.update(titles[heading[1]])
```

索引に無い見出しのページは、どの URL にも帰属させない。同じタイトルが索引に複数あれば、候補を両方返して個別ページで確かめる。見出しが 1 つも索引と一致しなければ、結果を返さずエラーにする。既定の検索は llms.txt の索引だけを対象にし、全文は `--full` を付けたときだけ読む。

同じ `ad hoc queries` を `--full` で検索すると、次の 3 件になった。

```
https://docs.getdbt.com/docs/dbt-apis/discovery-api.md
https://docs.getdbt.com/sql-reference/case.md
https://docs.getdbt.com/sql-reference/order-by.md
```

## キャッシュも壊れ得た

旧スクリプトは `curl -sL "$FULL_URL" -o "$CACHE_FILE"` で 24 時間のキャッシュを直接上書きしていた。`-f` が無いので、サーバーがエラーを返しても curl は終了コード 0 で応答本文をキャッシュに書く。更新時刻も新しくなるため、その後 24 時間は壊れたキャッシュで検索が続く。

新しい版は、取得した本文が `# dbt Developer Hub` で始まるかを確かめてから、一時ファイルに書いて `os.replace` で差し替える。HTTP 503 と HTML のエラーページを返す場合の 2 件をテストに入れ、どちらでも元のキャッシュが残ることを確認した。

テストは単体 10 件で、ページの誤帰属、先頭ページ、同名タイトル、キャッシュの再利用と保全を扱う。Hypothesis のプロパティテストでは、索引検索の結果が索引に存在する一致 URL だけであることと、同じキーワードを重ねても結果が変わらないことを確かめた。llms-full.txt から URL を取るときは、本文の中を探さず、見出しを llms.txt で引き直す。
