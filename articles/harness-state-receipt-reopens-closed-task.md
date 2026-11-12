---
title: "完了した作業状態が、receipt を読むたびに未完了へ戻っていた"
emoji: "🔁"
type: "tech"
topics: ["claudecode", "codex", "python", "agent", "testing"]
published: false
---

Claude Code と Codex で共有している dotfiles では、複数工程の作業を状態ファイルで管理している。2026 年 11 月 12 日、この dotfiles でセッションを始めるたびに、4 件の作業状態が「未完了」と案内されていた。どれも終わったか、別の形で片付いた作業である。

## 状態ファイルと起動時の案内

状態ファイルには、作業の目的と受入条件を書く。受入条件は、検証コマンドと、そのコマンドが見る対象のパスの組である。`harness-state.py verify` は検証コマンドを実行し、終了値とログに加えて、対象ファイルのハッシュを証拠として保存する。`harness-state.py receipt` は、全条件の証拠が有効で、blocker と結果不明の外部操作が無い場合だけ終了値 0 を返す。対象ファイルのハッシュが保存時と変わった証拠は、古い (stale) ものとして扱う。

起動時には SessionStart hook が、現在の workspace に紐づく状態を調べる。完了していないものがあれば、次の文面を会話に注入する。

```text
[TASK RECOVERY] このworkspaceには未完了の作業状態がある: <状態ファイルのパス>。今回の依頼に該当する場合は状態を読み、現在の差分・外部操作結果と照合して再開する。古い検証証拠や結果不明の操作を成功・未実行と推定しない。
```

hook が読むのは、workspace ごとのポインタファイルにある `closed` の値である。この値は、状態ファイルを更新するたびに呼ばれる `attach()` が書く。

## receipt を読むと閉じた作業が開き直る

修正前の `attach()` は、`closed` を次の式で決めていた。

```python
'closed': task.get('phase') == 'complete' and receipt(path)['complete']
```

作業を終えた時点では、全条件の証拠が有効なので `closed` は真になる。問題は、その後に同じ workspace で別の作業がファイルを変えた場合である。証拠は stale になり、`receipt()` の `complete` は偽を返す。証拠が古くなること自体は、後の変更として正常な動きである。

`receipt` は確認のためのコマンドだが、CLI からの実行では最後に `attach()` も呼んでいた。

```python
elif args.action == 'receipt':
    result = receipt(args.state); record_outcomes(args.state)
    attach(args.state)
```

このため、完了した作業の receipt を後から読むだけで `closed` が計算し直され、偽に戻る。11 月 12 日の調査中にも、11 月 10 日に完了した作業の状態が実際に開き直った。

この経路には 10 月 24 日に足したテスト `test_closed_task_is_not_reopened_by_unrelated_future_changes` があり、通っていた。中身は、完了時に `attach()` を 1 回呼び、対象ファイルを変え、起動時の案内に出ないことを確かめるものである。ファイルを変えた後に `attach()` をもう一度呼ぶ手順が無く、receipt を読んだときの再計算は試していなかった。

対策では、一度 `closed` になった作業は、`phase` が `complete` のあいだ閉じたままにした。

```python
delivered = any(row.get('state') == original and row.get('closed') for row in entries)
entry = {...,
         'closed': bool(task.get('closure'))
         or (task.get('phase') == 'complete' and (delivered or receipt(path)['complete']))}
```

作業を再開したいときは、`phase` を `complete` 以外へ戻す。追加したテストは、完了後にファイルを変えてから `attach()` を呼び直し、案内に出ないことを確かめる。続けて `phase` を `in_progress` に戻し、案内に出ることも確かめる。

## 残っていた 2 つの原因

4 件の案内には、別の原因も重なっていた。

1 つは、成功以外で終えた作業を閉じる手段が無かったことである。評価の結果で不採用にした作業、後続の作業へ引き継いだ作業、後の変更で置き換わった作業がこれにあたる。案内を止めるには `phase` を `complete` にするしかなく、それは成功と区別できない。そこで理由を必須にした `close` を足した。

```bash
python3 scripts/harness-state.py close <状態ファイル> --reason "評価の結果、不採用とした"
```

閉じた作業も、receipt の `complete` は偽のままで、終了理由を `closure` として返す。案内は止まるが、成功として報告されることはない。外部操作が `pending` か `unknown` のまま残っていれば、`close` はその操作名を出して失敗する。結果の分からない操作を、閉じることで見えなくしないためである。

もう 1 つは、作業の途中で状態ファイルの `workspace` を書き換えた場合である。新しい workspace のポインタには項目が入るが、旧 workspace のポインタには古い項目が残り、更新されないまま案内され続けた。起動時の hook は、状態ファイル本体の `workspace` が現在の作業ディレクトリと違えば、その項目を飛ばすようにした。

追加したテストは 4 関数で、`close` の検査は `pending` と `unknown` の 2 通りで実行する。

## 読むコマンドが書いていないか

3 つのうち、既存のテストをすり抜けていたのは 1 つ目である。`closed` はポインタファイルに保存された値だが、実際には `attach()` のたびに、その時点の証拠から計算し直していた。しかも計算のきっかけには、読み取り用のはずの receipt が含まれていた。

完了の判定を証拠から導く設計なら、確認するだけのコマンドが状態を書き換えていないかを見ておく。テストでは、完了後に対象を変えたうえで、その確認コマンドをもう一度通す手順を入れる。
