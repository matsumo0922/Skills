---
name: codex-orchestrator
description: 複数の作業単位（GitHub issue や任意のタスク）を Codex CLI に委任する /goal 指示ファイルを生成して pbcopy する。Codex を課長として、worker サブエージェント並列実装 + レビュアーサブエージェントによる PR レビュー & 修正ループを自走させる。Use when the user asks to delegate work to Codex, create a Codex batch/orchestration prompt, 次のバッチ, Codex に作業を投げる, /goal プロンプト作成, or to run parallel implementation with an autonomous review loop.
---

# Codex オーケストレーション指示ファイル生成

任意のリポジトリの複数作業を Codex CLI（課長）に委任するための `/goal` 指示ファイルを生成する。Codex は worker サブエージェントを並列に spawn して実装させ、PR ごとにレビュアーサブエージェントを立ててレビュー → 修正コミット → 返信のループを自走させ、全 PR が承認・検証済みになったら人間に報告して停止する。

成果物は 2 つ:
1. **指示ファイル** … `/tmp/codex-goals/<バッチ名>.md`（`/goal` の objective は 4,000 文字制限のためファイル参照方式にする）
2. **短い `/goal` 行** … クリップボード（pbcopy）。ユーザーが Codex TUI に貼り付ける

## 手順

### 1. 作業単位の確定

ユーザーから対象を聞き取るか、会話の文脈から確定する。作業単位は GitHub issue でも口頭指示でも良い:

- **GitHub issue の場合**: `gh issue view <N>` で本文を読み、スコープと受け入れ条件を把握。親 epic / tracking issue があれば直列制約・依存関係の正として参照する
- **issue がない作業の場合**: 指示ファイル内に作業内容・受け入れ条件を自分で明文化する（worker が迷わないレベルまで具体化。曖昧なら生成前にユーザーへ確認）

### 2. 対象リポジトリの偵察（生成のたびに行う）

テンプレートの穴を埋めるための情報をリポジトリから収集する:

```bash
gh pr list --state all --limit 15      # 直近のマージ状況・ブランチ命名慣習
git worktree list                       # worktree 名の衝突確認
git status --short                      # メイン checkout が clean か
```

- **検証コマンド**: CLAUDE.md / AGENTS.md / Makefile / package.json / CI 設定からビルド・lint・テストのコマンドを特定する（例: Gradle 系なら assembleDebug + detekt、Node 系なら lint + test + build）
- **規約**: CLAUDE.md / AGENTS.md のコーディング規約・コミット規約・PR 規約の存在を確認し、指示ファイルから参照させる
- **機密・命名ポリシー**: リポジトリの CLAUDE.md やプロジェクトメモリに「書いてはいけない語・秘匿情報」のポリシーがあるか確認する。ある場合はチェック用パターンを**生成時にメモリ等のリポジトリ外ソースから組み立てて**指示ファイルに埋め込む。**パターンや実名を skill・テンプレート・リポジトリ内のファイルに書いてはならない**（指示ファイルは必ず `/tmp/` 配下）
- **worktree ビルドに必要な untracked ファイル**: `local.properties` / `.env` / 認証ファイルなど、worktree にコピーしないとビルドが通らないものを特定する（`.gitignore` と build 設定から判断）

### 3. バッチ設計の原則

- **ファイル競合で worker を分ける**: 同じファイル群を触る作業は (a) 同一 worker に直列担当させる、(b) 先行作業のマージ待ちなら今回見送る。並列 worker 間で競合する組み合わせを作らない
- **人間にしかできない部分はスコープから切る**: 実機検証・デザイン確定・デフォルト値の最終判断などは「Phase 1 のみ」「設定部分のみ」のように機械的に完了可能な範囲を明示して委任し、残りは人間の判断待ちとして報告させる
- worker は最大 4（レビュアーと合わせて Codex の `agents.max_threads`=6 に収める）
- **1 作業 = 1 worktree = 1 ブランチ = 1 PR**。ブランチは `codex/<識別子>-<英語スラッグ>`、worktree はリポジトリの兄弟ディレクトリで既存と衝突しない名前

