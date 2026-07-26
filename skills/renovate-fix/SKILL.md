---
name: renovate-fix
description: RenovateボットによるPRのCI失敗・コンフリクトを解消する。「PR<番号>のCIを通す」「Renovateのコンフリクト解消」「依存更新PRを直す」のようなRenovate PRに対する作業依頼で使用する。複数PRの一括処理、ロックファイルの再生成、依存更新によるbreaking changeの修正、main取り込みも含む。pnpm/npm/yarn いずれのパッケージマネージャでも動作。
---

# Renovate PR Fixer

RenovateがオープンしたPRを安全にマージ可能な状態にするためのワークフロー。CIの失敗とmainとのコンフリクトの両方に対応する。

## 基本原則

1. **必ずローカルで再現してから修正する** — CIのログだけで原因を推測しない
2. **コードに手を入れる修正(breaking change対応など)を含む場合、コミット前のユーザーレビューは絶対に必須** — 詳細は下記の「ユーザーレビュー必須ルール」参照
3. **コンフリクトとCI失敗は別問題として切り分ける** — 順番に対処
4. **Renovateは作業中にもブランチを書き換える** — push reject に備える

## ユーザーレビュー必須ルール

「単純なロックファイル再生成 + main マージだけ」なら自動でコミット&push して良い。
それ以外、つまり**ソースコードに1行でも手を入れる場合は、コミット前に必ずユーザーに以下を提示してレビュー承認を待つ:**

- 変更したファイルと変更内容の要約 (diff の主要部分)
- なぜその修正にしたか (release note や upstream issue の根拠)
- 他の選択肢を検討した場合はその比較
- ローカル検証結果 (typecheck / lint / test)

承認が来るまで `git commit` を絶対に実行しない。「OK」「承認」「コミットして」など明示的な承認が必要。

**なぜ厳格にするか:** Renovate PRは依存更新のみが期待されている。コード変更が含まれると、レビュー観点が変わる(機能影響、互換性、別解の有無)。ユーザーが想定していなかった変更が勝手にcommitされると、PRレビュー時の負担が増え、本来別PRに分けるべき変更が混入するリスクもある。Breaking change 対応のような判断を伴う変更は、ユーザーの判断を介在させるべき。

**例外なし:** ロックファイル以外のファイルが1つでも `git status` に出ていたらレビュー依頼ステップに進む。タイプミス修正や軽微な lint fix も含む。

## 全体フロー

```
1. PRの状態確認 (CI / merge state)
2. mainとのコンフリクト解消 (必要なら)
3. CI失敗の調査と修正 (必要なら)
4. ローカル検証 (typecheck / lint / test)
5. ユーザーレビュー依頼 ← 必須
6. コミット & push
7. CI結果監視
```

## ステップ詳細

### 1. PR状態確認

```bash
gh pr checks <number>                                      # CI状態
gh pr view <number> --json mergeStateStatus,mergeable,headRefName,title
```

`mergeStateStatus`の値:

- `CLEAN` — 何もしなくて良い(マージ可能)
- `DIRTY` / `CONFLICTING` — コンフリクトあり
- `BLOCKED` — CI失敗または承認待ち
- `UNKNOWN` — GitHubがまだ判定中、少し待って再確認

### 2. ブランチに移動

```bash
gh pr view <number> --json headRefName --jq '.headRefName'  # ブランチ名
git fetch origin
git checkout <branch>
git pull origin <branch>   # Renovateが書き換えている可能性があるので最新化
```

### 3. mainとのコンフリクト解消

```bash
git merge origin/main
```

ロックファイル(`pnpm-lock.yaml` / `package-lock.json` / `yarn.lock`)のコンフリクトはほぼ確実に発生する。手動で直さず**再生成**する:

```bash
# pnpmの場合
git checkout --theirs pnpm-lock.yaml
pnpm install --no-frozen-lockfile

# npmの場合
git checkout --theirs package-lock.json
npm install

# yarnの場合
git checkout --theirs yarn.lock
yarn install
```

`--theirs`を選ぶ理由: mainの内容を基準にし、PRが追加するパッケージ差分を install で再適用したほうが安全。

`package.json`のコンフリクトは手動で解決(両方の変更を残す)。

### 4. CI失敗の原因調査

