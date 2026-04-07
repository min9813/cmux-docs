# リモートサーバーでの cmux セットアップ手順

## 概要

`cmux ssh` で接続したリモートサーバー上で、Claude Code の通知や `cmux claude-teams` を使うためのセットアップ手順。iTerm2 等の他ターミナルユーザーと共有するサーバーでも安全に動作するよう設定します。

## 前提

- ローカル Mac で cmux が起動済み
- `cmux ssh user@remote` でリモートに接続済み（`cmuxd-remote` がブートストラップ済み）
- リモートサーバーに Claude Code がインストール済み

## セットアップ手順

### 1. bashrc に alias を追加（リモート側）

リモートの `~/.bashrc`（zsh の場合は `~/.zshrc`）に追加:

```bash
# cmux claude-teams 用 alias（必要に応じて CLAUDE_CONFIG_DIR を変更）
alias cmux-kenny2='CLAUDE_CONFIG_DIR=~/.claude-kenny2 cmux'

# cmux ssh 再接続時に CMUX_SOCKET_PATH を自動更新
# （新しいシェルを開くたびに最新のリレーアドレスになる）
if [ -f ~/.cmux/socket_addr ]; then
  export CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)
fi

# cmux の PATH 確認（通常は cmux ssh 接続時に自動設定される）
# 手動で追加が必要な場合:
# export PATH="$HOME/.cmux/bin:$PATH"
```

### 2. Claude Code の settings.json にフックを追加（リモート側）

対象の Claude Code 設定ディレクトリの `settings.json` にフックを追加します。

**重要:**
- リモートの `cmux` (`~/.cmux/bin/cmux`) は `cmuxd-remote` の busybox シンボリンクであり、`claude-hook` サブコマンドは使えない。`cmux notify` を使う
- フックは `/bin/sh` で実行されるため、`if/fi` で囲んで cmux がない環境（iTerm2 等）でもエラーが出ないようにする
- `CMUX_SOCKET_PATH` はシェル環境変数だと `cmux ssh` 再接続時に古くなる。`~/.cmux/socket_addr` から毎回読む方式にすることで確実に最新のリレーアドレスを使う

`~/.claude/settings.json`（または `CLAUDE_CONFIG_DIR` で指定したディレクトリの `settings.json`）を編集:

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

`--workspace` なしだと現在表示中のワークスペースに通知が届きます。SSH接続元のワークスペースに正確に通知したい場合は、`cmux list-workspaces` で接続先名からワークスペースIDを動的取得します。

**`DEST` を `cmux ssh` に渡した接続先名に置き換えてください**（例: `cmux ssh ollo1` なら `ollo1`）:

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

ワークスペース指定版の動作:

| 条件 | 挙動 |
|------|------|
| cmux バイナリがない（iTerm2 等） | `if which` で失敗 → 無音、エラーなし |
| cmux はあるがソケット未接続 | `list-workspaces` が失敗 → `while` に入らず終了 |
| 1ワークスペースが接続中 | そのワークスペースに通知 |
| 2ワークスペースが同じ接続先 | 両方に通知（`while read` でループ） |
| 該当ワークスペースなし | `python3` の出力が空 → 何もしない |

**なぜ `CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)` が必要か:**

`cmux ssh` を再接続するとリレーポートが変わりますが、既存の tmux セッションや `/bin/sh` のフック環境には古い `CMUX_SOCKET_PATH` が残ったままになります。`~/.cmux/socket_addr` にはリレーデーモンが起動するたびに最新のアドレスが書き込まれるため、フック実行時にここから読むことで常に正しいアドレスに接続できます。

**destination の確認方法:**

`cmux ssh` に渡した引数がそのまま `destination` になります:

| コマンド | destination |
|---------|-------------|
| `cmux ssh ollo1` | `ollo1` |
| `cmux ssh user@192.168.1.10` | `user@192.168.1.10` |
| `cmux ssh myserver.example.com` | `myserver.example.com` |

### 3. tmux の設定（リモート側: `~/.tmux.conf`）

`~/.tmux.conf` がない場合はファイルを作成します:

```bash
cat >> ~/.tmux.conf << 'EOF'
# 環境変数を tmux セッションに取り込む（新規セッション作成時に有効）
set -g update-environment "DISPLAY SSH_AUTH_SOCK SSH_CONNECTION CMUX_SOCKET_PATH CMUX_TAB_ID CMUX_PANEL_ID CMUX_SURFACE_ID CMUX_WORKSPACE_ID"

# OSC タイトルを cmux に伝播する（サイドバー/パネルタイトルの反映に必要）
set -g set-titles on
set -g set-titles-string "#{pane_title}"
EOF
```

既に `~/.tmux.conf` がある場合は、上記の内容を追記してください。

各設定の役割:

| 設定 | 効果 | ないとどうなるか |
|------|------|-----------------|
| `update-environment` | `cmux ssh` の環境変数を tmux 内に引き継ぐ | `cmux notify` 等が動作しない |
| `set-titles on` | OSC タイトルシーケンスを外側のターミナルに伝播 | cmux のサイドバーやパネルタイトルが更新されない |
| `set-titles-string "#{pane_title}"` | 伝播するタイトルの内容を指定 | タイトルがセッション名等になり意図した表示にならない |

tmux 3.3a 以降では、タイトル以外の OSC シーケンスもパススルーしたい場合に追加可能:

```bash
# 任意の OSC/DCS シーケンスのパススルー（tmux 3.3a 以降のみ）
set -g allow-passthrough on
```

