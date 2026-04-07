# cmux ssh が新しいワークスペースを作成する理由

## Q: `cmux ssh` を実行すると別のパネルになるが、同じパネル内でSSH接続できないか？

**A:** `cmux ssh` は仕様として常に新しいワークスペースを作成します（`CLI/cmux.swift:4100`）。

```swift
let workspaceCreate = try client.sendV2(method: "workspace.create", params: workspaceCreateParams)
```

### 新しいワークスペースが必要な理由

SSH ワークスペースには以下の専用インフラがワークスペース単位で紐づくため、既存のローカルパネルに後付けできません:

- **リレーデーモン** (`cmuxd-remote`) のブートストラップと stdio 接続
- **逆方向TCPトンネル** (`ssh -N -R`) のライフサイクル管理
- **ブラウザプロキシ** (SOCKS5/CONNECT) のワークスペーススコープ設定
- **セッション永続化**と再接続管理
- **リモート専用の環境変数** (`CMUX_SOCKET_PATH` 等)

### 同じパネル内でSSHしたい場合

通常の `ssh` コマンドをターミナルで直接実行すれば、現在のパネル内でSSH接続できます:

```bash
ssh user@remote
```

ただし、この場合は cmux の SSH 機能（ブラウザプロキシ、CLI リレー、通知中継、再接続、ドラッグ&ドロップ等）は使えません。
