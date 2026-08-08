# src/components/map の潜在的な不具合

## 1. `useMapInstance`: `reuseMaps` が依存配列から漏れており、古い値でクリーンアップされる
**ファイル:** `src/components/map/use-map-instance.ts:128-246`
**確信度:** high

```ts
useEffect(() => {
  ...
  const cachedMap =
    reuseMaps && CachedMapStack.has(cacheKey)     // 生成時に reuseMaps を参照
      ? (CachedMapStack.pop(cacheKey) as google.maps.Map)
      : null;
  ...
  return () => {
    ...
    if (reuseMaps) {                              // クリーンアップ時にも reuseMaps を参照
      CachedMapStack.push(cacheKey, map);
    } else {
      google.maps.event.clearInstanceListeners(map);
    }
    ...
  };
},
// eslint-disable-next-line react-hooks/exhaustive-deps
[
  container,
  apiIsLoaded,
  id,
  props.mapId,
  props.renderingType,
  props.colorScheme
  // ← reuseMaps がここに含まれていない
]);
```

このエフェクトは `container` / `apiIsLoaded` / `id` / `mapId` / `renderingType` / `colorScheme` が変化したときにしか再実行されない。`reuseMaps` prop だけを途中で変更しても effect は再実行されず、クリーンアップのクロージャは effect が最後に実際に走った時点の `reuseMaps` の値を保持し続ける。

**再現条件:** `reuseMaps={true}` でマウントし、その後 `reuseMaps={false}` に変更してからアンマウントする。effect が再実行されていないため、クリーンアップは古い `reuseMaps=true` を見て `CachedMapStack.push()` してしまい、意図せず地図インスタンスが静的な `CachedMapStack` に残り続ける（実質的なメモリ／DOM リーク）。逆方向（true→false 後に false のつもりが実は push される、または false→true のつもりが listener が clear される）でも同様に、propの意図と実際の挙動が食い違う。
