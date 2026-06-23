---
name: todo-collect
description: 自分宛のPRレビュー依頼とSlackメンションを横断収集し、アクションが必要なものだけを重複なくApple純正リマインダーの専用リストに登録する
---

## 実行フロー

登録先のリマインダーリストは `Work TODO`。自分のアクションが必要なものだけを収集・登録する。

各エントリは `識別子` / `タイトル` / `リンク` / `収集元` を持つ。識別子は重複判定に使う不変のキー（番号・タイムスタンプ）にする。

---

### ステップ1: 自分の特定

- GitHubユーザー名: `gh api user --jq .login`
- Slackユーザー: `git config user.email` / `git config user.name` を `mcp__plugin_slack_slack__slack_search_users` で照合してuser_idを特定する。特定できなければAskUserQuestionでSlackユーザー名を聞く。

---

### ステップ2: PRレビュー依頼の収集

```bash
gh search prs --review-requested=@me --state=open --json number,title,url,repository
```

このうち、次を**すべて満たすPRだけ**を収集対象とする。識別子は `PR <owner>/<repo>#<番号>`。

1. **自分のレビューが必要な状態**: 自分のレビューが未提出（未レビュー）、または自分の最終レビュー後に新しいコミットがある（再レビュー待ち）。

   ```bash
   # 自分の最終レビュー
   gh api repos/<owner>/<repo>/pulls/<番号>/reviews \
     --jq '[.[] | select(.user.login == "<github_username>")] | sort_by(.submitted_at) | last | .submitted_at // "none"'
   # 最新コミットのタイムスタンプ
   gh api repos/<owner>/<repo>/pulls/<番号>/commits --jq 'last | .commit.committer.date'
   ```

2. **他者がまだレビューしていない**: 自分以外のユーザーによる `APPROVED` / `CHANGES_REQUESTED` レビューが存在しない（`COMMENTED` のみは未レビュー扱い）。

   ```bash
   gh api repos/<owner>/<repo>/pulls/<番号>/reviews \
     --jq '[.[] | select(.user.login != "<github_username>") | select(.state == "APPROVED" or .state == "CHANGES_REQUESTED")] | length'
   # 0 のものを対象にする
   ```

3. **レビュー可能な実装PR**: draft ではなく（`gh pr view <番号> --repo <repo> --json isDraft` が `false`）、タイトルが `chore(deps)` / `chore(deps-dev)`（依存更新bot）以外。

---

### ステップ3: PRの優先度付け（Jira連携）

収集したPRを次の優先度で並べる。

1. **現スプリントのチケットに紐づくPRを高優先度にする。** PRタイトル先頭のチケットキー（例: `WF-1198`）を抽出し、Atlassian MCPで現スプリント（openSprints）に含まれるか判定する。

   ```
   mcp__atlassian__getAccessibleAtlassianResources   # cloudIdを取得
   mcp__atlassian__searchJiraIssuesUsingJql
     cloudId: <上記cloudId>
     jql: "key in (<抽出したチケットキーをカンマ区切り>) AND sprint in openSprints()"
     fields: ["summary", "status"]
   ```
   返ってきたチケットに紐づくPRを「スプリント内」とする。

2. **同順位内は古いPRを優先する。** 各グループ内をPR番号の昇順（番号が小さい＝作成が古い）で並べる。

並び順は「スプリント内（古い順）→ それ以外（古い順）」。

---

### ステップ4: Slackメンションの収集

全チャンネルを横断して自分宛メンションを集める。

```
mcp__plugin_slack_slack__slack_search_public_and_private
  query: "to:me after:<YYYY-MM-DD>"   # 直近2週間程度
  sort: "timestamp"
```

このうち、**自分のアクション（返信・対応）が必要なメッセージだけ**を残す。依頼・質問・要対応を含むものが対象。本文の趣旨で判断し、自分が既に返信済み／リアクション済みのものは `slack_read_thread` で確認して対象から外す。

- 識別子: `Slack <channel>/<message_ts>`
- リンク: メッセージのpermalink。本文要約を30字程度でタイトルにする。

---

### ステップ5: 統合リストの提示と取捨選択

収集結果（PRレビュー依頼・Slackメンション）を収集元ごとにグルーピングして一覧表示する。各行に識別子・タイトル・リンクを含める。PR・Slackとも0件なら「登録対象なし」と報告して終了する。

一覧提示後、AskUserQuestionで登録から除外したいものを確認する。除外指定がなければ全件を登録対象とする。

---

### ステップ6: リマインダーへの登録

macOSの `osascript`（AppleScript）でApple純正リマインダーを操作する。ステップ3の優先度順（スプリント内→それ以外、各グループ古い順）に先頭から登録し、登録順がそのまま表示順になるようにする。スプリント内のPRは `priority:1`（高）、それ以外は `priority:9`（低）を設定する。

```applescript
tell application "Reminders"
  if not (exists list "<リスト名>") then make new list with properties {name:"<リスト名>"}
  set existingNames to name of reminders of list "<リスト名>"
  -- 優先度順に並べたエントリを先頭から登録する。各エントリについて:
  --   theName = "<タイトル> [<識別子>]"（識別子は末尾に置き視認性を確保）、theBody = "<リンクURL>"
  --   thePriority = スプリント内なら1、それ以外は9
  --   existingNames のいずれかが "[<識別子>]" を含むならスキップ、なければ作成
  make new reminder at end of list "<リスト名>" with properties {name:theName, body:theBody, priority:thePriority}
end tell
```

- タイトルは `<タイトル> [<識別子>]` 形式（識別子をsuffixにしてタイトルを先頭に置く）。
- 重複判定はタイトル末尾の `[<識別子>]` の照合で行う。
- 並び順を作り直すときは、対象の未完了リマインダーを一括削除してから優先度順に再登録する。一括削除は `delete (every reminder of list "<リスト名>" whose name contains "[PR ")` の形で行う。
- タイトル・本文の文字列はダブルクォート・改行をエスケープしてAppleScript構文を保つ。

---

### ステップ7: 結果サマリ

新規登録件数（収集元別内訳）・重複スキップ件数・ユーザー除外件数を報告する。

```
TODO登録完了（リスト: <リスト名>）

新規登録 <N>件
  ・<タイトル> [<識別子>]
重複スキップ <M>件
除外 <K>件
```

---

## 注意事項

- 収集・登録のみを行う。実際のレビューは [slack-pr-review](../slack-pr-review) スキルが担う。
- `gh search` やSlack検索の結果が空のときは、その収集元を0件として続行する。
- osascriptが失敗した場合はその旨を報告する。
