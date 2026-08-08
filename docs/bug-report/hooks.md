# src/hooks の潜在的な不具合

## 1. `useSuperclusterWorker`: `options` を変更してもワーカーに反映されない
**ファイル:** `src/hooks/use-supercluster-worker.ts:182-277`
**確信度:** high

```ts
// Update options ref in effect to avoid accessing during render
useEffect(() => {
  optionsRef.current = options;
}, [options]);

// Initialize worker
useEffect(() => {
  ...
  const initMessage: WorkerMessage = {
    type: 'init',
    options: optionsRef.current
  };
  worker.postMessage(initMessage);
  ...
}, [workerUrl]); // ← options の変更では再実行されない
```

`optionsRef.current` は最新の `options` に更新されるが、それをワーカーに送る `init` メッセージは `workerUrl` が変わったときにしか送信されない。そのため、マウント後に `radius` / `maxZoom` / `minPoints` などのクラスタリングオプションを変更しても、実行中のワーカーには一切反映されず、古い設定のままクラスタが計算され続ける。

**再現条件:** `useSuperclusterWorker` を使うコンポーネントで、マウント後に `options` prop（`radius` など）だけを変更する。`workerUrl` は変わらないため再初期化されず、クラスタリング結果が更新前の設定のまま変わらない。

---

## 2. `useSuperclusterWorker`: アンマウント/再初期化時に未解決の Promise が永遠にハングする
**ファイル:** `src/hooks/use-supercluster-worker.ts:270-276`
**確信度:** high

```ts
return () => {
  worker.terminate();
  workerRef.current = null;
  isReadyRef.current = false;
  dataLoadedRef.current = false;
  pendingRequests.clear(); // ← reject されずに Map から消えるだけ
};
```

`getLeaves` / `getChildren` / `getClusterExpansionZoom` などは `pendingRequestsRef` に `{resolve, reject}` を積んで、ワーカーからの応答で解決される設計になっている。しかし `worker.terminate()` 時のクリーンアップは `pendingRequests.clear()` するだけで、保留中の Promise を `reject()` しない。

**再現条件:** これらの非同期メソッドを呼び出した直後（応答が返る前）にコンポーネントがアンマウントされる、または `workerUrl` が変化してワーカーが再生成される。呼び出し元の `await` が永久に解決されずハングする。
