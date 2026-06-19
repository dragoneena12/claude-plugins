---
name: slack-pr-review
description: 自分宛のSlack PRレビュー依頼を取得し、未レビュー・再レビュー待ちのPRをworktreeでcode-reviewする
---

## 実行フロー

引数としてSlackチャンネル名を受け取る（例: `#dev-review`）。引数がない場合はAskUserQuestionで確認する。

---

### ステップ1: 自分のSlackユーザーIDの特定

`git config user.email` と `git config user.name` を取得し、`mcp__plugin_slack_slack__slack_search_users` で一致するSlackユーザーを検索してuser_idを特定する。

特定できなかった場合はAskUserQuestionでSlackユーザー名を聞く。

---

### ステップ2: 自分宛メンションの抽出

`mcp__plugin_slack_slack__slack_read_channel` で指定チャンネルの直近メッセージ（100件程度）を取得する。

以下の条件を**両方**満たすメッセージのみを残す:
- 自分のuser_idへのメンション（`<@USER_ID>` 形式）を含む
- GitHubのPull Request URL（`github.com/*/pull/*` パターン）を含む

1つのメッセージに複数のPR URLが含まれる場合は、それぞれ個別エントリとして扱う。

---

### ステップ3: PRの状態分類

GitHubのユーザー名は `gh api user --jq .login` で取得する。

ステップ2で抽出した各PRについて以下を取得し、4つに分類する:

```bash
# PRの状態（state, mergedAt）
gh pr view <番号> --repo <owner>/<repo> --json number,title,state,mergedAt

# 自分の最終レビュー（タイムスタンプ付き）
gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  --jq '[.[] | select(.user.login == "<github_username>")] | sort_by(.submitted_at) | last'

# PRの最新コミットのタイムスタンプ
gh api repos/{owner}/{repo}/pulls/{number}/commits \
  --jq 'last | .commit.committer.date'
```

分類ルール:

| 分類 | 条件 |
|------|------|
| **未レビュー** | PRがopenかつ自分のレビューが存在しない |
| **再レビュー待ち** | PRがopenかつ自分の最終レビュー後に新しいコミットが存在する |
| **参考情報（著者対応待ち）** | PRがopenかつ自分がレビュー済みで自分の最終レビュー後に新コミットがない |
| **対象外** | PRがclosed（未マージ）またはmerged |

---

### ステップ4: 候補一覧の提示

PR情報は `gh pr view <番号> --repo <owner>/<repo> --json number,title` で補完する。

**未レビュー・再レビュー待ち**が0件の場合は「対応待ちの依頼はありません」と報告し、参考情報があればそちらのみ表示して終了する。

以下の形式で表示する:

```
レビュー依頼（自分宛）

【未レビュー】
1. [#<番号>] <タイトル> — @<依頼者Slack名> より（<相対時間>）

【再レビュー待ち】
2. [#<番号>] <タイトル> — 前回レビュー後に <N> コミット追加

---
【参考: 著者対応待ち】
・[#<番号>] <タイトル> — レビュー済み、著者未対応

※ <除外件数>件はスキップ（クローズ済み・マージ済み）
```

一覧を表示したら、未レビュー・再レビュー待ちの全PRについてそのままステップ5に進む。ユーザー承認は不要。

---

### ステップ5: worktreeの作成とcode-reviewの実行

未レビュー・再レビュー待ちのPRを1件ずつ順番に処理する:

1. PRをworktreeとしてチェックアウトする:

   ```bash
   gh pr checkout <番号> --repo <owner>/<repo> -d --worktree /tmp/pr-review-<番号>
   ```

2. worktreeディレクトリをカレントディレクトリとして `code-review` スキルを実行する。ユーザー承認なしで完了させる。**レビューはpending（下書き）のままにすること。公開submitは絶対に行わない。**

3. code-review完了後にworktreeを削除する:

   ```bash
   git worktree remove /tmp/pr-review-<番号> --force
   ```

全PRの処理完了後にステップ6へ進む。

---

### ステップ6: 処理結果のサマリ表示

全PRの処理が完了したら、以下の形式でサマリを表示する:

```
レビュー完了サマリ

✅ [#<PR番号> <タイトル>](<PR URL>) — pending <N>件投稿
✅ [#<PR番号> <タイトル>](<PR URL>) — コメントなし（問題なし）
❌ [#<PR番号> <タイトル>](<PR URL>) — エラー: <エラー内容>

合計: <件数>件処理（うちコメントあり <N>件、なし <M>件、エラー <K>件）
```

- 各PRについて「pendingコメント件数」「コメントなし」「エラー」のいずれかを記録しておき、ここで一覧表示する
- エラーが発生したPRがある場合は、ユーザーに確認を促す

---

## 注意事項

- Slack MCP の読み取りはメッセージ本文の `text` フィールドにメンションとPR URLが含まれるかで判定する
- `gh` コマンドのオーナー・リポジトリ名は取得したPR URLから抽出する
- worktreeパスが既に存在する場合は `git worktree remove` で削除してから再作成する
- code-reviewスキルはworktreeのディレクトリをカレントディレクトリとして実行する
