# Tunnel URL確認

Cloudflare TunnelのURLを取得します。

## 引数

$ARGUMENTS にセッション名が指定されている場合はそれを使用。
指定がない場合は、ユーザーに確認するか、tmux list-sessions で確認。

## 実行手順

1. セッション名を確定

2. TunnelのURLを取得：

```bash
SESSION_NAME="セッション名"
PANE="1"  # Tunnelは通常ペイン1

# ペインの出力からURLを抽出
TUNNEL_URL=$(tmux capture-pane -t "${SESSION_NAME}:0.${PANE}" -p | grep -o 'https://[^ ]*\.trycloudflare\.com' | tail -1)

if [ -n "${TUNNEL_URL}" ]; then
  echo "Tunnel URL: ${TUNNEL_URL}"
else
  echo "Tunnel URLが見つかりません。Tunnelが起動しているか確認してください。"
  echo ""
  echo "Tunnelペインの内容:"
  tmux capture-pane -t "${SESSION_NAME}:0.${PANE}" -p | tail -20
fi
```

3. URLを報告

## Tunnelが見つからない場合

Tunnelを再起動：

```bash
tmux send-keys -t "${SESSION_NAME}:0.1" C-c
sleep 2
tmux send-keys -t "${SESSION_NAME}:0.1" "cloudflared tunnel --url http://localhost:3000" C-m
sleep 5
tmux capture-pane -t "${SESSION_NAME}:0.1" -p | grep -o 'https://[^ ]*\.trycloudflare\.com'
```