`gh pr checks`で失敗しているjobのURLからrun IDを抽出して:

```bash
gh run view <run_id> --log-failed | head -100
```

依存更新による破壊的変更が多い。よくあるパターン:

- **TypeScriptエラー** — 型定義の削除/変更。release noteで確認
- **ESLintエラー** — 新しいルールの有効化、ルール名変更、プラグイン移行
- **テストエラー** — APIの挙動変更
- **Lint設定エラー** — config形式の変更 (flat config移行など)

upstream のリリースノートを確認する:

```bash
gh release view v<version> --repo <owner>/<repo> --json body,name
# 例: gh release view v26.0.0 --repo i18next/i18next
```

### 5. 修正

依存更新で破壊的変更があった場合は、コードを修正する。
**コメントや変数名の自己修正は最小限に。タスクが明示的に求める変更のみ行う。**

### 6. ローカル検証

プロジェクトの`package.json`で実際のスクリプト名を確認してから実行:

```bash
pnpm typecheck   # または npm run typecheck
pnpm lint        # または pnpm _eslint (lintステップを個別に呼ぶ場合)
pnpm test        # または npm test
```

すべて通ってから次へ。

### 7. ユーザーレビュー依頼

ソースコード変更を**1ファイルでも**含む場合 → **必ず止まる**。

ロックファイルとmainからのマージコミットだけなら、このステップはスキップして良い。

レビュー依頼時は `git diff` から以下を抽出して提示する:

- 変更ファイル一覧
- 各変更の意図と根拠 (release note / upstream PR / issue URL)
- 検討した代替案があればその比較
- ローカル検証結果 (typecheck / lint / test の pass/fail)
- 「コミット&pushして良いか?」を明示的に確認

ユーザーから明示的な承認(「OK」「コミットして」など)が返るまで `git commit` 禁止。
「いいね」「良さそう」など曖昧な反応でも、コードに手を入れた場合は1度確認を取る。

### 8. コミット & push

承認を得てからコミット。

```bash
# ロックファイル再生成だけの場合
git add <lockfile>
git commit -m "Merge branch 'main' into <branch>"

# breaking change修正を含む場合
git add <変更ファイル>
git commit -m "fix: <修正内容の要約>"

git push origin <branch>
```

**push rejectされた場合** — Renovateが先にブランチを更新している。**force pushしない**。リモートをfetchして状況確認:

```bash
git fetch origin
git log --oneline origin/<branch> -5
```

Renovateが既に同等の修正を済ませている(例: mainにrebase済み)場合、自分の変更はそのままリモートにresetして良い:

```bash
git reset --hard origin/<branch>
```

ローカルの修正が必要なら、リモートを取り込んで再度マージ → push。

### 9. CI監視

push後すぐにはCIが起動しないことがある。少し待ってから:

```bash
gh pr checks <number>
```

長時間待つ場合は ScheduleWakeup で間隔を空けてポーリング(2-3分間隔)。

## 複数PRの並行処理

複数のRenovate PRを連続処理する場合、**依存関係を意識した順序**で対処する:

- Lint/フォーマッタ系のconfigを更新するPR (例: eslint-config) を先に処理
- それに依存する型/lintルール更新を含むPRは後で処理
- 別PRがmainにマージされると他PRに新たなコンフリクトが発生する。一巡したら再度全PRの状態を確認

## よくあるトラブル

### Renovateがマージコミットを上書きしてくる

CI待ち中にrenovateがブランチを force-update することがある。`origin/<branch>`が予期せず変わっていたら、`git reset --hard origin/<branch>`してから再度作業。force pushされたコミットは既に内容が等価ならresetしてOK。

### CIワークフローが起動しない

push後に`gh pr checks`で何も表示されない場合、ワークフロートリガーが効いていない可能性。空コミットや`workflow_dispatch`での再実行を試す:

```bash
gh workflow run <workflow_file> --ref <branch>
```

### `.serena/` などローカル限定ファイルのprettier警告

CI環境には存在しないファイルでlocalでだけ警告が出ることがある。CIに影響しないなら無視可。

## このスキルが扱わないこと

- Renovate設定 (`renovate.json`) の編集 — 別作業
- PRのapprove/merge — ユーザーが行う
- 重大な破壊的変更で実装方針の判断が必要なケース — 必ずユーザー相談
