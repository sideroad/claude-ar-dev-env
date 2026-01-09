# コントロールEC2 プロジェクトルール

## 役割

このEC2はAR開発環境の管理サーバーです。

- 案件EC2の作成・起動・停止・削除
- 案件EC2へのSSH接続（tmux window経由）
- 常時起動（自動停止なし）

---

## セキュリティ

```
インターネット ──✕──→ コントロールEC2（SSHポート閉鎖）
                         ↑
Tailscale VPN ──────→ コントロールEC2（100.x.x.x）
                         │
                         └──→ 作業用EC2（VPC内部IP）
```

- **SSHポート**: 閉鎖（Tailscale経由でのみ接続可能）
- **作業用EC2**: コントロールEC2からのみアクセス可能

---

## 構成

```
コントロールEC2 (t4g.micro)
├── Tailscale（セキュアな接続）
├── Claude Code（案件EC2の管理）
├── AWS CLI（EC2操作）
├── 管理スクリプト（~/bin/）
├── SSH鍵（~/.ssh/ar-dev-*-key.pem）
├── エイリアス（~/.project_aliases）
└── tmux [ctrl]
      ├── window 1: 管理シェル
      ├── window 2: 案件A EC2
      └── window 3: 案件B EC2
```

---

## カスタムコマンド

| コマンド | 用途 |
|----------|------|
| `/project:new-project` | 新規案件EC2作成 |
| `/project:delete-project` | 案件EC2削除 |

---

## シェルコマンド（~/bin/）

```bash
list-projects      # 案件一覧
start-project XX   # 案件XX起動
stop-project XX    # 案件XX停止
start-all          # 全案件起動
stop-all           # 全案件停止
```

---

## tmux操作

```
Alt + ]    → 次のwindow
Alt + [    → 前のwindow
```

> ネストされたtmux（作業用EC2内）では `Alt + Shift + [ / ]` を使用

---

## ドキュメント

| ファイル | 用途 |
|----------|------|
| `~/docs/03-claude-code-new-project-setup.md` | 案件EC2作成タスク |
| `~/docs/04-claude-code-operations.md` | 運用タスク詳細 |

---

## AWS命名規則

| リソース | パターン |
|----------|----------|
| 案件EC2 | `ar-dev-{client}-server` |
| セキュリティグループ | `ar-dev-{client}-sg` |
| キーペア | `ar-dev-{client}-key` |
| IAMロール | `ar-dev-{client}-role` |

---

## 重要なルール

1. **案件EC2の作成は必ず ~/docs/03-claude-code-new-project-setup.md を使用**
2. **SSH鍵は ~/.ssh/ に保存**（案件作成時に自動保存）
3. **エイリアスは ~/.project_aliases に追加**（案件作成時に自動追加）
