---
title: "Actions の検証を git の pre-push に移すなら、commit と環境を隔離する"
emoji: "🧪"
type: "tech"
topics: ["git", "bash", "githubactions", "shellscript", "claudecode"]
published: false
---

自分用の設定リポジトリでは、定期実行のエージェントが出す PR の一部を自動でマージしている。その条件は GitHub Actions の CI 成功だった。2026-10-24 に、この lint と全テストを Actions から git の pre-push hook へ移した。

テストを手元で走らせるだけなら hook に 1 行書けば済む。ただ、素朴な pre-push には 2 つのずれがある。テストが見るのは push する commit ではなく作業ツリーで、動く環境は push した人のシェルそのものになる。移行の翌日、定期実行の push がこの 2 つ目のずれで止まった。

## 作業ツリーではなく commit を検証する

pre-push hook は、送ろうとしている ref を標準入力から 1 行ずつ受け取る。

```text
<local ref> <local oid> <remote ref> <remote oid>
```

hook の中で `bash scripts/check-push.sh` をそのまま呼ぶと、テストは作業ツリーのファイルを読む。未 commit の修正があれば、壊れた commit でも通る。逆に、未 commit の壊れた変更があれば、正しい commit の push が止まる。

そこで `local oid` ごとに一時ディレクトリへ commit を取り出し、そこで検証する。中核部分を抜粋する。

```bash
OBJECTS="$COMMON/objects"   # COMMON は git rev-parse --git-common-dir の絶対パス
seen=" "
while read -r local_ref local_oid remote_ref remote_oid; do
  case "$local_oid" in *[!0]*) ;; *) continue ;; esac   # 全桁 0 は ref の削除
  commit="$(git rev-parse --verify "$local_oid^{commit}")"
  case "$seen" in *" $commit "*) continue ;; esac       # 同じ commit は 1 回だけ
  seen="$seen$commit "
  snapshot="$TEMP_ROOT/$commit"
  mkdir -p "$snapshot"
  git archive "$commit" | tar -x -C "$snapshot"
  local_env="$(git rev-parse --local-env-vars)"
  status=0
  (
    unset $local_env
    git init --quiet "$snapshot" &&
    printf '%s\n' "$OBJECTS" > "$snapshot/.git/objects/info/alternates" &&
    git -C "$snapshot" read-tree "$commit" &&
    git -C "$snapshot" update-ref HEAD "$commit" &&
    cd "$snapshot" &&
    bash scripts/check-push.sh
  ) > "$log" 2>&1 || status=$?
  # status が 0 以外ならログの末尾を出して exit する
done
```

ファイルを取り出すだけなら `git archive` で足りる。それでも一時ディレクトリを git リポジトリにしたのは、検証スクリプトが `git ls-files '*.json'` のように index から対象を列挙しているからだ。`read-tree` で index を作り、`update-ref` で HEAD を合わせる。objects は `alternates` で元リポジトリを参照させ、複製しない。

ref の削除 (`local oid` が全桁 0) では何も検証しない。同じ commit を複数の ref で送ったときは 1 回だけ検証する。

## hook の中の git は呼び出し元の GIT_DIR に従う

`unset $local_env` の行は、hook が GIT_DIR などを引き継いだ場合の備えである。`git rev-parse --local-env-vars` は、リポジトリの位置を決める環境変数の一覧を返す。

```console
$ git rev-parse --local-env-vars | paste -sd' ' -
GIT_ALTERNATE_OBJECT_DIRECTORIES GIT_CONFIG GIT_CONFIG_PARAMETERS GIT_CONFIG_COUNT GIT_OBJECT_DIRECTORY GIT_DIR GIT_WORK_TREE GIT_IMPLICIT_WORK_TREE GIT_GRAFT_FILE GIT_INDEX_FILE GIT_NO_REPLACE_OBJECTS GIT_REPLACE_REF_BASE GIT_PREFIX GIT_SHALLOW_FILE GIT_COMMON_DIR
```

GIT_DIR が export されていると、`git -C` で別のディレクトリを指しても、git は GIT_DIR のリポジトリを使う。2026-10-27 に git 2.50.1 で確かめた。

```console
$ (export GIT_DIR="$PWD/work/.git"; git -C other rev-parse --git-dir)
/.../work/.git
```

