# ワークツリー作成

新しいブランチ用のワークツリーを作成し、tmux windowとサービスを起動します。

## 使い方

```
/project:worktree-new ブランチ名
```

## 引数

- ブランチ名: 作成するブランチ名（例: feature/login）

## 実行内容

1. Git worktree作成
2. tmux window作成
3. サービス起動（次の空きポート）

```bash
BRANCH="${ARGUMENTS}"
BRANCH_SAFE=$(echo "$BRANCH" | tr '/' '-')
SESSION=$(tmux display-message -p '#S')
MAIN_DIR=$(git rev-parse --show-toplevel)
WORKTREE_DIR="${MAIN_DIR}/../${BRANCH_SAFE}"

# ポート決定（3000, 3010, 3020...）
USED_PORTS=$(ls ~/.logs/dev-*.pid 2>/dev/null | sed 's/.*dev-//' | sed 's/.pid//' | sort -n)
PORT=3000
while echo "$USED_PORTS" | grep -q "^$PORT$"; do
  PORT=$((PORT + 10))
done

echo "=== Creating worktree for $BRANCH ==="
echo "Directory: $WORKTREE_DIR"
echo "Port: $PORT"

# Git worktree作成
git worktree add -b "$BRANCH" "$WORKTREE_DIR" || git worktree add "$WORKTREE_DIR" "$BRANCH"

# 依存インストール
cd "$WORKTREE_DIR"
if [ -f package.json ]; then
  npm install
fi

# tmux window作成
tmux new-window -t "$SESSION" -n "$BRANCH_SAFE" -c "$WORKTREE_DIR"
tmux send-keys -t "$SESSION:$BRANCH_SAFE" "# Branch: $BRANCH (port $PORT)" C-m
tmux send-keys -t "$SESSION:$BRANCH_SAFE" "# Run: claude" C-m

# サービス起動
start-services $PORT "$WORKTREE_DIR"

echo ""
echo "=== Done ==="
echo "Window: $BRANCH_SAFE"
echo "Port: $PORT"
echo "Switch: Alt+Shift+]"
```
