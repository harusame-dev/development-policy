# コンポーネント（Next.js）

Next.js におけるコンポーネントの設計方針をまとめる。Page Component・Server Component・Container / Presentational・client component の扱いを対象とする。命名は [ファイル命名（Next.js）](./ファイル命名・nextjs.md)、テストの扱いは [テスト（Next.js）](./テスト・nextjs.md) を参照する。

## 1. Container / Presentational パターン

### 方針

- **Container / Presentational パターン**を基本とする

### 理由

- **テストしづらいサーバーコンポーネントから表示を切り離せる**: Next.js のサーバーコンポーネントは、エコシステムによるコンポーネントテストのサポートが十分でなくテストしづらい。データ取得・ロジックを Container に寄せ、表示を Presentational（純粋な表示）として切り離すことで、表示部分を通常のコンポーネントテストで検証できるようになる

### 実装方法

機能を Container（データ取得・ロジックを担う）と Presentational（props を受け取って表示するだけ）に分割する。Presentational はデータ取得や副作用を持たない純粋な表示層に保つ。

## 2. Container は Suspense でスケルトンを表示する

### 方針

- Container コンポーネントは `Suspense` でラップし、待機中はスケルトンを表示する

### 理由

- **ストリーミングと体感速度を両立できる**: Container を `Suspense` で囲みスケルトンを出すことで、データ取得を待たずに周辺 UI を先に表示でき、ローディング状態の扱いも宣言的に書ける

### 実装方法

```tsx
<Suspense fallback={<FooSkeleton />}>
  <FooContainer />
</Suspense>
```

## 3. Server Component はロジックを薄く保つ

### 方針

- サーバーコンポーネント内で直接データ取得やロジックを記載することは避け、別ファイルの関数に切り出す
- サーバーコンポーネント自体は、関数の呼び出しやコンポーネントの出し分けなど、薄い層に保つ

### 理由

- **テストが行いづらい**: 現在のエコシステムではサーバーコンポーネントの単体テストのサポートが十分でなく、テストが困難なため。ロジックを薄く保つことで、テスト可能な範囲（切り出した関数）に検証を寄せ、安定性を高める

### 実装方法

サーバーコンポーネントのファイルには「データ取得関数の呼び出し」「切り出した関数の呼び出し」「JSX の出し分け・レンダリング」のみを残し、データ取得やロジックの実装は別ファイルに分離する。

```tsx
// foo.server.tsx

import { getFoo } from "./foo.data";
import { notFound } from "next/navigation";
import { FooPresenter } from "./foo.presenter.universal";

type FooProps = {
  fooId: string;
};

export async function Foo({ fooId }: FooProps) {
  const foo = await getFoo(fooId);

  if (!foo) {
    notFound();
  }

  return <FooPresenter foo={foo} />;
}
```

- データ取得（`getFoo`）はサーバーコンポーネントの外（`foo.data.ts`）に切り出し、単体テストの対象とする
- 表示は純粋なプレゼンテーション用コンポーネント（`FooPresenter` / `foo.presenter.universal.tsx`）に委譲し、サーバーコンポーネント自体は「取得・分岐・委譲」のみを担う

## 4. client component は最小化する

### 方針

- ファイル全体に `"use client"` を付ける前に、client 要因（state / event handler / browser API）を別ファイルへ切り出せないか必ず検討する
- 親コンポーネントは「子に client を含むだけ」であれば universal のまま保つ
- client component を作成した後も、「これ以上分解して universal / server に逃がせる部分が残っていないか」を必ず再チェックする

### 理由

- **サーバーで描画できる範囲を最大化する**: `"use client"` を付けた範囲はクライアントへ JS が送られ、サーバーコンポーネントの利点（バンドル削減・サーバー側でのデータ取得）を失う。client 要因を末端に閉じ込めるほど、サーバーで処理できる範囲が広がる
- **client 境界は広がりやすい**: 安易に親へ `"use client"` を付けると、その配下すべてが client 扱いになり境界が肥大化する。最小単位に切り出して閉じ込めることで、境界の広がりを防ぐ

### 実装方法

client 要因（ここでは `useState` と `onClick`）だけを別コンポーネントに切り出し、親は universal のまま保つ。

