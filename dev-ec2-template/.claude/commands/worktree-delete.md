# ワークツリー削除

ブランチ用のワークツリー、tmux window、サービスを削除します。

## 使い方

```
/project:worktree-delete ブランチ名
```

## 引数

- ブランチ名: 削除するブランチ名（例: feature/login）

## 実行内容

1. サービス停止
2. tmux window削除
3. Git worktree削除

```bash
BRANCH="${ARGUMENTS}"
BRANCH_SAFE=$(echo "$BRANCH" | tr '/' '-')
SESSION=$(tmux display-message -p '#S')
MAIN_DIR=$(git rev-parse --show-toplevel)
WORKTREE_DIR="${MAIN_DIR}/../${BRANCH_SAFE}"

echo "=== Deleting worktree for $BRANCH ==="

# ポート特定（ログファイルから）
PORT=$(grep -l "$WORKTREE_DIR" ~/.logs/dev-*.log 2>/dev/null | head -1 | sed 's/.*dev-//' | sed 's/.log//')

# サービス停止
if [ -n "$PORT" ]; then
  echo "Stopping services on port $PORT..."
  stop-services $PORT
fi

# tmux window削除
if tmux list-windows -t "$SESSION" | grep -q "$BRANCH_SAFE"; then
  echo "Closing tmux window..."
  tmux kill-window -t "$SESSION:$BRANCH_SAFE"
fi

# Git worktree削除
if [ -d "$WORKTREE_DIR" ]; then
  echo "Removing worktree..."
  git worktree remove "$WORKTREE_DIR" --force
fi

# ブランチ削除確認
echo ""
read -p "Delete branch '$BRANCH' as well? (y/N): " DELETE_BRANCH
if [ "$DELETE_BRANCH" = "y" ]; then
  git branch -D "$BRANCH"
  echo "Branch deleted."
fi

echo ""
echo "=== Done ==="
```
