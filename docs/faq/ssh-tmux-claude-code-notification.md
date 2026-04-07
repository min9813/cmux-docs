# SSH先の tmux 内で Claude Code 終了時に通知する方法

## Q: ローカルでは何も設定しなくても Claude Code の通知が来るのに、SSH先では来ない。なぜか？

**A:** ローカルでは cmux が `claude` コマンドを**自動的にラップ**してフックを注入しているためです。

cmux の Settings > Automation > **Claude Code Integration** がデフォルトで ON になっており、cmux のシェル統合が PATH の先頭にラッパースクリプト (`Resources/bin/claude`) を配置します。ユーザーが `claude` を実行すると、ラッパーが `--settings` フラグを注入し、以下のフックを Claude Code に自動登録します:

| フック | 動作 |
|--------|------|
| `SessionStart` | サイドバーにセッション開始を表示 |
| `Stop` | 通知を送信 + ステータスを Idle に |
| `SessionEnd` | セッション終了のクリーンアップ |
| `Notification` | Claude の通知を中継 |
| `UserPromptSubmit` | "Running" ステータスに変更 |
| `PreToolUse` | ツール使用時のステータス更新 |

SSH先では cmux のラッパースクリプトが PATH にないため、`claude` を実行しても素の Claude Code が起動し、フックが注入されないので通知が来ません。

## Q: SSH先の tmux 内で Claude Code のセッションが終わったら通知できるか？

**A:** できます。以下の方法があります。

### 方法1: リモートの `settings.json` にフックを設定する（推奨）

リモートサーバーの `~/.claude/settings.json` に追加。Claude Code のフック形式は `matcher` + `hooks` 配列の構造が必要です。

**重要:**
- SSH先のリモート `cmux` (`~/.cmux/bin/cmux`) は `cmuxd-remote` の busybox シンボリンクであり、ローカルの cmux CLI とはサポートするコマンドが異なります。`claude-hook` サブコマンドはリモートでは使えないため、`cmux notify` を使います
- フックは `/bin/sh` で実行されるため、`which cmux` が失敗しても exit 0 になるよう `if/fi` で囲みます（iTerm2 等の cmux がない環境でもエラーが出ない）
- `CMUX_SOCKET_PATH` はシェル環境変数だと `cmux ssh` 再接続時に古くなるため、`~/.cmux/socket_addr` から毎回読みます

#### シンプル版（現在表示中のワークスペースに通知）

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "if which cmux >/dev/null 2>&1; then CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr 2>/dev/null) cmux notify --title 'Claude Code' --body 'Session complete' 2>/dev/null; fi"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "if which cmux >/dev/null 2>&1; then CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr 2>/dev/null) cmux notify --title 'Claude Code' --body 'Notification' 2>/dev/null; fi"
          }
        ]
      }
    ]
  }
}
```

#### ワークスペース指定版（正しいワークスペースに通知）

`cmux notify` に `--workspace` を指定しないと、現在表示中のワークスペースに通知が届きます。SSH接続元のワークスペースに正確に通知したい場合は、`cmux list-workspaces` で接続先名からワークスペースIDを動的取得します:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "if which cmux >/dev/null 2>&1; then S=$(cat ~/.cmux/socket_addr 2>/dev/null); CMUX_SOCKET_PATH=$S cmux list-workspaces --json 2>/dev/null | python3 -c \"import sys,json;d=json.load(sys.stdin);[print(w['id']) for w in d.get('workspaces',[]) if w.get('remote',{}).get('destination')=='DEST' and w.get('remote',{}).get('connected')]\" 2>/dev/null | while read WS; do CMUX_SOCKET_PATH=$S cmux notify --title 'Claude Code' --body 'Session complete' --workspace \"$WS\" 2>/dev/null; done; fi"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "if which cmux >/dev/null 2>&1; then S=$(cat ~/.cmux/socket_addr 2>/dev/null); CMUX_SOCKET_PATH=$S cmux list-workspaces --json 2>/dev/null | python3 -c \"import sys,json;d=json.load(sys.stdin);[print(w['id']) for w in d.get('workspaces',[]) if w.get('remote',{}).get('destination')=='DEST' and w.get('remote',{}).get('connected')]\" 2>/dev/null | while read WS; do CMUX_SOCKET_PATH=$S cmux notify --title 'Claude Code' --body 'Notification' --workspace \"$WS\" 2>/dev/null; done; fi"
          }
        ]
      }
    ]
  }
}
```

