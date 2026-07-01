---
name: todo-collect
description: 自分宛のPRレビュー依頼とSlackメンションを横断収集し、アクションが必要なものだけを重複なくApple純正リマインダーの専用リストに登録する
---

## 実行フロー

登録先のリマインダーリストは `Work TODO`。自分のアクションが必要なものだけを収集・登録する。

各エントリは `識別子` / `タイトル` / `リンク` / `収集元` を持つ。識別子は重複判定に使う不変のキー（番号・タイムスタンプ）にする。

**loopで繰り返し実行される前提で動く。** 前回の実行で見たPR・メンションでも、状態（レビュー要否・優先度・スプリント紐付け・解決済みか）は実行ごとに変わりうる。過去に収集・登録済みだからといって判定を省略せず、**毎回すべての候補についてステップ2〜3の条件と優先度を最新の状態から評価し直す**。キャッシュや前回結果を再利用しない。既存リマインダーは識別子一致でも「そのまま残す」ではなく、最新状態に合わせて優先度・タイトルを更新するか完了化するかを毎回判断する（ステップ6・7）。

---

### ステップ1: 自分の特定

- GitHubユーザー名: `gh api user --jq .login`
- Slackユーザー: `git config user.email` / `git config user.name` を `mcp__plugin_slack_slack__slack_search_users` で照合してuser_idを特定する。特定できなければSlackメンションの収集をスキップする。

---

### ステップ2: PRレビュー依頼の収集

`review-requested:@me`（未対応のレビュー依頼）と `reviewed-by:@me`（自分がレビュー済みのopen PR）の両方を収集する。前者だけでは、一度自分がレビューを出すと（明示的に再リクエストされない限り）候補から外れ、再レビュー待ち（自分のレビュー後に新コミットが来たPR）を取りこぼす。GitHubのsearch構文はORできないため、GraphQLの2エイリアスを1リクエストで投げてPR番号で重複排除する。

```bash
gh api graphql -f query='
query {
  requested: search(query: "is:pr is:open review-requested:@me", type: ISSUE, first: 100) {
    nodes { ... on PullRequest { number title url repository { nameWithOwner } } }
  }
  reviewed: search(query: "is:pr is:open reviewed-by:@me", type: ISSUE, first: 100) {
    nodes { ... on PullRequest { number title url repository { nameWithOwner } } }
  }
}' --jq '[.data.requested.nodes[], .data.reviewed.nodes[]] | unique_by(.repository.nameWithOwner + "#" + (.number|tostring))'
```

得られた候補のうち、次を**すべて満たすPRだけ**を収集対象とする。識別子は `PR <owner>/<repo>#<番号>`。

1. **自分のレビューが必要な状態**: 自分のレビューが未提出（未レビュー）、または自分の最終レビュー後に新しいコミットがある（再レビュー待ち）。

   ```bash
   # 自分の最終レビュー
   gh api repos/<owner>/<repo>/pulls/<番号>/reviews \
     --jq '[.[] | select(.user.login == "<github_username>")] | sort_by(.submitted_at) | last | .submitted_at // "none"'
   # 最新コミットのタイムスタンプ
   gh api repos/<owner>/<repo>/pulls/<番号>/commits --jq 'last | .commit.committer.date'
   ```

2. **（未レビューのPRのみ）他者がレビューに入っていない**: 自分が**まだ一度もレビューしていない**PR（`myLastReview == none`）については、自分・PR作成者本人・bot を除いたユーザーによるレビュー（`APPROVED` / `CHANGES_REQUESTED` / `COMMENTED` のいずれか）が1件も存在しないことを条件とする。他のレビュアーが既に着手しているPRに、自分が未着手のまま割り込むのを避けるための条件。

   一方、**自分が一度レビュー済みのPR（＝条件1の「再レビュー待ち」: 自分の最終レビュー後に新コミットがある）は、他者がレビューに入っていても収集対象とする**。一度自分が関与した以上は継続フォローするため、この場合は他者レビューの有無を問わない。

   ```bash
   AUTHOR=$(gh pr view <番号> --repo <owner>/<repo> --json author --jq .author.login)
   gh api repos/<owner>/<repo>/pulls/<番号>/reviews \
     --jq "[.[] | select(.user.login != \"<github_username>\") | select(.user.login != \"$AUTHOR\") | select(.user.type != \"Bot\")] | length"
   # 未レビュー（myLastReview == none）のPRは、この件数が 0 のものだけを対象にする。
   # 再レビュー待ち（自分のレビュー後に新コミットあり）のPRは、この件数を問わず対象にする。
   ```

