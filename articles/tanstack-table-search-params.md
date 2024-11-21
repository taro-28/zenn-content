---
title: "TanStack TableのstateをURLパラメータに同期するライブラリを作った"
emoji: "🐆"
type: "tech"
topics:
  - "tanstacktable"
  - "react"
  - "table"
  - "web"
  - "frontend"
published: false
---

[Tanstack Table](https://tanstack.com/table/latest)はReact, Vue, Solidなどの様々なライブラリ/フレームワークで使用できる、高機能なテーブルを作るためのHeadlessなUIライブラリです。

今回は、ReactでTanStack Tableを使用する際に検索やソートなどの状態をURLパラメータに同期できる[tanstack-table-search-params](https://github.com/taro-28/tanstack-table-search-params)というライブラリを作ったのでその紹介です。

https://github.com/taro-28/tanstack-table-search-params

https://x.com/taroro_tarotaro/status/1814550036392653023

# TanStack Table

まずReactでのTanStack Tableの使い方を簡単に紹介します。

TanStack Tableでは`useReactTable`というhookに、テーブルに表示するデータやカラムの定義などを渡し、その戻り値からテーブルのstateやハンドラーなどにアクセスしてテーブルを作ります。

```tsx
import {
  createColumnHelper,
  flexRender,
  getCoreRowModel,
  useReactTable,
} from "@tanstack/react-table";

type User = { id: string; name: string };

// テーブルに表示するデータ
const data: User[] = [
  { id: "1", name: "John" },
  { id: "2", name: "Sara" },
];

// カラムの定義
const columnHelper = createColumnHelper<User>();
const columns = [
  columnHelper.accessor("id", {}),
  columnHelper.accessor("name", {}),
];

export default function UserTable() {
  // テーブルを作成
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
  });

  return (
    <div>
      <table>
        <thead>
          <tr>
            {table.getFlatHeaders().map((header) => (
              <th key={header.id}>
                {flexRender(
                  header.column.columnDef.header,
                  header.getContext(),
                )}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {/* テーブルのstateにアクセスして表示する */}
          {table.getRowModel().rows.map((row) => (
            <tr key={row.id}>
              {row.getAllCells().map((cell) => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

![TanStack Tableで作ったテーブル](/images/tanstack-table-search-params/table.png)

さらにこのテーブルに検索を実装したのが以下です。

```diff tsx
import {
  // ...
+ getFilteredRowModel,
} from "@tanstack/react-table";

// ...

export default function UserTable() {
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
+   getFilteredRowModel: getFilteredRowModel(),
  });

  return (
    <div>
+     <input
+       placeholder="Search..."
+       value={table.getState().globalFilter}
+       onChange={(e) => table.setGlobalFilter(e.target.value)}
+     />
      <table>
```

![検索を追加したテーブル](/images/tanstack-table-search-params/table_with_search.png)

ここで`getCoreRowModel`と`getFilteredRowModel`は、テーブルの各機能を実装した[RowModel](https://tanstack.com/table/v8/docs/guide/row-models)という関数で、TanStack Tableでは`@tanstack/react-table`からimportできる`getFilteredRowModel`のような組み込みのRowModelか、自作したRowModelを使ってテーブルに検索やソートなどの機能を追加できます。

# tanstack-table-search-params

では次に今回作成したtanstack-table-search-paramsを使って、検索の状態をURLパラメータに同期します。

以下はNext.jsのPages Routerの例です。

tanstack-table-search-paramsでは、

1. `useTableSearchParams`をimportして、Next.jsの`useRouter`の戻り値のオブジェクトを引数に渡して呼び出す
2. `useTableSearchParams`の戻り値を`useReactTable`の引数に渡す

だけで、TanStack TableのstateをURLパラメータに同期できます。

```diff tsx
+ import { useRouter } from "next/router";
+ import { useTableSearchParams } from "tanstack-table-search-params";

// ...

export default function UserTable() {
+ const router = useRouter();
+ const stateAndOnChanges = useTableSearchParams(router);
  const table = useReactTable({
+   ...stateAndOnChanges,
    // ...
```

![検索条件をURLパラメータに同期したテーブル](/images/tanstack-table-search-params/table_with_search.png)

# URLパラメータに同期する仕組み

tanstack-table-search-paramsでTanStack TableのstateをURLパラメータに同期する仕組みを紹介するために、上のコードで`router`や`stateAndOnChanges`のオブジェクトを丸ごと受け渡す部分を、必要なプロパティに絞った形式に変更します。

```tsx
export default function UserTable() {
  const { query, pathname, replace} = useRouter();
  const stateAndOnChanges = useTableSearchParams({
    query: router.query,
    pathname: router.pathname,
    replace: router.replace,
  });
  const table = useReactTable({
    state: {
        globalFilter: stateAndOnChanges.state.globalFilter,
    },
    onGlobalFilterChange: stateAndOnChanges.onGlobalFilterChange,
    // ...
```

ここで`useReactTable`に`state.globalFilter`と`onGlobalFilterChange`を渡しているのは、TanStack Tableのstateを管理する場合の設定方法です。

https://tanstack.com/table/latest/docs/guide/global-filtering#global-filter-state

上のコードの通り、`useTableSearchParams`は

- `query`: URLパラメータのReact state
- `pathname`: 現在のURLのパス
- `replace`（or `push`）

https://github.com/taro-28/tanstack-table-search-params/blob/main/packages/tanstack-table-search-params/src/index.ts#L123-L151

を受け取って、

- `state.globalFilter`
- `onGlobalFilterChange`

を返すhookです。

https://github.com/taro-28/tanstack-table-search-params/blob/main/packages/tanstack-table-search-params/src/index.ts#L23-L44

引数と戻り値を見ていただけたら想像がつくかもしれませんが、このhookでやっていることはとてもシンプルです。

まず`state`は、`query`をTanStack Tableの以下の[`TableState`型](https://github.com/TanStack/table/blob/main/packages/table-core/src/types.ts#L179-L192)に変換（デコード）して返しています。

https://github.com/TanStack/table/blob/main/packages/table-core/src/types.ts#L179-L192
https://github.com/TanStack/table/blob/main/packages/table-core/src/features/GlobalFiltering.ts#L13-L15
https://github.com/TanStack/table/blob/main/packages/table-core/src/features/RowSorting.ts#L23-L32

次に`onGlobalFilterChange`は、各TanStack TableのstateをURLパラメータに変換（エンコード）して`replace`を実行する関数を返しているだけです。

`state`と`onGlobalFilterChange`を作る部分は、どちらもただの変換（オブジェクト→オブジェクト, 関数→関数）で、内部で別のstateなどを持っていないのが特徴です。（※後述するdebounceでのみ、内部でdebounce用のstateを持っています）

# 対応しているTanStack Tableのstate

現在(2024/11/20時点)は、以下の4つのTanStack Tableのstateに対応していますが、今後も増やしていく予定です。

- globalFilter: 行全体の検索
- sorting: ソート
- pagination: ページネーション
- columnFilters: 列ごとの検索

# 使用できるrouter

[URLパラメータに同期する仕組み](#URLパラメータに同期する仕組み)で紹介したように、tanstack-table-search-paramsはURLパラメータのReact stateをTanStack Tableのstateに変換しているだけなので、ReactのstateでURLパラメータを取得できるrouterであれば（たぶん）使用できます。

以下の3つは、examplesを用意しているのでよかったらご覧ください。

- [Next.js(Pages Router)](https://github.com/taro-28/tanstack-table-search-params/tree/main/examples/next-pages-router)
- [Next.js(App Router)](https://github.com/taro-28/tanstack-table-search-params/tree/main/examples/next-app-router)
- [TanStack Router](https://github.com/taro-28/tanstack-table-search-params/tree/main/examples/tanstack-router)

# カスタマイズ性

tanstack-table-search-paramsでは、URLパラメータ名やエンコード形式などをカスタマイズできます。

## URLパラメータ名

URLパラメータ名はデフォルトでは`globalFilter`、`sorting`のような`TableState`型と同じ名前が使われます。

URLパラメータ名を変更する場合は、`useTableSearchParams`の第2引数の`paramNames`で指定できます。

```tsx
const stateAndOnChanges = useTableSearchParams(router, {
  paramNames: {
    // URLパラメータ名を変更
    globalFilter: "search",
  },
});
```

prefixやsuffixを追加する場合は、関数を渡すこともできます。

```tsx
const stateAndOnChanges = useTableSearchParams(router, {
  paramNames: {
    // URLパラメータ名にprefixをつける
    globalFilter: (defaultName) => `userTable-${defaultName}`,
  },
});
```

## エンコード形式

URLパラメータの値のエンコード形式はデフォルトでは、以下のような素朴な形式です。

| stateの値        | URLパラメータの例          |
| ---------------- | -------------------------- |
| 行全体の検索     | `?globalFilter=John`       |
| ソート           | `?sorting=name.desc`       |
| ページネーション | `?pageIndex=2&pageSize=20` |

エンコード形式を変更する場合は、`useTableSearchParams`の第2引数の`encoders`と`docoders`で指定できます。

```tsx
const stateAndOnChanges = useTableSearchParams(router, {
  encoders: {
    // エンコード形式をJSON.stringifyに変更
    globalFilter: (globalFilter) => ({
      globalFilter: JSON.stringify(globalFilter),
    }),
  },
  decoders: {
    globalFilter: (query) =>
      query["globalFilter"]
        ? JSON.parse(query["globalFilter"])
        : (query["globalFilter"] ?? ""),
  },
});
```

エンコード形式とパラメータ名の両方の変更も可能です。

```tsx
const stateAndOnChanges = useTableSearchParams(router, {
  encoders: {
    sorting: (sorting) => ({
      // エンコード形式をJSON.stringifyに変更して、パラメータ名をmy-sortingに変更
      "my-sorting": JSON.stringify(sorting),
    }),
  },
  decoders: {
    sorting: (query) =>
      query["my-sorting"]
        ? JSON.parse(query["my-sorting"])
        : query["my-sorting"],
  },
});
```

## debounce

URLパラメータへの同期をdebounceさせることもできます。

```tsx
const stateAndOnChanges = useTableSearchParams(router, {
  debounceMilliseconds: 500,
});
```

特定のTanStack Tableのstateのみのdebounceも可能です。

```tsx
const stateAndOnChanges = useTableSearchParams(router, {
  debounceMilliseconds: {
    // globalFilterのURLパラメータへの同期を0.5秒debounceさせる
    globalFilter: 500,
  },
});
```

## デフォルト値

URLパラメータが存在しない場合のデフォルトのTanStack Tableのstateの値を指定できます。

例えばソートの場合、特にデフォルト値を設定しない場合の`state.sorting`の値（ソート条件）と対応するURLパラメータは以下です。

| `state.sorting`の値                  | URLパラメータ             |
| ------------------------------------ | ------------------------- |
| `[]`                                 | なし                      |
| `[{ id: "createdAt", desc: true }]`  | `?sorting=createdAt.desc` |
| `[{ id: "createdAt", desc: false }]` | `?sorting=createdAt.asc`  |

※`createdAt`はカラム名

対して、以下のようにデフォルトのソート順を降順に設定した場合の、

```tsx
const stateAndOnChanges = useTableSearchParams(router, {
  defaultValues: {
    sorting: [{ id: "createdAt", desc: true }],
  },
});
```

`state.sorting`の値とURLパラメータは以下のようになります。

| `state.sorting`の値                  | URLパラメータ            |
| ------------------------------------ | ------------------------ |
| `[]`                                 | `?sorting=none`          |
| `[{ id: "createdAt", desc: true }]`  | なし                     |
| `[{ id: "createdAt", desc: false }]` | `?sorting=createdAt.asc` |

# 最後に

元々は仕事でTanStack Tableを使っていて、楽にURLパラメータへの同期したかったのがきっかけで作りました。
（仕事でも使い始めていて、[サービスのお知らせページでも少し紹介](https://about.basemachina.com/news/feature-update-20241111#index_cXalpYnF)しています）

現状自分が欲しい機能は一通り揃ったので、あとは対応しているTanStack Tableのstateを増やしたりカスタマイズ性を高めたりして完成度を高めていきたいです。

ちなみにプライベートで自分の作ったpackageをnpmに公開したのが初めてで、npmのpublishの方法や[tsup](https://tsup.egoist.dev/)の便利さなどを知れたのもよかったです。

よかったら使ってみてください！（スターもいただけるととても喜びます！）

https://github.com/taro-28/tanstack-table-search-params
