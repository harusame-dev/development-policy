# GitHub Actions ワークフロー設定ガイドライン

## 概要

GitHub Actions ワークフローの品質・セキュリティ・メンテナンス性を統一するための設定ガイドライン。

## 1. パーミッション設定

### Organization / リポジトリレベルの設定

#### 方針

`Read repository contents and packages permissions` に設定する。

#### 理由

`Read and write permissions` にしてパーミッションブロックが設定されていないワークフローがある場合、不用意に広範囲な権限が付与されセキュリティリスクが発生するため。
ワークフローでも権限を設定する方針だが、設定漏れなどが発生しても問題ないように多段的に防衛する。

### 個別ワークフローのパーミッション設定

#### 方針

トップレベルの `permissions` ブロックで最小限のパーミッションを設定する。

#### 理由

セキュリティリスクを最小限に抑えるため。
Organization / リポジトリレベルでも設定するが、設定漏れなどが発生しても問題ないように多段的に防衛する。

#### 設定例

```yaml
permissions:
  contents: read
```

## 2. ステップ名

### 方針

各ステップには日本語でわかりやすい名前を必ず付けること。

### 理由

視認性を向上し、失敗時や調査時などに一目でどこでエラーが発生しているか分かるようにするため。

### 設定例

```yaml
- name: リポジトリのチェックアウト
  uses: actions/checkout@v4
```

## 3. pnpm/action-setup

### pnpm バージョン

#### 方針

`package.json` の `packageManager` フィールドのバージョンを使用すること。

#### 理由

ワークフローと開発環境の pnpm バージョンを統一し、バージョン違いによる動作が異ならないようにするため。また設定を DRY に保ち、設定の乖離が起こらないようにするため。

#### 設定例

```yaml
- name: pnpm のセットアップ
  uses: pnpm/action-setup@v4
  # version は省略（package.json の packageManager を使用）
```

## 4. actions/setup-node

### Node.js バージョン

#### 方針

`.node-version` ファイルを参照すること。

#### 理由

ワークフローと開発環境の Node.js バージョンを統一し、バージョン違いによる動作が異ならないようにするため。また設定を DRY に保ち、設定の乖離が起こらないようにするため。

#### 設定例

```yaml
- name: Node.js のセットアップ
  uses: actions/setup-node@v4
  with:
    node-version-file: .node-version
```

## 5. 重複実行防止

### 方針

重複実行を防ぐため、原則的に `concurrency` を設定し、設定しない場合はワークフローファイルにその理由をコメントで残すこと。

### 理由

直ぐに Push し直して古いコードでのワークフローが不要になった場合に最後まで実行されるなどの課金時間の無駄遣いを防止するため。

### 設定例

```yaml
concurrency:
  group: <workflow-name>-${{ github.ref }}
  cancel-in-progress: true
```

## 6. タイムアウト設定

### 方針

各ジョブには最低でも 10 分のタイムアウトを設定すること。

### 理由

何らかの理由でジョブが終わらない場合などに課金時間の無駄遣いを防止するため。

### 設定例

```yaml
jobs:
  build:
    timeout-minutes: 10
```