3. **レビュー可能な実装PR**: draft ではなく（`gh pr view <番号> --repo <repo> --json isDraft` が `false`）、**作者がbotでない**こと。

   除外はタイトルではなく**作者**で判定する。作者が bot（`author.is_bot == true`、または `dependabot[bot]` / `renovate[bot]` 等）の自動PRだけを除外する。

   ```bash
   gh pr view <番号> --repo <owner>/<repo> --json author --jq '.author.is_bot'
   # false（人間）のものを対象にする
   ```

---

### ステップ3: PRの優先度付け（Jira連携）

収集したPRを次の3段階の優先度に分類して並べる。

1. **現スプリントのチケットに紐づくPRを高優先度（中→高に格上げ）にする。** PRタイトル先頭のチケットキー（例: `WF-1198`）を抽出し、Atlassian MCPで現スプリント（openSprints）に含まれるか、また課題タイプ・優先度を判定する。

   ```
   mcp__atlassian__getAccessibleAtlassianResources   # cloudIdを取得
   mcp__atlassian__searchJiraIssuesUsingJql
     cloudId: <上記cloudId>
     jql: "key in (<抽出したチケットキーをカンマ区切り>) AND sprint in openSprints()"
     fields: ["summary", "status"]
   # 各チケットの課題タイプ・優先度も取得する（急ぎ判定に使う）
   mcp__atlassian__searchJiraIssuesUsingJql
     cloudId: <上記cloudId>
     jql: "key in (<抽出したチケットキーをカンマ区切り>)"
     fields: ["summary", "status", "issuetype", "priority"]
   ```
   現スプリントに含まれるチケットに紐づくPRを「スプリント内（高）」とする。

2. **スプリント外でも急ぎ対応すべきPRは優先度中にする。** スプリント内ではないが、次のいずれかに該当するものは「急ぎ（中）」とする。
   - 紐づくJiraチケットの課題タイプが `Bug`（バグ）。
   - 紐づくJiraチケットの優先度が `High` / `Highest`（緊急・重大）。
   - PRタイトル・本文にバグ修正や緊急対応を示す語（`fix` / `bug` / `hotfix` / `緊急` / `障害` / `バグ` など）を含む。

   いずれにも該当しないスプリント外PRは「通常（低）」とする。

3. **同順位内は古いPRを優先する。** 各グループ内をPR番号の昇順（番号が小さい＝作成が古い）で並べる。

並び順は「スプリント内・高（古い順）→ 急ぎ・中（古い順）→ 通常・低（古い順）」。

---

### ステップ4: Slackメンションの収集

DM・チャンネル（public/private）を横断して自分宛メンションを集める。`to:me` だけではチャンネルでの `@自分` メンションを取りこぼすことがあるため、**次の2系統の検索を必ず両方実行する**。

1. **DM・宛先メンション**（`to:me`）

   ```
   mcp__plugin_slack_slack__slack_search_public_and_private
     query: "to:me after:<YYYY-MM-DD>"   # 直近2週間程度
     sort: "timestamp"
   ```

2. **チャンネルでの直接メンション**（`<@自分のuser_id>`）

   ```
   mcp__plugin_slack_slack__slack_search_public_and_private
     query: "<@<自分のuser_id>> after:<YYYY-MM-DD>"
     sort: "timestamp"
     channel_types: "public_channel,private_channel"
   ```

   この検索は自分が参加しただけのスレッドや関連チャンネルも広く拾い、結果が大きくなりやすい。出力がトークン上限を超えてファイルに保存された場合は、サブエージェント（`general-purpose`）に「ファイルを80,000文字ずつpython（`open(...).read()[A:B]`）でスライスして全件読み、各結果のChannel/From/Time/Message_ts/Permalink/本文要約を抽出し、**本文中に `<@自分のuser_id>` を含み自分宛で自分のアクションが必要なものだけ** をフィルタして返す」よう依頼し、全文を自分のコンテキストに載せずに解析する。自分自身が送信したメッセージ（From が自分）は除外する。

