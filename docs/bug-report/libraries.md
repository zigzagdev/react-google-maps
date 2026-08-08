# src/libraries の潜在的な不具合

## 1. `createStaticMapsUrl`: `zoom: 0` を指定すると `zoom` パラメータが消える
**ファイル:** `src/libraries/create-static-maps-url/index.ts:105`
**確信度:** high

```ts
const params: Record<string, string | number | null> = {
  key: apiKey,
  size: `${width}x${height}`,
  ...(center && {center: formatLocation(center)}),
  ...(zoom && {zoom}),      // ← zoom === 0 は falsy なので条件から落ちる
  ...(scale && {scale}),
  ...
};
```

`zoom` は世界全体を表示する `0` が正当な値だが、`0 && {...}` は falsy と評価されるため、`zoom === 0` のときだけ `zoom` クエリパラメータがURLから欠落する。結果として、呼び出し側が明示的にズームレベル0を指定しても、Static Maps API のデフォルトズーム（自動調整）で描画されてしまう。

**再現条件:** `createStaticMapsUrl({ ..., zoom: 0 })` を呼び出す。生成されたURLに `zoom` パラメータが含まれない。

**参考:** 同じパターン（`value &&`）は `scale` などの他パラメータにも使われているが、`scale` に `0` を渡すケースは通常想定されないため実害は薄い。`zoom` は 0 が意味のある値である点が問題。