**`DEST` を `cmux ssh` に渡した接続先名に置き換えてください。** 例: `cmux ssh ollo1` なら `ollo1`、`cmux ssh user@myserver` なら `user@myserver`。

ワークスペース指定版の動作:

| 条件 | 挙動 |
|------|------|
| cmux がない（iTerm2 等） | `if which` で失敗 → 何もしない |
| cmux はあるがソケット未接続 | `list-workspaces` が失敗 → `while` に入らず終了 |
| 1ワークスペースが接続中 | そのワークスペースに通知 |
| 2ワークスペースが同じ接続先 | 両方に通知（`while read` でループ） |
| 該当ワークスペースなし | `python3` の出力が空 → 何もしない |

## Q: `--workspace` なしだとなぜ現在表示中のワークスペースに届くのか？

**A:** `notification.create` API のワークスペース解決ロジックが以下の優先順で動作するためです:

1. `workspace_id` (UUID) が指定されていれば → そのワークスペース
2. `surface_id` (UUID) が指定されていれば → そのパネルを含むワークスペース
3. どちらもなし → `tabManager.selectedTabId`（現在選択中のワークスペース）

`cmux notify --workspace` なしだと 3 に該当し、通知時にユーザーが見ているワークスペースに届きます。

## Q: ワークスペースの destination はどう決まるか？

**A:** `cmux ssh` に渡した引数がそのまま `destination` になります:

| コマンド | destination |
|---------|-------------|
| `cmux ssh ollo1` | `ollo1` |
| `cmux ssh user@192.168.1.10` | `user@192.168.1.10` |
| `cmux ssh myserver.example.com` | `myserver.example.com` |

SSH config の `Host` 名を使えばその名前が destination です。

## Q: タイトルでワークスペースを指定して通知できないか？

**A:** 現時点ではできません。`notification.create` API はワークスペースの解決に UUID (`workspace_id`, `surface_id`) のみ対応しており、タイトルや名前による検索はサポートされていません。

リモートの `cmux notify` がサポートするフラグ:

| フラグ | 用途 |
|--------|------|
| `--title` | 通知のタイトル（ワークスペースタイトルではない） |
| `--body` | 通知の本文 |
| `--workspace` | 通知先ワークスペースのUUID |

### 方法2: `cmux claude-teams` を使う

```bash
# リモート側の tmux 内で
cmux claude-teams
```

`cmux claude-teams` は `TMUX` 環境変数を偽装して Claude Code に tmux 互換の環境を提供します。ただし、**`--settings` によるフック注入は行わない**ため、通知を受け取るには方法1 の `settings.json` 設定が別途必要です。

### 方法3: 起動時に `--settings` で直接フックを注入する

`settings.json` を変更したくない場合、起動コマンドにフックを直接渡すこともできます:

```bash
HOOKS_JSON='{"hooks":{"Stop":[{"matcher":"","hooks":[{"type":"command","command":"cmux notify --title Claude\\ Code --body Session\\ complete","timeout":10}]}],"Notification":[{"matcher":"","hooks":[{"type":"command","command":"cmux notify --title Claude\\ Code --body Notification","timeout":10}]}]}}'

claude --settings "$HOOKS_JSON"
```

### 全体の仕組み

