# 開発サーバー再起動

tmuxペイン経由で開発サーバーを再起動します。

## 重要

**直接実行は禁止** - 必ず `tmux send-keys` を使用すること。
直接実行すると自動停止判定に影響します。

## 引数

$ARGUMENTS に以下の形式で指定可能：
- `セッション名` - セッション名のみ（コマンドは自動判定）
- `セッション名 コマンド` - セッション名とコマンド

例: `Acme-dev` または `Acme-dev pnpm dev`

## 実行手順

1. 引数がない場合、以下を確認：
   - tmuxセッション名
   - 起動コマンド（不明なら自動判定）

2. コマンドが不明な場合、package.jsonから判定：

```bash
# package.jsonのscriptsを確認
cat package.json | jq '.scripts'

# パッケージマネージャー判定
if [ -f "pnpm-lock.yaml" ]; then
    DEV_CMD="pnpm dev"
elif [ -f "yarn.lock" ]; then
    DEV_CMD="yarn dev"
else
    DEV_CMD="npm run dev"
fi
echo "判定結果: ${DEV_CMD}"
```

3. 再起動実行：

```bash
SESSION_NAME="セッション名"
PANE="0"
DEV_CMD="npm run dev"  # 判定結果または指定値

# 既存プロセス停止
tmux send-keys -t "${SESSION_NAME}:0.${PANE}" C-c
sleep 2

# 開発サーバー起動
tmux send-keys -t "${SESSION_NAME}:0.${PANE}" "${DEV_CMD}" C-m

echo "開発サーバーを再起動しました: ${DEV_CMD}"
```

4. 起動確認：

```bash
sleep 3
tmux capture-pane -t "${SESSION_NAME}:0.${PANE}" -p | tail -10
```
