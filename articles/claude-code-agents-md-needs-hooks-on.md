---
title: "Claude Code に AGENTS.md を読ませたら、hooks を切った実行だけ無視された"
emoji: "🗂️"
type: "tech"
topics: ["claudecode", "claude", "agentsmd", "dotfiles", "plugin"]
published: false
---

主に使うエンジンを Codex にしたので、リポジトリの指示ファイルを vendor 非依存の `AGENTS.md` 一本にしたかった。Claude Code は既定では `CLAUDE.md` しか読まないので、`AGENTS.md` しか持たないリポジトリでは Claude Code 側だけ指示が欠ける。

## instructionFiles を書き換える

Claude Code には builtin plugin `agents-md` があり、`instructionFiles` というオプションで挙動を選べる。既定値は `claude-md-or-agents-md` で、`CLAUDE.md` があるリポジトリでは `AGENTS.md` が完全に無視される。両方を読ませたいので `claude-md-and-agents-md` に変える。

```json
{
  "pluginConfigs": {
    "agents-md@builtin": {
      "options": { "instructionFiles": "claude-md-and-agents-md" }
    }
  }
}
```

`~/.claude/settings.json` に足した。`AGENTS.md` は plugin が `project` 種別の指示ファイルとしてエンジンへ返すので、`CLAUDE.md` と同じ位置・同じ枠で会話に入る。project 指示を省くサブエージェント (Explore や Plan など) では、`AGENTS.md` も `CLAUDE.md` と同じく省かれる。

## 読まれているかを確かめる

設定を足しただけでは、実際に読まれているかは分からない。一時ディレクトリを作り、指示ファイルの中身にしか書いていないトークンを `claude -p` (haiku) に答えさせて確認した。

| 条件 | 期待 | 結果 |
|---|---|---|
| `AGENTS.md` のみ、`instructionFiles` を `claude-md` に上書き | 読まれない | `UNKNOWN` |
| `AGENTS.md` のみ、`claude-md-and-agents-md` | 読まれる | `8831-QX` |
| `CLAUDE.md` のみ (対照) | 読まれる | `8831-QX` |
| `CLAUDE.md` と `AGENTS.md` の両方 | 両方読まれる | `AAA-111` と `BBB-222` |

ここまでは想定どおりだった。

## hooks を切った実行だけ結果が変わった

同じ確認を別の条件でも回した。`--settings '{"disableAllHooks": true}'` を付けて `hooks` を止めた状態で `AGENTS.md` のみのディレクトリを読ませたところ、期待していた `8831-QX` ではなく `UNKNOWN` が返った。`instructionFiles` はさっき設定した `claude-md-and-agents-md` のままで、他の条件は変えていない。

原因は `agents-md` plugin 側の `isAvailable` 判定にあった。plugin は自分が有効かどうかを hooks の無効化フラグで見ており、`disableAllHooks: true` の実行では plugin 自体が無効になる。無効になれば `instructionFiles` の値は関係なく、`CLAUDE.md` だけが読まれる挙動 (実質は何も読まれない、`AGENTS.md` しか無いリポジトリでは指示ゼロ) に戻る。

`disableAllHooks` は hooks を一時的に止めてデバッグしたいときに使うフラグで、指示ファイルの読み込みとは無関係に見える。実際には plugin の有効判定を経由してつながっていた。

## 対策

`AGENTS.md` だけのリポジトリで `--settings '{"disableAllHooks": true}'` を付けて実行すると、`AGENTS.md` の中身は一切コンテキストに入らない。この状態で動作確認をすると「指示が効いていない」のか「hooks を切ったから指示ファイルごと落ちた」のかを取り違える。

このハーネスでは既存の `CLAUDE.md` symlink を残す判断をした。他マシンや `instructionFiles` 未設定の環境、そして hooks を切った実行では `CLAUDE.md` だけが読まれるので、`AGENTS.md` へ一本化しても `CLAUDE.md` symlink は消さない。新規リポジトリでは作らないが、既存のものはそのまま残す。

`disableAllHooks` を付けた一時実行で指示ファイルの中身を前提にした検証をするときは、この 1 行を思い出す必要がある。フラグ名からは指示ファイルへの影響は読み取れない。