この状態で `git -C "$snapshot" read-tree` を実行すると、一時リポジトリではなく push 元の index を書き換える。手元で普通に `git push` したときは、pre-push に GIT_DIR は渡っていなかった。一方、`GIT_DIR=... git push` のように環境変数を付けて push すると、hook にもそのまま渡った。被害が push 元の index に及ぶため、サブシェルの中で一覧ごと外している。

## set -e は `|| status=$?` の左側で効かない

上のサブシェルは、最初の版ではコマンドを改行で並べていた。hook 全体は `set -euo pipefail` で始まるので、途中で失敗すれば止まるように見える。ところが bash は、`||` の左側にあるコマンドの中では `set -e` を無視する。サブシェルの中も例外ではない。

```bash
set -euo pipefail
status=0
(
  false
  echo "after false: still running"
) || status=$?
echo "status=$status"
```

macOS の bash 3.2.57 で実行した結果はこうなる。

```console
after false: still running
status=0
```

`false` で止まらず、サブシェルの終了コードは最後の `echo` の 0 になった。`read-tree` が失敗しても、後続のコマンドに進んでいた。index が空なら `git ls-files` は何も返さず、ファイルを列挙する検査は対象 0 件で終わるおそれがある。

各行を `&&` でつなぐと、`false &&` の時点でサブシェルが終了コード 1 で抜ける。回帰テストでは、`read-tree` を受け取ったら exit 23 で終わる偽の `git` を PATH の先頭に置いた。その状態で hook を起動し、push が止まり、検証スクリプトが 1 回も起動しないことを確かめている。

## 実際の push が 2 回止まった

hook を入れた直後の実際の push では、検証が 2 回 push を止めた。

1 回目は、新設済みの定期実行ジョブが監視側の期待表に未登録だった。2 回目は、構造テストがジョブの起動経路を認識できなかった。そのジョブは上限付きの実行関数を共有ライブラリ経由で呼んでいたが、テストはスクリプト内に実行関数の名前が直接現れることを要求していた。テストを、共有ライブラリを source して実行関数に届く経路まで照合する形に直した。

どちらの停止の後も、`git ls-remote` で remote の main が push 前の SHA のままであることを確かめてから、修正を push し直した。3 回目で全検証が通り、push 後の HEAD と `origin/main` が同じ SHA になった。

## pre-push は push した人の環境で動く

翌 10-25 の夜、定期実行のエージェントが別の Mac から push すると、内容に関係なく pre-push が必ず落ちていた。原因は 2 つあった。

* テストの 1 本が `rg` を使っていた。定期実行を担う Mac には ripgrep が入っておらず、`rg: command not found` で落ちた。対話で使う Claude Code のシェルでは `rg` が shell 関数として使えるため、手元の検証では気付きにくい
* 定期実行は起動時に `CLAUDE_CORRECTION_LOG_DISABLE=1` を設定している。訂正ログの記録先を確かめるテストがこの変数を引き継ぎ、記録停止側の分岐に入って落ちた

`rg` は `grep -F` に置き換え、テストの中で変数を外す修正を出した。2026-10-27 時点ではこの PR はレビュー待ちである。GIT_DIR の件と同じく、hook は呼び出し元の環境変数をそのまま受け取る。Actions では毎回新しいランナーで動いていたため、定期実行の環境変数がテストに届くことは無かった。

## 自動マージ側は検証記録で commit に結び付ける

Actions をやめると、自動マージの条件だった「CI 成功」が無くなる。代わりに、hook は検証が通った commit ごとに記録を残す。

```json
{"commit": "...", "tree": "...", "status": "passed", "log": ".../push-checks/<commit>.log", "log_hash": "...", "verified_at": "..."}
```

記録は `git rev-parse --git-common-dir` の下に置くので、worktree から push しても同じ場所に集まる。自動マージの前には、PR の head について次の 3 点を照合する。

* 記録の `commit` と `tree` が、その commit の値と一致する
* `status` が `passed` である
* ログを `git hash-object` で計算し直した値が `log_hash` と一致する

検証を始める前に古い記録を消し、成功したときだけ一時ファイルから `mv` で置く。これで、失敗した検証の後に以前の成功記録が残ることは無い。fixture のテストでは、成功後にログへ 1 行足すと照合が失敗することも確認した。

hook 側の対策は、commit の取り出しと GIT_DIR の除去と `&&` の連結で、どれも移行当日の 10-24 のうちに入れた。残ったのはテスト側である。push する可能性のあるマシンごとに依存コマンドがそろっているか、呼び出し元の環境変数で分岐が変わらないか。今回はこの 2 点を、定期実行の Mac から push して初めて見つけた。
