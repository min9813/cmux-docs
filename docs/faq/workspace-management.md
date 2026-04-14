# ワークスペース管理

## Q: cmux のウィンドウを二つ開き、片方からもう片方へワークスペースを移動できるか？

**A:** はい、`cmux move-workspace-to-window` コマンドで可能です。

```bash
cmux move-workspace-to-window --workspace <workspace-id|ref|index> --window <window-id|ref|index>
```

### 例

```bash
# ワークスペース2をウィンドウ1に移動
cmux move-workspace-to-window --workspace workspace:2 --window window:1
```

### ワークスペース・ウィンドウの ID 確認方法

```bash
cmux list-workspaces
cmux list-windows
```

## Q: window ID や workspace ID の確認方法は？GUI でできる？

**A:** 現時点では CLI のみで確認できます。GUI での ID 表示機能は未実装です。

```bash
# ウィンドウ一覧（ID付き）
cmux list-windows

# ワークスペース一覧（ID付き）
cmux list-workspaces

# JSON で詳細出力
cmux list-windows --json
cmux list-workspaces --json

# 現在のウィンドウ情報
cmux current-window
```
