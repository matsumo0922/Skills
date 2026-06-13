# /goal 指示ファイルテンプレート

`/goal` の objective には **4,000 文字制限**があるため、本テンプレートは「指示ファイル」として `{{...}}` を埋めて `/tmp/codex-goals/<バッチ名>.md` に書き出し、Codex にはファイルを参照する短い `/goal` 行だけを渡す。

ユーザーに pbcopy で渡す `/goal` 行（1 行、これだけがクリップボードに入る）:

```
/goal /tmp/codex-goals/<バッチ名>.md を読み、その指示に厳密に従って実行する。終了条件はファイル内 <completion_criteria> の全項目（{{PR_COUNT}} 本の PR が ready-for-review・self-assigned で存在し、各 PR にレビュアーの APPROVED コメントがあり、各ブランチで検証コマンドが成功し、人間の判断待ち事項が記録済み）を満たすこと。達成したら <reporting> の形式で人間に報告して停止する。
```

以下、指示ファイルの本文テンプレート:

---

ゴール: {{GOAL_SUMMARY}} をサブエージェント並列で実装し、全 PR がレビューループ（レビュアー指摘 → 修正コミット → PR 上で返信 → 再レビュー）を完走して APPROVED 宣言済み・検証済みになったら停止して人間に報告する。以下の指示に厳密に従うこと。

<role>
あなたはオーケストレーター（課長）。自分では実装・レビューをせず、サブエージェントに委任して進行管理・worktree 管理・結果検証・最終報告だけを行う。サブエージェントは明示的に spawn すること。
</role>

<language_policy>
**GitHub に書き込むコメントはすべて日本語**で書くこと（レビューコメント、指摘への返信、issue への質問コメント、PR 本文を含む）。例外: コミットメッセージと PR タイトルは英語（prefix 付き: feat:/fix:/refactor:/test:/docs:/chore:）。
</language_policy>

<setup>
オーケストレーターの責務:
1. cd {{REPO_PATH}}（メイン checkout。**working tree は変更禁止**。fetch と worktree 操作のみ）
2. `git fetch origin {{BASE_BRANCH}}`
3. **1 作業 = 1 worktree = 1 ブランチ = 1 PR**。worker が作業に着手するタイミングで worktree を作成する:
{{WORKTREE_COMMANDS}}
4. worktree 作成のたびに行うセットアップ: {{UNTRACKED_SETUP}}
5. 最初の worktree で検証コマンド（<verification> 参照）のベースライン成功を確認。依存解決等で失敗したらリポジトリの CLAUDE.md / AGENTS.md / Makefile を確認して解決を試み、不能なら中断して報告
</setup>

<delegation>
実装 worker サブエージェントを並列で spawn し、各 worker は担当作業を記載順に直列で処理する（順序はファイル競合回避のため厳守）:

{{WORKER_ASSIGNMENTS}}

各 worker への共通指示:
- 担当 worktree 内でのみ作業。GitHub issue がある作業は最初に `gh issue view <番号>` で本文（設計・受け入れ条件）を読む。worktree の CLAUDE.md / AGENTS.md の規約に従う
- スコープは上記の指定範囲のみ。issue に書かれていても今回スコープ外の範囲には手を出さない。無関係なリファクタ禁止
- ロジックを純粋関数に切り出せる箇所はユニットテストを追加
- 検証: <verification> のコマンドがすべて成功するまで完了とみなさない
- コミットは意味のある単位で分割、英語 + prefix。コミット前に <repo_policies> のチェック（セクションがある場合）
- PR: `gh pr create --base {{BASE_BRANCH}} --assignee @me`、draft 禁止。タイトル英語、本文日本語で「## 概要 / ## 変更内容 / ## 検証 / ## 人間の判断待ち（あれば）/ Closes #<番号>（issue がある場合）」
- PR 作成後、レビュー待ちの間に次の担当作業へ着手して良い。ただしレビュアーの指摘が来たら**修正対応を最優先**する
</delegation>

<verification>
各ブランチで以下がすべて成功することが完了条件:
{{VERIFY_COMMANDS}}
</verification>