```
┌─ ローカル Mac ─────────────────────┐     SSH      ┌─ リモートサーバー ──────────────────┐
│                                     │    tunnel    │                                     │
│  cmux アプリ                        │◄────────────►│  cmuxd-remote (デーモン)             │
│   ├ サイドバー通知表示              │  reverse TCP │   └ ~/.cmux/bin/cmux (シンボリンク)  │
│   └ リレーサーバー (localhost:PORT) │◄────────────►│                                     │
│                                     │              │  tmux                               │
│                                     │              │   └ Claude Code                     │
│                                     │              │      └ cmux notify (フック)          │
└─────────────────────────────────────┘              └─────────────────────────────────────┘
```

1. **ローカル側:** `cmux ssh` 実行時に、逆方向TCPトンネル (`ssh -N -R`) を張る
2. **リモート側:** `cmuxd-remote` がブートストラップされ、`~/.cmux/bin/cmux` がPATHに入る
3. **リモート側:** シェルに `CMUX_SOCKET_PATH=127.0.0.1:<relay_port>` が環境変数としてセットされる
4. **リモート側:** `cmux notify` を実行すると、逆方向トンネル経由でローカルのリレーサーバーに到達
5. **ローカル側:** cmux アプリがサイドバーに通知を表示

### 動作確認コマンド（リモート側、tmux 内で実行）

```bash
# 1. 環境変数が引き継がれているか確認
echo $CMUX_SOCKET_PATH
# → 「127.0.0.1:56080」のようなアドレスが表示されればOK

# 2. cmux CLI が使えるか確認
which cmux
# → 「~/.cmux/bin/cmux」が表示されればOK

# 3. リレー接続テスト
cmux ping
# → ローカルの cmux アプリから応答が返る

# 4. 通知テスト
cmux notify --title "test" --body "hello from tmux"
# → ローカルの cmux サイドバーに通知が表示される
```

### 必須: tmux の設定（リモート側: `~/.tmux.conf`）

tmux 内で cmux の機能を使うには、以下の設定が必要です:

```bash
# 環境変数を tmux セッションに取り込む（新規セッション作成時に有効）
set -g update-environment "DISPLAY SSH_AUTH_SOCK SSH_CONNECTION CMUX_SOCKET_PATH CMUX_TAB_ID CMUX_PANEL_ID CMUX_SURFACE_ID CMUX_WORKSPACE_ID"

# OSC タイトルを cmux に伝播する（サイドバー/パネルタイトルの反映に必要）
set -g set-titles on
set -g set-titles-string "#{pane_title}"
```

- `update-environment`: `cmux ssh` 接続時にセットされる `CMUX_SOCKET_PATH` を tmux 内のシェルに引き継ぐ。これがないと `cmux notify` 等が動作しない。
- `set-titles`: tmux が OSC タイトルシーケンスを外側のターミナルに伝播する。これがないと cmux のサイドバーやパネルタイトルに反映されない。

### 設定の反映方法（リモート側）

`~/.tmux.conf` を編集した後、反映方法は状況によって異なります。

**新しい tmux セッションを作る場合（推奨）:**

```bash
tmux new -s work
```

すべての設定（`set-titles`、`update-environment`）が反映される。

**既存の tmux セッション内で反映する場合:**

```bash
# set-titles は source で即座に反映される
tmux source ~/.tmux.conf

# update-environment は source では反映されない（tmux new 時のみ有効）
# 環境変数は手動でセットする
export CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)
```

| 設定 | `tmux source` で反映 | `tmux new` で反映 |
|------|---------------------|-------------------|
| `set-titles` | される | される |
| `update-environment` | されない | される |

### tmux で環境変数が引き継がれないケース

**既存の tmux セッションにアタッチした場合（リモート側）:**

`cmux ssh` より前に起動していた tmux セッションには `CMUX_SOCKET_PATH` が存在しない。`update-environment` は `tmux new` 時にしか効かないため、手動でセットする:

```bash
# リモート側: 既存 tmux セッション内で手動セット
export CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)
```
