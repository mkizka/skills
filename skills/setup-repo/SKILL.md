---
name: setup-repo
description: 引数で指定した既存リポジトリ(package.json、eslint、prettier、husky、lint-staged、tsconfig、vitest、turborepoなどの設定)を確認し、汎用的に使える部分だけを現在のリポジトリに導入する。「mkizka/xxxを参考にセットアップして」「このリポジトリの設定を真似して初期設定して」「/setup-repo」のような、既存リポジトリの構成を新しいプロジェクトに持ち込みたい依頼で使用する。
---

# setup-repo

引数で受け取ったリポジトリの内容を確認し、基本的な設定を模倣して現在のリポジトリに導入するスキル

使い方

```
/setup-repo https://github.com/mkizka/repository-name
/setup-repo repository-name # 省略形。↑と同じ意味
```

## 手順

1. 引数で受け取ったリポジトリの内容を確認する
2. 「対象設定」セクションに挙げられている設定を確認する
3. 確認した設定のうち、汎用的に使えるものだけを順番に現在のリポジトリにも導入する

## 対象設定

- package.json
  - 依存関係
- .npmrc
- pnpm-workspace.yaml
- prettier
  - 設定はpackage.jsonに書いている場合もある
  - .prettierignore
- eslint
- husky
- lint-staged
  - monorepoの場合は.lintstagedrc.jsonも確認
- tsconfig
  - tsconfig/basesが使える場合はそれを優先
  - その場合は以下を使用する
    - @tsconfig/nodexx(バージョンは現在最新のlts)
    - @tsconfig/strictest
- vitest
- turborepo
- mise.toml

## ルール

- Node.jsのバージョンはスキル実行時点のLTSの最新版を使用する
- pnpmのバージョンはスキル実行時点の最新版を使用する
- 依存関係はpnpm addでインストールして最新バージョンを使用する
- 汎用的に使用できる内容のみコピーする。引数のプロジェクト固有と思われる設定はコピーしない
