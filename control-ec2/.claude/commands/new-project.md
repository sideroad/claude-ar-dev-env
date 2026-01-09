# 新規案件EC2作成

新しい案件用のEC2インスタンスを作成します。

## 使い方

```
/project:new-project
```

## 実行内容

1. ~/docs/03-claude-code-new-project-setup.md を読み込む
2. ドキュメントの指示に従って案件EC2を作成
3. SSH鍵を ~/.ssh/ に保存
4. エイリアスを ~/.project_aliases に追加
5. tmux windowへの追加を案内

## 実行

以下のドキュメントを読み込んで実行してください：

```
~/docs/03-claude-code-new-project-setup.md
```

このドキュメントには以下のタスクが含まれています：
- EC2インスタンス作成
- セキュリティグループ作成
- IAMロール作成
- 環境構築（Node.js, Claude Code, tmux等）
- GitHub/Cloudflare認証
- 自動停止設定
