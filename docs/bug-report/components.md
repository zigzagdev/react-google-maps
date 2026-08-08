# src/components (直下) の潜在的な不具合

## 1. `AdvancedMarker`: anchor 系 prop を「設定 → 未設定」に変えても古いスタイルが残る
**ファイル:** `src/components/advanced-marker.tsx:349-411` (`useAdvancedMarkerAnchoring`)
**確信度:** medium-high

```ts
useEffect(() => {
  if (!marker || !hasChildren) return;
  ...
  if (anchorLeft !== undefined || anchorTop !== undefined) {
    marker.anchorLeft = anchorLeft;
    marker.anchorTop = anchorTop;
    ...
    return;
  }

  if (anchorPoint !== undefined) {
    ...
    contentElement.style.transform = ...; // または marker.anchorLeft/anchorTop
    marker.dataset.origin = 'rgm';
    ...
  }
  // anchorLeft/anchorTop/anchorPoint が全て undefined の場合、
  // どちらの分岐にも入らず「リセット処理」が一切ない
}, [marker, anchorPoint, anchorLeft, anchorTop, hasChildren]);
```

`anchorLeft` / `anchorTop` / `anchorPoint` のいずれかが設定された状態から、すべて `undefined` に変わった場合、どちらの `if` 分岐にも入らないため、以前に設定した `contentElement.style.transform`、`marker.anchorLeft` / `marker.anchorTop`、`marker.dataset.origin = 'rgm'` がクリアされずに残る。

**再現条件:** 実行時に `anchorPoint` prop（もしくは `anchorLeft`/`anchorTop`）を一度設定してから `undefined` に戻す。マーカーは古いオフセット・transform のまま表示され続ける。

---

## 2. `InfoWindow`: テキストのみの子要素を持つマーカーをアンカーにするとクラッシュしうる
**ファイル:** `src/components/info-window.tsx:203-214`
**確信度:** medium

```ts
if (anchorBcr && anchor.dataset.origin === 'rgm') {
  const anchorDomContent = anchor.content.firstElementChild
    ?.firstElementChild as Element;

  const contentBcr = anchorDomContent?.getBoundingClientRect(); // undefined になりうる

  const anchorOffsetX =
    contentBcr.x -                                   // ← null チェックなしで参照
    anchorBcr.x +
    (contentBcr.width - anchorBcr.width) / 2;

  const anchorOffsetY = contentBcr.y - anchorBcr.y;
  ...
}
```

`anchorDomContent`（＝ `firstElementChild.firstElementChild`）はテキストノードのみを子に持つマーカーなど、DOM 構造次第で `undefined` になりうる。その場合 `contentBcr` も `undefined` になるが、直後の `contentBcr.x` 等のプロパティアクセスに null チェックがないため `TypeError: Cannot read properties of undefined` が発生する。

**再現条件:** 3.62 未満の Google Maps API バージョン（`anchorLeft`/`anchorTop` 未対応）で、テキストノードのみを子要素とする `AdvancedMarker` に `InfoWindow` をアンカーする。

---

## 3. `Marker3D`: `useMemo` 内で DOM への副作用を行っており、StrictMode で要素がリークしうる
**ファイル:** `src/components/marker-3d.tsx:141-152`
**確信度:** medium

```ts
const contentContainer = useMemo(() => {
  const container = document.createElement('div');
  container.style.display = 'none';
  document.body.appendChild(container); // ← 副作用（DOM 追加）を useMemo 内で実行
  return container;
}, []);

// Remove the container on unmount
useEffect(() => {
  return () => contentContainer.remove(); // 最終的な contentContainer しか片付けない
}, [contentContainer]);
```

`useMemo` はメモ化の最適化用フックであり、Reactは同じレンダーで複数回呼び出すことを許容している（実際、開発モード／StrictMode では純粋性検証のためにレンダー関数ごと2回呼ばれることがある）。この場合、破棄された方の呼び出しで作られた `<div style="display:none">` は `document.body` に追加されたまま、クリーンアップの対象にならず残り続ける。

**再現条件:** React 18+ の StrictMode 環境で `Marker3D` をマウントする。マウントのたびに検知されない孤立した `div` が `document.body` にリークする可能性がある。

---

## 4. `Marker3D`: interactive/non-interactive の切り替えでホスト要素が作り直される
**ファイル:** `src/components/marker-3d.tsx:127,167,218-222`
**確信度:** low（意図的な設計の可能性あり）

`onClick` の有無で `isInteractive` が変わり、レンダリングされるタグが `gmp-marker-3d` ⇔ `gmp-marker-3d-interactive` で切り替わる。React はこれらを別要素として扱うためアンマウント→リマウントが発生し、`usePropBinding` などの命令的な状態は新しい要素に対して最初から設定し直される。API 上2つの別カスタム要素として提供されている以上、これは仕様上避けられない可能性が高いが、`onClick` を実行時に付け外しするとマーカーの内部状態（アニメーション状態等、React管理外のもの）が失われる点は利用者にとって驚きになりうる。
