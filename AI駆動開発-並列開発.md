# AI 駆動開発-並列開発

## 概要

AI 駆動開発では git worktree を用いて並列で開発することを基本とし、そのためのポリシーを規定する
基本的な考え方として、各 worktree ごとに独立した環境を用意する方針

## 開発サーバー

### 方針

開発サーバーは worktree ごとに立ち上げ、起動時に安定した名前付き URL を割り振る

モノレポ構成の場合

http://localhost:3000 -> https://package-name.project-name.worktree-name.local -> https://web.crm.feature-add-note.local/

### 理由

worktree ごとに開発サーバーを立ち上げないと、
エージェントが開発サーバーにアクセスして動作確認が行えない

そのため、ポート番号を開発サーバーごとに競合しないように割り当てを行うこともできるが、
どのポート番号がどのワークツリーのどのアプリなのか確認しづらい

また、cookie の状態が引き継がれてしまう可能性などもあるため、
明示的な名前をつけて分離する

### 実装方法

portless の使用を前提とする
以下 portless + turborepo 構成の参考

#### ルート dev:portless スクリプト作成

crm プロジェクトの web アプリの開発サーバーコマンドを追加する例。
フォルダ構成などにより調整すること。

```json
// package.json
{
  "scripts": {
    "web:dev:portless": "WORKTREE_NAME=$(basename $PWD) turbo dev:portless --filter @apps/web"
  }
}
```

#### Turbo 設定追加

```json
// turbo.json
{
  "globalEnv": ["WORKTREE_NAME"],
  "tasks": {
    "dev:portless": {
      "cache": false,
      "persistent": true
    }
  }
}
```


#### 各プロジェクト dev:portless スクリプト作成


```json
// apps/web/package.json
{
  "scripts": {
    "dev:portless": "portless web.crm.$WORKTREE_NAME next dev"
  }
}
```


参考：https://community.vercel.com/t/using-portless-with-conductor-git-worktrees/34557


## Supabase（DB）

### 方針

Supabase は worktree ごとにプロジェクト名とポートを変え、独立した環境を立ち上げる

### 理由

DB を共有してしまうと、他の worktree によって行った操作などが影響してしまう可能性があるため

### 実装方法

supabase 起動用の npm script を作成
（supabase が専用のモノレポパッケージになっている前提）

```json
{
  "scripts":{
    // 開発サーバー同様ルートパッケージで WORKTREE_NAME が定義してある前提
    // 必要なサービスに応じて環境変数を定義する。(DB_PORT,API_PORT,etc...)
    // プロジェクト名が crm の場合。適切なプロジェクト名に置き換える
    "dev:portless": "PROJECT_NAME=crm_$WORKTREE_NAME DB_PORT=$(get-port) supabase start"
  }
}
```

config.toml のプロジェクト名、ポート番号を環境変数によって設定する

```toml
project_id = "env(PROJECT_NAME)"


[db]
# Port to use for the local database URL.
port = "env(DB_PORT)"


```