### 4. テンプレートを埋めて出力

[prompt-template.md](prompt-template.md) の `{{...}}` を埋め、指示ファイルを `/tmp/codex-goals/<バッチ名>.md` に書き出す（`mkdir -p` を忘れない）。その後、テンプレート冒頭の**短い `/goal` 行**（指示ファイルパス + 終了条件の要約。4,000 文字以内）を pbcopy する。

| プレースホルダ | 内容 |
|---|---|
| `{{GOAL_SUMMARY}}` | バッチ名と作業一覧（スコープ注記付き） |
| `{{REPO_PATH}}` / `{{BASE_BRANCH}}` | メイン checkout の絶対パスとベースブランチ |
| `{{UNTRACKED_SETUP}}` | worktree 作成後にコピーする untracked ファイル等のセットアップ手順（不要なら「なし」） |
| `{{WORKTREE_COMMANDS}}` | 作業ごとの `git worktree add` コマンド |
| `{{WORKER_ASSIGNMENTS}}` | worker ごとの担当・順序・**やる範囲とやらない範囲**・固有の注意点。issue がない作業は受け入れ条件をここに明文化 |
| `{{VERIFY_COMMANDS}}` | 完了条件となる検証コマンド（lint / build / test） |
| `{{REPO_POLICIES}}` | リポジトリ固有ポリシー（機密・命名チェック等）。無ければセクションごと削除 |
| `{{PR_COUNT}}` / `{{COMPLETION_ITEMS}}` | completion_criteria 用の PR 本数と作業列 |

### 5. 生成前チェックリスト

- [ ] 各 worker の担当に「やらないこと」が明示されている
- [ ] 並列 worker 間でファイル競合がない / 同一 worker 内の重複ファイル作業に stacked PR 指示がある
- [ ] worktree 名が既存と衝突しない
- [ ] 検証コマンドが対象リポジトリの実際のコマンドになっている（別プロジェクトのコマンドの混入禁止）
- [ ] completion_criteria の PR 本数・作業列がテンプレ全体で一致
- [ ] 機密・命名ポリシーのパターンが skill やリポジトリ内ファイルに残っていない
- [ ] GitHub コメント全日本語 / コミットメッセージ・PR タイトル英語、の指示が残っている

### 6. ユーザーへの報告

バッチ構成（worker × 担当 + 選定理由）/ 見送った作業と理由 / 人間の判断待ちになりそうな点 / 指示ファイルのパス / 「クリップボードの `/goal` 行を Codex TUI に貼り付ければ開始」の案内。

## Codex 側の前提知識（テンプレートが依存している事実）

- `/goal <目的>` は検証可能な終了条件付きの自律ループ（codex-cli 0.128+、`goals` feature stable）。`/goal pause` / `resume` / `clear` で制御。**objective は 4,000 文字制限**のため長い指示はファイル参照にする
- サブエージェントは**明示的に spawn を指示**しないと作られない。worker / explorer / default の 3 種、並列上限 `agents.max_threads`（デフォルト 6）
- `request_user_input` は Plan モード専用（`default_mode_request_user_input` は under development で実用不可）。**`/goal` 中に人間へ対話質問はできない**前提で、仕様判断は issue/PR への日本語コメント + 仮決め続行プロトコルで処理する
- 実装もレビューも同一 GitHub アカウントのため**自分の PR に `gh pr review --approve` / `--request-changes` は実行不可**。`gh pr comment` ベースの「## レビュー (ラウンド N)」/「## レビュー結果: APPROVED」プロトコルで代替する
- Codex の起動はサンドボックス緩め（`codex --full-auto` 等）でないと gh / git / ビルドのたびに承認要求で自律性が死ぬ

## 運用ノート

- Codex セッションは**バッチごとに新規開始**が基本（指示ファイルは自己完結。前バッチの古い状態認識の混入を防ぐ）。同一バッチの PR 群への追加指示のみ resume を使う
- 指示ファイルは `/tmp` 配下のため再起動で消える。再実行が必要なら本 skill で再生成する
