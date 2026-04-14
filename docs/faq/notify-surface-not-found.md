# cmux notify の Surface not found エラー

## Q: cmux notify が "Surface not found" エラーで失敗する。ワークスペースは接続済みなのになぜ？

**A:** cmux 側のバグです。`cmux notify --workspace <WS>` でワークスペースのみ指定した場合、surface（パネル）の解決も必須になっているのが原因です。

リモート接続ワークスペースでは `CMUX_SURFACE_ID` 環境変数が設定されず、TTY フォールバックも効かないため surface が見つからずエラーになります。

本来はワークスペースだけ指定すればワークスペースレベルの通知として動作すべきですが、CLI 側（`CLI/cmux.swift`）とサーバー側（`Sources/TerminalController.swift` の `notifyTarget`）の両方で surface ID を必須にしています。

### 回避策

- hooks の `cmux notify` 呼び出しに `2>/dev/null` を付けてエラーを無視する（既にそうなっている場合は動作への影響なし）
- hooks セクションを空にして通知自体を止める
