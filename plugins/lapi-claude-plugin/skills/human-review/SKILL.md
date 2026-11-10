---
name: human-review
description: ユーザー（人間）がPRを読んで抱いた疑問・意見を壁打ちし、合意した指摘をpending reviewのインラインコメントに書き起こし、指示があればsubmitしてhuman-reviewedラベルを付ける
---

## 位置づけ

指摘の中身を決めるのはユーザー。このスキルは、ユーザーの疑問への調査・意見出しと、ユーザーが出した指摘をPRコメントとして投稿する作業を担う。AIが自分で指摘を探すレビューは `code-review` の担当であり、このスキルでは行わない。

## 実行フロー

### ステップ1: 対象PRの特定

引数でPR番号・URLが渡されればそれを、なければ現在のブランチに紐づくPRを対象にする。PR番号・タイトル・作者を宣言する。作者が自分の場合は、レビューコメントではなく修正を提案する方が適切なことを伝える。

### ステップ2: 壁打ち

ユーザーの疑問・意見（「〜でいいのでは？」「〜に入れるべきでは？」など）ごとに、コードやドキュメントを実際に読んで根拠を確かめ、意見を返す。

- 賛否を明確にし、根拠を `file:line` で示す。ユーザーの案に反対なら、代案と理由を示す
- 依存方向・前例・既存規約など、コメントに書けば作者が納得しやすい事実を集めておく
- この段階ではコメントを作らない

### ステップ3: コメントの下書きと pending review への追加

ユーザーが「コメントにして」などと指示したら、Skillツールで `write-review-comments` を呼び出し、その作法に従って本文を書く。

- **指摘の主張はユーザーのもの**。ユーザーが示した趣旨と、壁打ちで合意した根拠だけを書く。AIが挙げた反論・懸念をユーザーの同意なく混ぜない
- `[nit]` などの重要度ラベルや観点ラベルは、ユーザーが求めない限り付けない
- コメントを付ける行は、指摘の中心となる行（型定義・決定文など）にする。diffハンク外の行には付けられないため、その場合は review の body に回す
- 追加後、本文全文・文字数・URLを表示する。投稿したら終わりではなく、ユーザーが直しを指示したら編集する

pending review はユーザーごとに1つしか持てない。既存の自分の pending review があればそこに足す。

- 既存の pending review の確認: `gh api repos/{owner}/{repo}/pulls/<番号>/reviews` で `state == "PENDING"` かつ自分のログインのものを探す
- **pending review が無い場合**: REST の `POST /pulls/<番号>/reviews` で `event` を**指定せず**、`commit_id`（headRefOid）と `comments` を付けて作成する。本文は `jq --rawfile` で JSON を組み `--input` で渡す（改行・バッククォートを壊さないため）
- **pending review がある場合**: REST ではコメントを足せない。GraphQL の `addPullRequestReviewThread`（`pullRequestReviewId` は review の `node_id`、`path`・`line`・`side: RIGHT`・`body`）で足す
- **コメントの編集**: GraphQL の `updatePullRequestReviewComment` を使う。pending 中のコメントは REST の `GET /pulls/comments/<id>` が 404 になるので、ID は GraphQL で `reviews(states: PENDING)` → `comments { id databaseId }` から取る

### ステップ4: submit とラベル付与

ユーザーが submit を指示したら、GraphQL の `submitPullRequestReview` で submit する。

- `event` はユーザーの指定に従う。指定がなければ `COMMENT` とする（提案・質問を勝手に APPROVE / REQUEST_CHANGES にしない）
- リポジトリに `human-reviewed` ラベルが定義されていれば（`gh label list --search human-reviewed`）、`gh pr edit <番号> --add-label human-reviewed` で付ける。定義されていなければ新規作成しない
- submit した review のURL、イベント種別、付けたラベルを報告する

## 注意事項

- submit は作者に公開される操作なので、ユーザーの明示的な指示があるときだけ行う
- 一時ファイル（本文・JSON）はスクラッチパッドに置く
