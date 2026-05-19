# 命名規則 - nextjs（テスト）

[命名規則-nextjs.md](./命名規則-nextjs.md) のテストファイル向け命名規則。

## ファイル命名規則

### server テスト（node 環境）

| 種別                          | ファイル名パターン        |
| ----------------------------- | ------------------------- |
| small（外部依存なし）         | `*.small.server.test.ts`  |
| medium（DB など外部依存あり） | `*.medium.server.test.ts` |

### client テスト（browser mode）

| 対象             | ファイル名パターン  |
| ---------------- | ------------------- |
| hook・スクリプト | `*.client.test.ts`  |
| コンポーネント   | `*.client.test.tsx` |

## 対象外

- サーバーコンポーネント自体はテスト対象外
