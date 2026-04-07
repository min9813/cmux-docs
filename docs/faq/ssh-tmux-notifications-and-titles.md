# SSH + tmux での通知とタイトルの挙動

## Q: SSH先で tmux を開いた場合、通知やタイトルは変わるか？

**A:** 通知とタイトルで挙動が異なります。

| 機能 | tmux内での動作 |
|------|---------------|
| `cmux notify` 通知 | 動作する（リレーデーモン経由） |
| ワークスペースタイトル（サイドバー） | SSH接続名のまま変わらない |
| パネルタイトル (OSC) | tmux の設定次第（デフォルトでは伝播しない） |
| エージェント連携 | 動作する（tmux-compat変換あり） |

- 通知は OSC シーケンスではなく、リレーデーモン (`cmuxd-remote`) の逆方向TCPトンネル経由で送られるため、tmux のバージョンに関係なく動作する。
- タイトルは Ghostty の OSC シーケンス (OSC 0/2) に基づいて更新されるが、tmux はデフォルトで OSC タイトルを外側のターミナルに伝播しない（`set-titles` がデフォルト off のため）。

## Q: tmux 内のタイトルを cmux のサイドバーやパネルに反映するには？

**A:** リモート側の `~/.tmux.conf` に以下を追加する必要があります:

```bash
# OSC タイトルを外側のターミナル（cmux/Ghostty）に伝播する
set -g set-titles on
set -g set-titles-string "#{pane_title}"
```

これがないと、tmux が OSC 0/2 タイトルシーケンスを吸収してしまい、cmux 側にタイトル変更が届きません。

| 設定 | 効果 | tmux バージョン |
|------|------|----------------|
| `set -g set-titles on` | OSC タイトルを外側のターミナルに伝播 | 全バージョン |
| `set -g set-titles-string "#{pane_title}"` | 伝播するタイトルの内容を指定 | 全バージョン |
| `set -g allow-passthrough on` | 任意の OSC/DCS シーケンスのパススルー | 3.3a 以降 |

## Q: tmux 3.0a で `set -g allow-passthrough on` が使えないがどうすればいいか？

**A:** `allow-passthrough` は tmux 3.3a (2022年6月) で追加されたオプションのため、3.0a では存在しません。

タイトル伝播だけなら `allow-passthrough` は不要です。上記の `set-titles` で十分です。

任意の OSC/DCS シーケンス（タイトル以外）のパススルーが必要な場合は、レガシーの DCS パススルー (`\ePtmux;\e...\e\\`) を使うか、tmux を 3.3a 以降にアップグレードしてください。

通知 (`cmux notify`) は TCP リレー経由なので、tmux の設定やバージョンに無関係で動作します。
