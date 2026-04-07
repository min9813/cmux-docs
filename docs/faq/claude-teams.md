# cmux claude-teams

## Q: 特定の Claude Code セッションIDを指定して cmux から起動するには？

**A:** `cmux claude-teams` は引数をすべて `claude` コマンドに転送するので、Claude Code の `--resume` フラグが使えます。

```bash
cmux claude-teams --resume <session-id>
```

| フラグ | 説明 |
|--------|------|
| `--continue` | 直近のセッションを再開 |
| `--resume <session-id>` | 特定のセッションIDを指定して再開 |

## Q: cmux claude-teams でできることは？

**A:** `cmux claude-teams` は Claude Code をエージェントチームモードで起動するコマンドです。

### 自動的に行われること

- `--teammate-mode auto` をデフォルトで付与（明示的に `--teammate-mode` を指定した場合は上書きしない）
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` を設定
- tmux 互換シム（`~/.cmuxterm/claude-teams-bin/tmux`）を PATH に追加し、Claude の tmux コマンドを cmux のワークスペース・スプリット操作に変換
- cmux ソケットパスを環境変数（`CMUX_SOCKET_PATH`, `CMUX_SOCKET`）で渡す

### 使い方の例

```bash
# 基本起動（チームモード有効）
cmux claude-teams

# 直近セッションを再開
cmux claude-teams --continue

# 特定セッションを再開
cmux claude-teams --resume <session-id>

# モデル指定
cmux claude-teams --model sonnet

# teammate-mode を明示指定（auto 以外にしたい場合）
cmux claude-teams --teammate-mode full
```

### Claude Code の引数をそのまま使える

`cmux claude-teams` は `claude` コマンドの全引数を転送するため、Claude Code がサポートする任意のフラグ（`--model`, `--resume`, `--continue`, `--allowedTools` など）がそのまま使えます。

### Claude Code の実行パスの解決順

1. Settings > Automation > Claude Code で設定されたカスタムパス（環境変数 `CMUX_CUSTOM_CLAUDE_PATH` または UserDefaults）
2. PATH 上の `claude`（cmux 自身のラッパーは除外）
3. cmux にバンドルされた `claude`
