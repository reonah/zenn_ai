---
title: "Claude Code の skill を 125 個作ってから、どれが動いているのかを数えた"
emoji: "📏"
type: "tech"
topics: ["claudecode", "claude", "anthropic", "agentskills"]
published: false
---

`~/.claude/skills` を数えたら 125 個あった。2026 年 7 月の話である。

増やすのは簡単だった。同じ作業を 2 回繰り返すたびに手順を skill に落としてきた結果で、1 つ 1 つには作った理由がある。困ったのは、そのうちどれが実際に呼ばれているのかを答えられなかったことだ。減らす話をしようにも、材料が無い。

きっかけは Anthropic の [Lessons from building Claude Code: how we use skills](https://claude.com/blog/lessons-from-building-claude-code-how-we-use-skills) で、社内でどう書き、どう配り、どう計測しているかが書かれている。計測のくだりで、自分の側には測る手段が 1 つも無いことに気づいた。

## 増やすときに払っているもの

skill の本体は、必要になったときだけ読まれる。description は違う。全 skill 分がセッションの頭に並ぶ。

数えたら合計 38,648 字だった。125 個で割ると 1 つあたり平均 309 字。skill を 1 つ増やすというのは、以後のすべてのセッションに 300 字を足すということでもある。本体の行数は全部で 19,801 行あったが、こちらは読まれるときにだけ払うので、増やす判断で見るべきなのは description の側だった。

## まず遡って数えた

計測の仕組みを入れる前に、手元に残っているものを先に数えた。`~/.claude/projects/**/*.jsonl` の transcript に Skill ツールの呼び出しが残っている。

2026-05-02 以降の約 12 週間分で 67 回。skill 名でユニークにすると 18 個だった。

上位は harness-audit が 5 回、skill-import・pr-review-comment・obsidian-workflow・hook-authoring が各 4 回。ハーネス自体をいじる作業と、レビューと、Obsidian。自分が繰り返しているのは実際にその 3 つだったことになる。

もう 1 つ分かったのは、`~/.claude/skills` に存在しない名前でも呼ばれていたことだ。schedule が 4 回、loop が 3 回、code-review が 2 回。プラグインや組み込み由来のもので、自分が書いた skill ではない。改名前の名前で呼ばれた記録も混ざっていた。

### この数字では消せない

125 個のうち 18 個。残り 107 個は一度も起動していない、と読みたくなる。読まなかった。

transcript には保持期間がある。期間より前の起動は残っていない。この数字では「記録がない」と「使っていない」が分離できず、107 件の中には実際に使ったが記録の消えたものが混ざっている。削除候補の一覧としては使えない。

## 記録を残す側を作る

そこで PreToolUse hook (matcher は `Skill`) を書いて、起動のたびに `{ts, skill, args_len, cwd, session_id}` を JSONL に 1 行追記するようにした。49 行のシェルスクリプトである。

判断が要ったのは 4 か所。

- args の本文は書かない。長さだけ記録する。プロンプト本文がログに平文で残るのを避けたい。起動回数を数えるだけなら skill 名で足りる
- hook の中でも `tool_name` を自分で確認する
- 2MB を超えたらローテーションする。1 行 150 バイト前後なので 1 万回以上の起動を保持できて、無限には伸びない
- 書き込みに失敗しても jq が転んでも、常に exit 0 で返す。ログが取れないことで本体の作業が止まるほうが困る

2 つめは保険である。公式ドキュメントの PreToolUse matcher の例には Bash / Edit / Write と MCP のツール名しか挙がっておらず、`Skill` が matcher として効くという確証がなかった。効かずに別のツールへ配線された場合でも、素通りして何もしないようにしてある。

集計スクリプトは hook のログと transcript の両方から読み、`(skill, session_id, timestamp)` の集合で重複を除く。hook 以降は恒久記録、それ以前は保持期間内だけの遡及、という二重構造で当面は動く。

## 一覧に description が出ない skill があった

作業中に別のことが気になった。セッション冒頭の skill 一覧で、名前だけで並ぶものと description 付きで並ぶものが混ざっている。

description が長すぎて落とされているのだと思った。名前だけで出ていた databricks-model-serving、pdf、docx、grilling、md-to-html の SKILL.md を順に開いて確かめたところ、いずれも正常な description を持っていた。

長さでも説明がつかなかった。公式の上限は description と `when_to_use` の合計で 1,536 字。手元の最長は databricks-app-design の 847 字で、誰も上限には達していない。しかもその 847 字はフル表示されていて、719 字の databricks-model-serving が名前だけだった。表示を制御する `skillOverrides` 設定は `settings.json` にも `settings.local.json` にも無い。

決め手は同じセッションの途中で来た。md-to-html が、今度は description 付きの一覧として改めて注入された。skill 側は 1 文字も変えていない。

出し分けているのはハーネスのほうだった。つまり description を書き換えても、冒頭一覧での扱いは変えられない。

ただし、効く場所が別にあることも同時に分かった。起動トリガらしい語 (ユーザの発話例や「〜と言われたとき」) を description に持たない skill が 6 件あり、うち 5 件は外部由来 (pdf / skill-creator / theme-factory / web-artifacts-builder / webapp-testing) だった。英語の description のまま日本語で頼んでいれば、モデルの側で結びつかない。一覧にどう出るかを気にしていたが、直すべきなのは起動するかどうかのほうだった。

## when_to_use で 2 回引っかかった

`when_to_use` は skill 一覧で description に連結される frontmatter フィールドで、description 本文を触らずにトリガ語だけ足せる。外部由来の 5 件とローカルの grilling に追加し、追加後にこの 6 件が description 付きで一覧へ再提示されるところまで同じセッション内で確認した。合計は 300 から 450 字で、1,536 字の上限には余裕がある。

引っかかったのは 2 つ。

1 つは、上流を再同期すると消えること。外部由来の skill は上流と 1 対 1 で保ちたいので、いずれ上書きされる運命にある。README の出典テーブルに「when_to_use 追加」の列を足してどれに入れたかを残し、取り込み手順 (skill-import) の Step 5 に入れ直す工程を書いた。上流との差分を 1 フィールドに閉じ込めてあるのは、この作業を軽くするためでもある。

もう 1 つは、skill-creator に同梱されている `quick_validate.py` が `when_to_use` を未知キーとして弾いたことだった。このバリデータの許可キーは allowed-tools / description / license / metadata / name に限られている。一方で Claude Code の公式ドキュメントには `when_to_use` が記載されていて、実際に一覧へ連結されることもこの目で確認している。バリデータ側が追随していないと判断して、追加したフィールドのほうを残した。上流由来のツールが上流の仕様に追いついていないという、少し気持ちの悪い結論ではある。

## 2 回残すと決めたものを消した

重複も 1 組見つかった。grill-me と grilling で、どちらも計画を詰めるための追及インタビューをやる skill である。前者は `disable-model-invocation: true` でユーザの明示呼び出し専用、後者はモデルからも起動できる。

過去の spec を掘ると、この 2 つを分けて残す判断が 2 回記録されていた。意図的な user-invocable wrapper だから残す、と。

2026-07-25 にそれを覆して統合した。grill-me の中身は `/grilling` を呼ぶ 1 行で、grilling 自身も `/grilling` で直接起動できる。失われるのはコマンド名だけで、トリガ語は grilling の `when_to_use` に移した。2 回残すと書いてあったから今回も残す、では棚卸しにならない。

分割のほうは想像より健全だった。SKILL.md の行数は中央値 96。300 行を超えるものが 12 件あるが、そのうち references や scripts に分かれていないのは doc-coauthoring (376 行) の 1 件だけである。これは今も未着手のまま残っている。

## 基準をどこに置いたか

skill の良し悪しの判断基準は `rules/skill-authoring.md` に書いた。`paths: **/SKILL.md` でスコープしてあるので、SKILL.md を書くか直すセッションでしか読み込まれない。125 個の常時ロード分を数えた直後に、常時ロードされる rule を増やす選択は取りにくかった。

手順の側は既存の skill (skill-creator / skill-import / harness-audit) に残した。skill-creator は外部由来で上流と 1 対 1 に保ちたいため、そこへ追記する道は最初から無い。作り方の手順は skill-creator、良し悪しの判断基準は rules 側、と役割で分けた形になっている。

出典の記事にあって取り込まなかったものもいくつかある。ユーザ固有設定を `config.json` へ外出しする話は、判断基準として rules に書くに留めた (単一ユーザの dotfiles で実害が出ていない)。`${CLAUDE_PLUGIN_DATA}` への状態保存は、この環境が plugin ではないので変数自体が存在しない。plugin marketplace での配布に至っては、配布先が自分 1 人である。

## 数え直しはこれから

125 個のうちいくつが実際に動いているのか。これにはまだ答えが出ていない。67 回 / 18 個は保持期間内の観測であって、残り 107 個の判決ではない。hook を入れた日から先の期間で数え直して、そのときに初めて消す判断ができる。

今回はっきりしたのは、増やす側にだけコストが見えていなかったことのほうだ。skill を 1 つ書くのは 30 分で終わる。1 つ消すには計測が要る。125 個目のあとに書くべきだったのは 126 個目ではなく、125 個を数える仕組みのほうだった。
