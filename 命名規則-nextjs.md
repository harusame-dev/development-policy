# 命名規則 - nextjs

## 目的

サフィックスに識別子を付与する主目的は、ファイルの性質を**判別・検索しやすくする**こと。具体的には：

- **AI フレンドリー**: ファイル名だけでそのファイルがどのような性質（サーバー/クライアント、コンポーネント/ユーティリティ、テスト種別など）を持つかを判別できる
- **AI ルールファイルでの絞り込み**: 識別子ごとに glob で対象を絞り、該当ファイルだけに必要なコンテキスト・ルールを適用できる

## 対象外

以下は本命名規則の対象外とする：

- next.js の file convention（`page.tsx` など）
- shadcn/ui で生成されたコンポーネント

## ファイル命名規則

| 種別 | ファイル名パターン | 備考 |
| --- | --- | --- |
| サーバーコンポーネント | `*.server.tsx` | |
| クライアントコンポーネント | `*.client.tsx` | |
| ユニバーサルコンポーネント | `*.universal.tsx` | |
| スケルトンコンポーネント | `*.skeleton.universal.tsx` | 基本は universal 固定。`"use client"` を要求するライブラリ等で必要な場合のみ `*.skeleton.client.tsx` を許容する |
| container コンポーネント | `*.container.[server\|client\|universal].tsx` | |
| presenter コンポーネント | `*.presenter.[server\|client\|universal].tsx` | |
| form コンポーネント | `*.form.client.tsx` | |
| Server Action | `*.action.ts` | |
| RSC 用データフェッチ | `*.data.ts` | |
| サーバー専用ユーティリティ（コンポーネント以外） | `*.server.ts` | サーバーで実行される前提、`next/headers`、`next/navigation`、DB クライアントなどサーバーでのみ動く依存を含むモジュールに付与する |

## コンポーネントの種別判定フロー

上から順に評価し、最初に該当した区分を採用する：

| 優先度 | 判定条件 | 区分 |
| --- | --- | --- |
| 1 | ファイル先頭に `"use client"` がある | **client** |
| 2 | `async` 関数として宣言されている／サーバー専用モジュール（`next/headers`、 DB クライアント、 シークレット環境変数など）を参照する | **server** |
| 3 | React の hook を使う（`useState` / `useEffect` / `useRef` / `useContext` 等）／イベントハンドラ（`onClick` 等）を直接定義する／ブラウザ専用 API（`window` / `document` / `localStorage` 等）を使う／内部で `"use client"` を要求するライブラリを使う | **client**（`"use client"` を付与） |
| 4 | 上記いずれにも該当しない（props を受けて JSX を返すだけ、 子に client component を含むだけ） | **universal** |

## テストファイル

### server テスト（node 環境）

| 種別 | ファイル名パターン |
| --- | --- |
| small（外部依存なし） | `*.small.server.test.ts` |
| medium（DB など外部依存あり） | `*.medium.server.test.ts` |

### client テスト（browser mode）

| 対象 | ファイル名パターン |
| --- | --- |
| hook・スクリプト | `*.client.test.ts` |
| コンポーネント | `*.client.test.tsx` |

### 対象外

- サーバーコンポーネント自体はテスト対象外
- E2E は別チケットで対応
