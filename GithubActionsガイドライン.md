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

#### 実装方法

```yaml
permissions:
  contents: read
```

## 2. ステップ名

### 方針

各ステップには日本語でわかりやすい名前を必ず付けること。

### 理由

視認性を向上し、失敗時や調査時などに一目でどこでエラーが発生しているか分かるようにするため。

### 実装方法

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

#### 実装方法

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

#### 実装方法

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

### 実装方法

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

### 実装方法

```yaml
jobs:
  build:
    timeout-minutes: 10
```

## 7. 不要なワークフローのスキップ

### 方針

関係のあるファイルが変更された場合にのみ実行されるようにする。
例： web アプリのテストは apps/web と apps/web が依存している packages/database が変更された場合に実行する

### 理由

- GitHub Actions は実行時間で課金されるため、不必要な実行を削減するため
- 無関係なワークフロー実行によるマージ遅延を防ぎ、開発効率を低下させないため

### 実装方法

ファイル変更の検出方法は、ジョブがマージの必須条件に指定されているかどうかで使い分ける。

#### マージの必須条件に指定されていないジョブの場合

GitHub Actions の標準機能 `paths-ignore` を使用してパスを指定する。

```yaml
on:
  push:
    branches: [develop, main]
    paths-ignore:
      - "docs/**"
      - "**/*.md"
  pull_request:
    branches: ["**"]
    paths-ignore:
      - "docs/**"
      - "**/*.md"
```

#### マージの必須条件に指定されているジョブの場合

`paths-ignore` に設定したパスではワークフロー自体が起動しないため、マージ必須のジョブが pass とならず、マージできなくなる。

そのため、`dorny/paths-filter` を使用してステップレベルの制御を行う。

```yaml
# サーバーテストの例
jobs:
  need-ci:
    runs-on: ubuntu-latest
    steps:
      - name: チェックアウト
        uses: actions/checkout@v4

      - name: 変更ファイルの検出
        uses: dorny/paths-filter@v2
        id: filter
        permissions:  # dorny/paths-filter には pull-requests の read 権限が必要
          pull-requests: read
        with:
          predicate-quantifier: every # フィルターの条件を AND 条件にするために必要。 default は same で or 条件
          filters: |
            web:
              - 'apps/web/**'  # apps/web ディレクトリ内の任意のファイルを対象とする
              - '!**/*.md'     # And .md ファイルは除外（サーバーテストでドキュメントの更新は無関係なため
              - '!**/eslint.config.mjs' # And eslint.config.mjs は除外（サーバーテストでは ESLint の設定変更は無関係なため
            logger:
              - 'packages/logger/**'
              - '!**/*.md'
              
      - name: 実行要否の判定
        id: should-run
        run: echo "result=${{ steps.filter.outputs.web == 'true' || steps.filter.outputs.logger == 'true' }}" >> $GITHUB_OUTPUT

      - name: web のテスト実行
        if: steps.should-run.outputs.result == 'true'
        run: test
```

### 例外

フィルター条件のメンテナンスコストがかかることと、Github Actions は 1 分単位の課金のため 1 分未満の実行でも 1 分として課金されることから
以下の場合のどちらかに該当する場合は例外とし、コメントでその理由を記載してスキップ設定を省略すること

- ほとんどのファイルがワークフローの対象
- ワークフローが 3 分未満

## 8. サードパーティ Actions の SHA ピン留め

### 方針

サードパーティ Actions の指定には、`@v4` などのタグやブランチではなく、コミット SHA 形式での指定を行うこと。

### 理由

`@v4` などのタグ方式やブランチ指定の場合、対象の Action のリポジトリが悪意のあるコードに置き換えられた際に、その悪意のあるコードを実行してしまうリスクがある。
コミット SHA の場合は immutable なため、参照先の変更による影響を減らせる。

### 実装方法

#### ローカルでの追加・更新時

[suzuki-shunsuke/pinact](https://github.com/suzuki-shunsuke/pinact) を使用して SHA 形式にピン留めする。

```yaml
- name: リポジトリのチェックアウト
  uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

#### Renovate による自動更新

`renovate.json` に `pinGitHubActionDigests` を追加し、アップデート時に自動的に SHA 形式へ変換されるようにする。

```json
{
  "extends": ["config:recommended", "helpers:pinGitHubActionDigests"]
}
```
