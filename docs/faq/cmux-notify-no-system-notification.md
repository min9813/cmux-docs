# cmux notify で macOS システム通知が来ない

## Q: cmux notify でサイドバーには通知が来るが、macOS のシステム通知が来ない。作業完了時は来るのになぜか？

**A:** cmux アプリがフォーカス中かつ通知対象のワークスペース/パネルが表示中の場合、macOS システム通知は意図的に抑制されます。

### 抑制ロジック（`TerminalNotificationStore.swift:938-971`）

```swift
let isActiveTab = AppDelegate.shared?.tabManager?.selectedTabId == tabId
let isFocusedSurface = surfaceId == nil || focusedSurfaceId == surfaceId
let isFocusedPanel = isActiveTab && isFocusedSurface
let isAppFocused = AppFocusState.isAppFocused()
let shouldSuppressExternalDelivery = isAppFocused && isFocusedPanel
```

**`isAppFocused && isFocusedPanel` が true のとき、`UNUserNotification` の配信がスキップされる。**

### なぜ「作業完了」だとシステム通知が来るか

Claude Code のフック（Stop イベント等）が発火するタイミングでは、ユーザーが別のアプリ（エディタ等）にフォーカスを移していることが多いため、`isAppFocused` が `false` → システム通知が配信される。

### なぜ `cmux notify` だとシステム通知が来ないか

`cmux notify` は v1 コマンド `notify` → `notifyCurrent()` を呼ぶ（`TerminalController.swift:12615`）。これは**現在選択中のタブ・フォーカス中のパネル**に通知を送る。cmux を見ている最中 = そのタブにフォーカス中 = 常に抑制条件に合致する。

### 対処法

| 方法 | 説明 |
|------|------|
| cmux からフォーカスを外す | 別のアプリにフォーカスした状態で `cmux notify` を実行すればシステム通知が来る |
| 別のワークスペースに送る | `cmux notify --tab <別のtab>` で現在フォーカス中でないタブに送る |
| フックから発火させる | Claude Code の Stop フック等、ユーザーが cmux を見ていないタイミングで発火すれば抑制されない |