**注意:** `set-titles` で十分なケースがほとんどです。`allow-passthrough` は tmux 3.3a 未満（例: 3.0a）では存在しないオプションなのでエラーになります。

### 4. 設定を反映（リモート側）

```bash
# bashrc を反映
source ~/.bashrc

# tmux 設定を反映（既存セッション内の場合）
tmux source ~/.tmux.conf

# tmux の update-environment は source では反映されないため、
# 環境変数は手動セットするか、新しい tmux セッションを作る
export CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)

# または新しい tmux セッションを作る（推奨、すべて反映される）
tmux new -s work
```

### 5. 動作確認（リモート側、tmux 内で実行）

```bash
# 1. cmux CLI が使えるか
which cmux
# → ~/.cmux/bin/cmux

# 2. リレー接続テスト
cmux ping
# → pong

# 3. 通知テスト
cmux notify --title "test" --body "setup complete"
# → ローカルの cmux サイドバーに通知が表示

# 4. ワークスペース指定版を使う場合: destination の確認
CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr) cmux list-workspaces --json 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); [print(w['id'], w.get('remote',{}).get('destination','')) for w in d.get('workspaces',[]) if w.get('remote',{}).get('connected')]"
# → ワークスペースIDと destination が表示される

# 5. Claude Code を起動して Stop フックを確認
cmux claude-teams
# → セッション終了時にローカルに通知が来る
```

## 複数の CLAUDE_CONFIG_DIR を使う場合

サーバー上で用途別に Claude Code 設定を分けたい場合:

```bash
# ~/.bashrc に追加
alias cmux-kenny2='CLAUDE_CONFIG_DIR=~/.claude-kenny2 cmux'
alias cmux-dev='CLAUDE_CONFIG_DIR=~/.claude-dev cmux'
```

各ディレクトリにも `settings.json` のフック設定が必要:

```bash
# ディレクトリ作成
mkdir -p ~/.claude-kenny2

# settings.json をコピー（フック設定を含む）
cp ~/.claude/settings.json ~/.claude-kenny2/settings.json
```

## トラブルシューティング

### `cmux: not found`

```bash
# PATH に cmux があるか確認
which cmux
ls -la ~/.cmux/bin/cmux

# ない場合は cmux ssh で再接続（ブートストラップが実行される）
# ローカル側で:
cmux ssh user@remote
```

### `cmux ping` がタイムアウト

```bash
# CMUX_SOCKET_PATH が設定されているか
echo $CMUX_SOCKET_PATH
# → 空なら手動セット
export CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)

# socket_addr ファイルが存在するか
cat ~/.cmux/socket_addr
# → 空やファイルなしなら、リレートンネルが切れている
# → ローカル側で cmux ssh を再接続
```

### `cmux ping` は成功するがフックで `failed to connect` / `EOF` エラー

`cmux ssh` を再接続するとリレーポートが変わりますが、シェルの `CMUX_SOCKET_PATH` やフックの `/bin/sh` 環境には古いポートが残っています。

```bash
# 現在のシェルの値と最新の値を比較
echo "shell: $CMUX_SOCKET_PATH"
echo "latest: $(cat ~/.cmux/socket_addr)"
# → 値が異なっていればポートが変わっている

# シェルの環境変数を更新
export CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)
```

フック側の恒久対策として、フックコマンドで `CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)` を毎回読む方式にしてください（本ドキュメントのステップ2を参照）。

### tmux 内で `CMUX_SOCKET_PATH` が空

```bash
# tmux が環境変数を管理しているか確認
tmux show-environment | grep CMUX

# -CMUX_SOCKET_PATH と表示されたら unset されている
# 手動セット
export CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr)

# ~/.tmux.conf の update-environment を確認
tmux show-option -g update-environment
# → CMUX_SOCKET_PATH が含まれていなければ追加が必要
```

### フックエラー: `Stop hook error: Failed with non-blocking status code`

```bash
# cmux バイナリは存在するがソケットに接続できない
# → フックコマンドに if/fi が入っているか確認
# → 入っていなければ本ページの settings.json の形式に更新

# ping は通るが notify が失敗する場合
cmux notify --title "test" --body "debug"
# → エラーメッセージを確認
```

### フックエラー: `Hooks use a matcher + hooks array`

Claude Code のフック形式が古い。`matcher` + `hooks` 配列の構造が必要:

```json
// NG: 古い形式
{"Stop": [{"type": "command", "command": "..."}]}

// OK: 正しい形式
{"Stop": [{"matcher": "", "hooks": [{"type": "command", "command": "..."}]}]}
```

### フックが発火しない（エラーもなし）

```bash
# settings.json が正しいディレクトリにあるか確認
echo $CLAUDE_CONFIG_DIR
# → 値があればそのディレクトリの settings.json を確認
# → 空なら ~/.claude/settings.json

# settings.json の JSON が valid か確認
python3 -m json.tool < ~/.claude/settings.json
# → パースエラーがあればファイル全体がスキップされる
```

### iTerm2 ユーザーでフックエラーが出る

フックコマンドが `if/fi` で囲まれていない場合、cmux がない環境でエラーが出ます。`if which cmux ... fi` 方式に変更:

```json
{
  "type": "command",
  "command": "if which cmux >/dev/null 2>&1; then CMUX_SOCKET_PATH=$(cat ~/.cmux/socket_addr 2>/dev/null) cmux notify --title 'Claude Code' --body 'Session complete' 2>/dev/null; fi"
}
```

`if/fi` により、`which cmux` が失敗しても exit 0 が返るため Claude Code にエラーが表示されません。`&&` 方式だと `which` の非ゼロ exit がフックエラーとして扱われる場合があります。
