---
title: "Claude Code の認証エラー検出 hook が、自分の修正コミットを映した git log に反応した"
emoji: "🔁"
type: "tech"
topics: ["claudecode", "hooks", "bash", "jq", "自動化"]
published: false
---

Claude Code の PostToolUse hook で、Bash の出力から gcloud / BigQuery / Databricks の認証エラーを拾っている。認証切れは同じコマンドをリトライしても直らないので、検出したら「リトライせず、ユーザに `! gcloud auth login` 等の実行を依頼して完了報告を待て」という additionalContext を注入する。パターンは実際のエラー文言に基づく grep の正規表現で、不一致なら即 exit 0 の軽い hook だ。

この hook が 2026-07-06 に、`git log` を表示しただけで発火した。

## 何に一致したのか

同じ日の少し前に、この hook の誤検知を一度直していた。素の `databricks auth login` というパターンが、履歴のサンプルや hook ファイル自身を cat した出力にも一致していた (同じセッションで 2 回再現) ので、「run ... databricks auth login」のようにエラー文脈で実行を促される形だけに絞る修正だ。コミットメッセージには経緯として、直す前のパターン文字列を引用して書いた。

その後 `git log` で直近のコミットを表示した。出力には一次修正のメッセージが含まれ、そこに引用されたパターン文字列が hook の grep に一致した。認証エラーは 1 つも起きていない。hook は自分の修正履歴を読んで発火した。

## 原因は 2 つ重なっていた

1 つ目は行の潰し込み。tool_response を `jq -r '.tool_response | tostring'` で取り出していたが、tostring は JSON 表現の 1 行文字列を返すので、改行は `\n` のエスケープに潰れる。「run ... databricks auth login」や「Error: ... unauthenticated」のように行内の共起を意図したパターンが、本来は別々の行にある語をまたいで一致していた。

2 つ目は引用への反応。git log、cat、grep のような表示系コマンドの出力には、過去のコミットメッセージ、ドキュメント、hook ファイル自身の中身が入る。テキストとしてはパターンに一致するが、live な認証エラーではない。一次修正のようにパターンをエラー文脈に絞っても、パターン文字列そのものを引用するテキストは防げない。hook ファイルにはパターンがリテラルで書いてあるので、そのファイルを表示すれば必ず一致する。

## 対策

パターンではなく、入口と入力の形を直した。

```bash
CMD=$(printf '%s' "$INPUT" | jq -r '.tool_input.command // ""' 2>/dev/null || true)
SKIP_CMD='(^|&&|;)[[:space:]]*(git[[:space:]]+(log|show|diff|blame)|cat|bat|head|tail|less|more|grep|rg)([[:space:]]|$)'
printf '%s' "$CMD" | grep -qE "$SKIP_CMD" && exit 0

OUTPUT=$(printf '%s' "$INPUT" | jq -r '[.tool_response | .. | strings] | join("\n")' 2>/dev/null || printf '%s' "$INPUT")
```

表示系コマンドがステートメントの先頭 (行頭、`&&`、`;` の直後) にあれば、出力を見ずに exit 0 する。対象はローカル読み取りだけで認証エラーを起こし得ないコマンドに限り、fetch / pull / push や gh は入れていない。パイプの直後は先頭に含めないので、`databricks jobs list --output json | grep job_id` のような受け側の grep では skip されず、live のエラーは今までどおり拾う。

tool_response は文字列値を列挙して実改行で結合する形に変えた。grep の行単位マッチが働き、パターンは同一行内の共起に限定される。単一行を前提に書いていた `[^|]*` は `.*` に置き換えた。

表示系コマンドが複合コマンドの先頭に 1 つでもあれば全体が skip されるので、検出漏れは増える。それでもこの hook は fail-open で、検出を漏らして失うのはリトライ 1 回分だ。誤検知のたびに認証依頼の指示が注入されて作業が止まるほうが高くつくので、誤検知の削減を優先した。

## テストの出力にもパターンを出せない

回帰テストは正例 4 件と負例 4 件で、負例には git log が一次修正のコミットメッセージを映すケースと、hook ファイル自身を cat するケースをそのまま入れた。

テストの stdout には PASS / FAIL 以外を出せない。失敗時にフィクスチャの中身を表示すると、そこに含まれるパターン文字列が、そのテストを実行している Bash 呼び出しの tool_response に載る。すると live 環境で動いている同じ hook が、テストの実行に反応する。

コマンド出力を正規表現で監視する hook は、いずれ自分自身の痕跡 (修正のコミットメッセージ、hook のソース、テストのフィクスチャ) を読む。パターンを精密にする前に、どのコマンドの出力を見ないかを決めておくほうが早かった。
