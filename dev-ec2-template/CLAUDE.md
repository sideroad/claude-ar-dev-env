# 作業用EC2 プロジェクトルール

## 案件情報

- クライアント: ${CLIENT_NAME}
- EC2: ar-dev-${CLIENT_NAME}-server（スポットインスタンス）
- tmuxセッション: ${CLIENT_NAME^}-dev

---

## スポットインスタンス

このEC2はスポットインスタンスで動作しています。

- **コスト**: オンデマンドの約60-70%削減
- **中断**: 稀に発生（2分前に通知）
- **中断時**: 自動停止（EBSデータは保持）
- **復旧**: コントロールEC2から `start-project ${CLIENT_NAME}`

> 中断後はtmuxセッション・開発サーバー・Tunnelの再起動が必要です

---

## 構成

```
作業用EC2 (t4g.small スポット)
├── Node.js (fnm)
├── tmux セッション [${CLIENT_NAME^}-dev]
│   ├── Window 1 [main]: Claude Code
│   ├── Window 2 [feature/xxx]: Claude Code（並行開発時）
│   └── ...
├── バックグラウンドサービス（~/bin/）
│   ├── 開発サーバー (nohup)
│   └── Cloudflare Tunnel (nohup)
└── 自動停止（SSH切断後3時間）

tmux Window = Claude Code のみ（フルスクリーン）
┌─────────────────────────────────────┐
│                                     │
│           Claude Code               │
│                                     │
└─────────────────────────────────────┘

Alt+Shift+] で次のブランチ（Window）へ移動
```

---

## カスタムコマンド

| コマンド | 用途 |
|----------|------|
| `/project:worktree-new` | ワークツリー作成（Window+サービス込み） |
| `/project:worktree-delete` | ワークツリー削除 |
| `/project:worktree-list` | ワークツリー一覧 |
| `/project:dev-restart` | 開発サーバー再起動 |
| `/project:tunnel-url` | Tunnel URL確認 |
| `/project:pr-create` | PR作成 |

---

## tmux操作

### セッション操作
```
tmux attach -t ${CLIENT_NAME^}-dev    # セッションにアタッチ
Ctrl+B, d                              # デタッチ
```

### Window切り替え（Beam Pro対応）
```
Alt + Shift + ]  → 次のwindow
Alt + Shift + [  → 前のwindow
```

> コントロールEC2では `Alt + [ / ]` を使用（キーバインド衝突回避）

---

## 重要なルール

### 1. サービス管理コマンド（~/bin/）

```bash
start-services [ポート] [ディレクトリ]  # サービス起動
stop-services [ポート]                   # サービス停止
get-urls [ポート]                        # Tunnel URL取得
services-status                          # 状態確認
```

### 2. ポート割り当て

| ブランチ | ポート範囲 |
|----------|------------|
| main | 3000-3009 |
| feature/login | 3010-3019 |
| feature/payment | 3020-3029 |

### 3. よく使う操作

```bash
# サービス起動（mainブランチ）
start-services 3000 ~/project

# Tunnel URL確認
get-urls 3000

# サービス状態確認
services-status
```

---

## 自動停止

- 条件: SSH接続がない状態が3時間継続
- ログ: `sudo tail -f /var/log/auto-stop.log`
- 無効化: `sudo systemctl stop auto-stop`

---

## 接続方法

コントロールEC2から:
```bash
# エイリアスで接続
${CLIENT_NAME}

# または直接
ssh -i ~/.ssh/ar-dev-${CLIENT_NAME}-key.pem ubuntu@<Private IP>
```
