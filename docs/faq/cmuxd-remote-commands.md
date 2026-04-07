# cmuxd-remote のサブコマンド

## Q: SSH先で `cmuxd-remote` を実行すると `version`, `serve --stdio`, `cli <command>` の3つが表示されるが、それぞれ何か？

**A:** `cmuxd-remote` は SSH 先にアップロードされる Go バイナリで、3つのサブコマンドがあります。

### 1. `cmuxd-remote version`

バイナリのバージョンを表示。ローカルの cmux アプリがブートストラップ時にリモート側のバージョンを確認し、必要に応じてアップデートする際に使用。

### 2. `cmuxd-remote serve --stdio`

メインのデーモンモード。stdio 上で JSON-RPC を話し、ローカルの cmux アプリと通信する。

| カテゴリ | RPC メソッド | 内容 |
|---------|-------------|------|
| ハンドシェイク | `hello`, `ping` | 接続確立・死活監視 |
| ブラウザプロキシ | `proxy.open/close/write`, `proxy.stream.*` | リモートネットワークへの SOCKS5/HTTP CONNECT トンネル |
| セッション管理 | `session.open/close/attach/detach/resize/status` | PTY の永続化、複数接続時の smallest-screen-wins リサイズ |

ローカルの cmux アプリが `ssh -T host sh -c 'exec cmuxd-remote serve --stdio'` で起動し、stdio パイプ経由で通信する。

### 3. `cmuxd-remote cli <command> [args...]`

CLI リレー。リモートのシェルから cmux コマンドをローカルの cmux アプリに中継する。

仕組み:
1. **ローカル側:** cmux が `ssh -N -R 127.0.0.1:PORT:127.0.0.1:LOCAL_RELAY` で逆方向TCPトンネルを張る
2. **リモート側:** `cmuxd-remote cli` がそのポートに接続し、HMAC-SHA256 認証を経てコマンドを転送
3. **busybox 方式:** `~/.cmux/bin/cmux` というシンボリンクで呼ばれると自動的に `cli` サブコマンドとして動作

これにより SSH 先で以下のように使える:

```bash
cmux notify --title "Build done"    # ローカルに通知
cmux list-workspaces --json         # ローカルのワークスペース一覧
cmux new-workspace                  # ローカルに新ワークスペース作成
cmux ping                           # 接続確認
```
