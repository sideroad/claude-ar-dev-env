# ワークツリー一覧

現在のワークツリーとサービス状態を表示します。

## 使い方

```
/project:worktree-list
```

## 実行内容

```bash
echo "=== Git Worktrees ==="
git worktree list

echo ""
echo "=== Running Services ==="
services-status

echo ""
echo "=== tmux Windows ==="
tmux list-windows -F "  #I: #W"
```