<human_decision_protocol>
`/goal` 実行中は人間へ対話的に質問できない前提で動くこと（request_user_input は使えない）。仕様判断が必要になったら:
1. 担当 issue（無ければ PR）に**日本語コメント**で「質問 + 自分の仮決め案 + 根拠」を投稿する
2. 仮決め案で実装を続行し、PR 本文の「## 人間の判断待ち」セクションに同じ内容を列挙する
3. 受け入れ条件そのものを満たせない・複数の根本方針から選べない場合のみ、その worker を停止して最終報告に含める（他 worker は続行）
仮決めを確定事項としてコードコメントや KDoc/docstring に書かないこと（仮決めだと分かる場所 = issue / PR に書く）。
</human_decision_protocol>

<stacking>
同一 worker の 2 件目が 1 件目と同じファイルに触る場合は、1 件目のブランチから 2 件目のブランチを切り、PR の base も 1 件目のブランチにする（stacked PR）。PR 本文の冒頭に「⚠️ <1件目> のマージ後に base を {{BASE_BRANCH}} へ変更してください」と日本語で明記する。ファイルが重ならない場合は origin/{{BASE_BRANCH}} から切る。
</stacking>

<review_loop>
worker が PR を作成したら、PR ごとにレビュアーサブエージェント（実装 worker とは別エージェント）を spawn する。レビューと修正のやり取りは**すべて GitHub の PR 上・日本語**で行う。

制約: 同一アカウントのため `gh pr review --approve` / `--request-changes` は失敗する。使わず、以下のコメントプロトコルを使う:
- レビュアー: `gh pr diff` / `gh pr view` / worktree のコード読解で、(1) スコープと受け入れ条件の充足、(2) バグ・エッジケース、(3) リポジトリ規約違反、(4) <repo_policies> 違反、を検証。指摘は `gh pr comment` で `## レビュー (ラウンド N)` から始まる**日本語コメント**として投稿し、各指摘に severity（must-fix / should / nit）と file:line を付ける。根拠のない指摘・スコープ外の要求はしない。指摘ゼロなら `## レビュー結果: APPROVED` を投稿
- 実装 worker: must-fix / should に対応するコミットを push し、指摘ごとに対応内容（コミット SHA 付き）を**日本語で返信**。対応しない nit は理由を返信
- APPROVED まで繰り返す。**最大 3 ラウンド**。収束しなければ打ち切って未解決指摘を最終報告へ
- 各 PR の APPROVED 後、オーケストレーターが該当 worktree で <verification> を最終実行して成功を確認
</review_loop>

<repo_policies>
{{REPO_POLICIES}}
（リポジトリ固有の機密・命名ポリシー等。チェック用の grep パターンを使う場合は、**パターン自体をファイル・コミット・PR・スクリプトに残さずシェル直接実行のみ**とする。固有ポリシーが無い場合、生成時にこのセクションごと削除する）
</repo_policies>

<completion_criteria>
以下がすべて満たされたらゴール達成として停止:
1. {{PR_COUNT}} 本の PR（{{COMPLETION_ITEMS}}）が open・ready-for-review・self-assigned で存在（issue がある作業は `Closes #<番号>` 付き、stacked PR は base 注記付き）
2. 各 PR に `## レビュー結果: APPROVED` があり、must-fix が全対応済み
3. 各ブランチ最終 HEAD で <verification> のコマンドがすべて成功
4. <repo_policies> のチェックが全 PR 差分で違反 0 件（セクションがある場合）
5. 「人間の判断待ち」事項が各 PR / issue に日本語で記録され、最終報告に集約されている
6. worktree は削除せず残す（人間のレビュー・追修正用）
</completion_criteria>

<reporting>
完了時（またはブロッカー停止時）の人間への報告: 各 PR の URL / レビューラウンド数と主要指摘の要約 / 最終検証結果 / **人間の判断待ち事項の一覧（どの issue・PR の何か）** / stacked PR の base 変更が必要なもの / 未解決事項。箇条書きで簡潔に。
</reporting>

<blockers>
依存解決や検証コマンドが不能 / gh の認証・権限エラー / 作業指示と現行コードの根本矛盾、の場合は該当 worker のみ停止して報告し、他は続行。憶測で仕様を確定して進めないこと。
</blockers>