両系統の結果を統合し（識別子で重複排除）、**自分のアクション（返信・対応）が必要なメッセージだけ**を残す。依頼・質問・要対応を含むものが対象。本文の趣旨で判断し、自分が既に返信済み／リアクション済みのものは `slack_read_thread` で確認して対象から外す。分報・FYI・自動通知など返信不要なものも除外する。

- 識別子: `Slack <channel>/<message_ts>`
- リンク: メッセージのpermalink。本文要約を30字程度でタイトルにする。

---

### ステップ5: 登録対象の確定

収集結果（PRレビュー依頼・Slackメンション）の全件を登録対象とする。PR・Slackとも0件なら「登録対象なし」と報告して終了する。

---

### ステップ6: リマインダーへの登録

macOSの `osascript`（AppleScript）でApple純正リマインダーを操作する。ステップ3の優先度順（スプリント内→急ぎ→通常、各グループ古い順）に先頭から登録し、登録順がそのまま表示順になるようにする。優先度はステップ3の分類に従い、スプリント内のPRは `priority:1`（高）、スプリント外でも急ぎのPRは `priority:5`（中）、それ以外は `priority:9`（低）を設定する。

```applescript
tell application "Reminders"
  if not (exists list "<リスト名>") then make new list with properties {name:"<リスト名>"}
  set existingNames to name of reminders of list "<リスト名>"
  -- 優先度順に並べたエントリを先頭から登録する。各エントリについて:
  --   theName = "<タイトル> [<識別子>]"（識別子は末尾に置き視認性を確保）、theBody = "<リンクURL>"
  --   thePriority = スプリント内なら1、スプリント外でも急ぎなら5、それ以外は9
  --   既存に "[<識別子>]" を含む未完了リマインダーが無ければ新規作成する。
  --   既にある場合はスキップせず、今回算出した優先度・タイトルと差があれば
  --   その既存リマインダーの priority / name を更新する（変化が無ければ何もしない）。
  make new reminder at end of list "<リスト名>" with properties {name:theName, body:theBody, priority:thePriority}
end tell
```

- タイトルは `<タイトル> [<識別子>]` 形式（識別子をsuffixにしてタイトルを先頭に置く）。
- 重複判定はタイトル末尾の `[<識別子>]` の照合で行う。**同じ識別子が既にあっても「スキップして放置」ではなく、今回の最新評価に基づく優先度・タイトルへ更新する。** ループ実行の途中で低→高に格上げされた／再レビュー待ちになった等の変化を、既存リマインダーへ確実に反映させるため。
- 並び順を作り直すときは、対象の未完了リマインダーを一括削除してから優先度順に再登録する。一括削除は `delete (every reminder of list "<リスト名>" whose name contains "[PR ")` の形で行う。
- タイトル・本文の文字列はダブルクォート・改行をエスケープしてAppleScript構文を保つ。

---

### ステップ7: 解決済みリマインダーの完了化

リストの未完了リマインダーのうち、PR識別子（`[PR <owner>/<repo>#<番号>]`）を持つものを点検し、解決済みのPRを完了状態にする。

各PRリマインダーについて識別子から `<owner>/<repo>` と番号を取り出し、**その時点のGitHub上の最新状態を都度取得して**、現在もステップ2の収集条件を満たすかを判定する（前回の実行時に満たしていたかどうかは判断材料にしない）。満たさなくなったもの（マージ／クローズ済み、自分がレビュー済みで再レビュー不要、未レビューのまま他者がレビューに入った、draft化、など）を完了にする。なお、自分がレビュー済みで再レビュー待ち（自分のレビュー後に新コミットあり）のものは、他者がレビューに入っていても収集対象として残るため完了化しない。

```applescript
tell application "Reminders"
  set completed of (every reminder of list "<リスト名>" whose name contains "[PR <owner>/<repo>#<番号>]") to true
end tell
```

---

### ステップ8: 結果サマリ

新規登録件数（収集元別内訳）・重複スキップ件数・完了化した件数を報告する。

```
TODO登録完了（リスト: <リスト名>）

新規登録 <N>件
  ・<タイトル> [<識別子>]
重複スキップ <M>件
完了化 <L>件（解決済み）
```

---

## 注意事項

- 収集・登録のみを行う。実際のレビューは [slack-pr-review](../slack-pr-review) スキルが担う。
- `gh search` やSlack検索の結果が空のときは、その収集元を0件として続行する。
- osascriptが失敗した場合はその旨を報告する。