```tsx
// 非推奨: state のためにコンポーネント全体を client 化している
"use client";

export function Foo({ items }: FooProps) {
  const [open, setOpen] = useState(false);

  return (
    <section>
      <ItemList items={items} />
      <button onClick={() => setOpen((v) => !v)}>切り替え</button>
    </section>
  );
}
```

```tsx
// 推奨: client 要因だけを Toggle に切り出し、親は universal のまま
export function Foo({ items }: FooProps) {
  return (
    <section>
      <ItemList items={items} />
      <Toggle />
    </section>
  );
}

// toggle.client.tsx
"use client";

export function Toggle() {
  const [open, setOpen] = useState(false);
  return <button onClick={() => setOpen((v) => !v)}>切り替え</button>;
}
```

## 5. Page Component はサーバーコンポーネントにする

`app/**/page.tsx` を対象とする。

### 方針

- 必ずサーバーコンポーネントとして作成する
- インタラクションが必要な場合は別ファイルにクライアントコンポーネントを作成して、Page Component から呼び出す

### 理由

- **サーバーで処理できる範囲を最大化する**: ページ全体に `"use client"` を付けると配下すべてが client 扱いになり、サーバーコンポーネントの利点（バンドル削減・サーバー側でのデータ取得）を失う。インタラクションは末端の client component に閉じ込める（「client component は最小化する」と同じ考え方）

## 6. Page Component ではデータ取得を Container に委譲する

### 方針

- データ取得はページでは行わず、Container コンポーネントを作成して行う

### 理由

- **ページをルーティングの入口に徹する薄い層に保つ**: データ取得を Container に寄せ、Container を `Suspense` でラップすることで、ページはレイアウトの組み立てと子への受け渡しに集中でき、データ待ちは Container 単位のスケルトンに閉じ込められる

### 実装方法

```tsx
// app/foo/page.tsx
export default function Page() {
  return (
    <Suspense fallback={<FooSkeleton />}>
      <FooContainer />
    </Suspense>
  );
}
```

## 7. Page Component の props は `PageProps<'/path'>` を使う

### 方針

- ページコンポーネントの props は Next.js が提供するグローバル型 `PageProps<'/path'>` を使う

### 理由

- **ルート定義と型を一致させる**: `PageProps<'/path'>` を使うことで props 型を手書きせず、ルートと型のずれを防げる

### 実装方法

```tsx
// app/foo/[id]/page.tsx
export default function Page({ params }: PageProps<'/foo/[id]'>) {
  // ...
}
```

## 8. Page Component では `params` / `searchParams` を `await` しない

### 方針

- ページコンポーネントで `params` / `searchParams` を `await` しない。Promise のまま子（Container）コンポーネントへ渡す
- 子へ渡す前に必要なパラメータだけ取り出したい場合は `params.then((p) => p.foo)` のように `.then` で変換し、変換後の `Promise<T>` を渡す

### 理由

- **`await` を Container に閉じ込めてストリーミングを活かす**: ページで `await` するとページ全体がその解決を待ち、ページ自体が async になる。Promise のまま渡して Container 側（`Suspense` 配下）で解決すれば、ページは即座にレイアウトを返し、データ待ちを Container に閉じ込められる

### 実装方法

ページは async にせず、`params` / `searchParams` を Promise のまま Container へ渡す。必要なパラメータだけ取り出す場合は `.then` で変換する。`await` による解決は Container 側に閉じ込める。

```tsx
// app/foo/[id]/page.tsx
export default function Page({ params }: PageProps<'/foo/[id]'>) {
  // await しない。必要なパラメータだけ .then で変換して Promise のまま渡す
  const fooId = params.then((p) => p.id);

  return (
    <Suspense fallback={<FooSkeleton />}>
      <FooContainer fooId={fooId} />
    </Suspense>
  );
}
```

```tsx
// foo.container.server.tsx
type FooContainerProps = {
  fooId: Promise<string>;
};

export async function FooContainer({ fooId }: FooContainerProps) {
  // await による解決は Container 側に閉じ込める
  const foo = await getFoo(await fooId);

  if (!foo) {
    notFound();
  }

  return <FooPresenter foo={foo} />;
}
```
