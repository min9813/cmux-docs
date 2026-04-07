# SSH ワークスペースへのドラッグ&ドロップの保存先

## Q: ドラッグ&ドロップでファイルを転送した場合、リモート上のどこに保存されるか？

**A:** リモートの `/tmp/cmux-drop-<UUID>.<拡張子>` に保存されます（`Workspace.swift:4816-4819`）。

```swift
static func remoteDropPath(for fileURL: URL, uuid: UUID = UUID()) -> String {
    let extensionSuffix = fileURL.pathExtension.trimmingCharacters(in: .whitespacesAndNewlines)
    let lowercasedSuffix = extensionSuffix.isEmpty ? "" : ".\(extensionSuffix.lowercased())"
    return "/tmp/cmux-drop-\(uuid.uuidString.lowercased())\(lowercasedSuffix)"
}
```

### 具体例

| ドロップしたファイル | リモート上のパス |
|---|---|
| `screenshot.png` | `/tmp/cmux-drop-a1b2c3d4-e5f6-7890-abcd-ef1234567890.png` |
| `report.pdf` | `/tmp/cmux-drop-<UUID>.pdf` |
| `data` (拡張子なし) | `/tmp/cmux-drop-<UUID>` |

### 転送方法

- ローカルから `scp` で ControlMaster 共有接続経由でアップロード
- アップロード後、ターミナルにリモートパスがテキストとして挿入される
- アップロードがキャンセルまたは失敗した場合は `rm -f` でリモートのファイルを自動クリーンアップ

### 注意

`/tmp` に保存されるため、サーバー再起動や OS の tmp クリーナーで消える可能性があります。永続的に保持したい場合は、ドロップ後にリモート側で手動で移動が必要です。
